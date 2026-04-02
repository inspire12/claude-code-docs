<img src="./banner.svg" alt="Claude Code Docs" width="100%"/>

# Claude Code Docs (KO/EN)

> Claude Code 문서 중앙 허브 — 유용한 문서들을 한국어·영어로 정리합니다.

---

## 업데이트 기록

<table>
  <tr>
    <td valign="top" width="180">
      <img src="https://img.shields.io/badge/2026--04--02-📄 분석서 추가-0A93C8?style=flat-square" alt="2026-04-02"/><br>
      <sub><code>v1.1</code></sub>
    </td>
    <td valign="top">
      <b><a href="./WIKIDOCS-소스코드분석.md">WIKIDOCS-소스코드분석.md</a></b> 추가<br>
      <sub>wikidocs.net/338204 기반 소스코드 심층 분석서 · 쿼리 루프, 도구 시스템, 권한, 훅, UI 레이어 등 <b>24개 섹션</b></sub>
    </td>
  </tr>
  <tr><td colspan="2"><hr/></td></tr>
  <tr>
    <td valign="top" width="180">
      <img src="https://img.shields.io/badge/2026--04--01-🎉 초기 릴리즈-1A7F5E?style=flat-square" alt="2026-04-01"/><br>
      <sub><code>v1.0</code></sub>
    </td>
    <td valign="top">
      <b>초기 문서 아카이브 생성</b><br>
      <sub>Mintlify 원본 한국어 번역 <b>21개</b> + 영어 원본 <b>26개</b></sub>
    </td>
  </tr>
</table>

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

## 심층 분석 문서

번역 문서 외에, Claude Code 소스코드를 직접 분석한 심층 기술 문서가 포함되어 있습니다.

| 문서 | 출처 | 내용 |
|------|------|------|
| [WIKIDOCS-소스코드분석.md](./WIKIDOCS-소스코드분석.md) | [wikidocs.net/338204](https://wikidocs.net/338204) | ~1,884개 TypeScript 파일 분석 — 쿼리 루프, 도구 시스템, 권한, 훅, UI 레이어 등 24개 섹션 |

---

## 출처

- 원본 문서: https://vineetagarwal-code-claude-code.mintlify.app
- 관련 레포: https://github.com/instructkr/claw-code
- 최초 발견: [@Fried_rice (Chaofan Shou)](https://x.com/shouc001) — npm 소스맵 경유 R2 버킷 접근

---

> ℹ️ 이 레포는 공개된 분석 문서의 번역 아카이브입니다. Anthropic의 독점 소스코드를 포함하지 않습니다.
