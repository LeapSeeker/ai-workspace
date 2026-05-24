# E2 NO_MOTION Baseline Analysis

_Last updated: 2026-05-24 | Updated by: codex_

## 목적

E2에서 수집한 `NO_MOTION` 5분 x 2세션을 이용해, 사람이 없거나 정적인 운영 환경에서 정상적으로 발생하는 CSI 배경 노이즈와 순간 spike 범위를 확인한다.

이 분석은 단순 수집 품질 판정이 아니라, 낙상 오탐 억제를 위한 다음 기준을 잡는 데 목적이 있다.

- 정적/no_motion window의 raw amplitude 변화 범위
- spike 후보 window의 빈도와 크기
- pair_dt / ts_gap이 no_motion에서 어느 정도 자연스럽게 발생하는지
- 추론 hard skip 기준이 아니라 quality flag / energy gate 후보를 정하는 근거

## 입력 데이터

```text
data/raw/E2_S02_A_NO_MOTION_T001.csv
data/raw/E2_S02_A_NO_MOTION_T002.csv
```

수집 조건:

```text
환경: E2
대상자: S02
활동: NO_MOTION
길이: 5분 x 2세션
SNTP: RX1/RX2 동기화 완료 확인
```

주의:

E2는 완전한 빈 공간 확보가 쉽지 않다. 따라서 이 데이터는 "완전한 empty room"이라기보다 "실제 E2 운영 환경의 정적 배경 노이즈"로 해석한다.

## 분석 방법

- window length: 3초
- stride: 1초
- total windows: 596개
- raw CSV를 직접 사용했고, pandas/numpy 없이 표준 라이브러리 기반으로 계산했다.
- 각 window마다 아래 지표를 계산했다.

```text
count
rate_hz
pair_dt p95/p99/max
ts_gap p95/p99/max
amp_abs mean/p95
frame_delta mean/p95/p99/max
spike_ratio
spike_count
```

`frame_delta`는 인접 frame 사이 raw amplitude 평균 절대 변화량이다.

```text
frame_delta[t] = mean(abs(amplitude[t] - amplitude[t-1]))
```

`spike_ratio`는 window 내부 frame_delta가 robust threshold를 초과한 frame 비율이다.

```text
threshold = median(frame_delta) + 5 * 1.4826 * MAD(frame_delta)
spike_ratio = spike_frame_count / frame_delta_count
```

## 세션 단위 요약

```text
E2_S02_A_NO_MOTION_T001.csv
rows=28,828
duration=300.52s
pair_rate=95.93Hz
capture_ratio=0.959
loss=2.01%
pair_dt: p50=4.5ms p95=26.0ms p99=41.6ms max=50.0ms
ts_gap : p50=10.3ms p95=26.4ms p99=40.2ms max=122.8ms
amp_abs mean=9.821 p95=10.494 max=11.130
frame_delta_abs mean=0.681 p95=1.199 p99=1.448 max=2.798
```

```text
E2_S02_A_NO_MOTION_T002.csv
rows=29,086
duration=300.50s
pair_rate=96.79Hz
capture_ratio=0.968
loss=1.85%
pair_dt: p50=5.2ms p95=24.8ms p99=40.2ms max=50.0ms
ts_gap : p50=10.1ms p95=25.3ms p99=40.4ms max=115.6ms
amp_abs mean=9.911 p95=10.564 max=11.090
frame_delta_abs mean=0.675 p95=1.190 p99=1.442 max=2.037
```

해석:

- Hz와 손실률은 안정적이다.
- SNTP 동기화가 완료된 상태에서도 pair_dt p95는 약 25ms, pair_dt p99는 약 40ms 수준이다.
- 따라서 현재 E2 환경에서 pair_dt p95=25ms를 곧바로 불량 또는 추론 skip 근거로 보면 안 된다.
- ts_gap max는 115~123ms 수준으로, 큰 보간 artifact를 만들 가능성이 있는 200ms 이상 gap은 관찰되지 않았다.

## 3초 Window 전체 분포

전체 596개 window 기준:

```text
count: p50=289 p95=299 p99=304 max=309
rate_hz: p50=96.754 p95=100.003 p99=101.642 max=103.609

pair_dt_p95: p50=25.1ms p95=30.7ms p99=33.5ms max=36.1ms
pair_dt_p99: p50=39.0ms p95=45.4ms p99=46.5ms max=47.1ms
pair_dt_max: p50=47.1ms p95=49.8ms p99=50.0ms max=50.0ms

ts_gap_p95: p50=24.9ms p95=31.3ms p99=33.2ms max=38.0ms
ts_gap_p99: p50=38.2ms p95=46.9ms p99=50.5ms max=83.4ms
ts_gap_max: p50=49.3ms p95=63.8ms p99=81.5ms max=122.8ms

amp_abs_mean: p50=9.865 p95=9.935 p99=9.951 max=9.960
amp_abs_p95: p50=10.519 p95=10.628 p99=10.659 max=10.700

frame_delta_mean: p50=0.677 p95=0.710 p99=0.730 max=0.739
frame_delta_p95: p50=1.187 p95=1.267 p99=1.287 max=1.371
frame_delta_p99: p50=1.418 p95=1.558 p99=1.690 max=1.754
frame_delta_max: p50=1.625 p95=1.927 p99=2.039 max=2.798

spike_ratio: p50=0.000 p95=0.003 p99=0.004 max=0.007
spike_count: p50=0 p95=1 p99=1 max=2
```

