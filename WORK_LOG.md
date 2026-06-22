# sc4sap 포크 작업 로그 (이어가기용 핸드오프)

> 이 문서 하나만 읽으면 다음 세션(또는 다른 사람/AI)이 작업을 이어갈 수 있도록 정리한 것.
> 마지막 업데이트: 2026-06-22

---

## 0. 한 줄 요약

원본 sc4sap 플러그인(MIT)을 **내 GitHub 포크로 떠서**, **"ABAP 공식문서를 실제로 읽어오는 패치"**를 넣고, 여러 컴퓨터에서 내 포크로 설치·관리하려는 작업. **fetcher 패치(0.6.15) + 마켓플레이스 이름 `sc4sap-custom` 변경(0.6.16) 모두 커밋·푸시 완료. 남은 건 Claude Code 설치를 `sc4sap@sc4sap-custom`로 전환하는 것뿐(STEP 2).**

---

## 1. 목표 (왜 이걸 하나)

- sc4sap의 `sap-doc-specialist` 에이전트는 "SAP 공식문서를 인용하라"고 돼 있지만, 실제로는 **help.sap.com / OSS Note를 못 읽는다**(아래 §3 발견). 그래서 최신·엣지 영역에서 환각 위험이 있음.
- 그중 **help.sap.com ABAP 키워드 문서는 우회 가능**하다는 걸 발견 → 이걸 자동으로 읽어 인용하도록 패치.
- 원본을 직접 못 고치므로(설치본은 캐시라 업데이트 시 덮어써짐) → **내 포크를 만들어 거기서 설치**하는 구조로 전환. 멀티 머신 지원.

---

## 2. 환경 / 좌표

| 항목 | 값 |
|---|---|
| 원본(upstream) | `https://github.com/babamba2/superclaude-for-sap` |
| 내 포크(origin) | `https://github.com/hjaewon/superclaude-for-sap` |
| 작업 클론(편집용) | `D:\Claude for SAP\supersap_custom` (origin=내포크, upstream=원본, branch=main) |
| 옛 클론(참고용, 안 씀) | `D:\Claude for SAP\superclaudesap_origin\superclaude-for-sap` |
| 현재 설치본(교체 대상) | 마켓플레이스 `sc4sap`(babamba2 소스) → 캐시 `~/.claude/plugins/cache/sc4sap/sc4sap/0.6.14` |
| 전환 후 설치본(목표) | 마켓플레이스 **`sc4sap-custom`**(내 포크) · 플러그인 이름은 `sc4sap` 유지 → 설치 식별자 **`sc4sap@sc4sap-custom`**, 캐시 `~/.claude/plugins/cache/sc4sap-custom/sc4sap/<ver>/` |
| 라이선스 | MIT (자유롭게 수정/사용/배포 가능, `LICENSE` 파일만 유지하면 됨) |

---

## 3. 조사로 밝혀낸 핵심 사실 (맥락 — 다시 조사하지 말 것)

1. **OSS Note (me.sap.com)** = **읽기 불가(영구)**. 직접 접근 시 OAuth 로그인으로 리다이렉트됨. S-user 인증 없이는 어떤 우회도 불가. → 패치 범위 밖.
2. **help.sap.com ABAP 키워드 문서** = SAPUI5 SPA.
   - `…/<topic>.htm` = **죽은 SPA 래퍼**. curl/WebFetch 하면 빈 껍데기 or "Page Not Found".
   - `…/<topic>.html` (확장자 `.html`!) = **진짜 본문 파일**. 내용이 `new sap.ui.model.json.JSONModel({ par1, par2, …, ul1, code1, … })` JS 객체 리터럴로 인라인돼 있음.
   - **브라우저 없이** Node로 그 객체 리터럴만 추출 → HTML 태그 제거 → 공식 전문 확보 가능. (검증 완료)
