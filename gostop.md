# gostop.html 분석 문서

3인용 고스톱(점백) 게임. **단일 HTML 파일**(`gostop.html`, 총 4,552줄)로 HTML+CSS+JS가 전부
들어있고, 외부 의존성은 구글 폰트 하나뿐. 서버/빌드 없이 브라우저에서 바로 연다. 카드 그림도
이미지 파일 없이 전부 인라인 SVG를 JS로 생성한다.

## 파일 구조 개관

| 구간(줄) | 내용 |
|---|---|
| 1–290 | `<style>` — CSS. 테마(`[data-theme]`), 카드/보드 스타일, 각종 토스트(`#goToast`/`#bonusToast`/`#dealToast`/`#chongtongToast`), 강조 표시(`.score-highlight`/`.shake-target`/`.choose-target`/`.bonus-drawn`), 반응형(1280px+ 확대, 가로모드) |
| 291–462 | `<body>` — 규칙 설명(`.rule-box`, 299줄부터), 설정 화면(`#setup`), 갤러리 모달, 게임 화면(`#game`), 오버레이(`#overlay`) |
| 463–4552 | `<script>` — 게임 엔진 전체 (섹션 구분은 아래 표) |

### `<script>` 내부 섹션

| 줄 | 섹션 |
|---|---|
| 464–486 | 전역 JS 에러를 화면에 빨간 배너로 띄우는 안전장치 (디버그용, 정상 동작엔 무관) |
| 490–522 | 상수: `PLAYER_NAMES`, `HUMAN=0`(암묵적으로 0), `won()`, `escapeHtml()`, `labelOf()` |
| 523–1615 | **카드 SVG 그림 생성 함수들** — 꽃/새/리본 등 화투 그림을 그리는 헬퍼 다수. 게임 로직과 무관, 순수 렌더링. `cardSVG(card)`가 최종 진입점 |
| 1616–1670 | 카드/덱 데이터: `MONTH_CARDS`, `hiddenCardTypesForMonth()`, `buildDeck()`, `shuffle()` |
| 1671–2357 | **게임 로직 + AI 판단** (핵심, 아래 별도 절 참고) |
| 2363–2561 | 오버레이 자동 진행 타이머, 고/스톱 판단(`assessGoStopFactors`/`simulateStopOutcome`/`aiDecideGoStop`) |
| 2566–2822 | 전역 상태 `G`, 세션/타이머 유틸(`safeTimeout`), 갤러리, 테마·라벨·자동진행 설정 |
| 2823–2946 | 라운드 시작(`uiStartNewGame`, `startRound`, 초기 총통/보너스/4장/3장 처리) |
| 2946–3232 | 로그·토스트 큐, 폭탄, 흔들기 판정 함수들 |
| 3232–3618 | 턴 진행 본체: `proceedTurn`, 애니메이션 앵커 계산, 총통(중도)·보너스 카드 내기·`aiPlayTurn`·`humanPlayCard` |
| 3618–3961 | 피 이동(`ttadakGains`/`transferPiAnimated`), 덱 뒤집기·카드 내기/뽑기 해석(`doDraw`~`doDrawResolve`, 뻑/따닥/쪽/싹쓸이 핵심부), 9월 유동형 카드 확정 |
| 3962–4213 | `afterTurn`/`afterTurnDecide`(고/스톱 트리거), `advanceTurn`, 정산(`finalizeRound`, `endRoundByExhaustion`, `applyGobak`, `settleCoins`) |
| 4214–4357 | `render()` — DOM 전체 갱신 |
| 4358–4552 | 카드 엘리먼트 생성(`makeCardEl`, `renderCapturedGrid`), 오버레이 화면들(라운드 종료/게임 오버/나가리/나가기), 다음 판·리셋 |

## 게임 상태 객체 `G`

전역 단일 객체(`let G = null`, 2566줄). 라운드 시작 시 재생성. 주요 필드:

- `coins[3]`, `round`, `dealer`(선 플레이어 인덱스), `over`, `lastWinner`, `nagariMultiplier`(나가리 누적 배율)
- `deck`(배열, `pop()`으로 뽑음), `hands[3]`, `floor`(바닥패), `captured[3]`(각자 먹은 패)
- `goCount[3]`, `lastGoScore[3]`(고를 부른 시점 점수 — 이후 갱신 기준), `shakeCount[3]`, `shakenMonths[3]`, `shakenCards[3]`
- `bombSkip[3]`(폭탄으로 얻은 "패 내지 않고 넘기기" 잔여 횟수), `ppeokMakeCount[3]`(뻑 3회=즉시승 카운트)
- `turn`(현재 차례), `phase`(상태머신, 아래 참고), `log`(최근 로그), `roundOver`
- `scoreHighlight`: `{playerIdx, ids:Set}` — 지금 이 사람의 캡처 패널에서 점수 사유로 강조 표시할 카드 id 집합(고/스톱 선택창, 라운드 종료 화면 공용)
- 임시 컨텍스트: `chooseCtx`(같은 월 2장 중 선택 대기), `flexCtx`(9월 열끗 쌍피/열끗 선택 대기), `bonusPending`, `bombCandidates`, `pendingScore`(고/스톱 대기 점수)

