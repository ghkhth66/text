# 센서리스 모터 불량 검토 보고서

## 1. 문서 목적

본 보고서는 정상 운전 파형과 불량 운전 시 3상 전류 발산 파형을 비교하여, 기계적 부하 증가, 임펠러 간섭, 베어링 이상, 축 편심 및 기계 공진이 없는 조건에서 센서리스 모터의 전기적·제어적 불량 원인을 체계적으로 분리하기 위한 검토 기준을 제시한다.

본 문서의 원인 순위는 현재 확보된 파형을 바탕으로 한 **검증 가설**이다. 최종 원인은 불량 재현 시 동일 시간축으로 취득한 Observer, PLL, 전류, 속도, 제한 및 고장 상태변수로 판정한다.

---

## 2. 현재 현상 요약

### 2.1 정상 운전 파형

정상 PLL Observer 파형에서는 다음 특징이 확인된다.

- I/F 기동 이후 FOC 전환이 이루어진다.
- `wr_hat_mori`가 초기 오버슈트 후 정상 운전 속도 부근으로 수렴한다.
- `wo_hat_mori`는 필터 전 PLL 출력이므로 리플이 존재하지만 평균 운전점은 유지된다.
- `theta_err_mori`는 0 electrical degree 부근을 중심으로 제한된 범위에서 진동한다.
- `Error_sum_mori`는 전환 초기에 증가한 후 일정한 수준으로 안정된다.
- 정지 명령 이후 PLL 및 Observer 관련 값이 0으로 복귀한다.

### 2.2 불량 운전 파형

불량 3상 전류 파형에서는 다음 특징이 관찰된다.

- 기동 직후 즉시 과전류가 발생하는 형태는 아니다.
- 초기에는 비교적 일정한 전류가 형성된다.
- 운전 중 3상 전류 진동이 발생한다.
- 진동 포락선이 시간에 따라 점진적으로 증가한다.
- 최종적으로 상전류가 큰 값까지 확대된 후 출력이 차단된다.

### 2.3 현상명 제안

현재 현상은 다음과 같이 정의하는 것이 적절하다.

> **운전 중 Observer/PLL 추종 불안정을 동반한 전류 발산 및 탈조 의심 현상**

단, `theta_err_mori`가 먼저 증가했는지, 속도오차 또는 전류지령이 먼저 증가했는지 확인하기 전에는 PLL을 근본원인으로 단정하지 않는다.

---

## 3. 제어 관점의 기본 원칙

모든 폐루프 제어의 기본 구조는 다음과 같다.

```text
Error = Target - Feedback
제어 목표: Error → 0
```

센서리스 FOC에서 주요 오차는 다음과 같다.

```text
전류오차
Id_error = Id_ref - Id
Iq_error = Iq_ref - Iq

속도오차
Speed_error = Speed_ref - Speed_feedback

PLL 위상오차
theta_err_mori → 0 electrical degree
```

불량 분석에서는 마지막에 발생한 Fault 플래그보다 **가장 먼저 증가한 Error**를 찾아야 한다.

```text
최초 Error 증가
→ 제어기 보정량 증가
→ 포화 또는 진동
→ 다른 제어루프 교란
→ 전류 발산
→ Fault 및 보호 정지
```

---

## 4. 종합 Logic Tree

```text
[운전 중 3상 전류 진동 및 진폭 증가]
                |
                v
[기계 원인 배제 완료?]
  |                         |
  | 아니오                  | 예
  v                         v
기계계 재검증          [전기·제어 원인 분기]
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
   [Motor Parameter]   [Observer/PLL]    [센싱·인버터]
    불일치 여부          추종 불안정          이상 여부
          |                 |                  |
     Rs/Ld/Lq/Ke       EEMF 품질          ADC Offset/Gain
     EEPROM Scale      theta_err          PWM Sampling
     상간 L 편차       wo/wr_hat          상전류 불균형
          |                 |                  |
          +-----------------+------------------+
                            |
                            v
               [최초 이상시점 비교]
                            |
         +------------------+------------------+
         |                  |                  |
         v                  v                  v
 theta_err 선행       IqeRef 선행       상전류 한 상 선행
 Observer/PLL         속도오차/제어      센싱/인버터/권선
 원인 우선            포화 우선          원인 우선
         |                  |                  |
         +------------------+------------------+
                            |
                            v
                    [원인 확정 시험]
```