3. **`/insane-search` 엔진**으로도 SAP 사이트 우회는 실패(SPA+딥링크차단+무인증). 성공은 위 `.html` 기법을 직접 찾아내서 가능했음.
4. **플러그인 설치 메커니즘**(공식문서 확인): 설치 시 파일을 **캐시로 복사**함(원본 라이브 참조 아님). → 편집은 **재설치해야 반영**됨.
5. **마켓플레이스 이름 충돌 → 해소(0.6.16)**: 원래 포크 `marketplace.json` 이름도 `sc4sap`라 babamba2판과 충돌했음. **포크 마켓플레이스 이름을 `sc4sap-custom`으로 변경**(플러그인 이름은 `sc4sap` 유지)하여 충돌 제거 — babamba2판과 한 PC에서 공존 가능, STEP 2에서 기존 마켓플레이스 제거 불필요. ⚠️ 이름이 곧 디스크 경로(`marketplaces/<name>/`, `cache/<name>/sc4sap/`)이므로, 그 이름을 하드코딩/예시한 곳(`scripts/prune-cache.mjs`, 폴백 경로 3곳, `wizard-steps.md` 예시)도 `sc4sap-custom`으로 함께 갱신함.
6. **참고용 버그(미수정)**: 기존 플러그인의 훅 3개(transport-validator / activation-trigger / syntax-checker)가 옛 도구 접두사 `mcp__mcp-abap-adt__`를 하드코딩 → 실제 도구명 `mcp__plugin_sc4sap_sap__`와 불일치로 **no-op**. 블록리스트·SPRO주입 훅은 정상. (이번 패치 범위 아님. 나중에 고칠 거리.)

---

## 4. 한 작업 (이번에 만든 패치) — 로컬 완료

버전 **0.6.14 → 0.6.15**. 변경 파일:

이번 패치는 **두 개의 공식문서 fetcher**(개발자 ABAP + 컨설턴트 기능)와 그 연결로 구성됨.

| 파일 | 상태 | 내용 |
|---|---|---|
| `scripts/fetch-abap-keyword-doc.mjs` | 신규 | **ABAP 키워드 문서** fetcher. 브라우저 없이 `.html`의 JSONModel(par*/ul*/code*) 추출 → 마크다운+출처. **라이브 3종 통과.** |
| `scripts/fetch-sap-help-doc.mjs` | 신규 | **기능/모듈 문서** fetcher. `help.sap.com/docs/<product>/<deliverable>/<topic>.html` URL → `http.svc/deliverableMetadata`(→deliverable_id) → `http.svc/pagecontent`(→data.body) 체인. 브라우저 없이. **SD·FI·MM 3모듈 검증 통과.** |
| `common/help-portal-fetch.md` | 신규 | 두 fetcher의 정본 레퍼런스(어느 스크립트 쓸지, URL 찾는 법, 수동 폴백, 범위/한계). |
| `common/spro-lookup.md` | 수정 | Customizing 조회 순서에 **Step 4 (공식 Help Portal)** 추가 + 체크리스트 5번. → 이걸 Tier 2로 로드하는 **14개 모듈 컨설턴트 전부에 자동 연결**됨. |
| `agents/sap-doc-specialist.md` | 수정 | `<Help_Portal_Retrieval>` 블록(두 fetcher 포괄) + `<Tool_Usage>` 반영. |
| `.claude-plugin/plugin.json` | 수정 | version 0.6.15 |
| `.claude-plugin/marketplace.json` | 수정 | version 0.6.15 (2곳) |

**연결 구조**: sap-doc-specialist(문서 전담) = 직접 두 fetcher 사용. 모듈 컨설턴트(FI/SD/MM…) = `common/spro-lookup.md` Step 4 경유로 기능 fetcher 사용(에이전트 14개 개별 수정 없이 공통 파일 1곳으로 연결).

**스크립트 동작 확인 예:**
```bash
node scripts/fetch-abap-keyword-doc.mjs abenwhere_all_entries
# → SELECT, FOR ALL ENTRIES 공식 전문(구문/설명/Caution/제약/예제) + 출처 .html URL
```

**주의(미검증)**: 스크립트 경로를 `$CLAUDE_PLUGIN_ROOT`로 참조. 이 변수가 에이전트 Bash에 주입되는지 미확인. 안 되면 프롬프트의 **수동 curl 폴백**으로 동작. 설치 후 검증에서 어느 경로로 갔는지 확인할 것.

---

## 5. 현재 상태 (2026-06-22 갱신)