### `phase` 상태머신
`idle` → `humanPlay`(사람 차례, 카드 클릭 대기) → `resolving`/`drawing` → (`choosingCapture` 갈림 가능) →
`choosingPiMode`(9월 카드 확정 대기) → `goStopChoice`(3점 이상 달성 시) → 다음 턴, 또는 `bombSkipChoice`,
`roundEnd`.

## 카드 데이터 모델

- 카드 객체: `{id, month(1~12, 보너스는 0), type, ttiType?, bird?, flexible?, isBonus?, ppeokOwner?, ppeokMonth?, bonusDrawn?}`
- `type`: `gwang`(광) / `yeol`(열끗) / `tti`(띠) / `pi`(피) / `ssangpi`(쌍피)
- `ttiType`: `hong`(홍단, 1·2·3월) / `cho`(초단, 4·5·7월) / `cheong`(청단, 6·9·10월) / `rain`(12월 띠, 단 조합 불가)
- `bird:true`: 고도리 대상 열끗(2·4·8월)
- `flexible:true`: 9월 열끗 — 획득 시점에 열끗/쌍피 중 선택(`resolveFlexThenAfterTurn`, 3932줄). 확정되기 전에는
  화면에서도 **피 줄에 기본으로 자리잡은 채** 표시된다(`renderCapturedGrid`, 4387줄 — 쌍피가 대개 더 유리하다는
  전제). 열끗으로 확정될 때만 열끗 줄로 이동하는 애니메이션(`animateFlexTypeResolved`)이 재생된다.
- `bonusDrawn:true`: 보너스 카드를 낸 뒤 덱에서 받은 카드에 표시 — 손패에서 파란 테두리로 강조되고, 그 턴이
  끝나는 시점(`afterTurn`, 3965줄)에 지워진다.
- 월별 카드 구성(`MONTH_CARDS`, 1616줄 / `buildDeck`, 1642줄):

  | 월 | 구성 |
  |---|---|
  | 1·3월 | 광, 홍단 띠, 피×2 |
  | 2월 | 고도리 열끗, 홍단 띠, 피×2 |
  | 4·7월 | 고도리 열끗(4월만)/일반 열끗(7월), 초단 띠, 피×2 |
  | 5월 | 일반 열끗, 초단 띠, 피×2 |
  | 6월 | 일반 열끗, 청단 띠, 피×2 |
  | 8월 | 광, 고도리 열끗, 피×2 |
  | 9월 | 유동형 열끗(flexible), 청단 띠, 피×2 |
  | 10월 | 일반 열끗, 청단 띠, 피×2 |
  | 11월 | 광, 쌍피, 피×2 |
  | 12월 | 광, 일반 열끗, 비(rain) 띠, 쌍피 |

- 총 12개월×4장(48장) + 보너스 카드 1장(`month:0, type:'ssangpi', isBonus:true`) = **덱 49장**. 보너스는 항상
  쌍피 취급(피 점수 계산에 2장으로 카운트).
- `ppeokOwner`: 뻑으로 묶인 카드 표시(그 카드를 놓은 사람 인덱스, 시작패 우연히 3장 겹친 경우는 `-1`)

## 라운드 진행 흐름

1. `uiStartNewGame()`(2823줄) → `startRound()`(2839줄): 셔플, 각자 7장, 바닥 6장. `render()`로 방금 나눠준
   상태를 그대로 먼저 보여주고 **"카드 배분 완료" 안내 배너**(`showDealToast`, 3038줄)를 띄워 잠깐 텀을
   준 뒤에야 후속 처리를 시작한다(컴퓨터가 바로 진행해버려 초기 배분을 놓치는 것을 방지).
2. `processInitialChongtong()`(2861줄): 시작 손패 같은 월 4장(총통) 체크 → 있으면 **"총통!" 토스트**로 그
   4장을 보여준 뒤(`showChongtongToast`, 3060줄) 손패에서 캡처 더미로 옮기고 즉시 3점 승리로 라운드 종료.
   두 명 이상 동시 총통이면 월이 더 높은 쪽이 승리.
3. `processInitialFloorBonusCard()`(2893줄): 바닥에 보너스 카드가 있으면 선이 가져가고 한 장 더 보충(재귀
   — 보충한 카드도 보너스일 수 있음). → `processInitialFloorQuadMonth()`(2916줄): 바닥에 같은 월 4장이면
   선이 그대로 가져감. → `processInitialFloorTripleMonth()`(2934줄): 바닥에 같은 월 3장이면 뻑으로 묶음
   (`ppeokOwner=-1`). 끝나면 `proceedTurn()`으로 실제 턴 진행 시작.
4. `proceedTurn()`(3232줄): 현재 턴 플레이어 처리. 손 3장+바닥 매칭이 있으면 폭탄 후보 판정
   (`checkBombCandidates`)부터 하고, 없으면 사람은 `humanPlay`로 전환(카드 클릭 대기), AI는 700ms 후
   `aiPlayTurn`.
