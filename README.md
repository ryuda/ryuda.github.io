// tools/scan-apis.mjs
// Windows 11 + Node (ESM) 환경 가정. 외부 라이브러리 없이 동작.
// 목적: 프로젝트의 HTTP 엔드포인트를 스캔하고, 설명/요청 바디 샘플까지 추정하여 .api-scan.json에 저장.
//
// 지원(휴리스틱):
// - Express/Router/Fastify: app.METHOD()/router.METHOD()/fastify.METHOD()
//   · 설명: 바로 위 JSDoc /** ... */ 또는 // 한 줄 주석
//   · 요청바디: 근처 Joi/Zod 스키마, req.body의 사용 키, 미들웨어 body-parser/JSON 사용 가정
// - NestJS: @Controller(prefix) + @Get/@Post/.. + @Body() DTO
//   · 설명: 데코레이터 위 JSDoc/주석
//   · 요청바디: DTO 클래스(필드 타입, class-validator 데코레이터)로 샘플 추정
// - Spring MVC/Boot: @RestController/@Controller + @RequestMapping/@*Mapping
//   · 설명: 메서드/클래스 Javadoc/** */ 또는 // 주석
//   · 요청바디: @RequestBody 파라미터의 POJO 필드 타입으로 샘플 추정
//
// 한계: 정규식 기반이라 100%는 아님. 그래도 Continue 같은 LLM 도구가 후처리하기엔 충분한 피드가 되도록 설계.
// 출력: [{ framework, method, path, file, line, handler, description, contentType, requestBodySample }]
// -----------------------------------------------------------------------------------------------

import fs from 'fs';
import path from 'path';

const root = process.cwd();
const SRC_DIRS = ['src', 'src/main/java', 'src/main/kotlin']; // 필요시 추가
const TARGET_EXTS = new Set(['.js', '.ts', '.jsx', '.tsx', '.java', '.kt']);

// 간단 타입→더미 값 매핑
const typeSample = (t) => {
  const s = (t || '').toLowerCase();
  if (s.includes('int') || s.includes('long') || s.includes('double') || s.includes('float') || s === 'number') return 0;
  if (s.includes('bool') || s === 'boolean') return true;
  if (s.includes('date') || s.includes('localdate') || s.includes('instant')) return '2025-01-01T00:00:00Z';
  if (s.includes('uuid')) return '00000000-0000-0000-0000-000000000000';
  if (s.includes('string') || s.includes('char') || s === 'any' || s === '') return 'string';
  if (s.includes('list') || s.includes('array') || s.endsWith('[]')) return [];
  if (s.includes('map')) return {};
  return 'string';
};

// 결과 컨테이너
const endpoints = [];

// DTO/POJO 인덱스 (Nest/Spring용)
const dtoIndex = new Map(); // name -> { fields: [{name,type,notes}], file }

// 유틸: 디렉토리 순회
const walk = (dir, out = []) => {
  if (!fs.existsSync(dir)) return out;
  for (const ent of fs.readdirSync(dir, { withFileTypes: true })) {
    if (ent.name.startsWith('.')) continue;
    const p = path.join(dir, ent.name);
    if (ent.isDirectory()) walk(p, out);
    else {
      const ext = path.extname(ent.name);
      if (TARGET_EXTS.has(ext)) out.push(p);
    }
  }
  return out;
};

const readText = (file) => {
  try { return fs.readFileSync(file, 'utf8'); } catch { return ''; }
};

// 유틸: 파일에서 특정 인덱스 주변 주석/JSDoc 추출
const extractNearbyDescription = (text, idx) => {
  // 바로 위 /** ... */ 또는 // ...
  const upToIdx = text.slice(0, idx);
  // /** ... */ 마지막 블록
  const blockIdx = upToIdx.lastIndexOf('/**');
  const endIdx = upToIdx.lastIndexOf('*/');
  if (blockIdx !== -1 && endIdx !== -1 && endIdx > blockIdx) {
    const block = upToIdx.slice(blockIdx, endIdx + 2);
    const lines = block.split('\n').map((l) => l.replace(/^\s*\/?\**\s?/, '').trim()).filter(Boolean);
    if (lines.length) return lines.join(' ');
  }
  // // ... 연속 라인
  const lines = upToIdx.split('\n');
  const picked = [];
  for (let i = lines.length - 1; i >= 0; i--) {
    const m = lines[i].match(/^\s*\/\/\s?(.*)$/);
    if (m) picked.unshift(m[1].trim());
    else break;
  }
  if (picked.length) return picked.join(' ');
  return '';
};

