import fs from 'fs';
import path from 'path';

const exts = new Set(['.js', '.ts', '.jsx', '.tsx', '.java']);
const root = process.cwd();
const results = [];

const patterns = [
  // Express / Router
  { type: 'node', re: /(app|router)\.(get|post|put|delete|patch)\s*\(\s*['"`]([^'"`]+)['"`]/g },
  // Fastify
  { type: 'fastify', re: /fastify\.(get|post|put|delete|patch)\s*\(\s*['"`]([^'"`]+)['"`]/g },
  // NestJS
  { type: 'nest_controller', re: /@Controller\s*\(\s*['"`]([^'"`]+)['"`]?/g },
  { type: 'nest_method', re: /@(Get|Post|Put|Delete|Patch)\s*\(\s*['"`]?([^'"`)]*)['"`]?\s*\)/g },
  // Spring
  { type: 'spring_controller', re: /@(RestController|Controller)\b/g },
  { type: 'spring_class_map', re: /@RequestMapping\s*\(\s*value\s*=\s*['"`]([^'"`]+)['"`]|@RequestMapping\s*\(\s*['"`]([^'"`]+)['"`]/g },
  { type: 'spring_method_map', re: /@(GetMapping|PostMapping|PutMapping|DeleteMapping|PatchMapping)\s*\(\s*['"`]?([^'"`)]*)['"`]?\s*\)/g },
];

const walk = (dir) => {
  for (const f of fs.readdirSync(dir, { withFileTypes: true })) {
    if (f.name.startsWith('.')) continue;
    const p = path.join(dir, f.name);
    if (f.isDirectory()) walk(p);
    else {
      const ext = path.extname(f.name);
      if (!exts.has(ext)) continue;
      const text = fs.readFileSync(p, 'utf8');
      let rel = path.relative(root, p);

      // Node/Express/Fastify
      for (const { type, re } of patterns) {
        if (type === 'node') {
          for (const m of text.matchAll(re)) {
            results.push({ framework: 'express/router', method: m[2].toUpperCase(), path: m[3], file: rel });
          }
        } else if (type === 'fastify') {
          for (const m of text.matchAll(re)) {
            results.push({ framework: 'fastify', method: m[1].toUpperCase(), path: m[2], file: rel });
          }
        }
      }

      // NestJS (컨트롤러 prefix 기억)
      let nestPrefix = null;
      for (const m of text.matchAll(patterns.find(p => p.type === 'nest_controller').re)) {
        nestPrefix = (m[1] || '').trim();
      }
      for (const m of text.matchAll(patterns.find(p => p.type === 'nest_method').re)) {
        const method = m[1].toUpperCase();
        const sub = (m[2] || '').trim();
        const full = ('/' + [nestPrefix, sub].filter(Boolean).join('/')).replace(/\/+/g, '/');
        results.push({ framework: 'nest', method, path: full, file: rel });
      }

      // Spring (클래스 prefix 기억)
      const hasController = patterns.find(p => p.type === 'spring_controller').re.test(text);
      if (hasController) {
        let classPrefix = null;
        for (const m of text.matchAll(patterns.find(p => p.type === 'spring_class_map').re)) {
          classPrefix = (m[1] || m[2] || '').trim();
        }
        for (const m of text.matchAll(patterns.find(p => p.type === 'spring_method_map').re)) {
          const kind = m[1];
          const sub = (m[2] || '').trim();
          const method = kind.replace('Mapping','').toUpperCase();
          const full = ('/' + [classPrefix, sub].filter(Boolean).join('/')).replace(/\/+/g, '/');
          results.push({ framework: 'spring', method, path: full, file: rel });
        }
      }
    }
  }
};

walk(path.join(root, 'src'));
fs.writeFileSync('.api-scan.json', JSON.stringify(results, null, 2));
console.log(`Found ${results.length} endpoints -> .api-scan.json`);