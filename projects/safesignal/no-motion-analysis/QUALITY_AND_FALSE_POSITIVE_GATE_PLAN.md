# Quality Criteria and False Positive Gate Plan

_Last updated: 2026-05-24 | Updated by: codex_

## 배경

E2 `NO_MOTION` baseline 5분 x 2세션 분석 결과, 수집 품질과 오탐 방어를 같은 기준으로 처리하면 안 된다는 점이 명확해졌다.

- pair rate는 약 96Hz로 안정적이다.
- 손실률은 약 2%로 양호하다.
- SNTP 동기화 완료 상태에서도 pair_dt p95는 약 25ms, pair_dt p99는 약 40ms 수준이다.
- no_motion에서도 raw amplitude frame_delta spike는 드물게 발생한다.
- spike는 pair_dt/gap이 매우 나쁜 구간과 완전히 일치하지 않는다.

따라서 `pair_dt`나 `ts_gap`만으로 수집 폐기 또는 실시간 추론 skip을 강하게 걸면 정상 no_motion이나 실제 낙상 window를 잘못 제외할 수 있다.

## 1. 수집 품질 기준과 추론 기준 분리

### 수집 품질 기준

목적:

```text
학습 데이터에 심한 패킷 손실, 큰 gap, 페어링 오류가 섞이는 것을 줄인다.
```

성격:

```text
저장 전 판단 / 재수집 권고 / 학습 데이터 제외 여부 판단
```

초안:

```text
저장 가능:
  pair_rate >= 90Hz
  loss < 5%
  ts_gap max < 200ms
  pair_dt p99 < 60ms

검토:
  pair_rate 85~90Hz
  loss 5~10%
  ts_gap max 200~300ms
  pair_dt p99 60~100ms

재수집 권장:
  pair_rate < 85Hz
  loss >= 10%
  ts_gap max >= 300ms
  pair_dt p99 >= 100ms
```

주의:

- E2 baseline에서 pair_dt p95는 25ms 수준이므로, pair_dt p95=15~25ms를 곧바로 불량으로 보면 안 된다.
- 대시보드 색상 기준은 E2 baseline 기준으로 완화할 필요가 있다.
- 수집 기준은 학습 데이터 품질 보증용이며, 실제 추론 hard skip 기준과 다르다.

### 실시간 추론 quality flag 기준

목적:

```text
실시간 입력 품질을 기록하고, 오탐/미탐 사후 분석에 사용한다.
```

성격:

```text
대부분 warn/flag/log 용도
알림 차단 또는 추론 skip으로 직접 연결하지 않음
```

초안:

```text
quality_warn:
  pair_dt p99 > 60ms
  or ts_gap max > 200ms
  or raw_spike_warn=True

quality_high:
  pair_dt p99 > 100ms
  or ts_gap max > 300ms
  or raw_spike_high=True
```

추론 정책:

```text
quality_warn=True여도 추론 수행
quality_high=True여도 기본은 추론 수행 + 로그 강화
hard skip은 low-energy no_motion/static gate에 한정 검토
```

이유:

낙상 순간에 Wi-Fi 수신 상태가 나빠질 수 있으므로, gap/pair_dt만으로 window를 버리면 recall을 손상시킬 수 있다.

## 2. no_motion baseline 기반 raw spike 기준 후보

E2 no_motion 596개 window 기준:

```text
frame_delta_mean p99 ~= 0.730
frame_delta_p95 p99 ~= 1.287
frame_delta_p99 p99 ~= 1.690
frame_delta_max p95 ~= 1.927
frame_delta_max p99 ~= 2.039
frame_delta_max max ~= 2.798
spike_ratio p99 ~= 0.004
spike_ratio max ~= 0.007
```

후보 기준:

```text
raw_spike_warn:
  spike_ratio > 0.01
  or frame_delta_max > 2.1

raw_spike_high:
  spike_ratio > 0.03
  or frame_delta_max > 3.0
```

해석:

- no_motion에서도 frame_delta_max 2.0 부근은 드물지만 발생한다.
- frame_delta_max만 단독으로 알림 차단 기준에 쓰면 안 된다.
- raw_spike는 `이 window가 불안정했다`는 metadata로 남기는 것이 우선이다.
- fall/non-fall 동작 window와 비교한 뒤 threshold를 확정해야 한다.

## 3. no_motion 클래스 학습 반영 체크리스트

이미 완료:

```text
collect/labels.py에 NO_MOTION 추가
server dashboard class list에 no_motion 추가
collect selfcheck 목표 수 242로 조정
NO_MOTION duration 300s, target 2 반영
```

추가 확인 필요:

