---
layout: single
title:  "AI agent 직접 만들어보기: 비트코인 자동 매매 시스템"
excerpt: "GPT와 업비트 API를 활용하여 만든 자율 매매 시스템 개발기"
categories: study
tag: [AI, AI agent, Artificial Intelligence, Bitcoin, investment]
typora-root-url: ../
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
---

<style>
  .toc__menu li > ul {
    display: none !important;
  }
</style>

## 개요
단순한 알고리즘 매매를 넘어, 시장의 '맥락'을 이해하고 스스로 성장하는 AI 트레이더를 만들 수 있을까?
이번 글은 **GPT-4o**를 활용하여 차트뿐만 아니라 뉴스, 거시경제 지표까지 종합적으로 분석하는 **비트코인 자동매매 시스템**을 개발한 기록입니다.

단순히 매매만 반복하는 것이 아니라, 매매 결과를 **데이터베이스에 기록하고 스스로 반성**하여 다음 매매에 반영하는 '자율 진화형' 시스템을 목표로 했습니다.

---

## 전체 시스템 구조
이 시스템은 크게 **데이터 수집 → AI 투자 판단 → 실제 매매 → DB 기록 및 반성**의 4단계 순환 구조로 이루어져 있습니다.

1.  **데이터 수집:** 차트, 호가, 뉴스, 거시경제 지표 수집
2.  **투자 판단:** 수집된 데이터를 GPT-4o에 전송하여 매수/매도 결정
3.  **주문 실행:** 업비트 API를 통해 실제 거래 체결
4.  **매매 기록 및 피드백:** 매매 결과를 DB에 기록하고, 그 DB를 다시 데이터로 사용함으로써 AI가 스스로 복기 수행

![image-20260125163124639](/images/2026-01-25-aibitcoin/image-20260125163124639.png){: .align-center style="width: 50%;"}

---

## 1. 데이터 수집 (Inputs)
GPT에게 투자 판단을 맡기기 전에 먼저 GPT에게 넘겨줄 데이터를 수집해야하는데요. 저는 **차트데이터와 호가데이터, 시장심리, 거시경제** 관련 자료들을 가져왔습니다.