---

# 5. 원인별 Logic Tree 및 검사 방법

## 5.1 Motor Parameter 및 EEPROM 불일치

### 5.1.1 판단 Logic Tree

```text
[특정 모터 또는 특정 생산 Lot에서만 불량?]
              |
       +------+------+
       |             |
      예            아니오
       |             |
       v             v
[모터 실측값 비교]  [공통 FW/PCB/제어값 검토]
       |
       v
[Rs/Ld/Lq/Ke가 정상 모터와 다른가?]
       |
  +----+----+
  |         |
 예        아니오
  |         |
  v         v
[EEPROM     [Q-format/Scale 및
 등록값과    파생상수 계산 검토]
 실측 비교]
  |
  v
[OP_Ld/OP_Lq/Ld_Ts/OP_KE_Const 일치?]
  |
  +-- 아니오 → Parameter/Scale 근본원인 후보
  |
  +-- 예 → Observer 입력·센싱 검토로 이동
```

### 5.1.2 검사항목

- 상간 인덕턴스 `U-V`, `V-W`, `W-U`
- 정지 상태 LCR 측정값
- 운전 조건별 `Ld`, `Lq`
- 권선저항 `Rs`
- 역기전력 상수 `Ke`
- EEPROM의 `OP_Ld`, `OP_Lq`, `OP_Rs`, `OP_KE`
- Runtime 파생상수 `Ld_Ts`, `OP_KE_Const`
- Q-format, Shift, 자료형 폭, 부호

### 5.1.3 검사 방법

1. 정상 모터와 불량 모터를 동일 온도에서 측정한다.
2. 동일 LCR 주파수, 측정전압 및 Rotor 위치에서 상간 L을 측정한다.
3. 세 상의 평균값과 상간 편차를 비교한다.
4. 가능한 경우 Rotor angle을 고정하여 d축과 q축 방향의 L을 측정한다.
5. 전류 크기를 단계적으로 바꾸어 자기포화에 따른 L 감소 특성을 비교한다.
6. 실측 `Rs/Ld/Lq/Ke`와 EEPROM Raw 값을 대조한다.
7. EEPROM Raw가 Runtime 상수로 변환되는 전체 경로를 확인한다.
8. 정상 모터 파라미터를 불량 모터에 적용한 경우와 실측 파라미터를 적용한 경우를 비교한다.

### 5.1.4 판정 기준

```text
실측값과 EEPROM 값 불일치
+ Observer Error 증가
+ theta_err_mori 증가
```

위 세 조건이 같은 시점과 운전영역에서 재현되면 Motor Parameter 불일치를 우선 원인으로 판단한다.

### 5.1.5 주의사항

- L값은 측정 주파수, Rotor 위치, 전류 및 자기포화에 따라 달라진다.
- 정지 LCR 값 하나만으로 운전 중 `Ld/Lq`를 확정하지 않는다.
- `Ld/Lq`의 단위와 Firmware 내부 Q-format을 혼용하지 않는다.
- 특정 생산품만 불량이라면 Turn 수, 상간 L 편차, Air Gap 및 자석 조립 편차를 함께 확인한다.

---

## 5.2 Observer EEMF 계산 오류

### 5.2.1 판단 Logic Tree

```text
[Error_VDS_hat_sl / Error_VQS_hat_sl가 먼저 이상?]
                     |
              +------+------+
              |             |
             예            아니오
              |             |
              v             v
 [Observer 입력과 모델 검토] [PLL 또는 전류루프 검토]
              |
     +--------+---------+----------------+
     |                  |                |
     v                  v                v
전류 입력 이상     전압 추정 이상    모델상수 이상
Ia/Ib/Ic           Vdc/PWM Duty       Rs/Ld/Lq/Ke
Offset/Gain        Dead-time          Scale/Q-format
     |                  |                |
     +------------------+----------------+
                        |
                        v
             [EEMF 크기·방향 복원 확인]
```

