프로젝트 전체를 스캔해서 “HTTP API 엔드포인트 목록”을 만들어줘.

요구사항:
- 프레임워크 자동 감지(예: Spring MVC/Boot: @RestController, @Controller, @RequestMapping, @Get/Post/Put/Delete/PatchMapping;
  Express/Router: app.METHOD, router.METHOD; NestJS: @Controller + @Get/@Post...; Fastify/Koa/Hono 등도 유사 패턴)
- 엔드포인트 표 컬럼: [Method, Path, Source(File:Line), Handler/Function, Auth(있으면), Consumes/Produces(있으면), Notes]
- Spring일 경우 클래스/메서드 레벨 @RequestMapping, @*Mapping, @PathVariable/@RequestParam도 반영해서 최종 Path 합성
- Express/Nest 등은 미들웨어/라우터 prefix까지 합쳐 최종 Path로 표기
- 중복/동적 경로는 {param} 형태로 정규화
- 마지막에 OpenAPI 3.1(yaml) 초안도 생성(components.schemas는 간단히 추론)
- 출력은 ①Markdown 표, ②openapi.yaml 두 섹션으로 분리
- 참고한 파일 경로와 라인 넘버를 표에 꼭 포함