* **기술적 분석**   
`pyupbit` 라이브러리를 사용해 업비트에서 실시간 비트코인의 차트데이터와 호가데이터를 가져왔습니다. 보조지표를 계산하기 위해 차트 데이터는 30일봉, 24시간봉을 가져오지만, 차트데이터와 호가데이터는 토큰을 아끼기 위해서 각각 최근 5개의 값(5일봉, 5시간봉)만 GPT에게 넘겨주었습니다.   
`ta` 라이브러리를 사용해 보조지표(RSI, MACD, 볼린저밴드, 이동평균선)를 계산합니다.   
<br>
[pyupbit 라이브러리 사용법](https://github.com/sharebook-kr/pyupbit>)   
[ta 라이브러리 사용법](https://github.com/bukosabino/ta)

* **시장 심리**   
공포 탐욕 지수(Fear & Greed Index)와 Cryptopanic의 뉴스 데이터를 활용합니다.  
뉴스는 중요한 최신 5개의 뉴스만 필터링해서 가져왔습니다.   
<br>
[공포 탐욕 지수](https://alternative.me/crypto/fear-and-greed-index/)   
[Cryptopanic API 사용법](https://cryptopanic.com/developers/api/)   

* **거시 경제**    
그 외에도, 환율과 글로벌 시세를 비교하여 **김치 프리미엄**을 계산하고, **나스닥(QQQ)** 지수도 참고합니다.   
<br>
이렇게 수집된 데이터들은 JSON 형식으로 깔끔히 정리되어 GPT에게 넘겨집니다.   

---

## 2. AI 의사결정 엔진: GPT-4o
이 프로젝트의 핵심인 '투자자의 뇌' 역할로 **GPT-4o**를 채택했습니다. 제한된 API 토큰 비용 내에서 복잡한 시장 데이터를 가장 정확하게 해석하고, JSON 포맷을 완벽하게 준수하기 때문입니다.   
(사실은 가장 최신버전의 GPT-5.2를 사용할 수도 있었지만, $5의 토큰으로 시스템을 최대한 많이 돌려보고 싶어서 GPT-4o를 사용했습니다...)

### 프롬프트 엔지니어링 (System Prompt)
AI에게 단순히 **"사/팔아/보류해"**가 아니라 **근거(Reasoning), 확신도(Confidence), 비중(Percentage)**도 같이 산출하도록 지시했습니다. 특히 다음과 같은 **리스크 관리 로직**을 시스템 프롬프트에 강제로 주입했습니다.

> "김치 프리미엄이 5% 이상이면 매수를 주의하라."   
> "확신이 높아도 몰빵(100%) 투자는 금지한다."

```python
# 시스템 프롬프트 예시
system_prompt = """
You are an expert Bitcoin trader...
3. Macro & Kimchi Premium (Critical):
   - Kimchi Premium: If > 5%, Korean price is overheated (Risky to buy).
...
"""
```

---

## 3. 주문 실행 및 안전장치

AI가 아무리 좋은 판단을 내려도, 실제로 매매를 수행하는 '손과 발'이 부실하면 무용지물입니다. 이 시스템의 **주문 실행 모듈**은 AI의 판단을 구체적인 API 명령으로 변환하고, 그 과정에서 발생할 수 있는 사고를 방지합니다.

### 3-1. 안전한 주문 로직

**[pyupbit 라이브러리](https://github.com/sharebook-kr/pyupbit)**를 사용하여 업비트 API와 통신하며, 다음과 같은  안전장치를 거칩니다.

1.  **최소 주문 금액 필터링:** 업비트 정책상 건당 5,000원 미만 주문 시 발생하는 API 에러를 사전에 차단합니다.
2.  **비중(Percentage) 기반 분할 매매:** AI가 제안한 비중만큼만 자산을 투입하여 리스크를 분산합니다.
3.  **수수료:** 비트코인을 매수할 때 수수료가 발생하므로, 수수료를 제외한 금액만 매수합니다.

```python
def execute_trade(result):
    # ... (생략) ...
    if decision == "buy":
        my_krw = upbit.get_balance("KRW")
        # 수수료(0.05%)를 고려하여 0.9995를 곱함
        buy_amount = my_krw * (percentage / 100) * 0.9995
        
        # 최소 주문 금액(5000원) 체크
        if my_krw and buy_amount > 5000:
            print(f"Executing Buy: {int(buy_amount)} KRW")
            order_res = upbit.buy_market_order("KRW-BTC", buy_amount)
        else:
            print("Buy Failed: 금액이 5000원 미만입니다.")
```

---

## 4. 매매 기록 저장 및 자기 반성

 단순히 정해진 규칙대로만 움직이는 기계가 아니라, **경험을 통해 학습하고 스스로 진화하는 에이전트**를 구현했습니다.

### 4.1. DB 구축
이 프로젝트는 별도의 무거운 서버(MySQL 등) 설치가 필요 없는 **SQLite**를 사용하여 거래 기록을 관리합니다. Python 내장 라이브러리인 `sqlite3`를 사용하므로 설정이 간편하고 파일 하나(`bitcoin_trades.db`)로 모든 데이터를 관리할 수 있어 이식성이 뛰어납니다.

**테이블 생성 코드**

```python
import sqlite3

def init_db():
    # DB 파일에 연결 (없으면 자동 생성)
    conn = sqlite3.connect('bitcoin_trades.db')
    c = conn.cursor()
    
    # DB 스키마(Schema)
    c.execute('''
        CREATE TABLE IF NOT EXISTS trades (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp TEXT,          -- 거래 시간
            decision TEXT,           -- 매수/매도/홀드
            percentage INTEGER,      -- 비중
            reason TEXT,             -- GPT의 판단 근거
            btc_balance REAL,        -- 비트코인 잔고
            krw_balance REAL,        -- 원화 잔고
            result TEXT,             -- 결과 (성공/실패)
        )
    ''')
    conn.commit()
    conn.close()
```

### 4-2. reflection 칼럼 생성
사람이 주식 투자에 실패하면 오답 노트를 쓰듯이, 이 AI 에이전트도 매매가 끝난 후 자신의 행동을 복기합니다.

매매를 수행하고 나서 일정 시간이 지난 후, 진입 당시 가격과 현재 가격을 비교하여 수익/손실 여부를 판단합니다. GPT-4o에게 당시 상황과 결과를 입력하고 **"반성 일기(Reflection)"**를 작성하게 합니다. 그리고 이 작성된 반성 내용을 **다음번 매매의 시스템 프롬프트에 동적으로 주입**하여 동일한 실수를 방지합니다.

이를 구현하기 위해 DB에 `reflection` 칼럼을 추가하여 AI의 '반성일기'을 저장할 공간을 마련했습니다.    

| id | timestamp | decision | result | ... | **reflection (New!)** |
|:---:|:---:|:---:|:---:|:---:|:---|
| 101 | 2026-01-24 | buy | **Loss** | ... | "RSI가 75로 과매수 구간이었는데 뉴스 호재만 믿고 진입한 것이 패착이었다. 다음엔 기술적 지표를 더 신뢰하자." |

<br>
아래 표는 시스템 가동 초기 실제 DB 기록입니다. 1~3, 5번 로그에서는 '잔고 부족(Insufficient Balance)' 예외 처리 로직이 정상 작동하여 매매가 차단되었으며, 4번째 시도에서 조건이 충족되어 첫 매수가 정상 체결된 것을 확인할 수 있습니다.
<iframe 
  src="https://docs.google.com/spreadsheets/d/e/2PACX-1vTYuTYT1ks8V6ckBPbYmg1Z_n7y79YWq01gljuJ_NJ8KLxQr_MNgJYNubjn813V-I6VAV6cFV3jw48s/pubhtml?gid=1226098300&amp;single=true&amp;widget=true&amp;headers=false"
  width="100%" 
  height="265"
  frameborder="0">
</iframe>

---

## 5. 향후 발전 계획

현재 버전은 '기능 구현'에 초점을 맞춘 프로토타입입니다. 앞으로 다음과 같은 기능을 추가하여 시스템 안정성을 높여보고 싶은 생각이 있습니다.

1.  **클라우드 서버 이관:** 현재는 개인 PC에서 실행 중이라 24시간 가동이 어려우므로 클라우드 환경으로 서버를 옮겨 중단 없는 시스템을 구축하기
2.  **모바일 알림 연동:** 매매가 체결되거나 에러가 발생했을 때, 휴대폰 어플로 즉시 알림을 받는 기능을 추가하기


