# gostop.html 분석 문서

3인용 고스톱(점백) 게임. **단일 HTML 파일**(`gostop.html`, 약 4,100줄, ~900KB)로 HTML+CSS+JS가
전부 들어있고, 외부 의존성은 구글 폰트 하나뿐. 서버/빌드 없이 브라우저에서 바로 연다.
카드 그림도 이미지 파일 없이 전부 인라인 SVG를 JS로 생성한다.

## 파일 구조 개관

| 구간(줄) | 내용 |
|---|---|
| 1–257 | `<style>` — CSS. 테마(`[data-theme]`), 카드/보드 스타일, 반응형(1280px+ 확대, 가로모드) |
| 259–436 | `<body>` — 설정 화면(`#setup`), 갤러리 모달, 게임 화면(`#game`), 오버레이(`#overlay`) |
| 437–4097 | `<script>` — 게임 엔진 전체 (섹션 구분은 아래 표) |

### `<script>` 내부 섹션

| 줄 | 섹션 |
|---|---|
| 438–460 | 전역 JS 에러를 화면에 빨간 배너로 띄우는 안전장치 (디버그용, 정상 동작엔 무관) |
| 461–495 | 상수: `PLAYER_NAMES`, `HUMAN=0`, `FULL`(카드 이름 사전), `labelOf()` |
| 496–1657 | **카드 SVG 그림 생성 함수들** — 꽃/새/리본 등 화투 그림을 그리는 헬퍼 다수(`plumBlossom`, `sakuraPetal5`, `mapleLeaf`, `SCENE`, `SCENE_*_OVERRIDE`, `GWANG_OVERLAY` 등). 게임 로직과 무관, 순수 렌더링. `cardSVG(card)`(1567) 가 최종 진입점 |
| 1658–1706 | 카드/덱 데이터: `MONTH_CARDS`, `buildDeck()`, `shuffle()` |
| 1707–2427 | **게임 로직 + AI 판단** (핵심, 아래 별도 절 참고) |
| 2429–2688 | 전역 상태 `G`, 세션/타이머 유틸, 테마·라벨·자동진행 설정, 갤러리 |
| 2689–2793 | 라운드 시작(`uiStartNewGame`, `startRound`) |
| 2794–3630 | 턴 진행: 로그/토스트, 폭탄, 흔들기, `proceedTurn`, 카드 내기/뽑기(`playHandCard`~`doDrawResolve`), 애니메이션 앵커 계산 |
| 3630–3780 | 고/스톱, 정산(`finalizeRound`, `endRoundByExhaustion`, `applyGobak`, `settleCoins`) |
| 3781–3963 | `render()` — DOM 갱신 |
| 3964–4097 | 오버레이 화면들(라운드 종료/게임 오버/나가리/나가기), 다음 판·리셋 |

## 게임 상태 객체 `G`

전역 단일 객체(`let G = null`, 2432줄). 라운드 시작 시 재생성. 주요 필드:

- `coins[3]`, `round`, `dealer`(선 플레이어 인덱스), `over`, `lastWinner`, `nagariMultiplier`(나가리 누적 배율)
- `deck`(배열, `pop()`으로 뽑음), `hands[3]`, `floor`(바닥패), `captured[3]`(각자 먹은 패)
- `goCount[3]`, `lastGoScore[3]`(고를 부른 시점 점수 — 이후 갱신 기준), `shakeCount[3]`, `shakenMonths[3]`, `shakenCards[3]`
- `bombSkip[3]`(폭탄으로 얻은 "패 내지 않고 넘기기" 잔여 횟수), `ppeokMakeCount[3]`(뻑 3회=즉시승 카운트)
- `turn`(현재 차례), `phase`(상태머신, 아래 참고), `log`(최근 40줄), `roundOver`
- 임시 컨텍스트: `chooseCtx`(같은 월 2장 중 선택 대기), `flexCtx`(9월 열끗 쌍피/열끗 선택 대기), `bonusPending`, `bombCandidates`, `pendingScore`(고/스톱 대기 점수)

### `phase` 상태머신
`idle` → `humanPlay`(사람 차례, 카드 클릭 대기) → `resolving`/`drawing` → (`choosingCapture` 갈림 가능) →
`choosingPiMode`(9월 카드 확정 대기) → `goStopChoice`(3점 이상 달성 시) → 다음 턴, 또는 `bombSkipChoice`,
`roundEnd`. `AUTO_PLAY_PHASES`(2611줄)에 열거된 phase에서만 자동 진행 타이머가 작동한다.