핵심 해석:

- no_motion window에서도 frame_delta_max는 보통 1.6 전후까지 나온다.
- no_motion window의 frame_delta_max p95는 약 1.93, p99는 약 2.04다.
- 매우 튄 window의 최대 frame_delta는 2.798이다.
- spike_ratio는 대부분 0이고, 높아도 0.7% 수준이다.
- 3초 window 약 300 frame 기준 spike frame은 보통 0~1개, 최대로 2개 정도다.

## Spike 후보 Window

frame_delta_max 상위 window:

```text
T001 start=225.0s count=279 rate=93.6Hz delta_max=2.798 delta_p99=1.501 spike_ratio=0.007 gap_max=43.7ms pair_dt_p99=40.5ms
T001 start=226.0s count=278 rate=93.7Hz delta_max=2.798 delta_p99=1.502 spike_ratio=0.007 gap_max=43.7ms pair_dt_p99=39.0ms
T001 start=227.0s count=278 rate=93.3Hz delta_max=2.798 delta_p99=1.522 spike_ratio=0.007 gap_max=52.6ms pair_dt_p99=37.6ms
T001 start=141.0s count=291 rate=97.8Hz delta_max=2.066 delta_p99=1.690 spike_ratio=0.003 gap_max=42.7ms pair_dt_p99=33.0ms
T001 start=142.0s count=280 rate=93.7Hz delta_max=2.066 delta_p99=1.568 spike_ratio=0.004 gap_max=122.8ms pair_dt_p99=35.1ms
T001 start=143.0s count=278 rate=93.2Hz delta_max=2.066 delta_p99=1.375 spike_ratio=0.004 gap_max=122.8ms pair_dt_p99=34.5ms
T002 start=15.0s count=295 rate=98.7Hz delta_max=2.037 delta_p99=1.554 spike_ratio=0.007 gap_max=49.3ms pair_dt_p99=44.1ms
```

해석:

- 가장 큰 spike는 T001 225~227초 부근에 집중된다.
- 해당 구간도 gap_max는 43~53ms 수준으로 매우 나쁘지 않다.
- 따라서 spike 현상을 단순히 ts_gap/pair_dt 품질 문제로만 설명하기 어렵다.
- raw amplitude 변화량 또는 SDP/RPCA 이후 energy를 별도로 봐야 한다.

## 기준 후보

이 값들은 즉시 hard threshold로 확정하지 않고, fall/walking/sit_stand 등 실제 동작 window와 비교한 뒤 확정해야 한다.

현재 no_motion baseline만 기준으로 본 후보:

```text
no_motion 정상 범위 후보:
frame_delta_mean <= 0.75
frame_delta_p95 <= 1.35
frame_delta_p99 <= 1.75
frame_delta_max <= 2.1
spike_ratio <= 0.01
```

추론 로그용 quality flag 후보:

```text
raw_spike_warn:
  spike_ratio > 0.01
  or frame_delta_max > 2.1

raw_spike_high:
  spike_ratio > 0.03
  or frame_delta_max > 3.0
```

수집/추론 품질 flag 후보:

```text
pair_dt_warn:
  pair_dt_p99 > 60ms

pair_dt_high:
  pair_dt_p99 > 100ms

ts_gap_warn:
  ts_gap_max > 200ms

ts_gap_high:
  ts_gap_max > 300ms
```

주의:

- 위 품질 flag는 추론 hard skip 기준이 아니다.
- 특히 낙상 안전 시스템에서는 gap/pair_dt가 나쁘다는 이유만으로 추론을 skip하면 실제 낙상을 놓칠 수 있다.
- 실시간 추론에서 hard skip 후보는 no_motion/static 판단용 low-energy gate 정도로 제한하는 것이 안전하다.

## 오탐 억제 설계 시사점

1. `pair_dt p95=25ms`는 현재 E2 no_motion baseline의 정상 범위다.
2. 기존 dashboard 색상 기준은 E2 기준으로 과민할 수 있다.
3. no_motion에서도 작은 amplitude 변화는 지속적으로 발생한다.
4. 단일 spike는 드물지만 존재한다. spike_ratio는 보통 0~0.7%다.
5. 단일 패킷/단일 window threshold만으로 낙상 판단하면 오탐 가능성이 남는다.
6. 추론 단계에서는 아래 조합이 더 적절하다.

```text
raw quality flag
+ SDP/RPCA 전 energy gate
+ fall probability threshold/margin
+ consecutive window confirmation N=2
+ cooldown
```