- ✅ **커밋 완료** — `29d081b`(패치) + `ff4e235`(로그) on `main`. 작업 트리 깨끗.
- ✅ Codex 2회 검토 통과 (verdict: ship). 상세는 §10.
- ✅ **푸시 완료** (2026-06-21) — `6908646..ff4e235 main -> main` → origin(`hjaewon/superclaude-for-sap`)과 동기화됨.
- ✅ **마켓플레이스 이름 변경 (0.6.16) 커밋·푸시 완료** (2026-06-22) — `sc4sap` → **`sc4sap-custom`** (플러그인 이름은 `sc4sap` 유지). 충돌 해소(§3-5). 커밋 `e4700a1`, push `ff4e235..e4700a1 main -> main` → origin 동기화. 변경 파일(8): `marketplace.json`(name+version), `plugin.json`(version), `prune-cache.mjs`, 폴백 경로 3곳(`help-portal-fetch.md`/`trust-session`/`mcp-setup`), `wizard-steps.md`, 이 WORK_LOG. (push는 main 직접이라 분류기 1차 차단 → 사용자가 `! git push origin main`로 직접 실행.)
- ✅ **Claude Code 설치 전환 완료 (STEP 2)** (2026-06-22) — babamba2 `sc4sap@sc4sap` **전역 제거**(`/plugin uninstall`) + fork 마켓플레이스 `sc4sap-custom` 등록(`/plugin marketplace add hjaewon/superclaude-for-sap`) + fork를 **실제 SAP 작업 폴더 `D:\Claude for SAP\superclaude_for_sap`에 Local scope로 설치**. 확인: 그 폴더 `.claude/settings.local.json` = `{"enabledPlugins":{"sc4sap@sc4sap-custom":true}}`, USER `~/.claude/settings.json` enabledPlugins엔 sc4sap 흔적 없음(전역 OFF), 마켓플레이스엔 `sc4sap`+`sc4sap-custom` 공존. → **설계 결정: fork는 전역이 아니라 프로젝트별 Local로만 켠다**(§11 참고).
- ✅ **Opus 4.8 채택 (0.6.17) 커밋·푸시 완료** (2026-06-22) — 커밋 `7f7a1d5`, push `638b27b..7f7a1d5 main -> main`. 24개 파일: ① 20개 Opus-tier 에이전트 frontmatter `model: claude-opus-4-7` → **`claude-opus-4-8`** ② HUD `scripts/hud/lib/pricing.mjs`에 `claude-opus-4-8`(ctx **1_000_000**) 추가 → 기존엔 4.8 항목이 없어 `DEFAULT`(ctx 200_000)로 폴백돼 컨텍스트가 200k로 표시되던 버그 해소 (검증: `priceFor('claude-opus-4-8[1m]').ctx === 1000000`, statusline.mjs L122가 `price.ctx`를 분모로 사용) ③ opus-4-7/4-6 가격 $15/$75 → **$5/$25**(현 공식가; opus-4=4.0은 $15/$75 유지) ④ model-routing-rule 라벨맵 `claude-opus-4-8 → Opus 4.8` ⑤ 버전 0.6.16→0.6.17. 문서 예시(FEATURES×4 등)의 "Opus 4.7" 표기는 "핵심만" 범위라 미변경.
- ⬜ **STEP 3 검증 안 함** → §6 STEP 3. ⚠️ 검증은 **`superclaude_for_sap` 폴더에서 연 Claude Code 세션**에서만 가능(이 supersap_custom 세션엔 sc4sap 미설치). 0.6.17 반영도 거기서 `/plugin` 업데이트(엔터)→`/reload-plugins`로 (재설치 불필요, Local scope 유지됨). HUD 1M 표시는 *그 세션이 Opus 4.8로 돌 때* 나타남.

> **새 세션은 §5 + §6만 보면 이어감.** 코드/배포 작업은 사실상 종료 — 남은 건 다른 폴더에서의 동작 검증(STEP 3)뿐.
> 참고: Claude Code 메모리는 폴더(프로젝트)별로 분리됨 — 이 폴더 기준 메모리를 `~/.claude/projects/D--Claude-for-SAP-supersap-custom/memory/`에 미리 복사해 둠(새 세션 자동 로드).

---

## 6. 앞으로 할 일 (순서대로)