### 5.2.2 검사항목

- `Error_VDS_hat_sl`
- `Error_VQS_hat_sl`
- `IDS_mori`, `IQS_mori`
- `B_Ide`, `B_Iqe`
- `B_Ias`, `B_Ibs`, `B_Ics`
- `Vdc`
- `B_Ta`, `B_Tb`, `B_Tc`
- `Rs`, `Ld_Ts`, `Lq`, `OP_KE_Const`
- `Filter_emf_ob`

### 5.2.3 검사 방법

1. PWM OFF 상태에서 상전류 Offset 평균과 편차를 측정한다.
2. 정상 모터와 불량 모터의 `Error_VDS_hat_sl/Error_VQS_hat_sl` 크기, Offset, 리플을 비교한다.
3. 동일 운전점에서 EEMF 벡터 크기를 계산한다.

```text
EEMF magnitude = sqrt(Error_VDS_hat_sl² + Error_VQS_hat_sl²)
```

4. EEMF 벡터의 방향이 연속적으로 변하는지 확인한다.
5. 전류 차분항이 PWM 리플을 과도하게 증폭하는지 확인한다.
6. `Filter_emf_ob`를 제한된 범위에서 변경하여 노이즈와 위상지연의 변화를 비교한다.
7. 실측 전압 또는 DC Link 전압과 PWM 기반 추정전압의 차이를 비교한다.

### 5.2.4 원인 판정

- `Error_VDS/VQS`가 `theta_err_mori`보다 먼저 왜곡되면 Observer 입력 또는 모델 문제가 우선이다.
- `Error_VDS/VQS`는 안정적인데 `theta_err_mori`부터 발산하면 PLL 위상검출 및 게인 문제가 우선이다.
- 속도가 증가할수록 오차가 커지면 `L`, `Ke`, 교차결합 및 전압 추정 오차 가능성이 높다.

### 5.2.5 주의사항

`Error_VDS_hat_sl`과 `Error_VQS_hat_sl`은 이름에 Error가 들어가지만 단순 제어오차가 아니라, 모터 모델 성분을 제거한 뒤 남는 EEMF 잔차 벡터로 해석해야 한다.

---

## 5.3 PLL Lock 상실 및 게인 불안정

### 5.3.1 판단 Logic Tree

```text
[theta_err_mori가 0° 주변에서 이탈?]
                 |
          +------+------+
          |             |
         예            아니오
          |             |
          v             v
[wo_hat_mori 확인]   [속도/전류루프 검토]
          |
  +-------+--------+
  |                |
고주파 진동       한 방향 증가/포화
  |                |
  v                v
Kp 과대,         Ki/Scale/부호,
EEMF Noise       적분 포화 검토
  |                |
  +-------+--------+
          |
          v
[wr_hat_mori가 실제속도와 분리되는가?]
          |
    예 → PLL Lock 상실 우선
    아니오 → 위상오차 Scale/판정 기준 재검토
```

### 5.3.2 검사항목

- `theta_err_mori`
- `Error_sum_mori`
- `wo_hat_mori`
- `wr_hat_mori`
- `theta_mori`
- `Kp_mori`
- `Ki_mori`
- `Filter_wr`
- `Filter_emf_ob`
- `B_WeRef`, `B_WeEst_temp`

### 5.3.3 검사 방법

1. 불량 발생 전후 `theta_err_mori`의 평균, Peak, RMS 및 방향성을 확인한다.
2. `theta_err_mori` 증가와 `wo_hat_mori` 진동의 선후관계를 확인한다.
3. `Error_sum_mori`가 한 방향으로 지속 증가하거나 제한값에 머무는지 확인한다.
4. `wo_hat_mori`와 `wr_hat_mori`의 차이를 확인한다.
5. `wr_hat_mori`와 독립 속도계 또는 `mech_speed`를 비교한다.
6. PLL Kp를 단계적으로 낮추어 고주파 진동 변화 여부를 확인한다.
7. Ki를 단계적으로 낮추어 저주파 헌팅과 적분 포화 여부를 확인한다.
8. `Filter_wr` 변경 시 리플 감소와 응답지연을 함께 확인한다.