## 다음 작업

1. fall / walking / sit_stand / lying / standing / picking / running window와 동일 지표 비교
2. no_motion vs human activity의 `frame_delta` 분리 가능성 확인
3. 전처리 pipeline에서 SDP z-score 전 energy 분포 계산
4. energy gate 후보를 no_motion p95/p99와 fall p5 기준으로 결정
5. dashboard 품질 판정 기준을 수집용 기준과 추론 quality flag 기준으로 분리
6. fine-tune 8-class 설계에 no_motion class 반영

---

## Cross-Activity Raw Window Comparison (2026-05-25)

추가로 현재 `data/raw`에 있는 활동 CSV와 동일한 3초 window / 1초 stride 기준으로 raw amplitude 변화량을 비교했다.

사용 가능한 활동:

```text
NO_MOTION: files=2, windows=596
WALK: files=60, windows=360
SIT_STD: files=60, windows=277
STAND: files=61, windows=183
LIE: files=30, windows=171
RUN: files=49, windows=294
```

현재 raw 폴더에는 fall/pick 데이터가 없어서 비교에서 제외됐다.

### 핵심 지표 비교

```text
NO_MOTION
frame_delta_mean p50=0.677 p95=0.710 p99=0.730
frame_delta_max  p50=1.625 p95=1.927 p99=2.039 max=2.798
spike_ratio      p50=0.000 p95=0.003 p99=0.004 max=0.007

STAND
frame_delta_mean p50=0.612 p95=0.660 p99=0.703
frame_delta_max  p50=1.403 p95=1.739 p99=1.798 max=1.905
spike_ratio      p50=0.000 p95=0.003 p99=0.003 max=0.003

LIE
frame_delta_mean p50=0.641 p95=0.681 p99=0.704
frame_delta_max  p50=1.464 p95=1.866 p99=2.004 max=2.004
spike_ratio      p50=0.000 p95=0.003 p99=0.006 max=0.006

SIT_STD
frame_delta_mean p50=0.677 p95=0.749 p99=0.798
frame_delta_max  p50=1.564 p95=1.909 p99=1.981 max=2.022
spike_ratio      p50=0.000 p95=0.003 p99=0.007 max=0.010

WALK
frame_delta_mean p50=0.674 p95=0.772 p99=0.810
frame_delta_max  p50=1.531 p95=2.046 p99=2.183 max=2.261
spike_ratio      p50=0.000 p95=0.003 p99=0.006 max=0.009

RUN
frame_delta_mean p50=0.748 p95=0.838 p99=0.873
frame_delta_max  p50=1.866 p95=2.758 p99=3.361 max=4.259
spike_ratio      p50=0.000 p95=0.010 p99=0.013 max=0.016
```

### no_motion p99 threshold 초과 비율

no_motion p99 기준:

```text
frame_delta_max_p99 = 2.039
frame_delta_mean_p99 = 0.730
spike_ratio_p99 = 0.004
```

각 활동에서 이 기준을 초과하는 window 비율:

```text
LIE:     fdmax>0.000, fdmean>0.000, spike_ratio>0.023
STAND:   fdmax>0.000, fdmean>0.000, spike_ratio>0.000
SIT_STD: fdmax>0.000, fdmean>0.094, spike_ratio>0.029
WALK:    fdmax>0.056, fdmean>0.181, spike_ratio>0.022
RUN:     fdmax>0.327, fdmean>0.595, spike_ratio>0.184
```

### 해석

1. raw frame_delta만으로 `NO_MOTION`과 `STAND`, `LIE`는 거의 분리되지 않는다.
2. `SIT_STD`, `WALK`는 일부 window에서 no_motion p99를 넘지만, 전체적으로 겹침이 크다.
3. `RUN`은 raw 변화량이 뚜렷하게 크므로 no_motion과 어느 정도 분리된다.
4. `NO_MOTION`의 frame_delta_max max=2.798이 `WALK` max=2.261보다 크다. 따라서 단일 spike 크기만으로 사람 동작 여부를 판단하면 위험하다.
5. raw spike 지표는 추론 hard gate보다 quality metadata로 쓰는 것이 안전하다.

### 설계 영향

raw amplitude 기반 gate만으로 오탐 억제를 해결하기 어렵다. 특히 빈 공간/정적 오탐은 다음 조합으로 봐야 한다.

```text
raw spike flag: 입력 이상 여부 기록
SDP/RPCA 전 energy: 실제 움직임 에너지 후보
모델 no_motion class: 학습 기반 정적 배경 분류
fall threshold/margin: 불확실한 fall 억제
consecutive window N=2: 단일 window 오탐 억제
cooldown: 이벤트 반복 억제
```

다음 분석 우선순위:

```text
1. SDP z-score 전 energy를 활동별로 비교
2. fall 데이터 확보 후 fall p5 energy와 no_motion p95/p99 비교
3. no_motion class 학습 후 confusion matrix에서 no_motion -> fall 오탐 확인
```
