# 센서리스 모터 불량 검토 보고서 (Mermaid Edition)

> 권장 Viewer: VS Code Markdown Preview, Obsidian, GitHub, MkDocs
>
> 본 문서는 ASCII Tree 대신 Mermaid Diagram을 사용하여 Logic Tree가 깨지지 않도록 구성하였다.

---

# 1. 종합 Fault Logic Tree

```mermaid
flowchart TD

A[운전 중 전류 진동 및 진폭 증가]

A --> B{기계 원인 배제 완료}

B -->|아니오| C[기계계 재검증]
B -->|예| D[전기 제어 원인 분석]

D --> E[Motor Parameter]
D --> F[Observer PLL]
D --> G[센싱 인버터]

E --> E1[Rs Ld Lq Ke]
E --> E2[EEPROM Scale]
E --> E3[상간 L 편차]

F --> F1[EEMF 품질]
F --> F2[theta_err]
F --> F3[wo_hat wr_hat]

G --> G1[ADC Offset Gain]
G --> G2[PWM Sampling]
G --> G3[상전류 불균형]

E --> H[최초 이상시점 비교]
F --> H
G --> H

H --> I[theta_err 선행]
H --> J[IqeRef 선행]
H --> K[특정 상전류 선행]

I --> I1[Observer PLL 원인]
J --> J1[Speed PI 제어 포화]
K --> K1[센싱 인버터 권선]
```

---

# 2. Motor Parameter Root Cause Tree

```mermaid
flowchart TD

A[특정 모터에서만 발생]

A --> B{정상 모터 대비 차이 존재}

B -->|예| C[Rs 비교]
B -->|예| D[Ld 비교]
B -->|예| E[Lq 비교]
B -->|예| F[Ke 비교]

C --> G[EEPROM 비교]
D --> G
E --> G
F --> G

G --> H{Runtime 변환 일치}

H -->|아니오| I[Scale Q-format 오류]
H -->|예| J[Observer 입력 검증]
```

---

# 3. Observer Root Cause Tree

```mermaid
flowchart TD

A[Error_VDS VQS 이상]

A --> B{theta_err 이전 발생}

B -->|예| C[Observer 원인 우선]
B -->|아니오| D[PLL 원인 우선]

C --> E[전류센싱 검증]
C --> F[Vdc 추정 검증]
C --> G[Ld Lq Rs Ke 검증]

E --> H[EEMF 재계산]
F --> H
G --> H

H --> I[EEMF 방향 확인]
```

---

# 4. PLL Root Cause Tree

```mermaid
flowchart TD

A[theta_err 증가]

A --> B{wo_hat 진동}

B -->|예| C[Kp 과대]
B -->|아니오| D[적분 포화]

C --> E[Kp Sweep]
D --> F[Ki Sweep]

E --> G[PLL Lock 확인]
F --> G

G --> H{wr_hat 실제속도 일치}

H -->|아니오| I[PLL Lock 상실]
H -->|예| J[Observer 재검토]
```

---

# 5. Current Sensor Root Cause Tree

```mermaid
flowchart TD

A[상전류 왜곡]

A --> B{특정 상만 이상}

B -->|예| C[ADC Offset]
B -->|예| D[ADC Gain]
B -->|예| E[Sampling 위치]

B -->|아니오| F[공통 제어원인]

C --> G[Ia Ib Ic 합 확인]
D --> G
E --> G

G --> H{Ia+Ib+Ic=0}

H -->|아니오| I[센싱 불량]
H -->|예| J[dq 변환 검토]
```

---

# 6. Current PI Root Cause Tree

```mermaid
flowchart TD

A[B_Iqe 증가]

A --> B{B_IqeRef 증가 선행}

B -->|예| C[Speed PI 확인]
B -->|아니오| D[Current PI 확인]

C --> E[속도 Feedback 검증]
D --> F[dq 전류 오차 확인]

E --> G[전압 제한 진입]
F --> G

G --> H{limit_cond 활성}

H -->|예| I[Anti Windup 검토]
H -->|아니오| J[Gain 재조정]
```

---

# 7. 최종 원인 판정 Flow

```mermaid
flowchart TD

A[불량 재현]

A --> B[Error_VDS VQS]
B --> C[theta_err]
C --> D[wo_hat wr_hat]
D --> E[Id Iq]
E --> F[IqeRef]
F --> G[limit_cond]

G --> H{최초 이상신호}

H -->|EEMF| I[Observer]
H -->|theta_err| J[PLL]
H -->|IqeRef| K[Speed PI]
H -->|상전류| L[센싱 인버터]

I --> M[근본원인 확정]
J --> M
K --> M
L --> M
```

---

# 8. 추천 측정 변수

## Mode A : Observer

```text
Error_VDS_hat_sl
Error_VQS_hat_sl
IDS_mori
IQS_mori
```

## Mode B : PLL

```text
theta_err_mori
wo_hat_mori
wr_hat_mori
Error_sum_mori
```

## Mode C : Current Loop

```text
B_IdeRef
B_Ide
B_IqeRef
B_Iqe
```

## Mode D : Fault

```text
limit_cond
fdetect_pos_err
IF_Trans_fail
taljo_restart_cnt
```

---

# 핵심 결론

```text
기계원인 배제 상태

1순위
Motor Parameter 불일치
Rs Ld Lq Ke

2순위
Observer EEMF 계산 오류

3순위
PLL Lock 상실

4순위
센싱 Offset Gain Sampling

5순위
Current PI 및 Limit
```

최초 증가한 Error를 찾는 것이 핵심이다.
