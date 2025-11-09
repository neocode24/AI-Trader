# 🚀 AI-Trader 빠른 시작 가이드

## ✅ 완료된 설정
- [x] Python 패키지 설치 완료
- [x] .env 파일 생성 완료
- [x] 주식 데이터 파일 존재 (미국 주식 2.5MB)
- [x] 초보자용 테스트 설정 생성

## ⚠️ 아직 필요한 것
- [ ] API 키 입력 (.env 파일 수정 필요)

---

## 📋 단계별 실행 가이드

### 1단계: API 키 설정
```bash
# .env 파일 편집
nano /home/user/AI-Trader/.env

# 또는
vim /home/user/AI-Trader/.env
```

**아래 항목을 실제 API 키로 변경하세요:**
```
OPENAI_API_KEY=your_openai_api_key_here        # sk-로 시작하는 키
ALPHAADVANTAGE_API_KEY=your_alpha_vantage_key_here
JINA_API_KEY=your_jina_api_key_here
```

### 2단계: MCP 서비스 시작
```bash
cd /home/user/AI-Trader/agent_tools
python start_mcp_services.py &
```

### 3단계: AI 트레이딩 실행 (초보자 테스트)
```bash
cd /home/user/AI-Trader
python main.py configs/beginner_test_config.json
```

---

## 🎯 테스트 설정 상세

**파일:** `configs/beginner_test_config.json`

- **시장:** 미국 NASDAQ 100
- **기간:** 2025-10-01 ~ 2025-10-05 (5일간)
- **초기 자금:** $10,000
- **AI 모델:** GPT-4o-mini (가장 저렴한 모델)
- **예상 비용:** 약 $0.50 ~ $2.00 (테스트 기간 동안)

---

## 📊 결과 확인

### 거래 기록 위치
```bash
# 포지션 기록
cat data/agent_data/gpt-4o-mini/position/position.jsonl

# 상세 로그
ls -la data/agent_data/gpt-4o-mini/log/
```

### 성과 분석
```bash
bash calc_perf.sh
```

---

## 🔧 문제 해결

### API 키 오류
```
Error: Invalid API key
```
→ .env 파일의 API 키를 다시 확인하세요.

### 데이터 없음 오류
```
Error: No price data found
```
→ 아래 명령어로 데이터를 다시 다운로드:
```bash
cd data
python get_daily_price.py
python merge_jsonl.py
```

### MCP 서비스 포트 충돌
```
Error: Port 8000 already in use
```
→ .env 파일에서 포트 번호 변경

---

## 💡 다음 단계

테스트가 성공하면:
1. 기간을 늘려서 재실행 (예: 1개월)
2. 다른 AI 모델 테스트 (Claude, GPT-4 등)
3. 투자 전략 커스터마이징
4. 중국 A주 시장 테스트

---

## 📞 도움이 필요하면

1. README.md 참고
2. GitHub Issues: https://github.com/HKUDS/AI-Trader/issues
3. 저에게 질문하기!