```text
1. fine-tune class mapping을 8-class로 확장
   기존: fall, walking, sit_stand, lying, standing, running, picking
   추가: no_motion

2. pretrained 6-class checkpoint에서 head 확장 방식 결정
   기존 6개 class weight 이식
   running, no_motion head는 신규 초기화

3. Alsaify source에는 no_motion이 없음
   no_motion은 SafeSignal-only class로 처리
   Alsaify batch loss에서 no_motion target이 나오지 않도록 유지

4. cache builder가 NO_MOTION CSV를 읽는지 확인
   data/raw/E*_S*_A_NO_MOTION_T*.csv naming 인식 필요

5. evaluation metric 수정
   8-class confusion matrix
   fall recall
   no_motion precision/recall
   no_motion -> fall 오탐률
   event-level false alarm rate

6. threshold selection 수정
   fall probability threshold만 볼지
   no_motion probability가 높으면 fall 후보를 억제할지 결정
```

권장 class order:

```text
0 fall
1 walking
2 sit_stand
3 lying
4 standing
5 running
6 picking
7 no_motion
```

주의:

no_motion 데이터가 적은 상태에서 class를 추가하면 모델이 no_motion을 과소학습할 수 있다. 현재 E2 2세션은 baseline/분석용으로는 충분하지만, 학습용으로는 최소 5분 x 6세션, 권장 5분 x 10세션 이상이 필요하다.

## 4. 실시간 오탐 방어 후보

### 4.1 Low-energy no_motion/static gate

목적:

```text
아무도 없거나 정적인 window가 z-score 후 의미 있는 패턴처럼 과증폭되는 것을 줄인다.
```

후보 위치:

```text
raw window 또는 SDP 계산 후, z-score 적용 전
```

정책:

```text
SDP/raw energy가 no_motion baseline p95 이하이고 fall-like 변화가 없으면 no_motion/static으로 판단
재학습 전에는 skip 또는 fall suppression 후보
재학습 후에는 no_motion class probability와 함께 사용
```

주의:

실제 낙상 window는 에너지가 낮을 가능성이 작지만, threshold를 너무 높게 잡으면 recall을 손상시킬 수 있다. fall 세션의 energy p5와 비교 후 결정한다.

### 4.2 Consecutive window confirmation

목적:

```text
단일 window spike나 순간 오분류가 바로 낙상 알림으로 이어지는 것을 방지
```

후보:

```text
fall_prob >= threshold 조건이 2개 연속 window에서 만족될 때 event 발생
stride=1s이면 N=2는 약 1초 추가 지연
```

권장:

```text
데모: N=2
장기 운영: N=2 또는 N=3 비교
```

### 4.3 Probability margin

목적:

```text
fall이 1등이어도 2등 class와 차이가 작으면 불확실한 분류로 처리
```

후보:

```text
fall_prob >= threshold
and fall_prob - second_prob >= 0.15
```

주의:

fine-tune validation 결과로 margin을 조정해야 한다.

### 4.4 Cooldown

목적:

```text
낙상 1회 이후 일어나는 과정 또는 같은 이벤트 반복 감지를 억제
```

후보:

```text
cooldown 30~60s
```

이미 서버에 cooldown 유틸이 있으므로 기존 구조 활용 가능.

### 4.5 Quality flag logging

목적:

```text
오탐/미탐 발생 시 원인 분석을 가능하게 한다.
```

추론 결과에 함께 남길 metadata 후보:

```text
window_start_ts
pair_rate_hz
pair_dt_p99_us
ts_gap_max_us
raw_frame_delta_mean
raw_frame_delta_max
raw_spike_ratio
sdp_energy_pre_zscore
rpca_sparse_ratio
quality_warn
quality_high
```

## 5. 다음 코드 작업 후보

우선순위:

```text
1. 분석 스크립트 추가 또는 debug tool화
   debug/data_collect/analyze_no_motion_baseline.py 후보

2. dashboard 품질 기준 완화
   E2 baseline 기준으로 pair_dt 색상/판정 업데이트

3. 실시간 추론 quality metadata 계산 위치 조사
   server/inference/buffer.py
   server/inference/predictor.py
   server/inference/worker.py

4. consecutive window detector 추가
   worker 또는 main의 fall event 발생 직전

5. no_motion 8-class fine-tune 경로 점검
   model/finetune/train.py
   cache builder / dataset loader
```

## 결론

현재 가장 중요한 설계 방향:

```text
수집 데이터 품질 기준은 학습 데이터 관리용으로 유지한다.
실시간 추론에서는 pair_dt/gap을 hard skip으로 쓰지 않는다.
오탐 억제는 no_motion energy, raw spike flag, consecutive confirmation, cooldown 조합으로 설계한다.
no_motion class는 최종 학습에 반드시 포함하되, 현재 2세션은 baseline으로만 과신하지 않는다.
```
