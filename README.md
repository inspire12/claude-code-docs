# Claude Code Docs (KO/EN)

> 2026년 3월, Claude Code의 npm 소스맵을 통해 내부 TypeScript 소스코드가 유출되는 사건이 발생했습니다.  
> 이 레포는 그 과정에서 공개된 내부 구조 분석 문서를 한국어로 번역·정리한 아카이브입니다.

---

## 배경

2026년 3월 31일, Claude Code npm 패키지에 포함된 `.map` 파일이 Anthropic의 내부 R2 버킷에 있는 unobfuscated TypeScript 소스 zip을 가리키고 있음이 발견되었습니다. 이를 통해 약 1,900개 파일, 512,000줄 규모의 Claude Code 원본 소스가 외부에 노출되었습니다.

유출된 소스를 분석한 커뮤니티([@VineeTagarwaL](https://github.com/VineeTagarwaL-code))는 내부 아키텍처를 문서화하여 Mintlify 사이트로 공개했습니다. 이 레포는 해당 문서를 한국어로 번역한 것입니다.

---

## 주요 발견 사항

유출 소스 분석을 통해 드러난 Claude Code의 실제 아키텍처:

- **런타임**: Bun + React/Ink (터미널 UI 렌더링)
- **에이전틱 루프**: `query.ts`가 구동하는 연속 루프 — 도구 호출 → 권한 확인 → 결과 수집 → 반복
- **컨텍스트 조립**: `context.ts`의 `getSystemContext()` / `getUserContext()`가 git 상태, CLAUDE.md, 현재 날짜를 조립
- **도구 시스템**: ~40개 내장 도구 (`BashTool`, `FileEditTool`, `AgentTool` 등)
- **멀티에이전트**: `coordinator/`에서 팀 레벨 병렬 오케스트레이션
- **스킬 시스템**: `.claude/skills/`의 마크다운 파일 기반 온디맨드 기능
- **피처 플래그**: Bun 빌드타임 dead code elimination (`PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON` 등)

---

## 구조

```
/ (root)       — 한국어 번역본 (21개)
en/            — 영어 원본 (26개)
  concepts/
  configuration/
  guides/
  reference/
    commands/
    sdk/
    tools/
```

---

## 출처

- 원본 문서: https://vineetagarwal-code-claude-code.mintlify.app
- 관련 레포: https://github.com/instructkr/claw-code
- 최초 발견: [@Fried_rice (Chaofan Shou)](https://x.com/shouc001) — npm 소스맵 경유 R2 버킷 접근

---

> ℹ️ 이 레포는 공개된 분석 문서의 번역 아카이브입니다. Anthropic의 독점 소스코드를 포함하지 않습니다.
