# SafeSignal False Positive / Input Quality Improvement Notes

_Last updated: 2026-05-24 | Owner: codex review notes_

이 문서는 SafeSignal 낙상 오탐, empty/background 미학습, ESP32 입력 품질 문제를 별도로 추적하기 위한 개선 노트다.
코드의 진실 공급원은 `C:\Project\LastProject\wifi-csi-fall-detection` 저장소이며, 이 파일은 원인 가설과 개선 방향 정리용이다.

---

## 1. 현재 관찰된 문제

- 사람이 없거나 거의 정적인 환경에서도 `fall` 감지가 발생할 수 있다.
- 현재 모델 클래스에는 `empty`, `background`, `no_motion`, `invalid/noisy` 상태가 없다.
- 모델은 입력 window를 기존 행동 클래스 중 하나로 강제 분류하므로 OOD 입력에서 softmax fall 확률이 튈 수 있다.
- 자체수집 중 ESP32/무선 환경이 고르지 못하고, pair rate, timestamp gap, Rx1/Rx2 pair delay, 순간 spike가 불안정할 수 있음을 확인했다.
- 큰 timestamp gap이나 순간 spike가 보간/RPCA/SDP/z-score를 거치며 낙상 유사 특징으로 증폭될 가능성이 있다.

---

## 2. 원인 후보 우선순위

### 가능성 높음

1. **empty/background 클래스 부재**
   - 현재 `standing`, `lying`은 사람이 존재하는 상태의 클래스이며 무인 공간을 대체하지 못한다.
   - empty/no-motion은 별도 negative class 또는 gate로 다뤄야 한다.

2. **Rx1/Rx2 pairing 허용 오차 과대**
   - 현재 `server/utils/pairing.py`의 `PAIR_TOLERANCE_US = 50_000`은 100Hz 기준 약 5프레임 차이다.
   - 서로 다른 시점의 Rx1/Rx2를 concat하면 실제 존재하지 않는 공간 패턴이 생길 수 있다.

3. **실시간 품질 gate 부재**
   - `collect/quality.py`에는 `pair_rate_hz`, `capture_ratio`, `loss_rate`, `pair_dt`, `gap` 요약이 있다.
   - 하지만 inference 결과에는 품질 지표가 붙지 않고, `is_fall=True`면 쿨다운만 거쳐 알림으로 승격된다.

4. **RPCA / z-score가 잡음 또는 보간 artifact를 강조할 가능성**
   - RPCA sparse 성분은 움직임뿐 아니라 패킷 드롭, RSSI 변동, 보간 artifact도 강조할 수 있다.
   - window 단위 z-score는 절대 motion energy 정보를 제거하므로 작은 정적 잡음도 상대적으로 커질 수 있다.

5. **단일 window fall 판단**
   - 현재 fall 판단은 threshold 통과 window 하나로 알림까지 이어질 수 있다.
   - consecutive detection, confidence margin, no-motion gate, quality gate가 필요하다.

### 가능성 중간

1. **fall 세션 라벨 경계가 넓음**
   - 예: `앉기 2초 + 낙상 3초` 전체가 `fall` 라벨로 저장된다.
   - 낙상 직전의 앉기/서기/걷기 구간까지 fall로 학습되어 일반 동작 오탐을 키울 수 있다.

2. **Alsaify와 ESP32 SafeSignal 도메인 불일치**
   - Intel 5300 vs ESP32-S3, 90 feature vs Rx1+Rx2 104 feature, 320Hz downsample vs timestamp resampling 차이가 있다.

3. **offline/realtime resampling 구현 분리**
   - offline은 `model/preprocessing/resample.py`, realtime은 `server/inference/buffer.py`에 별도 구현되어 있다.
   - realtime 쪽은 `max_gap_ms`를 실제 reject 조건으로 쓰지 않는다.

4. **오탐 분석 로그의 window 불일치 가능성**
   - `last_fall_pair`는 최신 pair를 저장하므로 InferenceWorker 지연이 있으면 실제 fall 판단 window와 다를 수 있다.

---

## 3. 개선 방향

### A. 학습 설계

