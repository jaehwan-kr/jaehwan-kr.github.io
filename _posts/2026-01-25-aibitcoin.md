---
layout: single
title:  "AI agent 직접 만들어보기: 비트코인 자동 매매 시스템"
categories: study
tag: [AI, AI agent, Artificial Intelligence, Bitcoin, investment]
typora-root-url: ../
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
---

## 🚀 프로젝트 개요
단순한 알고리즘 매매를 넘어, 시장의 '맥락'을 이해하고 스스로 성장하는 AI 트레이더를 만들 수 있을까?
이번 프로젝트는 **GPT-4o**를 활용하여 차트뿐만 아니라 뉴스, 거시경제 지표까지 종합적으로 분석하는 **비트코인 자동매매 시스템**을 개발한 기록입니다.

단순히 매매만 반복하는 것이 아니라, 매매 결과를 **데이터베이스(DB)에 기록하고 스스로 반성(Reflection)**하여 다음 매매에 반영하는 '자율 진화형' 시스템을 목표로 했습니다.

---

## 🏗️ 전체 시스템 구조
이 시스템은 크게 **데이터 수집 → AI 투자 판단 → 실제 매매 → DB 기록 및 반성**의 4단계 순환 구조로 이루어져 있습니다.

1.  **데이터 수집:** 차트, 호가, 뉴스, 거시경제 지표 수집
2.  **투자 판단:** 수집된 데이터를 GPT-4o에 전송하여 매수/매도 결정
3.  **주문 실행:** 업비트 API를 통해 실제 거래 체결
4.  **매매 기록 및 피드백:** 매매 결과를 DB에 기록하고, 그 DB를 다시 데이터로 사용함으로써 AI가 스스로 복기 수행

![image-20260125163124639](/images/2026-01-25-aibitcoin/image-20260125163124639.png)

---

## 👁️ 1. 데이터 수집 (Inputs)
기존의 퀀트 봇들이 수치 데이터(가격, 거래량)만 봤다면, 이 에이전트는 **차트데이터와 호가데이터 거시경제**를 함께 봅니다.

* **기술적 분석**   
`pyupbit` 라이브러리를 사용해 업비트에서 실시간 비트코인의 차트데이터와 호가데이터를 가져왔습니다. 보조지표를 계산하기 위해 차트 데이터는 30일봉, 24시간봉을 가져오지만, 차트데이터와 호가데이터는 토큰을 아끼기 위해서 각각 최근 5개의 값(5일봉, 5시간봉)만 GPT에게 넘겨주었습니다.   
<br>
`ta` 라이브러리를 사용해 보조지표(RSI, MACD, 볼린저밴드, 이동평균선)를 계산합니다.   
<br>
pyupbit 라이브러리 사용법 : <https://github.com/sharebook-kr/pyupbit>   
ta 라이브러리 사용법 : <https://github.com/bukosabino/ta>

* **시장 심리**   
공포 탐욕 지수(Fear & Greed Index)와 Cryptopanic의 뉴스 투표 데이터를 활용합니다.
* **거시 경제 (Critical):** 환율과 글로벌 시세를 비교하여 **김치 프리미엄**을 계산하고, 비트코인과 커플링 되는 **나스닥(QQQ)** 지수도 참고합니다.

---

## 🧠 2. AI 의사결정 엔진: GPT-4o
이 프로젝트의 핵심인 '투자자의 뇌' 역할로 **GPT-4o**를 채택했습니다. 제한된 API 토큰 비용 내에서 복잡한 시장 데이터를 가장 정확하게 해석하고, JSON 포맷을 완벽하게 준수하기 때문입니다.

### 프롬프트 엔지니어링 (System Prompt)
AI에게 단순히 "사/팔아"가 아니라 **근거(Reasoning), 확신도(Confidence), 비중(Percentage)**을 산출하도록 지시했습니다. 특히 다음과 같은 **리스크 관리 로직**을 시스템 프롬프트에 강제로 주입했습니다.

> "김치 프리미엄이 5% 이상이면 매수를 주의하라."   
> "확신이 높아도 몰빵(100%) 투자는 금지한다."

```python
# 시스템 프롬프트 예시
system_prompt = """
You are an expert Bitcoin trader...
3. Macro & Kimchi Premium (Critical):
   - Kimchi Premium: If > 5%, Korean price is overheated (Risky to buy).
...
4. Decision Strategy (IMPORTANT):
   - Output 'confidence' (0-100)
   - Output 'percentage' (0-100)
"""
```