// 유틸: 라인 번호 계산
const indexToLine = (text, idx) => text.slice(0, idx).split('\n').length;

// Nest DTO 간단 파싱: class XxxDto { field: type; @IsString() ... }
const parseNestDtoFromText = (text, file) => {
  // class FooDto { ... }
  const classRe = /export\s+class\s+([A-Za-z0-9_]+)\s*\{([\s\S]*?)\}/g;
  for (const m of text.matchAll(classRe)) {
    const name = m[1];
    const body = m[2];
    const fields = [];
    // 데코레이터 포함 라인 분리
    const lines = body.split('\n');
    let pendingNotes = [];
    for (const raw of lines) {
      const line = raw.trim();
      if (!line) continue;
      // 주석 누적
      const c1 = line.match(/^\/\/\s*(.*)$/);
      if (c1) { pendingNotes.push(c1[1]); continue; }
      // 필드: foo?: string;  foo: number;
      const f = line.match(/^([A-Za-z0-9_]+)\??:\s*([^;{]+);/);
      if (f) {
        const fname = f[1];
        const ftype = f[2].trim();
        const note = pendingNotes.join(' ');
        fields.push({ name: fname, type: ftype, notes: note });
        pendingNotes = [];
        continue;
      }
      // 데코레이터는 notes에 포함
      const deco = line.match(/^@([A-Za-z0-9_]+)\b(.*)/);
      if (deco) {
        pendingNotes.push('@' + deco[1] + (deco[2] || ''));
      }
    }
    dtoIndex.set(name, { fields, file });
  }
};

// Spring POJO 간단 파싱: class Xxx { private String name; ... }
const parseJavaPojoFromText = (text, file) => {
  const classRe = /class\s+([A-Za-z0-9_]+)\s*\{([\s\S]*?)\}/g;
  for (const m of text.matchAll(classRe)) {
    const name = m[1];
    const body = m[2];
    const fields = [];
    // private/protected/public Type name;
    const fieldRe = /(private|protected|public)\s+([A-Za-z0-9_<>\[\]]+)\s+([A-Za-z0-9_]+)\s*;/g;
    for (const fm of body.matchAll(fieldRe)) {
      const ftype = fm[2].trim();
      const fname = fm[3].trim();
      fields.push({ name: fname, type: ftype, notes: '' });
    }
    if (fields.length) dtoIndex.set(name, { fields, file });
  }
};

// 요청 바디 샘플 생성 (DTO/POJO 또는 주변 힌트 기반)
const buildSampleFromDtoName = (dtoName) => {
  const dto = dtoIndex.get(dtoName);
  if (!dto) return null;
  const obj = {};
  for (const f of dto.fields) {
    obj[f.name] = typeSample(f.type);
  }
  return obj;
};

const buildSampleFromNearbyJoiZod = (text, idx) => {
  // 아주 단순: const schema = z.object({ key: z.string(), age: z.number() })
  const around = text.slice(Math.max(0, idx - 2000), idx + 2000);
  // zod
  const z = around.match(/z\.object\s*\(\s*\{([\s\S]*?)\}\s*\)/);
  if (z) {
    const body = z[1];
    const obj = {};
    const propRe = /([A-Za-z0-9_]+)\s*:\s*z\.([A-Za-z0-9_]+)/g;
    for (const m of body.matchAll(propRe)) {
      obj[m[1]] = typeSample(m[2]);
    }
    if (Object.keys(obj).length) return obj;
  }
  // Joi
  const j = around.match(/Joi\.object\s*\(\s*\{([\s\S]*?)\}\s*\)/);
  if (j) {
    const body = j[1];
    const obj = {};
    const propRe = /([A-Za-z0-9_]+)\s*:\s*Joi\.([A-Za-z0-9_]+)/g;
    for (const m of body.matchAll(propRe)) {
      obj[m[1]] = typeSample(m[2]);
    }
    if (Object.keys(obj).length) return obj;
  }
  // req.body 키 사용 흔적 (req.body.foo)
  const rb = around.match(/req\.body\.[A-Za-z0-9_]+/g);
  if (rb) {
    const obj = {};
    for (const hit of rb) {
      const k = hit.split('.').pop();
      obj[k] = 'string';
    }
    if (Object.keys(obj).length) return obj;
  }
  return null;
};

const contentTypeFromNearby = (text, idx) => {
  const around = text.slice(Math.max(0, idx - 1500), idx + 1500);
  if (/multipart\/form-data/i.test(around)) return 'multipart/form-data';
  if (/application\/x-www-form-urlencoded/i.test(around)) return 'application/x-www-form-urlencoded';
  return 'application/json';
};

// 핸들러 이름 추정 (함수명/메서드명)
const guessHandlerName = (text, idx) => {
  // 직후 괄호 내부의 식별자 or function 이름
  const after = text.slice(idx, idx + 400);
  // function foo(req,res) {..}
  const f1 = after.match(/function\s+([A-Za-z0-9_]+)/);
  if (f1) return f1[1];
  // (req,res)=> or handler 변수
  const f2 = after.match(/([A-Za-z0-9_]+)\s*[,)]/);
  if (f2) return f2[1];
  return '';
};