- 7-class fine-tune에 `empty` 또는 `background/no_motion`을 추가하는 8-class 설계를 검토한다.
- `empty`와 `static_person/no_motion`을 하나로 묶을지 분리할지 Claude AI와 결정한다.
- 각 환경별 empty/static 데이터를 별도 수집한다.
  - 아무도 없는 방
  - 사람이 있지만 거의 움직이지 않는 상태
  - ESP32/공유기만 켜진 상태
  - 주변 약한 움직임만 있는 상태
- Alsaify는 기본 행동 표현 유지 용도로 재사용하되, empty 오탐 억제의 핵심은 SafeSignal 자체 negative 데이터로 둔다.

### B. 입력 품질 gate

- 수집 저장 시점과 실시간 추론 시점에 같은 계열의 품질 지표를 계산한다.
- 후보 지표:
  - pair_rate_hz
  - capture_ratio
  - loss_rate
  - pair_dt p50/p95/p99/max
  - timestamp gap p95/max
  - gap_count
  - resampling 전후 sample count
- 큰 gap은 보간으로 메우기보다 window 폐기 또는 invalid 처리한다.
- `PAIR_TOLERANCE_US`는 실측 pair_dt 분포를 본 뒤 축소 검토한다.

### C. spike / outlier gate

- raw window `(300, 104)` 기준으로 추론 전 이상치를 계산한다.
- 후보 지표:
  - frame-to-frame delta energy
  - median/MAD 기반 robust z-score
  - spike frame ratio
  - spike duration
  - Rx1/Rx2 동시성
  - subcarrier consistency
  - RPCA sparse energy
  - SDP pre-zscore energy

### D. 후처리 / event-level 판단

- 단일 window fall 판단을 바로 알림으로 승격하지 않는다.
- 후보 조건:
  - quality gate 통과
  - no-motion gate 통과
  - fall threshold 통과
  - fall probability와 2등 클래스 margin 통과
  - 2~3개 연속 window fall
  - cooldown 적용
- window-level FAR와 event-level false alarm per minute/hour를 분리 평가한다.
- 빈 공간 30분 negative soak test를 별도 평가 항목으로 추가한다.

---

## 4. 바로 보완할 코드 후보

1. `server/inference/buffer.py`
   - realtime resampling 과정에서 gap/pair quality metadata를 유지한다.
   - `max_gap_ms`를 실제 reject 또는 invalid flag로 연결한다.

2. `server/inference/worker.py`
   - Inference result에 quality metadata를 포함한다.
   - 품질 불량 window는 predictor 호출 전 skip하거나 result에 `invalid_reason`을 붙인다.

3. `server/main.py`
   - `handle_inference_result()`에서 quality gate, consecutive gate, margin gate를 통과한 경우만 `on_fall_detected()` 호출한다.

4. `server/utils/pairing.py`
   - `PAIR_TOLERANCE_US`를 실측 기반으로 재검토한다.
   - pair_dt 분포를 실시간 window 품질에 전달할 방법을 마련한다.

5. `collect/labels.py`
   - `EMPTY` 또는 `BACKGROUND` 활동 코드를 추가할지 검토한다.
   - fall 세션 stage 중 실제 fall 구간만 label로 쓸 수 있는지 검토한다.

6. `model/finetune/train.py`
   - 8-class 확장 가능성 검토.
   - SafeSignal cache builder에서 empty/background와 raw-only split 정책을 반영한다.

---

## 5. Claude AI에 확인할 핵심 질문

- empty/background/no_motion을 별도 클래스로 추가해야 하는가, 아니면 no-motion gate로 충분한가?
- invalid/noisy input은 모델 클래스에 포함해야 하는가, 추론 전 quality gate로만 처리해야 하는가?
- ESP32 CSI 환경에서 spike/outlier 판별에 가장 타당한 신호 지표는 무엇인가?
- 큰 timestamp gap을 어느 기준에서 보간하지 않고 폐기해야 하는가?
- pair_dt 허용 기준은 얼마가 적절한가?
- fall 세션의 사전 자세 구간이 fall label로 들어가는 것이 오탐을 유발할 수 있는가?
- 데모 일정상 최소 구현안과 최종 발표용 clean 설계를 어떻게 나눌 것인가?