### 5.3.4 파형별 판단

| 파형 특징 | 우선 검토 |
|---|---|
| `theta_err` 고주파 진동, 평균 0 근처 | Kp 과대, EEMF 노이즈 |
| `theta_err` 한 방향 Bias | 좌표계, EEMF Offset, 파라미터 Bias |
| `Error_sum` 지속 증가 | Ki, 적분 포화, 지속 위상오차 |
| `wo_hat`만 크게 진동 | PLL PI 출력 및 EEMF 노이즈 |
| `wo_hat` 진동 후 `wr_hat` 지연 | Filter 설정 |
| `wr_hat`와 실제속도 분리 | PLL Lock 상실 또는 Scale 오류 |

### 5.3.5 주의사항

- PLL Lock은 속도가 한 번 일치했다는 의미가 아니다.
- `theta_err_mori`와 보정량이 허용범위 내에서 지속 유지되어야 한다.
- 필터를 크게 하면 리플은 줄지만 위상지연으로 오히려 고속 안정성이 나빠질 수 있다.
- PLL 게인은 센싱과 Observer 품질을 확인한 뒤 조정한다.

---

## 5.4 전류 센싱 Offset, Gain 및 Sampling 문제

### 5.4.1 판단 Logic Tree

```text
[특정 상전류가 먼저 비정상?]
                |
         +------+------+
         |             |
        예            아니오
         |             |
         v             v
[해당 채널 점검]   [3상 공통 원인 검토]
ADC Offset/Gain     각도/전류지령/부하
Shunt/OP Amp
Connector
         |
         v
[Ia + Ib + Ic ≈ 0 만족?]
         |
    +----+----+
    |         |
  아니오      예
    |         |
센싱/복원    좌표변환 및
오류 우선    PLL 검토
```

### 5.4.2 검사항목

- ADC Raw 값
- PWM OFF Offset
- `B_Ias`, `B_Ibs`, `B_Ics`
- `B_Ide`, `B_Iqe`
- Ia/Ib/Ic 합
- 상별 Gain
- ADC Sampling 시점
- PWM Duty와 최소 펄스
- Shunt 및 OP Amp 출력

### 5.4.3 검사 방법

1. PWM OFF에서 각 채널의 평균과 표준편차를 측정한다.
2. 무전류 상태에서 온도 변화에 따른 Offset Drift를 확인한다.
3. DC 전류 주입 또는 기준 전류를 사용해 상별 Gain을 비교한다.
4. 동일 전류를 각 채널에 입력하여 ADC Count가 동일한지 확인한다.
5. 운전 중 `Ia + Ib + Ic` 잔차를 계산한다.
6. PWM Duty에 따라 특정 채널이 왜곡되는지 확인한다.
7. ADC Sampling 위치가 스위칭 Edge 또는 최소펄스 구간과 겹치는지 확인한다.
8. 정상 PCB와 불량 PCB를 교환하여 모터 종속성과 PCB 종속성을 분리한다.

### 5.4.4 판정 기준

- 특정 상만 먼저 커지거나 납작해지면 해당 상의 센싱 또는 인버터 채널을 우선 확인한다.
- 세 상이 균형을 유지하며 동시에 확대되면 공통 전류지령, 추정각 또는 속도루프 발산을 우선 확인한다.
- dq 전류는 불안정하지만 abc 전류 합과 대칭성이 정상이라면 좌표변환 각도 오류 가능성이 증가한다.

---

## 5.5 인버터 출력, 상순서 및 Dead-time 문제

### 5.5.1 판단 Logic Tree

