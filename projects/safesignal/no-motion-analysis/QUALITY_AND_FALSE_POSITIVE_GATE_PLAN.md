# Quality Criteria and False Positive Gate Plan

_Last updated: 2026-05-25 | Updated by: codex_

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
hard skip은 금지. low-energy no_motion/static은 우선 flag/log로만 기록하고, 추후 fall 데이터 검증 후 fall threshold 상향 방식으로 검토
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

no_motion 데이터가 적은 상태에서 class를 추가하면 모델이 no_motion을 과소학습할 수 있다. 현재 정책은 E2/E3/E4 환경별 5분 x 10세션을 목표로 하되, 실제 수집 상황에 따라 내일 조정한다. E1은 우선 no_motion 미수집으로 명시하고, 이후 ESP32 추가 사용 필요성이 낮아지면 5분 x 2세션 또는 5분 x 6세션을 검토한다.

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

실제 낙상 window는 에너지가 낮을 가능성이 작지만, threshold를 너무 높게 잡으면 recall을 손상시킬 수 있다. fall 세션의 energy p5와 비교 후 결정한다. 이 단계에서는 hard skip을 적용하지 않는다.

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

margin은 selective classification / reject option 계열의 후보 조건으로 유지한다. fine-tune validation 결과로 적용 여부와 값을 조정한다. 후보값은 0.10~0.20, 초기 비교값은 0.15.

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
오탐 억제는 no_motion energy flag, raw spike metadata, consecutive confirmation, margin 후보, no_motion class 학습 조합으로 설계한다. event cooldown은 현재 단계에서 제외/보류한다.
no_motion class는 최종 학습에 반드시 포함하되, 현재 2세션은 baseline으로만 과신하지 않는다.
```

---

## 6. 확정된 Calibration 운영 정책 (2026-05-25)

### Calibration 단위와 대상

```text
단위: 환경별 calibration
우선 대상: E2, E3, E4
E1: 우선 미수집으로 명시. 상황에 따라 5분 x 2세션 또는 5분 x 6세션 추가 검토.
```

### 수집 개수

```text
목표: 환경별 5분 x 10세션
단, 실제 수집 상황에 따라 내일 조정 가능
```

### Duration

```text
개발/데모: 5분
실제 설치/최종 발표 설계: 초기 설치 시 10분 calibration 권장
```

### Baseline 의미와 저장

baseline은 해당 환경에서 no_motion/static 상태일 때의 z-score 전 SDP energy 분포 요약이다. 원본 CSI가 아니라 통계 요약값이다.

저장 정책:

```text
저장 위치: data/calibration/*.json
Git 포함: 허용
원본 no_motion CSV: Git 미포함
주의: 주소, 개인 식별 정보, 상세 방 구조는 JSON notes에 기록하지 않음
```

예상 JSON 필드:

```json
{
  "environment": "E2",
  "duration_sec": 300,
  "metric": "sdp_mean_abs",
  "p50": 0.0,
  "p95": 0.0,
  "p99": 0.0,
  "rpca_max_iter": 200,
  "window_size": 300,
  "stride": 100,
  "created_at": "..."
}
```

### Threshold 정책

```text
p95/p99 중 최종 기준은 fall 데이터 확보 후 결정
no_motion p99와 fall p5가 충분히 분리되는지 확인 전까지 추론 hard gate 금지
```

p95/p99 의미:

```text
p95: no_motion window 95%가 이 값 이하. 더 민감하지만 spike에 덜 보수적.
p99: no_motion window 99%가 이 값 이하. 더 보수적이지만 약한 움직임을 no_motion처럼 볼 위험이 있음.
```

### Low-energy 처리

```text
초기: flag/log만 남기고 event 판단에는 미적용
추후: fall 데이터 검증 후 low_energy일 때 fall threshold를 높이는 A안 검토
N 증가 방식은 우선 적용하지 않음
hard skip은 금지
```

### Margin 조건

```text
후보로 유지
fall_prob - second_prob >= margin
후보값: 0.10~0.20, 초기 비교값 0.15
최종 적용 여부와 값은 validation 후 결정
```

margin은 fall이 1등이어도 2등 클래스와 차이가 작으면 불확실한 예측으로 보는 selective classification / reject option 계열 후처리다.

### 사용성 개선

```text
우선 CLI만 개선
dashboard 연동은 후순위
```

CLI 개선 목표:

```text
긴 명령어를 직접 외우지 않고 선택형 또는 단일 진입점으로 실행
CSV 품질 검사
SDP energy 분석
baseline JSON 생성
Drive mirror 다운로드/검증은 후순위
```

---

## 7. 2026-05-25 정식 baseline/compare 결과에 따른 정책 업데이트

### 7.1 E2 no_motion baseline 확정

정식 pipeline 기준 결과:

```text
command:
python tools/safesignal_debug.py calibrate --env E2 --workers 1 --rpca-max-iter 200 --progress-every 10
python tools/safesignal_debug.py baseline-summary --env E2 --strict

source: 2 files / 596 windows
rpca_max_iter: 200

sdp_mean_abs   p50=0.02783 p95=0.04062 p99=0.04352 min=0.02387 max=0.04444
sdp_std        p50=0.03103 p95=0.06065 p99=0.06725 min=0.02207 max=0.06930
sparse_ratio   p50=0.01831 p95=0.01888 p99=0.02057 min=0.01513 max=0.02147
raw_delta_mean p50=0.50943 p95=0.54972 p99=0.56612 min=0.35535 max=0.58622
```

### 7.2 E1 fall vs E2 no_motion 비교 결과

```text
command:
python tools/safesignal_debug.py compare-energy --env E2 --target-env E1 --target-class fall --workers 1 --rpca-max-iter 200 --progress-every 10

target: 90 files / 299 windows

[sdp_mean_abs]
no_motion p99=0.04352
fall p5=0.02152
target_ratio_le_nm_p99=0.84615

[sdp_std]
no_motion p99=0.06725
fall p5=0.01839
target_ratio_le_nm_p99=0.85619

[sparse_ratio]
no_motion p99=0.02057
fall p5=0.01713
target_ratio_le_nm_p99=0.28428
```

### 7.3 결론

```text
Energy hard skip/gate 금지.
SDP pre-zscore energy는 no_motion과 fall 전체 세션 window를 분리하지 못했다.
fall CSV 전체 window에는 실제 낙상 순간이 아닌 준비/정지/전후 구간이 섞였을 가능성이 크다.
```

정책 업데이트:

```text
1. energy는 metadata/log로만 유지한다.
2. pair_dt/ts_gap도 실시간 추론 hard reject가 아니라 quality metadata/log로 유지한다.
3. no_motion 오탐 억제의 1차 해법은 no_motion class 학습이다.
4. fall 학습 품질 개선의 1차 해법은 fall event 중심 수집/trim/window 선별이다.
5. 추론 hard skip은 계속 금지한다.
```

### 7.4 수집 서버 운영 정책

수집 중 추론 RPCA 부하가 UDP 수신/페어링/대시보드 품질 측정에 영향을 줄 수 있으므로, 수집용 서버는 추론을 끄고 실행한다.

```bash
train
```

`train` alias 동작:

```text
cd /c/Project/LastProject/wifi-csi-fall-detection
SAFESIGNAL_DISABLE_INFERENCE=1
SAFESIGNAL_DRIVE_UPLOAD=1
SAFESIGNAL_DRIVE_REMOTE=gdrive:SafeSignal_Dataset
SAFESIGNAL_RCLONE_BIN=repo 내부 rclone.exe
rclone lsd gdrive:
python server/main.py
```

확인해야 할 로그:

```text
[Inference] disabled by SAFESIGNAL_DISABLE_INFERENCE=1
```

수집 중 아래 로그가 나오면 안 된다.

```text
[InferenceWorker] input_queue full
```

### 7.5 fall 수집 stage 정책

fall 6종은 event 중심 4초 프로토콜로 확정한다.

```text
FALL_SIT_F/B:  앉은 상태 대기 1초 -> 낙상 2초 -> 낙상 후 정지 1초
FALL_STD_F/B:  선 상태 대기 1초 -> 낙상 2초 -> 낙상 후 정지 1초
FALL_WALK_F/B: 1~2걸음 걷기 1초 -> 낙상 2초 -> 낙상 후 정지 1초
```

운영 주의:

```text
낙상 후 일어나는 동작은 포함하지 않는다.
준비 동작을 길게 넣지 않는다.
fall 세션은 4초로 짧게 수집한다.
```

### 7.6 네트워크 운영 메모

```text
휴대폰/인터넷 uplink 랜선은 공유기 WAN/Internet 포트에 연결한다.
LAN 1~4 포트는 PC/유선기기용이다.
노트북 Wi-Fi IPv4가 바뀌면 RX1/RX2/TX의 서버/타겟 IP를 다시 설정한다.
RX1/RX2 NTP 동기화 확인은 수집 전 필수다.
```

후속 후보:

```text
ESP32 status heartbeat UDP 패킷 추가
- RX1/RX2/TX online
- IP/channel/RSSI
- NTP synced
- uptime
- tx/rx stats
을 USB monitor 없이 대시보드에서 확인한다.
```