---

## 6. Pending Items

- [ ] Claude AI 원인 우선순위 답변 반영
- [ ] empty/background 수집 설계 결정
- [ ] 품질 gate threshold 초안 결정
- [ ] spike/outlier 지표 1차 구현 후보 결정
- [ ] inference event-level gate 정책 결정
- [ ] STATE.md Decisions Log 또는 Pending Items 반영 여부 결정

---

## 7. Claude AI 1차 분석 반영 (2026-05-24)

### 7.1 Claude AI 원인 우선순위

#### 가능성 높음

1. **빈 공간/정적 환경 OOD 입력**
   - 현재 모델은 `empty`, `background`, `no_motion` 상태를 학습하지 않았다.
   - softmax 구조상 빈 공간도 기존 행동 클래스 중 하나로 강제 분류된다.
   - `standing`/`lying`은 사람이 존재하는 상태이므로 무인 공간을 대체하지 못한다.

2. **Global z-score의 정적/빈 공간 노이즈 과증폭**
   - z-score는 SDP의 절대 에너지 크기를 제거한다.
   - 에너지가 매우 작은 정적 window도 미세 분산이 있으면 상대적 패턴처럼 보일 수 있다.
   - D-020의 degenerate 방어는 `std < 1e-8` 수준만 막으므로 작은 노이즈 window는 통과한다.

3. **큰 timestamp gap 보간 artifact**
   - 현재 `max_gap_ms`는 report-only에 가깝고 실시간 skip/reject로 연결되어 있지 않다.
   - 100Hz 기준 190ms gap은 약 19개 샘플 구간을 보간으로 채운다.
   - 보간 전후 불연속이 RPCA sparse 성분에서 낙상 유사 변화로 강조될 수 있다.

4. **수집 세션 contamination**
   - fall 세션의 앞 2~3초 준비 동작이 fall label로 저장된다.
   - 낙상 후 일어나는 구간도 포함될 수 있다.
   - 일반 앉기/서기/걷기 전이가 fall confidence를 높이는 방향으로 학습될 수 있다.

#### 가능성 중간

1. **Alsaify ↔ ESP32 SafeSignal 도메인 갭**
   - Intel 5300 vs ESP32-S3, 90 feature vs 104 feature, 320Hz downsample vs 70Hz→100Hz 보간 차이가 있다.
   - fine-tuning 전 pretrained 모델만 실시간 사용하면 도메인 갭이 그대로 남는다.

2. **Rx1/Rx2 pair_dt가 큰 상태의 concat**
   - `PAIR_TOLERANCE_US=50ms`는 70Hz 기준 3~4 패킷 어긋남이다.
   - 서로 다른 순간의 Rx1/Rx2를 concat하면 물리적으로 없는 feature가 생길 수 있다.

3. **single-window threshold 의존**
   - 현재는 1개 window의 `fall_prob >= threshold`가 이벤트로 이어질 수 있다.
   - temporal smoother 또는 consecutive confirmation이 없다.

#### 가능성 낮음

1. **RPCA 단독 원인**
   - 단일 패킷 spike가 300×104 전체에 미치는 영향은 제한적일 수 있다.
   - 다만 resampling artifact와 결합하면 영향이 커질 수 있다.

2. **SDP subcarrier mean 집계의 Rx별 이상 희석**
   - Rx 한쪽 이상을 희석하는 방향이라 직접 오탐 원인으로는 낮게 평가.

### 7.2 학습 설계 결정 후보

- `empty`와 `static_person`을 분리하기보다 **`no_motion` 단일 클래스 추가**가 권장된다.
- 최종 fine-tune 구조 후보:
  - `fall`, `walking`, `sit_stand`, `lying`, `standing`, `running`, `picking`, `no_motion`
- `invalid/noisy`는 학습 클래스에 포함하지 않고 quality gate로 처리한다.
- no_motion 수집은 env-subject 조합별 30세션 또는 장시간 empty recording을 window 분할하는 방식이 후보.
- Alsaify는 기본 행동 표현 유지 용도로 combined training에 계속 포함하되, `no_motion`은 SafeSignal-only 클래스가 된다.