```text
[한 상 파형만 찌그러지거나 결상 형태?]
                 |
          +------+------+
          |             |
         예            아니오
          |             |
          v             v
[Gate/PWM 채널]     [공통 제어원인]
[상배선/소자]        검토로 이동
          |
          v
[B_Ta/Tb/Tc와 실제 상전압 매핑 일치?]
          |
   +------+------+
   |             |
  아니오          예
   |             |
채널/상순서       Dead-time,
오류              Minimum Pulse,
                  Vdc Ripple 확인
```

### 5.5.2 검사항목

- `B_Ta`, `B_Tb`, `B_Tc`
- 실제 Gate U/V/W 파형
- 실제 상전압
- DC Link 전압 및 Ripple
- Dead-time
- 최소 On/Off Pulse
- Gate Driver Fault
- 상배선 순서

### 5.5.3 검사 방법

1. Firmware의 U/V/W Duty와 실제 Gate 채널을 대조한다.
2. PWM Enable ON/OFF 시 Safe State를 확인한다.
3. 상·하측 Gate가 동시에 켜지는 구간이 없는지 확인한다.
4. 불량 발생 직전 Duty가 0% 또는 100%에 고정되는지 확인한다.
5. 최소펄스 소실과 상전류 Zero Crossing 왜곡을 확인한다.
6. DC Link Dip 또는 Ripple가 불량 발생과 동시에 나타나는지 확인한다.
7. 정상 보드와 불량 보드를 교차 시험한다.

---

## 5.6 Current PI, Voltage Limit 및 Anti-Windup

### 5.6.1 판단 Logic Tree

```text
[B_IqeRef가 먼저 증가?]
            |
     +------+------+
     |             |
    예            아니오
     |             |
     v             v
[속도오차/부하]  [각도/센싱/전류루프]
[Speed PI]       우선 검토
     |
     v
[Vd/Vq 또는 limit_cond 포화?]
     |
  +--+--+
  |     |
 예    아니오
  |     |
  v     v
전압제한/       Current PI
Anti-Windup     Gain/Scale
검토            검토
```

### 5.6.2 검사항목

- `B_IdeRef`, `B_IqeRef`
- `B_Ide`, `B_Iqe`
- `B_VdeRef`, `B_VqeRef`
- `limit_cond`
- Current PI P/I State
- 전압 제한값
- Anti-windup 상태
- `B_WeRef`, `B_WeEst_temp`

### 5.6.3 검사 방법

1. Ref와 Feedback 오차를 같은 축에서 비교한다.
2. `B_IqeRef` 증가 전 속도 Feedback이 먼저 떨어졌는지 확인한다.
3. `B_IqeRef`는 일정한데 `B_Iqe`만 발산하는지 확인한다.
4. `B_IdeRef ≈ 0`인데 `B_Ide`가 증가하는지 확인한다.
5. Voltage Limit 진입 시 PI 적분값이 계속 증가하는지 확인한다.
6. 출력 제한 해제 후 전류가 정상으로 신속히 복귀하는지 확인한다.
7. Kp/Ki와 PWM 주기 및 L값의 관계를 재확인한다.

### 5.6.4 원인 판정

- `B_IqeRef`가 먼저 증가하면 Speed PI 또는 속도 Feedback 문제를 우선 검토한다.
- `B_IqeRef`는 정상인데 `B_Iqe`가 진동하면 Current PI, 각도오차 또는 인버터 문제를 우선 검토한다.
- `B_Ide`가 먼저 증가하면 dq축 정렬 또는 PLL 추정각 문제 가능성이 높다.

---

## 5.7 생산품 모터 전기적 편차 및 권선 불량

### 5.7.1 판단 Logic Tree

```text
[같은 PCB/FW에서 특정 모터만 불량?]
                 |
          +------+------+
          |             |
         예            아니오
          |             |
          v             v
[모터 전기특성 비교] [PCB/FW 공통원인 검토]
          |
   +------+------+----------------+
   |             |                |
   v             v                v
상간 R 편차    상간 L 편차      Ke/파형 편차
   |             |                |
   +-------------+----------------+
                 |
                 v
[권선 Turn/단락/접속/철심 및 자석 공정 추적]
```