// ------------------------- 1) 인덱싱(DTO/POJO) -------------------------
const allFiles = SRC_DIRS.flatMap((d) => walk(path.join(root, d)));
for (const file of allFiles) {
  const ext = path.extname(file);
  const text = readText(file);
  if (ext === '.ts' || ext === '.tsx' || ext === '.js' || ext === '.jsx') {
    // Nest DTO
    parseNestDtoFromText(text, file);
  }
  if (ext === '.java') {
    // Spring POJO
    parseJavaPojoFromText(text, file);
  }
}

// ------------------------- 2) 엔드포인트 스캔 -------------------------

for (const file of allFiles) {
  const ext = path.extname(file);
  const text = readText(file);
  if (!text) continue;

  // Express/Router
  {
    const re = /(app|router)\.(get|post|put|delete|patch)\s*\(\s*['"`]([^'"`]+)['"`]\s*,/gi;
    for (const m of text.matchAll(re)) {
      const idx = m.index || 0;
      endpoints.push({
        framework: 'express/router',
        method: m[2].toUpperCase(),
        path: m[3],
        file: path.relative(root, file),
        line: indexToLine(text, idx),
        handler: guessHandlerName(text, idx),
        description: extractNearbyDescription(text, idx),
        contentType: contentTypeFromNearby(text, idx),
        requestBodySample: buildSampleFromNearbyJoiZod(text, idx),
      });
    }
  }

  // Fastify
  {
    const re = /fastify\.(get|post|put|delete|patch)\s*\(\s*['"`]([^'"`]+)['"`]\s*,/gi;
    for (const m of text.matchAll(re)) {
      const idx = m.index || 0;
      endpoints.push({
        framework: 'fastify',
        method: m[1].toUpperCase(),
        path: m[2],
        file: path.relative(root, file),
        line: indexToLine(text, idx),
        handler: guessHandlerName(text, idx),
        description: extractNearbyDescription(text, idx),
        contentType: contentTypeFromNearby(text, idx),
        requestBodySample: buildSampleFromNearbyJoiZod(text, idx),
      });
    }
  }

  // NestJS: @Controller('/users') / @Post(':id')
  {
    // 파일 내 controller prefix (가장 마지막 선언을 사용)
    let controllerPrefix = '';
    const cRe = /@Controller\s*\(\s*['"`]?([^'"`)]*)['"`]?\s*\)/g;
    for (const c of text.matchAll(cRe)) controllerPrefix = (c[1] || '').trim();

    const mRe = /@(Get|Post|Put|Delete|Patch)\s*\(\s*['"`]?([^'"`)]*)['"`]?\s*\)\s*[\r\n\t ]*([A-Za-z0-9_\s@():,<>\[\]?=]*)\{/g;
    for (const m of text.matchAll(mRe)) {
      const idx = m.index || 0;
      const method = m[1].toUpperCase();
      const sub = (m[2] || '').trim();
      const fullPath = ('/' + [controllerPrefix, sub].filter(Boolean).join('/')).replace(/\/+/g, '/');

      // @Body() FooDto 탐색
      const tail = text.slice(idx, idx + 600);
      const b1 = tail.match(/@Body\(\s*\)\s*([A-Za-z0-9_]+)\s*:\s*([A-Za-z0-9_]+)/);
      const b2 = tail.match(/@Body\(\s*\)\s*\{([\s\S]*?)\}/); // 구조분해 바디
      let sample = null;
      if (b1) {
        const dtoName = b1[2];
        sample = buildSampleFromDtoName(dtoName);
      }
      if (!sample && b2) {
        // {name,age} → 샘플
        const props = b2[1].split(',').map(s => s.trim()).filter(Boolean);
        const obj = {};
        for (const p of props) obj[p] = 'string';
        if (Object.keys(obj).length) sample = obj;
      }
      if (!sample) sample = buildSampleFromNearbyJoiZod(text, idx);

      endpoints.push({
        framework: 'nest',
        method,
        path: fullPath,
        file: path.relative(root, file),
        line: indexToLine(text, idx),
        handler: guessHandlerName(text, idx),
        description: extractNearbyDescription(text, idx),
        contentType: contentTypeFromNearby(text, idx),
        requestBodySample: sample,
      });
    }
  }

  // Spring: @RestController / @Controller + @RequestMapping/@*Mapping
  {
    const hasCtrl = /@(RestController|Controller)\b/.test(text);
    if (hasCtrl) {
      // 클래스 레벨 prefix
      let classPrefix = '';
      // @RequestMapping(value="/x") 또는 @RequestMapping("/x")
      const cpRe = /@RequestMapping\s*\(\s*(?:value\s*=\s*)?['"`]([^'"`]+)['"`]/g;
      const cp2Re = /@RequestMapping\s*\(\s*['"`]([^'"`]+)['"`]/g;
      const cpm = [...text.matchAll(cpRe), ...text.matchAll(cp2Re)].pop();
      if (cpm) classPrefix = (cpm[1] || '').trim();

      const mapRe = /@(GetMapping|PostMapping|PutMapping|DeleteMapping|PatchMapping)\s*\(\s*['"`]?([^'"`)]*)['"`]?\s*\)[\s\S]*?\{/g;
      for (const m of text.matchAll(mapRe)) {
        const idx = m.index || 0;
        const kind = m[1];
        const sub = (m[2] || '').trim();
        const method = kind.replace('Mapping', '').toUpperCase();
        const fullPath = ('/' + [classPrefix, sub].filter(Boolean).join('/')).replace(/\/+/g, '/');

        // 메서드 시그니처 근처 @RequestBody 타입 찾기
        // public Resp create(@RequestBody CreateReq req) {
        const after = text.slice(idx, idx + 600);
        const rb = after.match(/@RequestBody\s+([A-Za-z0-9_<>$begin:math:display$$end:math:display$]+)\s+([A-Za-z0-9_]+)/);
        let sample = null;
        if (rb) {
          // 제네릭 제거
          const dtoName = rb[1].replace(/<.*?>/g, '').trim();
          sample = buildSampleFromDtoName(dtoName);
        }
        if (!sample) sample = buildSampleFromNearbyJoiZod(text, idx);

        endpoints.push({
          framework: 'spring',
          method,
          path: fullPath,
          file: path.relative(root, file),
          line: indexToLine(text, idx),
          handler: guessHandlerName(text, idx),
          description: extractNearbyDescription(text, idx),
          contentType: contentTypeFromNearby(text, idx),
          requestBodySample: sample,
        });
      }
    }
  }
}

// 후처리: path 정규화, 중복 제거(dedup by file+line or method+path+handler)
const norm = (p) => (p || '').replace(/\/+/g, '/').replace(/\/$/, '') || '/';
const keySet = new Set();
const deduped = [];
for (const e of endpoints) {
  e.path = norm(e.path);
  const k = `${e.framework}|${e.method}|${e.path}|${e.file}|${e.line}|${e.handler}`;
  if (keySet.has(k)) continue;
  keySet.add(k);
  deduped.push(e);
}

// 저장
const outPath = path.join(root, '.api-scan.json');
fs.writeFileSync(outPath, JSON.stringify(deduped, null, 2));
console.log(`Found ${deduped.length} endpoints -> ${outPath}`);