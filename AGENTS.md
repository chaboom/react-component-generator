# AGENTS.md

## Operational Commands

- 패키지 매니저는 bun 고정. npm/yarn/pnpm 사용 금지 (`bun.lock` 기준).
- 개발 서버: `bun run dev` (API 서버 `:3002` + Vite `:5173` 동시 실행)
- 테스트: `bun run test` (vitest). `bun test`는 Bun 내장 러너를 호출하므로 jsdom/setup 설정이 적용되지 않아 사용 금지.
- 테스트 감시: `bun run test:watch`
- 린트: `bun run lint`
- 빌드/타입체크: `bun run build` (`tsc -b && vite build`)
- 변경 후 최소 검증: `bun run test`, `bun run lint`, `bun run build`

## Golden Rules

### Immutable

- API 키를 클라이언트 응답으로 내보내지 마라. `/api/config`는 키 존재 여부(boolean)만 반환한다. (`server/index.ts:150-153`)
- `.env`는 커밋하지 마라. gitignore 대상이며 워크트리에는 `.worktreeinclude`로 복사된다. (`.gitignore`, `.worktreeinclude`)
- Gemini 호출 URL에는 API 키가 쿼리스트링으로 들어간다. 이 URL을 에러 메시지·로그·응답에 포함하지 마라. 현재 에러는 상태 코드만 노출한다. (`server/index.ts:99`, `server/index.ts:112`)

### 미리보기 코드 계약 (Hard Constraint + Double Defense)

- 생성 코드는 react-live `noInline`으로 실행된다. `import` 금지, 인라인 스타일만 사용, 순수 JS(TypeScript 문법 금지), 마지막에 `render(<Component />)` 호출이 필수다. (`server/index.ts:7-20`, `src/components/LivePreview.tsx:14`)
- 이 계약은 두 곳에서 지킨다. 시스템 프롬프트가 모델에 지시하고, `stripCodeFences`/`ensureRenderCall`이 응답을 사후 보정한다. 한쪽만 고치거나 제거하지 마라. (`server/index.ts:16`, `server/generator.ts:5`, `server/generator.ts:16`)
- 시스템 프롬프트의 규칙을 바꾸면 `server/generator.ts`의 보정 로직과 `server/generator.test.ts`도 함께 확인하라.

### Provider 비대칭 (Asymmetry)

- 모델 폴백(`withModelFallback`)과 `MAX_TOKENS` 잘림 검사는 Google 경로에만 있다. Anthropic 경로(`callAnthropic`)에는 없다. Provider 동작을 수정할 때 두 경로를 각각 확인하고, 의도된 차이인지 판단하라. (`server/index.ts:68-96` vs `server/index.ts:98-136`)
- 서버는 에러 문자열에 `'503'`/`'429'` 포함 여부로 상태 코드를 매핑한다. 에러 메시지 포맷(`... error: ${status}`)을 바꾸면 이 매핑이 조용히 깨진다. (`server/index.ts:84-86`, `server/index.ts:111-113`, `server/index.ts:194-206`)
- 폴백은 중간 에러를 버리고 마지막 에러만 던진다. 첫 모델의 실패 원인이 필요하면 `server/fallback.ts`를 수정해야 한다. (`server/fallback.ts:12-19`)

### 테스트 경계

- 테스트가 있는 곳: `server/generator.ts`, `server/fallback.ts`, `src/components/PromptInput.tsx`. 테스트하려는 로직은 부수효과 없는 순수 함수로 분리해 이 패턴을 따른다. (`server/generator.ts:1-2`)
- 테스트가 없는 곳: `server/index.ts`(Bun.serve 부수효과, import 시 서버가 뜬다), `src/hooks/useComponentGenerator.ts`, `src/App.tsx`. 이곳의 로직을 늘리지 말고 순수 함수로 빼서 테스트하라.
- vitest는 `src/**`와 `server/**`의 `*.test.ts(x)`를 모두 jsdom 환경으로 실행한다. (`vite.config.ts:16-21`)

### 타입체크 범위

- `tsc -b`는 `src`와 `vite.config.ts`만 검사한다. `server/` 디렉토리는 어떤 tsconfig에도 포함되지 않아 `bun run build`로 타입 오류가 잡히지 않는다. 서버 코드를 수정하면 직접 타입을 점검하라. (`tsconfig.app.json:27`, `tsconfig.node.json:25`)

### 포트 결합

- API 포트 `3002`는 `server/index.ts:139`와 Vite 프록시 `vite.config.ts:11`에 각각 하드코딩돼 있다. 한쪽만 바꾸지 마라.

## Project Context

- 프롬프트로 React 컴포넌트를 생성하고 실시간 미리보기와 코드를 보여주는 도구.
- Stack: React 19, TypeScript, Vite, react-live, Bun(API 프록시), Vitest, Testing Library.

## Standards & References

- 설치·실행·기능 설명은 `README.md` 참고.
- 커밋: 한국어, `type(scope): 요약` (type: feat/fix/refactor/chore/docs/test/style). scope는 기존 로그처럼 필요할 때만 사용. 상세 절차는 `.claude/skills/commit`.
- UI 스타일은 `src/App.css`의 CSS 변수와 클래스를 사용한다. 인라인 스타일 규칙은 미리보기로 생성되는 코드에만 적용된다.
- Maintenance Policy: 이 문서의 규칙과 코드가 어긋나면 작업을 마치기 전에 AGENTS.md 업데이트를 제안하라.