### 5.7.2 검사항목

- 상간 저항
- 상간 인덕턴스
- 절연저항
- Surge Test
- 역기전력 파형과 상간 균형
- Turn 수
- 권선 접속 상태
- 부분 단락
- Rotor 자석 착자 및 위치
- Air Gap 편차

### 5.7.3 검사 방법

1. 정상품과 불량품을 동일 계측 조건에서 비교한다.
2. 모터와 PCB를 교차 조합해 불량이 모터를 따라가는지 확인한다.
3. 상간 R/L 편차를 측정한다.
4. Surge 또는 Impulse 시험으로 부분 단락을 확인한다.
5. 외부 구동으로 역기전력 파형과 상간 균형을 측정한다.
6. Rotor angle별 L 변화와 Cogging 특성을 비교한다.
7. 제조 Lot, 권선기, 작업조건, 함침 및 조립 이력을 추적한다.

---

# 6. 최초 이상신호 기반 판정표

| 최초 이상신호 | 우선 원인 | 다음 검사 |
|---|---|---|
| `Error_VDS_hat_sl/Error_VQS_hat_sl` 왜곡 | Observer 입력, L/R/Ke, 전압 추정 | 실측 파라미터 및 센싱 비교 |
| `theta_err_mori` 급증 | PLL Lock 상실, EEMF 방향 오류 | `wo_hat`, `Error_sum`, EEMF 확인 |
| `wo_hat_mori` 고주파 진동 | PLL Kp 과대, EEMF Noise | Kp 및 Filter_emf_ob Sweep |
| `Error_sum_mori` 단방향 증가 | 지속 위상오차, Ki/포화 | 적분 제한과 Scale 확인 |
| `wr_hat_mori` 급락 | PLL 추정 실패 또는 실제 속도 저하 | 독립 속도와 비교 |
| `B_Ide` 증가 | dq축 정렬 불량, 추정각 오류 | theta_err와 선후 비교 |
| `B_IqeRef` 증가 | 속도오차, Speed PI 요구 증가 | Ref/Fbk 속도 비교 |
| `B_Iqe`만 발산 | Current PI, 각도, 인버터 | Ref와 Feedback 비교 |
| `limit_cond` 활성 후 발산 | 전류·전압 포화, Anti-windup | PI State와 제한값 확인 |
| 특정 상전류 먼저 왜곡 | 센싱, Gate, 권선, 채널 | 상별 교차시험 |
| 3상 균형 유지하며 확대 | 공통 지령 또는 추정각 발산 | Speed/PLL/Error 순서 확인 |
| `fdetect_pos_err` 증가 | 결과 플래그 가능성 | 발생 직전 Error를 추적 |
| `taljo_restart_cnt` 증가 | 탈조 후 재기동 | 최초 이상시점까지 역추적 |

---

# 7. 권장 측정 Mode 및 변수 구성

## Mode A. Observer Input Quality

```text
Error_VDS_hat_sl
Error_VQS_hat_sl
IDS_mori
IQS_mori
```

목적:

- EEMF 잔차 벡터 품질 확인
- 노이즈, Offset, 방향 왜곡 확인
- Observer 문제가 PLL보다 먼저 발생하는지 확인

## Mode B. PLL Lock

```text
theta_err_mori
Error_sum_mori
wo_hat_mori
wr_hat_mori
```

목적:

- 위상오차 수렴 확인
- PLL PI 출력과 적분상태 확인
- 필터 전후 속도 비교

## Mode C. dq Current Tracking

```text
B_IdeRef
B_Ide
B_IqeRef
B_Iqe
```

목적:

- d/q 전류 오차의 선후관계 확인
- Current PI 불안정과 각도오류 분리

## Mode D. Speed Tracking

```text
B_WeRef
B_WeEst_temp
mech_speed
wr_hat_mori
```

목적:

- 실제 속도 저하와 추정속도 오류 분리
- Speed PI가 전류 증가를 지시했는지 확인

