# 모터 제조 공정에서 L값(Inductance) 관리 방법과 영향 인자

---

# 1. 개요

인덕턴스(Inductance)는 모터의 전기적 및 자기적 특성을 나타내는 핵심 파라미터이다.

PMSM / BLDC Sensorless 제어에서 다음 항목의 성능에 직접 영향을 준다.

- Current PI 제어기
- Observer
- PLL
- 속도 추정
- FOC 안정성
- 기동 성능
- 탈조(Stall) 특성

특히 Sensorless FOC에서는

- Rs
- Ld
- Lq
- Ke

4개의 파라미터가 매우 중요하다.

---

# 2. L값이란?

## 일반적 정의

인덕턴스는 전류 변화에 저항하는 특성이다.

수식:

V = L × dI/dt

전류를 빠르게 변화시키려면 더 높은 전압이 필요하다.

---

## PMSM의 Ld, Lq

PMSM은 d축과 q축의 인덕턴스가 존재한다.

### d축 인덕턴스

Ld

회전자 자석 방향 축

---

### q축 인덕턴스

Lq

토크 생성 축

---

SPM 모터

```text
Ld ≈ Lq
```

IPM 모터

```text
Ld < Lq
```

---

# 3. 제조 공정에서 L값 관리 방법

---

## Step 1. 설계 목표 설정

예)

```text
Ld = 3.2mH ±10%

Lq = 3.4mH ±10%
```

또는

```text
Ld/Lq Ratio

0.95 ~ 1.05
```

관리 기준 설정

---

## Step 2. 권선 완료 후 측정

일반적으로

- LCR Meter
- Impedance Analyzer

사용

---

### 측정 예

```text
U-V = 3.25mH

V-W = 3.19mH

W-U = 3.27mH
```

---

### 판정 기준

상간 편차

```text
±3% 이내
```

또는

```text
±5% 이내
```

---

## Step 3. EOL(End Of Line) 검사

실제 인버터를 사용

다음 파라미터 측정

```text
Rs

Ld

Lq

Ke
```

---

### 장점

실제 운전 조건에 가장 가깝다.

Sensorless 제어기에 사용할 실제 값을 확보 가능

---

## Step 4. EEPROM 등록

측정값 저장

```text
Rs

Ld

Lq

Ke

Pole Pair
```

↓

Observer

↓

PLL

↓

Current PI

↓

FOC

에서 사용

---

# 4. L값을 결정하는 요소

영향도가 큰 순서

---

# 4.1 권선수(Turn 수)

가장 영향이 크다.

인덕턴스는

```text
L ∝ N²
```

---

예)

```text
100 Turn

↓

110 Turn
```

인 경우

```text
L 증가 약 21%
```

---

대표 불량

```text
권선 누락

권선 단락

권선 오배치
```

---

# 4.2 Air Gap

매우 중요

---

Air Gap 증가

```text
자기저항 증가

↓

자속 감소

↓

L 감소
```

---

원인

```text
축 편심

베어링 조립 오차

하우징 공차
```

---

# 4.3 철심 적층(Stack)

Stack Length 증가

```text
자속 증가

↓

L 증가
```

---

예)

```text
20mm

↓

22mm
```

---

# 4.4 Rotor 구조

SPM

```text
Ld ≈ Lq
```

---

IPM

```text
Ld < Lq
```

---

Spoke Type

```text
Ld << Lq
```

가능

---

# 4.5 자석 특성

영향 요소

```text
Br

Hc

자석 위치

착자 상태
```

---

결과

```text
Ld 변화

Lq 변화

Ld/Lq 변화
```

---

# 4.6 측정 전류

중요

Ld와 Lq는 일정한 값이 아니다.

---

전류 증가

↓

철심 포화

↓

Ld 감소

↓

Lq 감소

---

따라서

```text
무부하 측정값

운전 중 측정값
```

은 다를 수 있다.

---

# 5. Sensorless 제어기에서 L값이 중요한 이유

Observer는

```text
Ld × dId/dt

Lq × dIq/dt
```

를 사용한다.

---

L이 틀리면

```text
EEMF 추정 오차

↓

Error_VDS 증가

↓

Error_VQS 증가

↓

theta_err 증가

↓

PLL 오차 증가

↓

탈조 가능
```

---

# 6. 실제 불량 사례

---

## Case 1

실제

```text
Ld = 2.5mH
```

---

EEPROM

```text
Ld = 3.5mH
```

---

결과

```text
Observer 계산 오차

↓

각도 추정 오차

↓

토크 리플

↓

탈조
```

---

## Case 2

실제

```text
Ld = 2.5mH

Lq = 4.0mH
```

---

제어기

```text
Ld = Lq
```

가정

---

결과

```text
Decoupling 오차

↓

Current PI 성능 저하

↓

고속 안정성 저하
```

---

# 7. 생산라인 관리 항목

우선순위

---

## 1순위

권선수

```text
Turn 수
```

---

## 2순위

상간 편차

```text
U-V

V-W

W-U
```

---

## 3순위

Air Gap

---

## 4순위

Rs

---

## 5순위

Ke

---

## 6순위

Ld

---

## 7순위

Lq

---

## 8순위

Ld/Lq Ratio

---

# 8. Sensorless BLDC 프로젝트 관점 정리

Observer와 PLL은

```text
Rs

Ld

Lq

Ke
```

를 기반으로 회전자 위치를 추정한다.

따라서 생산 공정에서

```text
권선

Air Gap

철심

자석

Rs

Ld

Lq

Ke
```

의 편차가 커질수록 추정 오차가 증가한다.

결국

```text
theta_err 증가

↓

PLL Gain 증가

↓

Iq 증가

↓

추종 실패

↓

탈조
```

로 연결될 수 있다.

---

# 핵심 요약

제조 관점

```text
Turn 수
Air Gap
Stack
Rotor 구조
자석 특성
```

이 L값을 결정한다.

---

제어 관점

```text
L값 오차

↓

Observer 오차

↓

PLL 오차

↓

FOC 성능 저하

↓

탈조 가능성 증가
```

Sensorless 제어에서는

```text
Rs
Ld
Lq
Ke
```

관리 품질이 곧 제어 성능이다.