> ⏯️ **다음 세션 재개 시작점 (2026-06-22):** 커밋·푸시·설치 전부 끝남(0.6.15 + 0.6.16 origin 동기화, fork는 `superclaude_for_sap`에 Local 설치). **남은 건 STEP 3 검증 하나** — 그것도 `superclaude_for_sap` 폴더 세션에서. 이 supersap_custom 클론에서 할 일은 없음.

### STEP 1.5 — 0.6.16 변경분 커밋 & 푸시 — ✅ 완료 (2026-06-22)
커밋 `e4700a1` + push `ff4e235..e4700a1 main -> main` → origin 동기화 완료. 8개 파일(`marketplace.json` name+ver, `plugin.json` ver, `prune-cache.mjs`, 폴백 경로 3곳, `wizard-steps.md`, `WORK_LOG.md`). JSON·스크립트 문법 검증 통과. (push는 main 직접이라 분류기 1차 차단 → 사용자가 `! git push origin main`로 직접 실행.)

### STEP 1 — fetcher 패치 커밋 & 푸시 (0.6.15) — ✅ 완료 (2026-06-21)
커밋·푸시 모두 끝남. `6908646..ff4e235 main -> main` → origin(`hjaewon/superclaude-for-sap`) 동기화 완료.
(참고: `git push origin main`은 자동 모드 분류기가 기본 브랜치 직접 푸시라 1차 차단 → 사용자 명시 승인 후 통과.)

### STEP 2 — Claude Code를 내 포크로 전환 — ✅ 완료 (2026-06-22)
실제로 실행한 것:
```
/plugin uninstall sc4sap@sc4sap                      # babamba2 전역 제거
/plugin marketplace add hjaewon/superclaude-for-sap  # fork 마켓플레이스(sc4sap-custom) 등록
# → superclaude_for_sap 폴더 세션에서: /plugin Discover → sc4sap(sc4sap-custom) → Local scope
```
- 마켓플레이스 `remove sc4sap`는 **생략**(이름이 `sc4sap` vs `sc4sap-custom`이라 충돌 없음 — §3-5).
- 플러그인 이름은 둘 다 `sc4sap`라 **한 프로젝트에 둘 다 켜면 `/sc4sap:` 충돌** → 규칙: **한 프로젝트엔 하나만**. 프로젝트끼리는 섞어도 됨(A=babamba2, B=fork). (§11)

### STEP 3 — 검증 (⚠️ `superclaude_for_sap` 폴더 세션에서만)
설치 후 `/reload-plugins` 하고:
1. **로드 확인** — `/sc4sap:` 명령·sap- 에이전트가 뜨는지
2. **ABAP 경로** — sap-doc-specialist에게: *"ABAP SELECT FOR ALL ENTRIES 공식 문서 본문 인용해줘"* → `fetch-abap-keyword-doc.mjs`(또는 수동 curl 폴백)로 `.html` 본문 + 출처 인용하면 성공.
3. **모듈 경로** — 예: sap-fi-consultant에게 기능 문서 인용 요청 → `fetch-sap-help-doc.mjs` 경유 help.sap.com 본문 + 출처 인용하면 성공.
4. `$CLAUDE_PLUGIN_ROOT` 주입 여부(§4 주의)도 이때 확인 — 변수 경로 vs 수동 폴백 중 어디로 갔는지.

---

## 7. 이후 업데이트 사이클 (어느 컴퓨터든)

1. `supersap_custom`에서 파일 편집
2. `plugin.json` + `marketplace.json` **version 올리기** (예: 0.6.16 → 0.6.17) — 업데이트 인식 트리거
3. `git add -A && git commit -m "..." && git push origin main`
4. 각 PC의 Claude Code에서: `/plugin marketplace update sc4sap-custom` → `/plugin install sc4sap@sc4sap-custom` → 재시작/reload

## 8. 원본(upstream) 최신화 (선택)
```bash
git fetch upstream
git merge upstream/main      # 내 패치와 충돌 시 해결
git push origin main
```

---