### 7.3 입력 품질 gate 후보 threshold

#### 수집 저장 기준

| 지표 | RECOLLECT | WARN |
|---|---:|---:|
| loss_rate | >= 30% | >= 15% |
| pair_dt p50 | > 20ms | > 10ms |
| pair_dt p99 | > 100ms | > 50ms |
| gap_p95 | > 50ms | > 25ms |
| gap_max | > 500ms | > 200ms |

#### 실시간 추론 기준

- `gap_max > 200ms`: 해당 window skip.
- `gap_count > 5`: warn flag, 추론은 수행 가능.
- `pair_dt p99 > 80ms`: warn flag 또는 event 승격 차단 후보.
- 가장 중요한 단일 지표는 `gap_max`로 본다.

### 7.4 spike/outlier 판별 후보

#### raw amplitude 기준

- frame-to-frame delta energy: `np.diff(window, axis=0)` 기반.
- spike frame ratio: `delta_energy > median + 5*MAD` frame 비율.
- robust z-score: `abs((x - median) / (1.4826 * MAD)) > 8` frame 탐지.

#### RPCA / SDP 기준

- RPCA sparse energy ratio: `||S||_F / ||M||_F`.
- SDP pre-zscore energy: `mean(abs(sdp))` 또는 `||SDP||_F`.
- no_motion 판단에는 z-score 이후 입력이 아니라 **z-score 전 energy**를 사용해야 한다.

#### Rx / subcarrier consistency

- Rx1-Rx2 correlation이 낮으면 한쪽 Rx artifact 가능성.
- 변화가 일부 subcarrier에만 몰리면 하드웨어/무선 spike 가능성.
- threshold는 실측 데이터로 결정해야 한다.

### 7.5 전처리 수정 핵심

- 가장 즉각적인 방어책은 **SDP energy gate**.
- `window_to_model_input()`에서 z-score 직전 `sdp_energy = mean(abs(sdp))`를 계산한다.
- `sdp_energy < SDP_ENERGY_THRESHOLD`면 추론 skip 또는 no_motion 처리.
- threshold는 no_motion 세션의 SDP energy p95와 fall 세션 p5 사이에서 정한다.
- RPCA 자체는 현재 degenerate 방어 유지. 추가로 sparse energy ratio를 metadata로 로깅하는 것을 권장.

### 7.6 후처리 / event-level 정책

- validation 기준으로 `FALL_THRESHOLD`를 선택한다.
- 초기 데모 방어로는 `FALL_THRESHOLD=0.65` 임시 상향이 후보.
- stride=1초 기준 consecutive confirmation:
  - N=2: 데모 권장. 응답 지연 +1초.
  - N=3: 오탐 억제는 강하지만 지연 증가.
- margin 조건 후보: `fall_prob - second_prob > 0.15`.
- cooldown은 30~60초.
- 평가 지표는 window-level FAR와 event-level FAR를 분리한다.
- 빈 공간 30분 soak test에서 false alarm count를 별도 측정한다.

### 7.7 데모용 최소 구현안

재학습 없이 적용 가능한 순서:

1. **SDP energy gate**
   - z-score 전 SDP energy가 너무 낮으면 skip/no_motion.
2. **N=2 consecutive window confirmation**
   - `server/main.py` 또는 `server/inference/worker.py`에 event detector 추가.
3. **gap_max window skip**
   - `server/inference/buffer.py`에서 window 내 max gap을 보존하고 200ms 초과 시 skip.
4. **FALL_THRESHOLD 임시 상향**
   - `0.5 -> 0.65` 후보. recall 하락 리스크는 있음.

### 7.8 최종 구조 후보

#### demo pragmatic version

```text
패킷 수신
→ SlidingWindowBuffer
→ gap_max check (>200ms skip)
→ RPCA → ACF → SDP
→ SDP energy gate
→ Global z-score
→ pretrained CNN-GRU-Attention
→ fall_prob >= 0.65 and margin >= 0.15
→ N=2 consecutive + cooldown
→ Pi4/SMS alert
```

