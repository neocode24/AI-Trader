# 🧠 AI-Trader 학습 구조 분석

## 핵심 결론
**❌ 재귀적 자동 학습 아님**
**✅ 순차적 의사결정 (과거 포지션 참고)**

---

## 📊 실제 작동 방식

### 1. 백테스팅 흐름 (5일 예시)

```
Day 1 (2025-10-01):
├─ AI 입력:
│  ├─ 초기 자금: $10,000
│  ├─ 포지션: CASH = $10,000
│  └─ 오늘 주식 가격: AAPL=$255, MSFT=$420...
├─ AI 사고:
│  ├─ 뉴스 검색: "AAPL earnings good"
│  └─ 판단: AAPL 매수
└─ AI 결정:
   └─ 매수: AAPL 30주 ($7,650)
   └─ 결과: AAPL=30주, CASH=$2,350

Day 2 (2025-10-02):
├─ AI 입력:
│  ├─ 어제 포지션: AAPL=30주, CASH=$2,350  ← 과거 결과 참고
│  ├─ 어제 종가: AAPL=$262 (평가액 $7,860)
│  └─ 오늘 주식 가격: AAPL=$265, MSFT=$425...
├─ AI 사고:
│  ├─ "어제 AAPL 샀더니 +$210 수익"
│  └─ 판단: AAPL 보유, MSFT 추가 매수
└─ AI 결정:
   └─ 매수: MSFT 5주 ($2,125)
   └─ 결과: AAPL=30주, MSFT=5주, CASH=$225

Day 3, 4, 5...
(계속 반복)
```

---

## 🔄 과거 정보 활용 방식

### AI가 매일 받는 정보 (prompts/agent_prompt.py:25-59)

```python
agent_system_prompt = """
Current time: {date}                          # 1. 오늘 날짜

Your current positions:                        # 2. 어제 종료 시점 포지션
{positions}                                    #    예: AAPL=30, CASH=$2,350

The current value of your stocks:             # 3. 어제 종가 기준 평가액
{yesterday_close_price}                        #    예: AAPL=$262 → $7,860

Current buying prices:                         # 4. 오늘 시가 (매수 가능 가격)
{today_buy_price}                              #    예: AAPL=$265
"""
```

### 예시 프롬프트 (실제)

```
Current time: 2025-10-02

Your current positions:
AAPL: 30 shares
CASH: $2,350

The current value of your stocks:
AAPL: $262 (total: $7,860)

Current buying prices:
AAPL: $265
MSFT: $425
NVDA: $820
...
```

AI는 이 정보를 보고:
- "어제 AAPL을 30주 샀구나"
- "어제 $255에 샀는데 지금 $265네? +$10/주 수익!"
- "계속 보유할까, 팔까, 다른 주식 살까?"

---

## ❌ 없는 것들 (재귀적 학습이 아닌 이유)

### 1. 모델 파라미터 업데이트 없음
```
일반 머신러닝:
Day 1 → 학습 → 가중치 업데이트 → Day 2 → 학습 → 가중치 업데이트...

AI-Trader:
Day 1 → 결정 (모델 그대로) → Day 2 → 결정 (모델 그대로)...
```

**GPT-4o-mini는 매일 동일한 모델 사용 (학습 안 함)**

### 2. 과거 거래 이유 기억 안 함
```
Day 1: "AAPL 매수 (이유: 실적 좋음)"
Day 2: AI는 "왜 샀는지" 모름, 단지 "30주 보유 중" 만 앎
```

### 3. 장기 전략 기억 없음
```
AI는 매일 새로운 대화 세션:
- Day 1: 새 대화 시작
- Day 2: 새 대화 시작 (Day 1 대화 내용 모름)
- Day 3: 새 대화 시작 (과거 대화 전부 모름)
```

---

## ✅ 있는 것들

### 1. 포지션 정보 (tool_trade.py가 기록)
```python
# data/agent_data/gpt-4o-mini/position/position.jsonl
{
  "date": "2025-10-01",
  "positions": {"AAPL": 30, "CASH": 2350}
}
```

### 2. 가격 정보 (Alpha Vantage 데이터)
```python
# data/merged.jsonl
{
  "AAPL": {
    "2025-10-01": {"open": 255, "close": 262},
    "2025-10-02": {"open": 265, "close": 270}
  }
}
```

### 3. 매일 독립적 추론
```
AI는 매일:
1. 현재 포지션 확인
2. 오늘 가격 확인
3. 뉴스 검색
4. "오늘만" 최선의 결정 내림
```

---

## 🎯 실제 AI 의사결정 프로세스

### Day 2 시나리오 (코드 추적)

#### Step 1: 시스템 프롬프트 생성
```python
# prompts/agent_prompt.py:62-88
def get_agent_system_prompt(today_date="2025-10-02", signature="gpt-4o-mini"):
    # 1. 어제 포지션 읽기
    today_init_position = get_today_init_position(today_date, signature)
    # → {"AAPL": 30, "CASH": 2350}

    # 2. 어제 종가 읽기
    yesterday_sell_prices = get_yesterday_open_and_close_price(today_date)
    # → {"AAPL": 262}

    # 3. 오늘 시가 읽기
    today_buy_price = get_open_prices(today_date)
    # → {"AAPL": 265, "MSFT": 425, ...}
```