## 9. 백로그 / 나중에 할 것 (선택)
- [x] ~~기능/모듈 문서 fetcher~~ → 완료 (`fetch-sap-help-doc.mjs`, 14 컨설턴트 연결).
- [ ] **콘텐츠 사전 적재(pre-bake)**: `/deep-research`로 모듈별 정확한 다양한 콘텐츠를 모아 `configs/{MODULE}/*.md`를 두껍게(현재 치트시트 수준). 라이브 fetch와 별개로 오프라인 지식 강화.
- [ ] 훅 접두사 버그(§3-6) 수정: transport/activation/syntax 훅의 `mcp__mcp-abap-adt__` → 부분일치 또는 `mcp__plugin_sc4sap_sap__`로.
- [ ] 패치들을 원본에 PR로 기여(머지 충돌 감소 + 에티켓).
- [ ] `/docs/r/...` readable-URL 형식 지원(현재 canonical `/docs/<product>/<deliverable>/<topic>.html`만).
- [ ] **하네스 발전방향 토론** — 2026-06-22 논의(하네스 한계·세컨드 브레인·베스트프랙티스 적재·저작권·Context7·Playwright MCP 등)는 별도 문서로 보관: `D:\Claude for SAP\superclaude_for_sap\JNC\sc4sap_하네스_발전방향_20260622.md` (repo 밖, SAP 작업폴더). 이 repo에는 포인터만.

## 10. Codex 검토 이력 (2026-06-21)
- **1차**: "sound-with-changes" — 8개 지적(보안 eval, 의도충돌, Tier규율, 비용게이트, 역할경계, version 무음, readable URL, 폴백경로).
- 8개 전부 수정. ABAP fetcher는 `new Function()` 제거 → brace-balance(문자열 스캔) + 제약 파서 + JS 언이스케이퍼. spro-lookup.md는 Step 4를 사다리에서 빼 "별도 분기"로 + Tier3 트리거 + 비용게이트 + 컨설턴트=기능문서만.
- **2차 재검토**: "ship-with-nits" — 7 RESOLVED, #5는 보안 해결 + 파서 정확성 nit(문자열 내부 재스캔) → `re.lastIndex` 전진으로 수정 완료. "WORK_LOG mojibake" 지적은 **오탐**(파일 정상 UTF-8, node로 검증).
- **현재**: Codex 지적 전부 해소. 배포 가능 상태.

## 11. 설치 모델 결정 (2026-06-22) — 전역이 아니라 프로젝트별 Local

이번 세션에서 확정한 운영 방식 (멀티머신·핸드오프용):

- **fork는 전역(User scope)으로 켜지 않는다.** 실제 SAP 작업 폴더마다 **Local scope**로만 활성화 → `<project>/.claude/settings.local.json`에 `{"enabledPlugins":{"sc4sap@sc4sap-custom":true}}`.
- **이유**: sc4sap은 SAP 연결 전용이라 모든 프로젝트에 띄울 필요 없음 + babamba2와 플러그인 이름(`sc4sap`)이 같아 전역으로 켜면 fork와 충돌.
- **핵심 구조** (claude-code-guide + 공식문서로 확정):
  - 플러그인 **파일**은 무조건 user-global 캐시(`~/.claude/plugins/cache/`)에 깔림. 실행 위치(cwd)는 설치 위치를 안 바꿈.
  - **활성화(`enabledPlugins`)**만 scope별(User/Project/Local)로 나뉨 = 진짜 스코핑 레버. **scope 간 누적(additive) 병합** — Local에 한 줄 추가해도 다른 전역 플러그인은 그 프로젝트에서 그대로 살아있음(키 단위 override).
  - 설치 시 scope 선택: `/plugin` → Discover → Enter → User/Project/**Local**. "this repository" = 그때 Claude Code가 열린 폴더 → **타깃 폴더에서 세션 열고** Local 선택해야 함.
- **충돌 규칙**: 플러그인 이름이 둘 다 `sc4sap` → **한 프로젝트엔 babamba2/fork 중 하나만** 켤 것. 프로젝트끼리는 섞어도 무방.
- **현재 설치처**: fork = `D:\Claude for SAP\superclaude_for_sap` (Local). babamba2 = 전역 제거됨(어디서도 안 뜸). 마켓플레이스는 `sc4sap`+`sc4sap-custom` 둘 다 등록(공존).
- **다른 PC/폴더에 깔 때**: ① `/plugin marketplace add hjaewon/superclaude-for-sap` (1회) → ② 타깃 폴더 세션에서 `/plugin` Discover → Local 설치 → ③ `/reload-plugins`.