#### final clean version

```text
Alsaify 6-class + SafeSignal 8-class(no_motion 포함)
→ combined training
→ full unfreeze + 5 epoch warmup
→ 3-fold cross-subject validation
→ global threshold selection
→ realtime gap gate + SDP energy gate
→ fine-tuned 8-class model
→ no_motion probability suppresses fall
→ N=2 consecutive + cooldown
→ event-level FAR soak test
```

### 7.9 Codex 추가 판단

Claude AI 분석은 현재 Codex 검토와 대부분 정렬된다. 차이가 있는 부분은 다음과 같다.

- Claude AI는 RPCA 단독 원인을 낮게 평가했지만, Codex는 RPCA와 z-score/gap artifact의 결합 원인은 여전히 높은 실무 리스크로 본다.
- Claude AI는 수집 저장 기준을 기존 UI 기준보다 완화할 것을 제안했다. 다만 실시간 추론에서는 `gap_max > 200ms`를 강하게 skip하는 방향이 합리적이다.
- `no_motion` 단일 클래스 제안은 데이터 수집 부담과 모델 단순성 측면에서 타당하다.
- 우선 구현은 `gap_max skip` + `SDP pre-zscore energy gate` + `N=2 consecutive` 조합이 가장 비용 대비 효과가 크다.

### 7.10 Pending Updates

- [ ] `server/inference/buffer.py`가 window별 gap metadata를 보존하도록 설계.
- [ ] `window_to_model_input()`에서 SDP pre-zscore energy를 반환하거나 별도 helper로 노출하는 방안 결정.
- [ ] `server/main.py`에 N=2 consecutive event detector를 둘지, worker 내부에 둘지 결정.
- [ ] `no_motion` class 추가를 STATE.md Decision으로 확정할지 사용자/Claude AI와 합의.
- [ ] empty/no_motion 수집 절차를 `collect/labels.py`에 반영할지 결정.

---

## 8. 공유기 확보 이후 gate 정책 재평가 (2026-05-24)

### 8.1 새 관찰

- 공유기 확보 이후 수집 pair rate가 100Hz에 가깝게 안정화되었다.
- rate가 낮아져도 90Hz 아래로는 거의 내려가지 않는 상황이다.
- 따라서 Windows hotspot 환경에서 확인된 70Hz 천장과 큰 보간 artifact 리스크는 이전보다 낮아졌다.
- D-018/D-026의 리샘플 필요성은 재평가 대상이다. 다만 timestamp 기반 100Hz 정렬 자체는 유지하되, 실제로는 거의 identity resampling에 가까울 가능성이 높다.

### 8.2 추가 실측 필요

공유기 환경에서 다음 분포를 새로 산출해야 한다.

- pair_rate_hz
- capture_ratio
- gap_count
- gap_p95_us
- gap_max_us
- pair_dt p50/p95/p99/max
- loss_rate

공유기 환경에서 `gap_max <= 30ms` 수준으로 안정된다면, 보간 artifact는 주요 원인 후보에서 우선순위를 낮춘다.

### 8.3 중요한 설계 정정: 수집 gate와 추론 gate 분리

사용자 우려:

> 실제 낙상이 일어났는데 ESP32 페어링/수신 상태 때문에 window가 제외되면, 평가 recall은 좋아져도 실용성은 약해지는 것 아닌가?

결론: 맞는 우려다. 수집(training data) gate와 실시간 inference gate는 목적이 다르므로 같은 기준으로 hard reject하면 안 된다.

#### 수집 gate

목적: 학습 데이터 품질 보증.

- 품질 불량 세션은 artifact를 모델이 학습하지 않도록 재수집/제외하는 것이 맞다.
- gap_max, loss_rate, pair_dt 기준을 비교적 엄격하게 적용 가능하다.
- 예: gap_max > 200ms, loss_rate > 30% 등은 재수집 후보.

#### 실시간 추론 gate

목적: 실제 안전 시스템에서 낙상 미탐을 최소화하면서 오탐을 줄이는 것.