## Mode E. Limit and Fault

```text
limit_cond
IF_Trans_fail
fdetect_pos_err
taljo_restart_cnt
rdc_restart_cnt
```

목적:

- 제한 및 Fault 발생순서 확인
- 보호 플래그를 원인이 아닌 결과로 구분

## Mode F. Phase Current and PWM

```text
B_Ias
B_Ibs
B_Ics
B_Ta
B_Tb
B_Tc
Vdc
```

목적:

- 상별 불균형 확인
- Duty/상전류 매핑 확인
- DC Link 및 PWM 출력 이상 확인

---

# 8. 시험 Matrix

| 시험 ID | 시험 내용 | 변경 변수 | 고정 조건 | 판정 목적 |
|---|---|---|---|---|
| T01 | 정상/불량 모터 교차시험 | 모터 | PCB, FW, EEPROM | 모터 종속성 분리 |
| T02 | 정상/불량 PCB 교차시험 | PCB | 모터, FW, EEPROM | 보드 종속성 분리 |
| T03 | 실측 파라미터 적용 | Rs/Ld/Lq/Ke | 모터, 부하 | 모델 불일치 확인 |
| T04 | PLL Kp Sweep | Kp_mori | Ki, Filter | 고주파 진동 민감도 |
| T05 | PLL Ki Sweep | Ki_mori | Kp, Filter | 저주파 헌팅/적분포화 |
| T06 | EEMF Filter Sweep | Filter_emf_ob | PLL Gain | Noise와 위상지연 Trade-off |
| T07 | Speed Filter Sweep | Filter_wr | PLL Gain | 속도 리플과 응답지연 |
| T08 | Current PI Sweep | Current Kp/Ki | L, PWM | 전류루프 안정성 |
| T09 | PWM OFF Offset | 온도 | 전원, Board | ADC Offset Drift |
| T10 | 상별 Gain 검사 | 기준 전류 | ADC 조건 | 센싱 Gain 편차 |
| T11 | LCR 비교 | Rotor 위치/주파수 | 온도 | 상간 L 편차 및 위치 의존성 |
| T12 | BEMF 비교 | 외부 구동속도 | 측정 배선 | Ke와 상간 균형 |
| T13 | Voltage Limit | 속도/전류 지령 | 모터 | Anti-windup 복귀성 |
| T14 | 반복기동 | 동일 조건 반복 | 모든 설정 | 재현성 및 최초 이상신호 |

---

# 9. 시험 수행 순서

```text
Step 1
모터/PCB 교차시험으로 불량 귀속 분리

Step 2
PWM OFF 전류 Offset 및 상별 Gain 확인

Step 3
상간 R/L, Ke 및 EEPROM/Runtime Parameter 비교

Step 4
Observer EEMF d/q 파형 확인

Step 5
theta_err, wo_hat, wr_hat, Error_sum 확인

Step 6
Id/Iq Ref와 Feedback 확인

Step 7
limit_cond와 Current/Voltage PI State 확인

Step 8
PLL Gain 및 Filter를 제한된 범위에서 Sweep

Step 9
반복시험으로 동일 최초 이상신호 재현

Step 10
원인 후보 변경 전/후 파형과 KPI 비교
```

---

# 10. 주의 깊게 살펴야 할 내용

## 10.1 원인과 결과를 뒤집지 않는다

다음 변수는 최종 Fault 결과일 수 있다.

```text
fdetect_pos_err
taljo_restart_cnt
IF_Trans_fail
limit_cond
```

Fault 플래그가 켜진 시점이 아니라, 그 전에 처음 변화한 EEMF, theta error, 속도 또는 전류를 확인한다.

## 10.2 불균일 Sampling을 고려한다

CS+ Analysis Chart는 Sample 간격이 일정하지 않을 수 있다. 행 번호가 아니라 실제 `Time` 값을 사용해 이벤트 순서를 비교한다.

## 10.3 Raw/Q-format을 임의로 물리 단위로 환산하지 않는다