5. 카드 내기: `playHandCard()`(1792줄) — 바닥에 같은 월이 1장뿐이면 **즉시 먹지 않고 보류**(다음에 뽑을
   패로 뻑 여부를 확정), 0장이면 바닥에 내려놓음, 2장(뻑 아님, 피-피 제외)이면 어느 걸 먹을지 **이어지는
   덱 뒤집기 결과를 본 뒤에** 결정(`resolvePendingFloorChoice`, 1836줄), 3장 이상/뻑 묶음이면 전체 캡처.
6. 덱에서 한 장 뽑기: `doDraw()`→`doDrawResolve()`(3695~3917줄). 보류했던 카드와 뽑은 카드가 같은 월이면
   **뻑** 성립(3장 모두 바닥에 묶임, `ppeokMakeCount` 3회째면 즉시 3점 승리). 다르면 보류패 확정 캡처 후
   뽑은 카드도 `resolvePlayAsync`(1763줄)로 별도 처리. 이 시점에 **따닥**(낸 패+뽑은 패 모두 같은 월로
   캡처) / **쪽**(낸 패는 못 먹었는데 뽑은 패가 같은 월이라 먹음, 손패가 더 남아있을 때만) 판정.
7. `afterTurn()`(3962줄): 싹쓸이(캡처 후 바닥 0장, 마지막 손패가 아닐 때) 체크 → `afterTurnDecide()`
   (3996줄): 이번에 캡처가 있었고 점수 3점 이상이며 직전 고 시점 점수보다 올랐으면 고/스톱 대상. 낼 패가
   더 없으면 자동 스톱, 있으면 사람은 `goStopChoice` UI, AI는 `aiDecideGoStop()`.
8. `advanceTurn()`(4029줄) → 다음 사람 `proceedTurn()`. 모두 손패 소진(+폭탄 스킵 잔여도 없음)이면
   `endRoundByExhaustion()`(4124줄) — 3점 넘긴 사람 없으면 **나가리**(배율 누적, 무효), 있으면 그 사람 승리
   (단 고를 부르고 이후 득점 못했으면 무효 처리, `hadVoidedLeader`).
9. 승부가 나면 `finalizeRound()`(4081줄) 또는 `endRoundByExhaustion`이 정산 후 `showRoundEndOverlay`/
   `showGameOverOverlay`(누군가 코인 부족)/`showVoidOverlay`(나가리) 표시.

## 특수 규칙 구현 위치

| 규칙 | 구현 |
|---|---|
| 뻑(ppeok) | `doDrawResolve`의 `handRes.deferred` 분기(3756줄~), `ppeokOwner` 필드로 표시. 3회 시 즉시승(`G.ppeokMakeCount`) |
| 따닥/쪽/싹쓸이 | `doDrawResolve`(3851~3917줄), `afterTurn`(3979~) — 모두 `ttadakGains()`(3618줄)→`transferPiAnimated()`(1856줄)로 상대에게서 피 1장씩(없으면 쌍피) 받음 |
| 폭탄 | `checkBombCandidates`(3114줄), `shouldExecuteBombNow`(3135줄, 즉시 이득 없으면 미룸), `executeBomb`(3178줄) — 흔들기와 동일 배율로 취급, `bombSkip` +2 부여. **사람도 폭탄 후보 카드를 탭하면 확인창 없이 바로 실행**(`humanPlayCard`, 3576줄) |
| 흔들기 | `checkShakeCandidates`(3147줄), `declareShake`(3163줄), 3장 중 아무거나 실제로 낼 때 자동 인정하는 `autoShakeOnPlay`(3172줄) — 안내 문구 대신 손패에서 해당 3장을 주황색으로 강조 표시(`render`, `.shake-target`) |
| 총통 | 시작패는 `processInitialChongtong`(2861줄), 중도(손패에 4장이 모이는 경우, 예: 3장 들고 있다가 보너스 카드 보상으로 4번째를 뽑음)는 `checkMidRoundChongtong`(3494줄) |
| 나가리 | `endRoundByExhaustion`의 `best<3` 분기, `G.nagariMultiplier` 2배씩 누적 |
| 고박 | `applyGobak()`(4063줄) — 고를 부른 사람이 지고 상대 한쪽은 고를 안 불렀다면, 고를 부른 쪽이 두 사람 몫을 전액 부담 |
| 광박/피박 | `finalizeRound`/`endRoundByExhaustion`/`simulateStopOutcome` 내 `oppG===0`(광박), `oppPi 1~5`(피박) → 각각 2배, 중첩 시 4배 |
| 먹은 패 0장 면제 | 같은 곳, `oc.length===0 && !isChongtong` → `pay=0`(단 총통 승리는 상대가 먹은 게 없어도 면제 없음) |
| 보너스 카드 | 초기 배분(`processInitialFloorBonusCard`), 손패로 냄(`playBonusHandCard`, 3512줄 — 내면 정상 턴을 잃지 않고 덱에서 한 장 더 받은 뒤 이어서 정상 턴 진행), 덱에서 나옴(`doDraw` 내 `drawn.isBonus`), 끝까지 안 나오면 `endRoundByExhaustion` 시작부에서 마지막 턴 플레이어가 획득 |
| 9월 열끗 열끗/쌍피 선택 | `resolveFlexThenAfterTurn`(3932줄), `chooseFlexMode`(3952줄), AI는 `resolveFlexCardAuto`(3945줄) |
| 점수 계산 | `computeScore(capturedCards)`(1872줄) — 광 3/4/5장(비광 포함 3광은 2점, 아니면 3점), 띠 5장+(1점씩)/홍단·초단·청단 각 3점, 열끗 5장+(1점씩)/고도리(새 3장) 5점, 피 10장+(1점씩, 쌍피=2장) |
| 멍따(열끗 7장+) 2배 | `finalizeRound`/`simulateStopOutcome`의 `mungttaMultiplier`(`yCount>=7`) |
| 점수 강조 표시 | 고/스톱 선택창·라운드 종료 화면에서 실제로 점수에 기여한 캡처 카드를 초록색으로 강조(`G.scoreHighlight`, `scoringCardIds`/`.score-highlight`). 종료 팝업은 화면 전체를 덮어 뒤쪽 패널이 안 보이므로 팝업 안에도 직접 작게 보여준다(`showRoundEndOverlay`, 4409줄) |