#### Step 2: AI 추론 (agent/base_agent/base_agent.py:400)
```python
async def run_trading_session(today_date: str):
    # AI에게 위 정보 전달
    prompt = get_agent_system_prompt(today_date)

    # AI 사고 시작
    response = await ai_agent.invoke(prompt)
    # AI: "음... AAPL 수익 났네? 보유할까?"
    # AI: "뉴스 검색해보자"
    # AI: call_tool("get_information", "AAPL news")
    # AI: "긍정적이네, 계속 보유하고 MSFT도 사자"
    # AI: call_tool("buy", {"symbol": "MSFT", "amount": 5})
```

#### Step 3: 포지션 기록
```python
# agent_tools/tool_trade.py
def buy(symbol: str, amount: int):
    # 거래 실행
    new_position = {"AAPL": 30, "MSFT": 5, "CASH": 225}

    # 파일에 저장
    with open("position.jsonl", "a") as f:
        f.write(json.dumps(new_position))
```

---

## 🔁 루프 구조 (전체 백테스팅)

```python
# main.py
for date in ["2025-10-01", "2025-10-02", "2025-10-03", ...]:
    # 매일 새로운 AI 세션 시작
    ai_agent = create_new_agent()  # 모델 리셋 (학습 없음)

    # 오늘 거래 실행
    await run_trading_session(date)

    # AI 메모리 삭제 (다음 날은 새 시작)
    del ai_agent
```

**핵심:** 매일 AI는 "새로 태어남" → 과거 대화 기억 못함

---

## 📚 비유로 이해하기

### 1. 재귀적 학습 (없음)
```
❌ 학생이 시험 보고 → 오답 분석 → 공부 → 실력 향상 → 다음 시험
```

### 2. AI-Trader 방식 (실제)
```
✅ 매일 새로운 전문가 고용:
Day 1: 전문가 A "오늘 뭐 살까?" → AAPL 매수
Day 2: 전문가 B "어제 누가 AAPL 샀네? 나라면 어떻게 하지?" → 보유
Day 3: 전문가 C "AAPL 있네, 팔까?" → 판매

각 전문가는 동일한 지식(GPT-4o-mini) 가진 별개 인물
```

### 3. 포지션 = 메모
```
AI는 매일 초기화되지만, "메모(position.jsonl)"는 남음
다음 날 AI는 메모를 읽고 판단

마치:
- Day 1: "AAPL 샀음" (메모 남김)
- Day 2: (새 AI) "메모 읽어보니 AAPL 있네"
```

---

## 🆚 비교: 재귀적 학습 vs AI-Trader

| 항목 | 재귀적 학습 | AI-Trader |
|------|-----------|-----------|
| **모델 업데이트** | ✅ 매일 학습 | ❌ 모델 고정 |
| **과거 기억** | ✅ 전체 기억 | ❌ 포지션만 앎 |
| **전략 진화** | ✅ 점점 개선 | ❌ 일관된 추론 |
| **비용** | 높음 (학습 비용) | 낮음 (추론만) |
| **재현성** | 낮음 (계속 변함) | 높음 (동일 모델) |

---

## 💡 장단점

### 장점 (현재 방식)
1. **재현 가능:** 같은 데이터 → 같은 결과
2. **저렴함:** 학습 비용 없음
3. **안정적:** 모델이 이상하게 학습될 위험 없음

### 단점
1. **전략 개선 없음:** 손해 봐도 다음 날 동일한 실수 가능
2. **맥락 부족:** "왜 샀는지" 기억 못함
3. **장기 전략 불가:** 매일 단기 결정만

---

## 🚀 개선 가능성

만약 학습을 추가하고 싶다면:

### 방법 1: 프롬프트에 과거 로그 포함
```python
prompt = f"""
과거 거래 내역:
Day 1: AAPL 매수 (이유: 실적 좋음) → +$210
Day 2: MSFT 매수 (이유: 클라우드 성장) → -$50

오늘은? (과거 성공/실패 참고해서 결정)
"""
```

### 방법 2: RAG (Retrieval-Augmented Generation)
```
AI 결정 전:
1. 과거 성공 사례 검색
2. 유사 상황 찾기
3. 참고해서 결정
```

### 방법 3: Fine-tuning (진짜 학습)
```
90일 백테스팅 결과로 모델 재학습
→ 새로운 "투자 전문 GPT" 만들기
```

---

## ✅ 결론

**AI-Trader는:**
- ❌ 재귀적 자동 학습 아님
- ✅ 매일 독립적 의사결정
- ✅ 과거 포지션만 참고
- ✅ 동일한 AI 모델 반복 사용

**비유:** "매일 같은 투자 전문가에게 컨설팅 받는 것"
- 전문가는 어제 대화 기억 못함
- 단, "현재 내 주식 목록" 보고 조언
- 전문가 실력은 변하지 않음 (GPT-4o-mini 그대로)

---

**따라서 "백테스팅"은 "AI가 과거 데이터로 어떻게 투자했을지 시뮬레이션"일 뿐, "학습"이 아닙니다!**