## 카드 데이터 모델

- 카드 객체: `{id, month(1~12, 보너스는 0), type, ttiType?, bird?, flexible?, isBonus?, ppeokOwner?, ppeokMonth?}`
- `type`: `gwang`(광) / `yeol`(열끗) / `tti`(띠) / `pi`(피) / `ssangpi`(쌍피)
- `ttiType`: `hong`(홍단)/`cho`(초단)/`cheong`(청단)/`rain`(비, 단 조합 불가)
- `bird:true`: 고도리 대상 열끗(2·4·8월)
- `flexible:true`: 9월 열끗 — 획득 시점에 열끗/쌍피 중 선택(`resolveFlexThenAfterTurn`, 3550줄)
- 총 48장 = 12개월×4장 + 보너스 1장(`buildDeck`, 1684줄). 보너스는 항상 쌍피 취급.
- `ppeokOwner`: 뻑으로 묶인 카드 표시(그 카드를 놓은 사람 인덱스, 시작패 3장 우연 일치는 `-1`)

## 라운드 진행 흐름

1. `uiStartNewGame()`(2689) → `startRound()`(2705): 셔플, 각자 7장, 바닥 6장. 바닥에 보너스카드가 나오면
   선이 즉시 가져가고 한 장 더 보충. 총통(시작패 같은 월 4장) 체크 → 즉시 3점 승리. 바닥에 같은 월
   4장이면 선이 가져가고, 3장이면 뻑으로 묶음.
2. `proceedTurn()`(2963): 현재 턴 플레이어 처리. 폭탄 스킵 잔여 턴이면 `bombSkipChoice`/AI 자동판단.
   사람이면 `humanPlay`로 전환해 클릭 대기, AI면 700ms 후 `aiPlayTurn`.
3. 카드 내기: `playHandCard()`(1831) — 바닥에 같은 월이 1장뿐이면 **즉시 먹지 않고 보류**(다음 뽑을
   패로 뻑 여부 확정), 0장이면 바닥에 내려놓음, 2장(뻑 아님)이면 선택 UI, 3장 이상/뻑 묶음이면 전체 캡처.
4. 덱에서 한 장 뽑기: `doDraw`→`doDrawResolve`(3371~3548). 보류했던 카드와 뽑은 카드가 같은 월이면
   **뻑** 성립(3장 모두 바닥에 묶임, `ppeokMakeCount` 3회째면 즉시 3점 승리). 다르면 보류패 확정 캡처 후
   뽑은 카드도 `resolvePlayAsync`(1802)로 별도 처리. 이 시점에 **따닥**(낸 패+뽑은 패 모두 캡처) /
   **쪽**(낸 패는 못 먹었는데 뽑은 패가 같은 월이라 먹음) 판정.
5. `afterTurn()`(3581): 싹쓸이(바닥 0장) 체크 → 점수 3점 이상이고 `lastGoScore`보다 올랐으면 고/스톱
   대상. 낼 패가 더 없으면 자동 스톱, 있으면 사람은 `goStopChoice` UI, AI는 `aiDecideGoStop()`.
6. `advanceTurn()`(3630) → 다음 사람 `proceedTurn()`. 모두 손패 소진(+폭탄 스킵 잔여도 없음)이면
   `endRoundByExhaustion()`(3719) — 3점 넘긴 사람 없으면 **나가리**(배율 누적, 무효), 있으면 그 사람 승리
   (단 고를 부르고 이후 득점 못했으면 무효 처리, `hadVoidedLeader`).
7. 승부가 나면 `finalizeRound()`(3683) 또는 `endRoundByExhaustion`이 정산 후 `showRoundEndOverlay`/
   `showGameOverOverlay`(누군가 코인 부족)/`showVoidOverlay`(나가리) 표시.

## 특수 규칙 구현 위치