## AI 판단 로직 (사람도 동률 매칭 선택엔 같은 함수 공유)

AI는 미래를 탐색하지 않는 **휴리스틱 가중치 방식**이며, 사람 좌석(HUMAN)도 같은 월 2장 중 하나를
직접 선택하는 경우가 아닌 이상 대부분의 보조 판단(캡처 대상 결정 등)에 동일한 함수를 공유한다.

### 보조 판단 함수

- `cardWeight(card, playerIdx)`(1687줄): 카드 한 장의 "기본 가치"를 숫자로 환산. (3차 튜닝: 광/쌍피/피/
  띠achievable/열끗achievable 5개 기본값을 각각 **독립적으로**(다른 축은 고정) -1/-0.5/+0.5/+1로 흔들었을
  때는 5개 전부 95% 유의성 문턱(z=1.96)을 재현 가능하게 넘지 못했다(광 z=2.04→재검증 시 0.84로 무너짐,
  쌍피 z=1.59, 피 z=1.92~2.02 사이를 맴돎, 띠/열끗achievable 미달). 4차 튜닝: 그런데 **쌍피 5→6 + 피
  3→3.5 + 열끗achievable 1.2→1.5를 동시에** 바꾼 결합 후보는 표본을 늘릴수록 z가 계속 커지는(N=1500
  z=1.55 → N=4000 z=2.06 → N=8000 z=2.29) 패턴을 보여 진짜 개선으로 판단해 **채택**했다 — 개별로는 약한
  신호들이 합쳐지며 뚜렷해진 사례. 홍단 방어·확정 짝 방어·초단 긴급도 등 정성적 회귀 테스트와 자동 진행
  안정성도 전부 통과 확인.)
  - 광 = 5(12월 비광만 2.8 — 3광 점수가 3점→2점으로 깎이기 때문)
  - 쌍피 = 6, 피 = 3.5(4차 튜닝으로 5/3에서 상향)
  - 띠/열끗(고도리)은 그 조합이 **이 플레이어 본인 기준으로** 아직 완성 가능한지(`ttiComboAchievable`/
    `godoriAchievable`)에 따라 갈림: 가능하면 띠=2, 고도리열끗=1.5(4차 튜닝으로 1.2에서 상향). 이미 죽은
    조합이면 1.2(띠)/1.25(열끗)를 기본으로 하되, `flatThresholdBonus()`(1713줄 — 5장 이상 평문 점수 문턱에
    얼마나 가까운지, 이미 4장 이상 캡처했으면 +1.4까지)만큼 가산. 9월 유동형 카드는 열끗/쌍피 중 낮은
    쪽이라도 최소 쌍피 가치는 보장(`Math.max(6, asYeol)` — 쌍피가 6으로 오른 것에 맞춰 하한도 5→6으로
    같이 올림).
- `ttiComboAchievable(ttiType, playerIdx)` / `godoriAchievable(playerIdx)`(1671/1680줄): 이 색 단(또는
  고도리)을 **이 플레이어가 직접** 완성할 수 있는 상태인지 — 이미 캡처된 패의 소유자 집합(`owners`)을
  구해서, `playerIdx`가 주어지면 "나 아닌 다른 사람이 단 한 명이라도 이미 캡처했으면 불가능"(그 카드는
  나에게 영영 돌아오지 않으므로), 주어지지 않으면(전역 관점) "서로 다른 두 명에게 갈렸을 때만 불가능"으로
  더 느슨하게 판단.
- `scoreProximityValue(capturedPile, addedCards, handPile, playerIdx)`(1949줄): 이 카드(들)를 이 캡처
  더미에 더하면 점수 문턱에 얼마나 가까워지는지 추정. 실제 총점 증가분은 `(after-before)*3`으로 크게
  반영하고, 그 외에 광 1~4장/단 1·2장/고도리 1·2장/피 5·8장 근접 시 문턱 직전이라는 신호를 별도 가산
  (예: 단 2장째엔 +3, 광 4장째엔 +5). `handPile`을 넘기면(내 관점 판단에서만) 아직 손에만 있는 같은 종류
  패도 절반 가중치(`HAND_W=0.5`)로 반영 — 상대 위협 판단에는 상대 손패를 볼 수 없으므로 넘기지 않는다.
  `ttiComboAchievable`/`godoriAchievable`로 이미 죽은 조합이면 신호를 끈다.
