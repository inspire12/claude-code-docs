# CLAUDE.md 설정

> Claude에게 세션마다 자동으로 로드되는 지속적인 프로젝트별 지시사항을 주기 위한 CLAUDE.md 파일 작성 및 구성 방법.

`CLAUDE.md` 파일로 Claude가 모든 세션 시작 시 로드하는 프로젝트 지식을 인코딩할 수 있습니다. 매번 프로젝트 컨벤션, 빌드 시스템, 아키텍처를 설명하는 대신, 한 번 작성하면 Claude가 자동으로 읽습니다.

## CLAUDE.md에 들어가야 할 것

빠진 경우 실수를 일으킬 지시사항을 작성하세요. 그 외에는 노이즈입니다.

**포함할 것:**
- 빌드, 테스트, lint 커맨드 (도구 이름만 아닌 정확한 호출)
- 코드를 어떻게 작성하고 구성해야 하는지에 영향을 미치는 아키텍처 결정
- 프로젝트 고유의 코딩 컨벤션 (네이밍 패턴, 파일 구조 규칙)
- 환경 설정 요구사항 (필요한 env 변수, 예상 서비스)
- Claude가 알아야 하는 일반적인 함정 또는 패턴
- 모노레포 구조와 어떤 패키지가 어떤 책임을 가지는지

**제외할 것:**
- Claude가 이미 아는 것 (표준 TypeScript 구문, 일반 라이브러리 API)
- 자명한 알림 ("깔끔한 코드를 작성하세요", "주석을 추가하세요")
- 민감한 데이터 — API 키, 비밀번호, 토큰, 또는 어떤 시크릿도
- 자주 변경되어 오래될 정보

> 💡 테스트: "이 줄을 제거하면 Claude가 이 코드베이스에서 실수를 할까?" 그렇지 않다면 삭제하세요.

## 파일 위치

Claude는 현재 디렉토리에서 파일시스템 루트까지 올라가며 메모리 파일을 발견합니다. 현재 디렉토리에 더 가까운 파일이 더 높은 우선순위를 가집니다.

| 파일 | 타입 | 용도 |
|------|------|------|
| `/etc/claude-code/CLAUDE.md` | Managed | 관리자가 설정한 모든 사용자의 시스템 전체 지시사항 |
| `~/.claude/CLAUDE.md` | User | 모든 프로젝트에 적용되는 개인 전역 지시사항 |
| `~/.claude/rules/*.md` | User | 모듈식 전역 규칙, 각 파일이 별도로 로드됨 |
| `CLAUDE.md` (프로젝트 루트) | Project | 소스 컨트롤에 커밋된 팀 공유 지시사항 |
| `.claude/CLAUDE.md` (프로젝트 루트) | Project | 팀 공유 프로젝트 지시사항의 대안 위치 |
| `.claude/rules/*.md` (프로젝트 루트) | Project | 주제별로 구성된 모듈식 프로젝트 규칙 |
| `CLAUDE.local.md` (프로젝트 루트) | Local | 커밋되지 않는 개인 프로젝트별 지시사항 |

## 로딩 순서와 우선순위

파일은 다음 순서로 로드됩니다. 모델이 컨텍스트에서 나중에 나타나는 내용에 더 주의를 기울이기 때문에, 나중에 로드된 파일이 더 높은 실효 우선순위를 가집니다.

1. **Managed 메모리** — `/etc/claude-code/CLAUDE.md`와 `rules/*.md` — 항상 로드됨
2. **User 메모리** — `~/.claude/CLAUDE.md`와 `rules/*.md` — 모든 프로젝트의 개인 전역 지시사항
3. **Project 메모리 (루트에서 CWD까지)** — 파일시스템 루트에서 현재 디렉토리까지 각 디렉토리의 `CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`. CWD에 더 가까운 파일이 나중에 로드됨 (높은 우선순위)
4. **Local 메모리** — 루트에서 CWD까지 각 디렉토리의 `CLAUDE.local.md`. 같은 순회 순서. 기본적으로 gitignore됨

## @include 지시어

메모리 파일은 `@` 표기법으로 다른 파일을 include할 수 있습니다:

```markdown
# CLAUDE.md

@./docs/architecture.md
@~/shared/style-guide.md
@/etc/company-standards.md
```

| 구문 | 해석 |
|------|------|
| `@filename` | 현재 파일 디렉토리 기준 상대 경로 |
| `@./relative/path` | 현재 파일 디렉토리 기준 명시적 상대 경로 |
| `@~/path/in/home` | 홈 디렉토리 아래 경로 |
| `@/absolute/path` | 절대 파일시스템 경로 |