| 규칙 | 구현 |
|---|---|
| 뻑(ppeok) | `doDrawResolve`의 `handRes.deferred` 분기(3433~), `ppeokOwner` 필드로 표시. 3회 시 즉시승 |
| 따닥/쪽/싹쓸이 | `doDrawResolve`(3516~3532), `afterTurn`(3593~) — 모두 `ttadakBonus()`→`transferPi()`(1875) 호출 |
| 폭탄 | `checkBombCandidates`(2873), `executeBomb`(2942) — 흔들기와 동일 배율 취급, `bombSkip` +2 부여 |
| 흔들기 | `checkShakeCandidates`(2906), `declareShake`(2922), 패를 실제로 낼 때 자동 인정 `autoShakeOnPlay`(2931) |
| 총통 | `startRound` 내부 `finalizeInitialSetup`(2734) |
| 나가리 | `endRoundByExhaustion`의 `best<3` 분기, `G.nagariMultiplier` 2배씩 누적 |
| 고박 | `applyGobak()`(3665) — 고 부른 사람이 진 경우 안 부른 쪽 몫까지 전액 부담 |
| 광박/피박 | `finalizeRound`/`endRoundByExhaustion` 내 `oppG===0`(광박), `oppPi 1~5`(피박) → 2배, 중첩 시 4배 |
| 먹은 패 0장 면제 | 같은 곳, `oc.length===0` → `pay=0` |
| 보너스 카드 | 초기 배분(`startRound`), 손패로 냄(`playBonusHandCard`, 3200), 덱에서 나옴(`doDraw` 내 `drawn.isBonus`), 끝까지 안 나오면 `endRoundByExhaustion` 시작부에서 마지막 턴 플레이어가 획득 |
| 9월 열끗 열끗/쌍피 선택 | `resolveFlexThenAfterTurn`(3550), `chooseFlexMode`(3570), AI는 `resolveFlexCardAuto`(3563) |
| 점수 계산 | `computeScore(capturedCards)`(1888) — 광 3/4/5장, 띠 5장+/홍단·초단·청단, 열끗 5장+/고도리, 피 10장+ |
| 멍따(열끗 7장+) 2배 | `finalizeRound`/`simulateStopOutcome`의 `mungttaMultiplier` |

## AI 판단 로직 (사람도 동률 매칭 선택엔 같은 함수 공유)

AI는 미래를 탐색하지 않는 **휴리스틱 가중치 방식**. 핵심 함수:

- `cardWeight(card, playerIdx)`(1724): 카드 한 장의 "기본 가치"를 숫자로. 광=5(비광 2.8), 쌍피=5,
  피=3, 단/고도리는 그 조합이 아직 완성 가능한지(`ttiComboAchievable`/`godoriAchievable`, 1710/1718)에
  따라 가중치가 크게 달라짐(이미 두 명에게 갈린 색은 "죽은 조합"으로 취급).
- `scoreProximityValue(pile, addedCards)`(1927): 이 카드(들)를 먹으면 점수 문턱(광/단/고도리/피)에
  얼마나 가까워지는지 추정 — 내 이득 판단과 "상대에게 넘기면 위험한 정도" 판단에 공용으로 쓰임.
- `opponentRiskIfLeft(playerIdx, cards)`(1966): 지금 안 먹고 바닥에 남기면 상대에게 얼마나 위험한지.
  단순 위험 크기뿐 아니라, 그 결과 **누군가 파산해서 게임이 끝난다면 그게 나한테 유리한지**까지
  `simulateStopOutcome`으로 미리 시뮬레이션해서 반영(파산이 나에게 유리하면 위험도를 낮춤).
- `aiPickCaptureTarget(matches, playerIdx)`(1765): 바닥에 같은 월 2장 있을 때 어느 걸 가져올지 —
  `chooseCaptureTarget`(사람이 클릭)도 이 함수와 별개로 사람 선택을 그대로 받되, 자동 진행 시엔 이
  함수가 대신 고른다.
- `aiChooseHandCard(hand, floor, allCaptured, playerIdx, excludeMonths)`(2042): 이번 턴에 낼 카드
  선택 — 가장 복잡한 함수. 즉시 득점 여부 → 상대 방치 위험(`blockRisk`) → 내 점수 근접도 → 손패
  지원도 → 기본 가중치 순으로 비교. 뻑 스윕은 안전하면 일부러 미루고, 흔든 상대의 패를 가로챌
  타이밍도 계산. 먹을 패가 없으면 폴백으로 "무엇을 버릴지"(상대 위험 최소화 우선) 결정.