- `opponentRiskIfLeft(playerIdx, cards)`(1994줄): 지금 안 먹고 바닥에 남기면 상대에게 얼마나 위험한지 —
  상대별로 `scoreProximityValue`를 구해 최댓값을 쓴다. 단순 위험 크기뿐 아니라, 그 결과 상대가 3점 이상을
  채워서 **누군가 파산해 게임이 끝난다면 그게 나한테 유리한지**까지 `simulateStopOutcome`으로 미리
  시뮬레이션해서 반영 — 나에게 유리하면 위험을 0.3배로 낮추고, 불리하면 2.5배로 키운다. 이미 다른 누군가
  고를 부른 상태에서 고 안 부른 상대를 도와주면 고박으로 그 사람이 손해를 보므로 0.6배로 할인.
- `estimateTtadakValue(playerIdx)`(2031줄): 뻑/폭탄 완성 시 상대에게서 받을 피 기대값(일반 피 있으면 1,
  없고 쌍피만 있으면 2, 상대별 합산).
- `shouldPlayBonusNow(hand, floor, playerIdx)`(2052줄): 보너스 카드를 지금 낼지 판단 — 기본은 즉시 내되,
  ①아직 일반 피가 0장이면 미룸(유일한 피인 쌍피를 뜯길 위험) ②상대가 피박 조건(10장+)을 갖췄고 내가
  피박 구간(1~5장)이면 위험을 감수하고 즉시 ③바닥에 손패로는 못 먹는 고가치 패(`cardWeight>=3.5`)가
  있으면 한 장 더 뽑아 노려볼 가치가 있어 즉시.

### 먹을 패 선택 — `aiChooseHandCard(hand, floor, allCaptured, playerIdx, excludeMonths)`(2070~2357줄)

가장 복잡한 함수. 손패를 순회하며 바닥과 매칭되는 카드마다 다음 우선순위 체인(`better` 비교, 2210~2216줄)
으로 "이번 턴에 낼 카드"를 고른다:

0. 보너스 카드가 있고 `shouldPlayBonusNow`가 참이면 그걸 최우선으로 바로 반환.
1. **즉시 득점 여부**(`immediateScore`) — 이 캡처로 내 점수가 3점 이상이 되고 직전 고 점수보다 오르는지.
2. **상대 방치 위험**(`blockRisk`) — `opponentRiskIfLeft(taken)`에 "이 월 4장 중 안 보이는 장수"
   (`hiddenCount`, 0~2)를 절반씩 스케일(`certaintyScale=hiddenCount/2`)해서 곱한다(확정된 패일수록 위험 0에
   가까움). 여기에 ①경합 중인 패 자체의 잔존 위험(`hiddenCount*0.5`) ②남기는 패 자체의 원가치
   (`taken 합 * 0.2`)를 더하고, 상대가 피박 조건인데 내가 피박 구간을 벗어날 수 있는 캡처라면 +5로 시급
   처리.
   - 뻑 스윕(taken 3장 이상)이 지금 당장 득점도 아니고 상대에게 줄 피도 없으면(`estimateTtadakValue===0`)
     — 어차피 아무도 못 끼어드는 확정 패이므로 서두르지 않고 일부러 미룬다(`suppressed`로 보류).
   - 상대가 이 월로 흔들었는데(`shakenMonths`) 아직 안 보이는 패 중 지금 잡는 것보다 더 값진 게 있고
     급하지 않다면, 상대가 그 값진 패를 낼 때 가로챌 기회를 노리고 마찬가지로 미룬다.
