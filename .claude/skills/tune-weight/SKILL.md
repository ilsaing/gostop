---
name: tune-weight
description: 고스톱 게임 내 최적 가중치 조정을 위한 SKILL
disable-model-invocation: true
---

너는 유능한 코드 분석 전문가이자 통계 검증 전문가이다.
라운드별 평균획득점수를 높이고, 최종적으로 게임에 승리 확률을 높이는 목표 달성을 위해 AI판단로직 가중치 개선을 수행한다.
다음과 같은 단계로 문제를 해결한다.

### 작업 내용

- gostop.md 문서의 내용을 읽어 게임의 룰과 AI판단로직을 분석한다.
- 기존 코드 분석 : gostop.html 코드를 분석한다.
- 계획 수립 : 독립적인 서브에이전트를 활용해 기존 코드와 문서를 분석한 결과를 바탕으로 문제를 어떻게 해결해나갈지 계획을 수립한다.
- 환경 구축 : 수립된 계획을 바탕으로 통계적 검증을 위한 시뮬레이션 환경을 구축한다(아래 "실행 템플릿" 참고).
- 가중치 변경 : 손패를 내는 선택과 고/스톱 판단 선택을 위한 각종 가중치를 조정한다.
- 시뮬레이션 수행 : 3인 AI모드 시뮬레이션 수행을 하며, 2인은 기존 가중치로 1인은 변경 가중치로 시뮬레이션 한다.
- 가중치 검증 : 미리 작성한 시뮬레이션 환경을 이용해 충분한 횟수의 반복 수행을 통해 가중치의 통계적 유효성을 검증한다. N 스케일링·계층화·역방향 검증 원칙은 `CLAUDE.md`의 "검증 실행 방법론" 절을 따른다.
- 합리적 AI판단로직 개선을 위한 가중치 변경 시에는 목표 달성 가능성을 높이기 위해 상황에 따른 확률적 분석을 바탕으로 한다.
- 채택 여부와 무관하게 모든 시도(N/z/mean/결론)를 `gostop.md`에 "N차 튜닝"으로 기록한다.

## 실행 템플릿 (헤드리스 자기대국 하네스)

### 1. 하네스 준비

`gostop.html` 자체에는 라운드를 애니메이션 없이 끝까지 진행시키는 헤드리스 엔진이 없다.
대신 `gostop-mcs.html`에 이미 그 엔진(`continueRoundHeadless` 및 관련 함수들)이 있으므로,
매번 이걸 스크래치 디렉터리로 복사해 하네스로 쓴다(원본 `gostop-mcs.html`은 건드리지 않음):

```powershell
Copy-Item "D:\_vibe\gostop-mcs.html" "<scratchpad>\validate_xxx.html"
```

**중요**: `gostop-mcs.html`은 AI 로직 튜닝을 미러링하지 않으므로(관례상 AI 튜닝은
`gostop.html` 전용), 시간이 지날수록 두 파일의 `cardWeight`/`aiChooseHandCard`/
`assessGoStopFactors`/`opponentRiskIfLeft` 등이 벌어져 있다. **검증을 시작하기 전에 반드시
현재 `gostop.html`의 최신 내용을 읽어서, 복사한 하네스 파일의 해당 함수들을 그 내용으로
덮어써야 한다** — 안 그러면 대조군(A)이 실제 라이브 동작과 다른 걸 기준으로 잡혀 결과가
왜곡된다.

### 2. 토글 패턴

검증하려는 후보 로직을 전역 `let` 변수(기본값=기존 동작과 동일)로 빼고, 좌석 제한을 둬서
"한 좌석만 새 로직, 나머지 두 좌석은 기존 로직"인 비대칭 자기대국을 만든다(동일
알고리즘끼리 붙이면 항상 33%로 수렴해 무의미함):

```js
let USE_NEW_LOGIC = false;
let NEW_LOGIC_K = 1.0;
let NEW_LOGIC_SEAT = -1; // -1이면 전원 적용(대칭), 아니면 이 좌석의 판단에만 적용(비대칭 검증용)
// 함수 내부에서: const apply = USE_NEW_LOGIC && (NEW_LOGIC_SEAT<0 || playerIdx===NEW_LOGIC_SEAT);
```