- `aiDecideGoStop(score, deckRemaining, playerIdx)`(2401): 먼저 `simulateStopOutcome`으로 지금
  스톱하면 파산이 발생하는지 확인 — 발생하고 내가 그 순간 최종 승자면 무조건 스톱, 아니면 무조건 고.
  그 외엔 `assessGoStopFactors`(2295: 상대 위협/내 잠재력/업사이드)와 현재 코인 격차를 반영한
  확률(`goChance`)로 랜덤 결정.
- 코드 곳곳의 숫자 상수(예: `z=3.8`, `0.5→0.3` 같은 주석)는 **대량 시뮬레이션으로 튜닝된 값**이라는
  표시. 이 값들을 감으로 바꾸지 말고, 바꾸려면 근거(시뮬레이션)를 갖고 접근할 것.

## 렌더링 & 애니메이션

- `render()`(3809)가 유일한 DOM 갱신 함수 — 상태(`G`)를 바꾼 뒤 항상 이 함수를 호출해 화면을 동기화.
- 카드 이동은 `flyCardVisual`(3137)이 담당: 실제 DOM 카드는 그대로 두고 복제한 `.flyer-card`를
  화면에 띄워 좌표 애니메이션 후 제거. 좌표는 `rectAnchor`류 함수들(`floorAnchor`, `panelAnchor`,
  `capturedPanelAnchor`, `handCardAnchor` 등, 3053~3129)이 실제 DOM 요소 위치를 읽어 계산.
- `safeTimeout(fn, delay)`(2444)와 `gameSession` 카운터: "나가기"나 "새 게임"으로 세션이 바뀐 뒤에도
  이전 라운드의 `setTimeout` 체인이 뒤늦게 실행되며 새 게임 상태를 건드리는 사고를 막는 안전장치.
  **턴 진행 중 지연 실행되는 코드를 추가할 때는 반드시 `setTimeout` 대신 `safeTimeout`을 쓸 것.**
- 자동 진행(AUTO_PLAY) 두 종류: 평소 턴(`armAutoPlayTimer`/`performAutoPlayAction`, 2625/2639)과
  오버레이 팝업(`armOverlayAutoTimer`, 2261) — 둘 다 설정 화면의 "자동 진행 시간"(0=끄기)을 따름.

## UI 설정 (localStorage에 저장)

- 테마: `uiSetTheme` → `gostop-theme` (`navy`/`green`/`wine`/`charcoal`, CSS `[data-theme]`)
- 카드 라벨 표시: `uiSetLabelPref` → `gostop-show-labels` (`html[data-hide-labels]`)
- 자동 진행 시간: `uiSetAutoPlayDelay` → `gostop-autoplay-delay`
- 코인/이름은 저장 안 되고 시작 화면 입력값을 그대로 씀. 사람이 입력한 이름은 `escapeHtml()`로
  이스케이프해 `PLAYER_NAMES[0]`에 저장(로그 등 innerHTML 삽입 위험 방지), 원본은 `PLAYER_RAW_NAME`.

## 반응형 레이아웃 (CSS, 201~257줄)

- 기본(모바일): 세로 1단 배치.
- `min-width:1280px and min-height:640px`: `body{zoom:1.5}` + 좌우 상대 패널·바닥·내 패를 가로 3단
  배치로 재배열(PC용).
- `orientation:landscape and max-height:560px`: 확대 없이 가로 3단 배치(휴대폰 가로모드용).

## 향후 수정 시 참고할 점

- **로직 추가/수정은 대부분 1707~2427줄(게임 로직+AI) 또는 2793~3780줄(턴 진행/정산)에 집중**된다.
  496~1657줄(카드 그림)과 CSS는 룰과 무관하므로 시각적 요청이 아니면 건드릴 필요 없음.
- 새 규칙을 추가하면 `computeScore`, `finalizeRound`/`endRoundByExhaustion`(정산), 그리고 AI의
  `cardWeight`/`aiChooseHandCard`/`aiDecideGoStop` 세 곳에 **일관되게** 반영해야 사람전/AI전에서
  같은 규칙이 적용된다.
- 상태 변경 후에는 항상 `render()` 호출, 지연 실행은 `safeTimeout` 사용, 사용자 입력 문자열은
  `escapeHtml()`을 거쳐야 하는 곳인지 확인(현재는 이름 입력 하나뿐).
- 규칙 문서 자체가 `#setup .rule-box`(267~312줄)에 한국어로 이미 상세히 들어있어, 규칙 문구를
  바꿀 때는 이 HTML과 실제 로직 두 곳을 함께 수정해야 함(자동 동기화 없음).