3. **내 점수 근접도**(`mineValue`) — `scoreProximityValue`(손에 남는 패도 절반 가중).
4. **동일 월 라이벌 여부**(`sameMonthRival`) — 손패 여러 장이 캡처 결과(`taken`)가 완전히 같은 후보(예:
   11월 쌍피 vs 피, 9월 유동형 열끗 vs 피)일 때는 **손패 지원도(`handSupport`) 단계를 건너뛰고** 바로
   `cardWeight`가 더 비싼 쪽을 우선한다 — 아껴봐야 나중에 더 좋아질 게 없고, 라운드가 끝날 때까지 못 내면
   값어치를 통째로 날리는 손해만 있기 때문. (`handSupport`는 "내는 손패 타입이 손에 남는 다른 카드와
   겹치는 정도"인데, 피는 워낙 흔해서 이 비교에서는 무관한 잡음이 되기 쉬워 라이벌 비교일 때만 제외한다.)
5. (라이벌이 아닐 때만) **손패 지원도**(`handSupport`) — 이 캡처 결과가 남는 손패와 타입이 얼마나 겹치는지.
6. **기본 가중치**(`baseWeight`) — `floorGain(taken 원가치 합) - handCardCost*0.3(내 손패 가치의 30%만
   손해로 침, 값싼 패부터 먼저 소모하도록) - deadCardPenalty(캡처 후 손에 남는 같은 월 중복 패 중 비싼
   쪽을 죽은 패로 손해 처리) - ppeokFormRisk(1장 매칭이라 보류될 경우 다음 드로우로 뻑이 되어 이번 캡처
   기회를 날릴 확률×가치, 고를 부른 상태면 1.5배) + ttadakEstimate(뻑 완성 시 실제 기대 피 보너스)`.

먹을 패가 하나도 없으면(`best`가 null) 폴백으로 **버릴 패**를 정한다(2230~2356줄):

0. 보너스 카드가 있으면 그대로 낸다.
1. 손에 같은 월 3장이 있고 바닥에 그 월이 아직 없으면(폭탄 대기) 그 3장은 최대한 보존.
2. 손에 같은 월 2장이 있는데 나머지 2장이 이미 전부 캡처되어 아무도 못 맞추는 상태면, 그중 싼 쪽을
   먼저 낸다(나머지 한 장으로 나중에 확실히 페어 확보).
3. 남은 후보들을 아래 기준으로 비교:
   - `oppRisk`: 기본은 `opponentRiskIfLeft([card])`. **"확정된 짝"(대리 방어패) 개념**: 이 카드가 직접
     단 색깔이거나, 이 월에서 안 보이는 카드가 정확히 1장이고 그게 하필 그 색 단이면(=나머지 3장이 이미
     다 설명됨), 그 숨은 단 자체를 손에 쥔 것과 동일한 방어 카드로 취급한다. 단, 같은 색을 막아줄 다른
     손패(직접 단이든 마찬가지로 확정된 대리 방어패든)가 이미 하나 더 있으면 — 그 색은 어차피 그 하나로
     상대에게 막혀 있으므로 이 카드까지 같이 지킬 필요는 없다고 보고 위험을 0으로 낮춘다.
   - 이 월에서 안 보이는 카드(`hiddenTypes`)가 있으면, 그게 나타났을 때의 내 이득/상대 위험을
     `hiddenTypes`가 정확히 1장이면 그 값을 그대로(확정된 짝이므로 두 카드 가치를 합친 것과 동일), 2장
     이상이면 **최댓값이 아니라 후보들의 평균**으로 기대값을 잡아 `mineVal`/`oppRisk`에 반영(실제로 나타날
     확률 = `hiddenTypes.length/deck.length`만큼만 가중). 상대 쪽은 1장짜리 조기 신호까지 반영하면
     과민반응이 되므로, `computeScore` 실제 문턱 돌파분(×3)만 쓴다.
   - `mineVal`: `scoreProximityValue(capturedPile, [card], restHand)` + 위 평균 보정.
   - `w`: `cardWeight(card) - jjokBonus`(지금 버리면 다음 드로우가 우연히 같은 월이라 "쪽" 보너스가 터질
     확률×기대 피 가치만큼 할인 — 버리기 더 매력적으로).
   - 비교 순서: `oppRisk` 낮은 것 → `mineVal` 낮은 것 → `w` 낮은 것.
4. 그래도 못 정하면 `cardWeight`가 가장 낮은 카드 중 무작위.

### 고/스톱 판단

- `aiDecideGoStop(score, deckRemaining, playerIdx)`(2535줄): 먼저 `simulateStopOutcome`(2495줄, 정산
  규칙을 그대로 미리 계산)으로 **지금 스톱하면 누군가 파산해서 게임이 끝나는지** 확인. 파산이 발생한다면
  — 그 순간 내가 코인 기준 최종 1위면 무조건 `stop`, 아니면(내가 진다면) 무조건 `go`(역전을 노림).
  파산이 없으면 점수 15점 이상은 무조건 `stop`. 그 외엔 아래 `goChance`로 확률적 결정
  (`Math.random()<goChance`, 최종적으로 0.05~0.9로 클램프):

  ```
  goChance = 0.23 - oppRisk*1.1 + myPotential*0.45 + upside*0.4 - deckPenalty - relStanding*0.3
  ```
  - `deckPenalty = (1-deckRemaining/21)*0.15` — 덱이 거의 바닥나면 나가리 위험까지 감안해 스톱 쪽으로.
  - `relStanding = (내 코인 - 상대 평균 코인)/전체 코인` — 앞서 있으면(양수) `goChance`를 낮추고(보수적),
    뒤처져 있으면(음수) 높인다(공격적). 게임의 목표는 이번 판 승리가 아니라 최종적으로 더 부유하게
    남는 것이라는 원칙.
  - 상수들(`0.23`, `1.1`, `0.45`, `0.4`, `0.15`, `0.3` 등)은 헤드리스 시뮬레이터로 **비대칭 자기대국**
    (한 좌석만 후보 값, 나머지는 기존 값 — 동일 알고리즘끼리 붙이면 항상 33%로 수렴해 무의미하므로)
    방식으로 튜닝한 값. `0.15→0.23`은 라운드 EV(N=12,000)·매치 승률(N=3,000) 양쪽에서 유의미하게
    검증됨(코드 주석에 근거 명시). 튜닝 지표는 승률이 아니라 **평균 획득 점수/코인**을 우선한다(자세한
    원칙은 `CLAUDE.md`의 "AI 가중치 통계 튜닝 지침" 참고).

- `assessGoStopFactors(playerIdx)`(2397줄) — 세 가지 요인을 0~1로 정규화해서 반환:
  - **`oppRisk`**: 상대 중 위협이 가장 큰 쪽의 `computeScore` 총점(이미 고를 불렀으면 +2) + 바닥에
    노출된 카드로 즉시 문턱을 넘는 경우(고도리 2장 먹고 3번째 새가 바닥에 있는 등, 3점 이상 점프만
    반영) + 바닥에 아직 안 걷힌 뻑/보너스 카드가 있는데 내가 그 월 패를 못 쥐고 있으면 위험 가산(쥐고
    있으면 오히려 `myPotential` 쪽에 가산). `/3.84`로 정규화.
  - **`myPotential`**: 손패 중 아직 다른 사람에게 3장 이상 잠식되지 않은 "살아있는" 월 비율
    (`liveMonthHits/hand.length * 0.384`) + 문턱 근접 가산(`nearThreshold`: 광 2~4장 +0.3, 띠/열끗
    정확히 4장 +0.15씩, 피 8장 이상 +0.16, 뻑 더미/미획득 보너스 카드를 내가 쥐고 있으면 +0.3씩).
  - **`upside`**: 계속 고를 했을 때 배율이 커질 여지.
    - 흔들기 1회 이상이면 +0.1875.
    - **3고 이상**(`goCount>=2`, 즉 이번에 고 하면 3고 달성 또는 이미 3고 이상)이면 +0.25 — 3고부터는
      한 번 더 고할 때마다 배율이 다시 2배가 되므로(3고 x2 → 4고 x4 → 5고 x8 …), 3고를 막 넘긴 뒤에도
      똑같이 반영(처음 한 번만 반영하지 않음).
    - **3고 이상까지 갈 잠재력**을 goCount와 무관하게 첫 고 판단부터 반영: 남은 손패 수
      (`turnsLeft=hand.length`, `/6`로 정규화)만큼 기회가 많다고 보고, 피 문턱(10장)에 "곧 확실히
      채울 수 있는" 상태(`easyPiScoring`)면 가중치를 0.15→0.3으로 크게. `easyPiScoring`은 이미 피
      9장 이상이거나, 손에 보너스 카드가 있거나(+2) 바닥에 내 손패와 같은 월의 피/쌍피가 있어서
      (단, 그 월 4장이 전부 설명되어 상대가 가로챌 수 없는 확정 상황만) 다음에 그대로 가져올 수 있는
      양(`nearCertainPiGain` — 바닥 카드 자체 값 + 그걸 잡는 데 쓸 손패 자체의 피 값까지 합산)을 더해
      10장을 채우는 경우.
    - 위와 별개로, **다음 내 차례에 상대가 가로챌 수 없는 상태로 피 문턱을 확실히 넘는 게 확정적**이면
      (`pN+nearCertainPiGain>=10`) upside에 **+0.7**을 추가로 얹는다 — 막연한 가능성이 아니라 사실상
      정해진 결과이므로 상대 위협이 크지 않은 한 고를 안 부를 이유가 없다는 논리. (2차 헤드리스 시뮬레이션
      튜닝: 세션 기반 비대칭 자기대국, N=1000·세션당 40라운드=총 4만 라운드 페어 비교, 0.5→0.7 z=3.33로
      유의미하게 채택. `goCountUpside`=0.25와 `easyPiScoring`의 0.3/0.15는 더 큰 값 쪽으로 개선 여지가
      있어 보였으나(z≈1.2~1.9) 95% 유의성 문턱을 넘지 못해 원래 값 유지 — 한 라운드 안에서 드물게만
      발동해 더 많은 표본이 필요할 수 있음)
    - 상대 중 광이 0장(광박 대상)이면 상대별 +0.08, 피가 1~5장(피박 대상)이면 상대별 +0.064.

## 렌더링 & 애니메이션

- `render()`(4214줄)가 유일한 DOM 갱신 함수 — 상태(`G`)를 바꾼 뒤 항상 이 함수를 호출해 화면을 동기화.
- 카드 이동은 `flyCardVisual`(3428줄)이 담당: 실제 DOM 카드는 그대로 두고 복제한 `.flyer-card`를
  화면에 띄워 좌표 애니메이션 후 제거. 좌표는 `rectAnchor`류 함수들이 실제 DOM 요소 위치를 읽어 계산
  (`floorAnchor`/`floorRevealAnchor`/`floorOverlapAnchor`/`handPlayFloorAnchor`/`panelAnchor`/
  `capturedPanelAnchor`/`capturedPiAnchor`/`capturedCardAnchor`/`handCardAnchor`, 3323~3427줄).
  - `handPlayFloorAnchor(card)`: 손패를 낼 때 바닥에서 겹쳐 보일 위치. 같은 월 바닥 카드가 여러 장(2장
    중 하나를 나중에 고르는 경우, 3장이 뻑으로 묶이는 경우)이면 그중 어느 한 장과 겹치면 "이미 짝지어졌다"는
    오해를 주므로, 해당 카드들의 **정중앙(좌표 평균)** 위치로 겹치게 한다. 매칭이 아예 없어 그냥 버려지는
    패라면 `floorRevealAnchor()`(덱 카드를 뒤집을 때 쓰는 바닥 중앙 하단 자리)와 동일한 위치를 쓴다.
- `safeTimeout(fn, delay)`(2578줄)와 `gameSession` 카운터: "나가기"나 "새 게임"으로 세션이 바뀐 뒤에도
  이전 라운드의 `setTimeout` 체인이 뒤늦게 실행되며 새 게임 상태를 건드리는 사고를 막는 안전장치.
  **턴 진행 중 지연 실행되는 코드를 추가할 때는 반드시 `setTimeout` 대신 `safeTimeout`을 쓸 것.** 단,
  `safeTimeout`으로 감싼 콜백 내부에서 그 자체로 상태를 관리하는 공유 큐(예: 토스트 큐의 `busy` 플래그)가
  있다면, 세션이 바뀌는 순간 콜백 전체가 조용히 스킵되면서 그 플래그가 영원히 풀리지 않을 수 있다 —
  `runToastQueue`(2967줄)는 이 문제를 피하려고 큐 자체의 정리(`hideFn`/`busy=false`)는 일반 `setTimeout`
  으로, 호출자에게 알리는 `onDone`만 `gameSession` 체크로 감싼다.
- 토스트 4종(`showGoToast`/`showPiBonusToast`/`showDealToast`/`showChongtongToast`, 2986~3080줄)은 모두
  `queueToast()`(2963줄)로 직렬화되어 동시에 겹쳐 뜨지 않는다.
- 자동 진행(AUTO_PLAY) 두 종류: 평소 턴(`armAutoPlayTimer`/`performAutoPlayAction`)과 오버레이 팝업
  (`armOverlayAutoTimer`, 2363줄) — 둘 다 설정 화면의 "자동 진행 시간"(0=끄기)을 따름.

## UI 설정 (localStorage에 저장)

- 테마: `uiSetTheme` → `gostop-theme` (`navy`/`green`/`wine`/`charcoal`, CSS `[data-theme]`)
- 카드 라벨 표시: `uiSetLabelPref` → `gostop-show-labels` (`html[data-hide-labels]`)
- 자동 진행 시간: `uiSetAutoPlayDelay` → `gostop-autoplay-delay`
- 코인/이름은 저장 안 되고 시작 화면 입력값을 그대로 씀. 사람이 입력한 이름은 `escapeHtml()`로
  이스케이프해 `PLAYER_NAMES[0]`에 저장(로그 등 innerHTML 삽입 위험 방지), 원본은 `PLAYER_RAW_NAME`.

## 반응형 레이아웃 (CSS)

- 기본(모바일): 세로 1단 배치.
- `min-width:1280px and min-height:640px`: `body{zoom:1.5}` + 좌우 상대 패널·바닥·내 패를 가로 3단
  배치로 재배열(PC용).
- `orientation:landscape and max-height:560px`: 확대 없이 가로 3단 배치(휴대폰 가로모드용).

## 향후 수정 시 참고할 점

- **로직 추가/수정은 대부분 1671~2357줄(게임 로직+AI 캡처/버릴패 선택), 2397~2561줄(고/스톱 판단),
  또는 3232~4213줄(턴 진행/정산)에 집중**된다. 523~1615줄(카드 그림)과 CSS는 룰과 무관하므로 시각적
  요청이 아니면 건드릴 필요 없음.
- 새 규칙을 추가하면 `computeScore`/`scoringCardIds`(점수·강조 표시), `finalizeRound`/
  `endRoundByExhaustion`(정산), 그리고 AI의 `cardWeight`/`aiChooseHandCard`/`assessGoStopFactors` 네
  곳에 **일관되게** 반영해야 사람전/AI전에서 같은 규칙이 적용된다.
- 상태 변경 후에는 항상 `render()` 호출, 지연 실행은 `safeTimeout` 사용, 사용자 입력 문자열은
  `escapeHtml()`을 거쳐야 하는 곳인지 확인(현재는 이름 입력 하나뿐).
- 규칙 문서 자체가 `.rule-box`(299줄부터)에 한국어로 이미 상세히 들어있어, 규칙 문구를 바꿀 때는 이
  HTML과 실제 로직 두 곳을 함께 수정해야 함(자동 동기화 없음).
- AI 가중치를 통계적으로 튜닝할 때의 방법론(비대칭 자기대국, 평균 점수/코인을 기본 지표로, 파산 임박
  시엔 상대적 코인 우열로 판단)은 `CLAUDE.md`에 별도로 정리되어 있으니 함께 참고할 것.
- **알려진 한계**: `aiChooseHandCard`가 손패의 같은 월 카드가 정확히 3장이고 바닥에 그 월이 1장 있는
  상태로 직접 호출되면(예: 테스트에서 `checkBombCandidates`를 거치지 않고 바로 호출) 3장 중 무엇을
  낼지를 놓고 값비싼 카드(광 등)를 바로 써버리는 경우가 있다. 하지만 실제 게임에서는 AI/사람 턴 모두
  `aiChooseHandCard` 호출 **이전에** 항상 폭탄 후보(`checkBombCandidates`)를 먼저 확인해서 그 월을
  `excludeMonths`로 제외하거나 폭탄으로 처리하므로, 이 경로는 실제 플레이에서는 도달하지 않는다.
