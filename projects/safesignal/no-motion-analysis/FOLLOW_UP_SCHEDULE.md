# SafeSignal no_motion / energy baseline 후속 일정

_Last updated: 2026-05-25 | Updated by: codex_

## 목적

정적/빈 공간 오탐 문제를 no_motion baseline과 fall energy 비교로 먼저 검증한 뒤, 그 결과를 근거로 모델/추론 수정을 진행한다. 현재 단계에서는 추론 hard skip, threshold, margin, cooldown을 코드에 적용하지 않는다.

## 진행 순서

### 1. E2 no_motion baseline 확정

진행 중인 명령이 완료될 때까지 대기한다.

```bash
python tools/safesignal_debug.py calibrate --env E2 --workers 1 --rpca-max-iter 200 --progress-every 10
```

완료 후 검증한다.

```bash
python tools/safesignal_debug.py baseline-summary --env E2 --strict
```

확인 항목:

- `data/calibration/E2_no_motion_baseline.json` 생성 여부
- `n_files >= 2`
- `n_windows > 0`
- `rpca_max_iter >= 100`
- `sdp_mean_abs`, `sdp_std`, `sparse_ratio`, `raw_delta_mean` 포함 여부

### 2. E1 fall 대비 energy 비교

E1 fall CSV는 Google Drive `SafeSignal_Dataset/E1/S01`에서 로컬 `data/raw/`로 90개 다운로드 완료.

E2 no_motion baseline 생성 후 아래 명령으로 비교한다.

```bash
python tools/safesignal_debug.py compare-energy --env E2 --target-env E1 --target-class fall --workers 1 --rpca-max-iter 200 --progress-every 10
```

판단 기준:

- `no_motion sdp_mean_abs p99`와 `fall sdp_mean_abs p5`의 간격 확인
- `fall p5 > no_motion p99`이면 sampled data 기준 분리 가능성이 있음
- 겹치면 energy 단독 hard gate 금지, metadata/flag 또는 보조 조건으로만 유지

### 3. 내일 수집 전 장비 점검

수집 직전 확인한다.

- RX1 DevID `0x01`, RX2 DevID `0x02`, TX DevID `0x00`
- RX1/RX2/TX `show_config`의 서버/타겟 IP가 노트북 Wi-Fi IPv4와 일치
- RX1/RX2 모두 `NTP 동기화 완료`
- 대시보드에서 30초 이상 품질 확인
  - pair rate 90Hz 이상 권장
  - loss 5% 이하 권장
  - pair_dt / ts_gap이 비정상적으로 튀지 않는지 확인

### 4. 내일 수집 우선순위

우선순위:

1. E2/E3/E4 no_motion baseline 수집
2. 부족한 일반 동작 수집
3. 부족한 fall 세션 보강
4. E1 no_motion은 시간과 장비 여유가 있으면 진행

no_motion 수집 계획:

- 충분 조건: 환경별 5분 x 10세션
- 시간이 부족하면 우선 5분 x 2세션으로 시작
- 수집 후 환경별 baseline 생성 및 summary 검증

예:

```bash
python tools/safesignal_debug.py calibrate --env E3 --workers 1 --rpca-max-iter 200 --progress-every 10
python tools/safesignal_debug.py baseline-summary --env E3 --strict
```

### 5. 환경별 baseline / fall 비교 후 결정할 것

- E2/E3/E4 no_motion baseline 차이가 큰지
- 환경별 threshold가 필요한지
- `sdp_mean_abs`가 안정적인 1차 지표인지
- `sdp_std`, `sparse_ratio`, `raw_delta_mean`이 보조 지표로 의미 있는지
- energy를 추론 결과 metadata로만 둘지, fall confidence 보조 조건으로 쓸지
- hard skip은 계속 금지할지 재확인

현재 기본 방침:

- pair_dt / ts_gap은 hard reject가 아니라 quality metadata/log로 유지
- energy도 fall p5 비교 전에는 hard gate로 적용하지 않음
- event cooldown은 오탐 원인 해결책이 아니므로 현재 개선 범위에서 제외

### 6. 모델 작업 진입 조건

baseline/compare-energy 결과를 확보한 뒤 모델 코드를 수정한다.

후보 작업:

- no_motion class 포함 fine-tune class map 확정
- 기존 pretrained checkpoint head 확장 / migration 점검
- SafeSignal-only class 처리
- Alsaify + SafeSignal combined training에서 class mask/weight 정책 정리
- validation metric에 no_motion false alarm rate 추가
- side-fall 제외 정책과 excluded file count 로깅 확인

사전학습 재수행은 현재 우선순위에서 제외한다. 기존 pretrained checkpoint를 유지하고, SafeSignal 자체 데이터와 no_motion class를 포함한 fine-tune을 우선한다.

### 7. 학습 후 평가

평가 순서:

1. window-level metric
2. no_motion false alarm rate
3. fall recall
4. class confusion matrix
5. event-level false alarm

### 8. 추론 적용

모델/분석 결과가 나온 뒤 마지막에 적용한다.

후보:

- `window_to_model_input()`에서 z-score 전 energy metadata 노출
- `FallPredictor.predict()` 결과에 energy/quality metadata 포함
- N=2 consecutive confirmation 독립 적용 검토
- threshold/margin 후보 검증
- hard skip은 현재 금지

## 현재 결론

현재 순서는 다음으로 고정한다.

```text
baseline 분석 -> 내일 수집 -> 환경별 baseline -> fall 비교 -> 모델 수정 -> 학습/평가 -> 추론 보조 로직
```