```text
IDS_mori
IQS_mori
Error_VDS_hat_sl
Error_VQS_hat_sl
Error_sum_mori
```

위 변수는 내부 Scale 확인 전 Raw 또는 Q-format으로 비교한다.

## 10.4 정상 평균값만 보지 않는다

평균 속도가 정상이어도 순간 리플이 증가할 수 있다. 다음을 함께 본다.

```text
평균
Peak-to-Peak
RMS
최대 절대값
정착시간
Bias
포화 지속시간
```

## 10.5 한 번에 여러 파라미터를 바꾸지 않는다

PLL Gain, EEMF Filter, Speed Filter, Current PI를 동시에 변경하면 원인 분리가 불가능하다. 한 번에 한 항목만 변경하고 정상/불량 파형을 같은 축에서 비교한다.

---

# 11. 현재 우선순위와 권장 결론

기계적 원인이 배제된 현재 조건에서 우선순위는 다음과 같다.

1. 실제 모터 `Rs/Ld/Lq/Ke`와 EEPROM/Runtime Parameter 불일치
2. 전류 및 전압 입력 품질에 의한 Observer EEMF 방향 오차
3. `theta_err_mori` 발산을 동반한 PLL Lock 상실
4. PLL Gain과 EEMF/Speed Filter의 조합 불안정
5. 전류센서 Offset/Gain 및 PWM 동기 Sampling 문제
6. Current PI, Voltage Limit 및 Anti-windup 문제
7. 인버터 상채널 또는 모터 권선의 전기적 불균형

현재 가장 중요한 판정은 다음 두 질문이다.

```text
Q1. theta_err_mori가 B_IqeRef보다 먼저 증가했는가?

예:
Observer/PLL 원인을 우선한다.

아니오, B_IqeRef가 먼저 증가:
속도 Feedback, Speed PI 및 제한조건을 우선한다.
```

```text
Q2. Error_VDS_hat_sl/Error_VQS_hat_sl가 theta_err_mori보다 먼저 왜곡됐는가?

예:
센싱, 전압추정, Motor Parameter 및 Observer를 우선한다.

아니오:
PLL 위상검출, Gain, 적분 및 Filter를 우선한다.
```

---

# 12. 최종 판정 기록 양식

## 시험 정보

```text
Motor Serial:
PCB Serial:
FW Version:
EEPROM Version:
시험 일시:
시험 조건:
부하 조건:
주변 온도:
```

## 최초 이상신호

```text
최초 이상 변수:
최초 이상 시각:
정상 범위:
불량 측정값:
후속 이상 변수:
Fault 발생 시각:
```

## 최종 판정

```text
근본원인:
직접 원인:
결과 현상:
재현 여부:
교차시험 결과:
파라미터 변경 전:
파라미터 변경 후:
최종 PASS 기준:
```

---

# 13. 핵심 요약

```text
불량 파형
= 운전 중 전류 진동이 점진적으로 증가한 뒤 보호 정지

기계원인
= 배제됨

우선 원인
= Motor Parameter 또는 Observer 입력 오차
  → EEMF 방향 왜곡
  → theta_err 증가
  → PLL Lock 상실
  → dq축 혼합
  → 유효토크 저하
  → Iq 요구 및 상전류 증가
  → 제한 또는 탈조 보호

최종 원인 분리 기준
= 가장 먼저 증가한 Error를 찾는다.
```

---

## 참고 근거

- 내부 모터제어 교재의 기본 전압방정식 및 Observer 구조에서는 `Rs`, `Ld`, `Lq`, 전류 변화량과 속도 교차결합 성분이 EEMF 잔차 계산에 사용된다.
- 내부 교재는 튜닝 순서를 센싱/상순서/Scale, 전류루프, Observer, PLL, I/F 및 전환, 속도루프 순으로 제시한다.
- Ld와 Lq는 Rotor 구조와 운전 전류에 따라 달라질 수 있으므로, 정지 LCR 측정값과 실제 운전상태 파라미터를 구분해야 한다.