- gap_max 또는 pair_dt 불량만으로 window를 hard skip하면 실제 낙상을 놓칠 수 있다.
- 실시간에서는 품질 불량을 `quality_flag=True`로 기록하고 추론은 수행하는 것이 더 안전하다.
- fall event 발생 시 quality_flag를 함께 로그에 남겨 사후 분석에 활용한다.

### 8.4 gate별 정책 정리

| gate | 수집 단계 | 실시간 추론 단계 | 비고 |
|---|---|---|---|
| pair_rate / capture_ratio | WARN/RECOLLECT 가능 | flag only | 낮은 rate만으로 실제 낙상 skip 금지 |
| gap_max | 엄격 적용 가능 | flag only 권장 | 아주 극단적인 경우만 별도 검토 |
| pair_dt | 엄격 적용 가능 | flag only 권장 | Rx concat 신뢰도 metadata로 사용 |
| SDP pre-zscore energy | no_motion 분석용 | skip/no_motion 가능 | 실제 낙상 차단 가능성 낮음 |
| no_motion class probability | 학습 후 평가 | fall suppression 가능 | final clean 구조에서 사용 |
| consecutive fall | 해당 없음 | alert 승격 조건 | skip이 아니라 event confirmation |

### 8.5 실시간 hard skip 후보 재정의

기존 후보였던 `gap_max > 200ms -> skip`은 실시간 안전성 관점에서 `quality_flag=True`로 완화한다.

실시간에서 hard skip 또는 no-alert 처리가 정당한 후보는 다음으로 제한한다.

1. **SDP pre-zscore energy가 no_motion threshold 미만**
   - window 내 변화량이 empty/static 수준이면 모델 추론 또는 alert 승격을 막는다.
   - 실제 낙상 window가 이 기준을 통과하지 못할 가능성은 낮다.

2. **명백한 packet/parser invalid**
   - packet shape/length/magic/device_id 오류처럼 모델 입력 자체가 성립하지 않는 경우.

그 외 gap/pair_dt/loss 관련 품질 문제는 우선 `quality_warn`으로 남기고, alert를 막지는 않는다.

### 8.6 Codex 판단 업데이트

- 공유기 안정화 이후에는 보간 artifact보다 **empty/no_motion 미학습**, **z-score 에너지 소실**, **single-window alert 승격**, **라벨 contamination**의 우선순위가 더 높아진다.
- 실시간 추론에서 품질 gate를 hard skip으로 설계하면 safety recall을 해칠 수 있으므로, quality metadata logging + event confirmation 방식이 더 적절하다.
- 데모용 최소 구현 순서도 조정한다.
  1. `N=2 consecutive window confirmation`
  2. `SDP pre-zscore energy/no_motion gate`
  3. `quality metadata logging` with gap/pair_dt flag
  4. threshold/margin 조정
- `gap_max skip`은 수집/학습 데이터 정제에서는 유지 후보지만, 실시간 알림 차단 조건에서는 제외한다.

### 8.7 Pending Updates

- [ ] 공유기 환경 수집 CSV의 gap/pair_dt/rate 분포 재측정.
- [ ] D-018 리샘플 정책을 공유기 환경 기준으로 재평가.
- [ ] 실시간 `gap_max skip` 대신 `quality_flag` 로깅 구조 설계.
- [ ] SDP energy gate threshold 산정을 위한 no_motion baseline 수집.
- [ ] 데모용 event detector 우선순위 재정렬: consecutive 먼저, hard quality skip 제외.

---

## 9. User Planning Decisions (2026-05-24)

### 9.1 확정된 방향

1. **`no_motion` 클래스 추가 확정**
   - `empty`, `static_person`을 분리하지 않고 단일 `no_motion` 클래스로 시작한다.
   - fine-tune 클래스는 기존 7-class + `no_motion` = 8-class 방향으로 간다.

2. **E2 no_motion baseline 수집 확정**
   - 형식: 5분 × 2세션.
   - 목적:
     - 공유기 환경 pair_rate / pair_dt / ts_gap / loss 분포 확인.
     - SDP pre-zscore energy baseline 산정.
     - pretrained 모델의 empty/no_motion false alarm 빈도 확인.