**동작:**
- 비존재 파일은 조용히 무시됨
- 순환 참조는 감지되고 방지됨
- Include는 최대 5단계 깊이까지 중첩
- 텍스트 파일 형식만 지원 — 이진 파일은 건너뜀
- 코드 블록 안의 `@include`는 처리되지 않음

## 프론트매터 경로 타겟팅

`.claude/rules/`의 파일은 YAML 프론트매터로 어떤 파일 경로에 적용할지 제한할 수 있습니다:

```markdown
---
paths:
  - "src/api/**"
  - "*.graphql"
---

# API 컨벤션

모든 API 핸들러는 공유 `validate()` 헬퍼로 입력을 검증해야 합니다.
GraphQL 리졸버는 직접 데이터베이스 쿼리를 수행하면 안 됩니다 — 데이터 레이어를 사용하세요.
```

프론트매터가 없는 규칙은 무조건 적용됩니다. `paths` 프론트매터가 있는 규칙은 Claude가 glob 패턴과 일치하는 파일을 작업할 때만 적용됩니다.

## TypeScript 프로젝트용 CLAUDE.md 예시

```markdown
# Project: Payments API

## 빌드 및 테스트

- 빌드: `bun run build`
- 테스트: `bun test` (Bun 내장 테스트 러너 사용 — Jest 사용하지 말 것)
- Lint: `bun run lint` (biome, eslint 아님)
- 타입 체크: `bun run typecheck`

변경 사항을 완료하기 전에 항상 `bun run typecheck`를 실행하세요.

## 아키텍처

- `src/handlers/` — HTTP 핸들러, 라우트 그룹당 하나의 파일
- `src/services/` — 비즈니스 로직, 직접 DB 접근 없음
- `src/db/` — 데이터베이스 레이어 (Drizzle ORM); 모든 쿼리가 여기에 있음
- `src/schemas/` — 핸들러 검증과 DB 타입 간에 공유되는 Zod 스키마

핸들러는 서비스를 호출합니다. 서비스는 DB 레이어를 호출합니다. 레이어를 건너뛰지 마세요.

## 컨벤션

- 모든 입력 검증 스키마에 `z.object().strict()` 사용
- 에러는 `Result<T, AppError>`로 전파 — 서비스 코드에서 절대 throw하지 말 것
- 모든 금액 값은 센트 단위 정수
- 타임스탬프는 Unix 초(number), Date 객체 아님

## 환경

필요한 env 변수: `DATABASE_URL`, `STRIPE_SECRET_KEY`, `JWT_SECRET`
로컬 개발: `.env.example`을 `.env.local`로 복사하고 값 입력

## 피해야 할 일반적인 실수

- `new Date()`를 직접 사용하지 마세요 — `src/utils/time.ts`의 `getCurrentTimestamp()` 사용
- `console.log` 추가하지 마세요 — `src/utils/logger.ts`의 `logger` 사용
- Raw SQL 작성하지 마세요 — Drizzle 쿼리 빌더 사용
```

## /init으로 CLAUDE.md 생성

Claude Code 세션에서 `/init`을 실행하면 프로젝트의 `CLAUDE.md`를 자동 생성합니다:

```
/init
```

Claude가 코드베이스를 분석하고 프로젝트와 가장 관련된 커맨드와 컨텍스트를 담은 파일을 생성합니다. 생성된 파일은 시작점입니다 — 검토하고, 진정으로 유용하지 않은 것은 제거하고, 코드에서 추론할 수 없는 프로젝트별 지식을 추가하세요.

## /memory로 CLAUDE.md 편집

`/memory`를 실행해 메모리 에디터를 열면 현재 로드된 모든 메모리 파일을 나열하고 세션 내에서 직접 편집할 수 있습니다:

```
/memory
```

변경 사항은 즉시 적용됩니다 — Claude가 업데이트된 파일을 리로드하고 새 지시사항을 현재 세션에 적용합니다.

## 파일 제외

Claude가 로드하지 않도록 할 CLAUDE.md 파일이 있다면(예: 벤더 의존성이나 생성된 코드에 있는 경우), 설정에 제외 패턴을 추가하세요:

```json
{
  "claudeMdExcludes": [
    "/absolute/path/to/vendor/CLAUDE.md",
    "**/generated/**",
    "**/third-party/.claude/rules/**"
  ]
}
```

패턴은 picomatch를 사용해 절대 경로와 매칭됩니다. User, Project, Local 메모리 타입만 제외 가능합니다 — Managed(관리자) 파일은 항상 로드됩니다.