### 3. Python/Playwright 검증 스크립트 골격

```python
from playwright.sync_api import sync_playwright
import pathlib

path = pathlib.Path("<scratchpad>/validate_xxx.html").resolve().as_uri()

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page(viewport={"width": 420, "height": 700})
    page.goto(path)
    page.wait_for_load_state("networkidle")
    page.click("#startBtn")
    page.wait_for_selector("#handRow", state="visible")

    result = page.evaluate("""([N, CANDIDATES]) => {
      // 결정적 PRNG — 딜(dealSeed)과 진행(playSeed)을 분리해서 같은 판을
      // 대조군(A)·실험군(B) 양쪽에 공통 난수로 재생시킨다(공통 난수법).
      function mulberry32(seed){
        return function(){
          seed |= 0; seed = seed + 0x6D2B79F5 | 0;
          let t = Math.imul(seed ^ seed >>> 15, 1 | seed);
          t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
          return ((t ^ t >>> 14) >>> 0) / 4294967296;
        };
      }
      function cloneCard2(c){ return Object.assign({}, c); }
      function makeInitialDeal(dealerIdx){
        const deck = shuffle(buildDeck());
        const hands = [[],[],[]];
        for(let i=0;i<7;i++) for(let p=0;p<3;p++) hands[p].push(deck.pop());
        const floor = [];
        for(let i=0;i<6;i++) floor.push(deck.pop());
        return {deck, hands, floor, dealerIdx};
      }
      function buildTestG(deal, coins){
        return {
          deck: deal.deck.map(cloneCard2), hands: deal.hands.map(h=>h.map(cloneCard2)),
          floor: deal.floor.map(cloneCard2), captured:[[],[],[]], dealer: deal.dealerIdx,
          goCount:[0,0,0], lastGoScore:[null,null,null], ppeokMakeCount:[0,0,0],
          shakeCount:[0,0,0], shakenMonths:[[],[],[]], shakenCards:[[],[],[]],
          coins: coins.slice(), bombSkip:[0,0,0], bonusPending: null, nagariMultiplier: 1,
          turn: deal.dealerIdx, log: [],
        };
      }

      const TEST_SEAT = 1;
      const savedG = G, savedAddLog = addLog, savedStatSeat = statSeatMcs;
      const originalRandom = Math.random;
      statSeatMcs = -1; // MCS 전부 끄고 순수 휴리스틱끼리만 비교

      function runVariant(deal, playSeed, use, k){
        Math.random = mulberry32(playSeed);
        const g = buildTestG(deal, [50,50,50]);
        G = g; addLog = function(){};
        USE_NEW_LOGIC = use; NEW_LOGIC_K = k; NEW_LOGIC_SEAT = TEST_SEAT;
        continueRoundHeadless(null);
        return g.coins[TEST_SEAT] - 50;
      }

      const results = {}; CANDIDATES.forEach(k=>results[k]=[]);
      for(let i=0;i<N;i++){
        const dealSeed = 1000003 + i*104729, playSeed = 9000003 + i*104729;
        Math.random = mulberry32(dealSeed);
        const deal = makeInitialDeal(i % 3);
        const deltaA = runVariant(deal, playSeed, false, 0);
        CANDIDATES.forEach(k=>{
          const deltaB = runVariant(deal, playSeed, true, k);
          results[k].push(deltaB - deltaA);
        });
      }
      Math.random = originalRandom;
      G = savedG; addLog = savedAddLog; statSeatMcs = savedStatSeat;

      const summary = {};
      CANDIDATES.forEach(k=>{
        const arr = results[k], n = arr.length;
        const mean = arr.reduce((a,b)=>a+b,0)/n;
        const variance = arr.reduce((a,b)=>a+(b-mean)*(b-mean),0)/(n-1);
        const stderr = Math.sqrt(variance/n);
        summary[k] = {n, mean, stderr, z: mean/stderr};
      });
      return summary;
    }""", [8000, [0.5, 1.0, 1.5]])
    print(result)
    browser.close()
```

`|z| > 1.96`이면 95% 유의. N/후보값 스케일링, 계층화, 역방향 검증 원칙은
`CLAUDE.md`의 "검증 실행 방법론" 절 참고.