3. **실시간 품질 gate 정책 확정 방향**
   - gap/pair_dt/loss 기반 hard skip은 하지 않는다.
   - 실시간에서는 quality_flag/logging 중심으로 간다.
   - 사용자 설치/실사용 시점에 5~10분 no_motion baseline calibration 절차를 manual로 문서화하는 방향을 검토한다.

4. **수집 데이터 품질 기준**
   - 최종 WARN/RECOLLECT threshold는 E2 baseline 측정 후 논의한다.

5. **데모용 방어책 우선순위**
   - 전반적으로 동의.
   - 실제 코드 작업 시 `consecutive`, `SDP energy gate`, `quality metadata logging`, threshold/margin을 다시 세부 논의한다.

### 9.2 보류 / 추가 논의 필요

1. **E1 이미 수집 완료 데이터 처리**
   - E1은 재수집이 불가능하므로, fall 세션 라벨 contamination 대응을 재수집 없이 어떻게 완화할지 논의 필요.

2. **pair_dt / ts_gap 개선 접근**
   - baseline 측정 결과를 보고 고정 offset / 랜덤 지터 / drift / raw gap vs paired gap을 분리 분석한다.

3. **추론 모델 경로 전환 원칙**
   - fine-tune 모델 생성 후 inference config가 pretrained best.pt를 계속 보지 않도록 하는 문제는 별도 심층 논의가 필요하다.

### 9.3 Codex follow-up notes

- `no_motion` 클래스 추가는 running 추가와 유사하게 head migration이 필요하지만, 단순 행 추가만으로 끝나지 않는다.
  - 기존 pretrained 6-class: `fall, walking, sit_stand, lying, standing, picking`
  - 기존 fine-tune 7-class: `fall, walking, sit_stand, lying, standing, running, picking`
  - 신규 fine-tune 8-class 후보: `fall, walking, sit_stand, lying, standing, running, picking, no_motion`
  - migration 정책:
    - old[0:5] -> new[0:5]
    - running(new[5]) 새 초기화
    - picking(old[5]) -> new[6]
    - no_motion(new[7]) 새 초기화
- `no_motion`은 Alsaify에 없는 클래스이므로 SafeSignal-only class로 취급한다.
- E2 baseline 수집을 하려면 `collect/labels.py`와 대시보드 label list에 `NO_MOTION` 활동 코드 추가가 필요할 가능성이 높다.
- 긴 2분 세션은 기존 3~8초 activity duration 구조와 다르므로 collection UI/manager가 duration을 그대로 처리할 수 있는지 확인해야 한다.


---

## 10. Implementation Update - NO_MOTION Collection Label (2026-05-24)

- Branch: `codex/no-motion-baseline`
- Implemented collection-only `NO_MOTION` activity for E2 baseline collection.
- Scope intentionally limited to collection labels/dashboard/self-check; model fine-tune 8-class migration is not included in this step.

### Changed Files

- `collect/labels.py`
  - Added `CLASS_NAMES[7] = "no_motion"`.
  - Added `ACTIVITY_INFO["NO_MOTION"]`:
    - display: `무동작/빈 공간`
    - class_idx: `7`
    - target: `2`
    - duration: `300`
  - Updated collection target documentation from 240 to 242 sessions.

- `server/dashboard/templates/index.html`
  - Added `no_motion` to the collection table class-name display array.

- `collect/_selfcheck.py`
  - Updated label expectations to 242 sessions and 7 non-fall activity codes including NO_MOTION.
  - Added `NO_MOTION` target/class/duration assertions.

- `collect/collect_main.py`
  - Updated CLI docstring collection target from 240 to 242 sessions.

### Verification

- `python -m py_compile collect\labels.py collect\_selfcheck.py server\dashboard\app.py` passed.
- Label-only import assertion passed:
  - `CLASS_NAMES[7] == "no_motion"`
  - `ACTIVITY_INFO["NO_MOTION"]["target"] == 2`
  - `get_duration("NO_MOTION") == 300`
  - `total_target_sessions() == 242`
- Full `python collect\_selfcheck.py` could not run in the current shell because `pandas` is not installed in this Python environment. This is an environment dependency issue, not a label-code failure.
