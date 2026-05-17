# SafeSignal Project State

_Last updated: 2026-05-17 (RPCA degenerate 입력 방어 추가) | Updated by: claude-code_

---

## Project Overview

- **프로젝트명:** SafeSignal
- **팀:** MATE
- **목표:** WiFi CSI 기반 낙상 감지 및 보호자 알림 시스템
- **타겟:** 독거 노인 가구 (카메라 미사용, 프라이버시 보호)
- **Demo:** 2026-06-04
- **Final Presentation:** 2026-06-11
- **GitHub:** https://github.com/LeapSeeker/wifi-csi-fall-detection

---

## Decisions Log

### [D-001] 핵심 기능 범위 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** 낙상 감지 + 보호자 알림으로 범위 확정. STT/LLM 미사용. TTS는 사전 녹음 음성 파일 생성 용도로만 1회 사용 (라이브 파이프라인 없음). 다중 인원 감지는 Out of Scope.
- **Status:** confirmed

### [D-002] 하드웨어 구성 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** ESP32-S3 3대 (Tx 1 + Rx 2), 외장 헬리컬 안테나, 배터리 전원. 추론 서버: 팀원 노트북 (RTX 4060, 64GB RAM). I/O 허브: Raspberry Pi 4. Pi4 전원: 유선 어댑터 (잠정 확정).
- **Status:** confirmed

### [D-003] 신호 처리 파이프라인 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** RPCA → ACF → SDP 파이프라인 확정. STFT 미사용. ESP32 위상 데이터 불안정 (CFO/STO 문제) → 진폭만 사용: sqrt(I²+Q²). 샘플링: 100Hz.
- **Ref:** Armenta-Garcia et al. (Sensors 2025, PMC12526573)
- **Status:** confirmed

### [D-004] 전처리 파라미터 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:**
  - 샘플링: 100Hz, 윈도우 3s = 300 packets
  - RPCA: λ = 1/√max(N_T, N_S)
  - ACF: NΔ = 20
  - SDP: subwindow W=30, stride=10, W_T=28
  - 다운샘플링 (320Hz→100Hz): resample_poly(up=5, down=16), Kaiser FIR
  - 최종 모델 입력 shape: (N, 1, 28, 20)
- **Status:** confirmed

### [D-005] 모델 아키텍처 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** CNN + GRU + Attention (Temporal Attention) 확정. CNN-LSTM 아님. Attention은 필수 구성요소.
- **Ref:** XFall (IEEE JSAC 2024)
- **Status:** confirmed

### [D-006] 사전학습 데이터 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** Alsaify LOS (E1+E2) 사용. UT-HAR 제외. NLOS (E3) 제외 (향후 확장 가능성만 보존).
  - Intel 5300, 90 subcarriers, 320Hz, 30 subjects × 5 experiments × 20 trials
  - 클래스 매핑: fall(A2+A5)=800, walking(A6+A8)=800, sit_stand(A10+A11)=800, lying(A3,C1+C2)=800, standing(A4,C2+C4)=800, picking(A12)=400 → 총 4,400 samples
  - running 클래스: Alsaify 해당 없음 → fine-tuning 전용
- **Ref:** https://data.mendeley.com/datasets/v38wjmz6f6/1 / https://doi.org/10.1016/j.dib.2020.106534
- **Status:** confirmed

### [D-007] UDP 패킷 구조 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** 224B 패킷. `[magic(1B,0xAB) | device_id(1B) | rssi(1B,int8_t) | reserved(1B) | seq(4B) | timestamp_us(8B) | amplitude×52(208B)]`. Rx1=0x01, Rx2=0x02, Tx=0x00, port 5005. Rx1/Rx2 SNTP 동기화, 고정 IP.
- **Status:** confirmed

### [D-008] 서버-Pi4 통신 방식 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** 추론 서버 ↔ Pi4: WebSocket (Pi4 outbound), JSON 포맷 (event/class/confidence/timestamp_us).
- **Status:** confirmed

### [D-009] 보호자 알림 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** SOLAPI SMS 확정 (동석 담당). 자동+수동 이중 전송 원칙 합의. Pi4 하드웨어 버튼 2개: ① 알람 취소, ② 사전 등록 번호 긴급 SMS.
- **Status:** confirmed

### [D-010] 자체 수집 목표 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:** 240 세션 목표.
  - 낙상: 앉다→낙상(앞/뒤/옆 10회씩=30), 서다→낙상(앞/뒤/옆 10회씩=30)
  - 비낙상: walking/sit-stand/lying/standing/running(빠른걷기)/picking 각 30세션
  - 증강: 5× (Jittering + Scaling + TimeWarping + NoiseScale) → ~1,200 samples
- **Status:** confirmed

### [D-011] 성능 목표 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:**
  - Fall Recall: MVG ≥85% / Stretch ≥90% (1차 지표)
  - FAR: ≤15% (제약)
  - F1: ≥0.85 (균형 확인)
  - Accuracy: 목표 지표 아님 (불균형 데이터에서 misleading)
  - 시스템 응답: ≤1.5s, 사용자 체감 지연: ≤3s, SMS API: ≤1s
- **Status:** confirmed

### [D-012] 기술 스택 확정
- **Date:** 2026-05-09
- **Decided by:** claude-ai
- **Content:**
  - ESP32-S3 펌웨어: ESP-IDF v5 + esp-csi (C), LLTF only, 52 subcarriers
  - Python: 3.11.9, PyTorch 2.6.0+cu124
  - 로컬 GPU: MX450 (2GB VRAM, 코드 검증용) / 학습: RTX 4060
- **Status:** confirmed

### [D-013] Rx1/Rx2 진폭 결합 방식 확정
- **Date:** 2026-05-11
- **Decided by:** claude-ai
- **Content:** Rx1(52 sc)과 Rx2(52 sc)를 서브캐리어 축으로 concatenate하여 (300, 104) 단일 행렬로 구성. SDP 단계에서 서브캐리어 축을 mean 집계하므로 최종 모델 입력 (1, 28, 20)은 n_sc 변화에 무관하게 유지됨. 두 수신기의 공간 다양성(spatial diversity) 정보를 손실 없이 보존하기 위해 average(b안) 대신 concatenate(a안) 채택.
- **Status:** confirmed

### [D-014] 실시간 추론 슬라이딩 윈도우 stride 확정
- **Date:** 2026-05-11
- **Decided by:** claude-ai
- **Content:** 기본 stride=100 (1초 간격 추론). stride=300은 시스템 응답 목표(≤1.5s)와 충돌 가능성, stride=30은 RPCA 연산 부하 미검증으로 기각. RTX4060에서 RPCA 단독 벤치마크(단일 윈도우 latency) 실측 후 조정. stride 값은 설정 파일 상수로 분리하여 코드 수정 없이 변경 가능하도록 구현.
- **Pending benchmark:** RTX4060에서 window_to_model_input() 단일 호출 latency 측정 후 stride 재검토. 목표: 추론 1회 < stride 구간(stride=100이면 1s 이내).
- **Status:** confirmed (stride 값은 실측 후 조정 예정)

### [D-015] 추론 모듈 구조 확정
- **Date:** 2026-05-11
- **Decided by:** claude-ai
- **Content:**
  - 추론 전용 별도 프로세스로 분리. on_paired(rx1, rx2) 콜백은 페어 데이터를 Queue에 넣고 즉시 반환(논블로킹). InferenceWorker 프로세스가 Queue에서 꺼내 슬라이딩 윈도우 버퍼 관리 → RPCA → ACF → SDP → 모델 추론 → fall 시 결과 반환.
  - server/dongseok 브랜치와 feature/pretrained-model 브랜치를 main으로 통합한 후 inference/ 서브모듈 신규 추가.
  - 플로우: on_paired → Queue.put(pair) / InferenceWorker: Queue.get() → 윈도우 버퍼 → stride 조건 충족 시 전처리+추론 → fall이면 on_fall_detected() 호출.
- **Status:** confirmed

### [D-016] 서버→Pi4 timestamp_us 기준 확정
- **Date:** 2026-05-11
- **Decided by:** claude-ai / codex review
- **Content:** 서버→Pi4 WebSocket 낙상 알림의 `timestamp_us`는 낙상으로 판정된 윈도우의 기준 Rx1 패킷 `timestamp_us`로 확정. 실험/성능 측정 및 서버 fall history와 같은 기준점을 유지하기 위함. 값이 0이거나 누락된 경우에만 서버 현재 시각(Unix μs)을 fallback으로 사용.
- **Status:** confirmed

### [D-017] 자체수집 무선 환경 + WIFI_PS_NONE 확정
- **Date:** 2026-05-11
- **Decided by:** user / claude-code (진단 결과)
- **Content:**
  - 자체 수집 환경: 일반 무선 공유기 미보유 → **휴대폰 핫스팟** 사용 (잠정). Windows 모바일 핫스팟(192.168.137.x ICS 대역)에서 broadcast forwarding이 매우 불안정, DTIM=3(307ms) + ESP-IDF 기본 `WIFI_PS_MIN_MODEM` 조합으로 air에 100Hz가 burst→idle 패턴으로만 나타남.
  - TX/RX1/RX2 모든 펌웨어 `wifi_init()` 끝에 `ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));` 추가. 부팅 로그에서 `pm start, type: 0` 확인.
  - 3순위 fix (TX target `255.255.255.255` → 노트북 LAN IP `192.168.137.1` 직접 unicast)는 별도 자문 후 결정. RX는 promiscuous + `channel_filter_en=false`라 unicast 패킷도 air sniff 가능하나 무선 air-time 특성이 broadcast와 다를 수 있어 실측 필요.
- **Status:** confirmed (2026-05-12 검증 완료 — PS_NONE + unicast 전환 모두 적용. broadcast 3Hz → unicast ~70Hz로 20배 개선. 잔여 천장은 [D-018]에서 처리)

### [D-018] 자체수집 데이터 100Hz 리샘플링 적용
- **Date:** 2026-05-12
- **Decided by:** user / claude-code (진단 결과)
- **Content:**
  - 자체수집은 Windows 모바일 핫스팟 환경에서 페어 rate ~70Hz 천장. STAND/WALK 활동 무관, 거리·안테나(1m 밀집 + 정렬)·채널(11→1) 변경 모두 ±5/s 안(세션 변동 범위). 펌웨어 STATS 로그상 `cb≈125~150/s, match≈60~75/s, sent==match, qfull=0, fail=0` — 큐/UDP/페어링 무손실, ~30% TX 프레임이 RX 라디오 demod 단계 이전에 손실. 약 5~7초 주기의 cb dip(절반 수준)이 채널 11/1 동일 발생 → Windows 모바일 핫스팟 스택의 주기 백그라운드 작업으로 추정. 펌웨어/채널 변경으로 회피 불가.
  - 결정: self-collected CSV는 **전처리 로더 단계에서 timestamp 기반 100Hz 균일 격자 보간 리샘플** 적용. scipy `interp1d` 선형 보간, 캡처 범위 안쪽만 채움(외삽 없음). Alsaify는 이미 100Hz라 미적용.
  - 적용 위치: `model/preprocessing/loader.py` (또는 self-collected 전용 로더). 원본 CSV는 그대로 두고 로딩 단계에서만 변환. 실시간 추론 측은 별도 검토 — 페어 incoming 시점에 동일 보간 적용 필요(stride 누적 + 시점 보간).
  - 가시화 보강: `collect/collect_main.py` `_run_session` 출력에 `pair_rate` / `capture_ratio` 추가, 운용 중 천장 변동 즉시 확인.
  - 라우터 확보 시 리샘플 필요성 재평가 — Pending Items에 등록.
- **Status:** confirmed (2026-05-13 구현 완료 — `f25f018 SafeSignal 자체수집 100Hz 리샘플 전처리 추가`)

### [D-019] 자체수집 평가 분할 + 베이스라인 측정 흐름 확정
- **Date:** 2026-05-14
- **Decided by:** user / claude-code
- **Content:**
  - **분할:** 자체수집 240세션은 **subject 단위 2명 train + 1명 test 봉인** 분할. test 1명의 raw 데이터는 학습/증강/모델 선정/하이퍼파라미터 튜닝/threshold 결정에 일체 사용 금지. 어떤 subject(S01/S02/S03)를 test로 봉인할지는 **자체수집 본격 진입 시점에 사전 지정** (수집 종료 후 결정 금지 — 진행 중 leakage 차단 보장 불가).
  - **흐름:** 3명 수집 완료 → 봉인 test로 현 best.pt 베이스라인 평가(6-class 기준) → train split만 증강(5×, D-010) → 파인튜닝(7-class) → 동일 test로 비교 평가. 베이스라인 vs 파인튜닝 개선폭은 6-class 교집합으로 보고하고 7-class 전체 metrics는 보조 지표.
  - **증강 적용 범위:** 학습 파이프라인의 train split 내부에서만 호출. test/val 에는 raw 그대로. `augment/augment.py` 호출 위치가 split 이후인지 코드 점검 항목으로 추가.
  - **running 클래스:** 사전학습 best.pt는 6-class (running 없음, D-006) → 베이스라인 평가에서 자체수집의 running 세션은 제외하고 6-class만 채점. 파인튜닝 모델은 7-class로 학습·평가.
  - **fall 수집 우선순위:** D-011 핵심 지표가 fall_recall이고 fall 세션 부족 시 평가 자체가 무의미 → **수집 일정상 240세션 전부 못 채워도 fall 60(앉다→낙 30 + 서다→낙 30) 우선, walking/sit_stand 차순위, picking/lying 최후순위**.
  - **베이스라인 평가 산출 항목 (필수):** 클래스별 confusion matrix, FALL_THRESHOLD sweep(0.3~0.7), subject별 metrics. 파인튜닝 설계 입력으로 사용.
- **Status:** **임시 확정 (추가 논의 필요)** — 자체수집 본격 진입 전 재검토 결정. 검토 트리거:
  - **검토 사유:** test=1명 구성의 통계적 검정력 문제(fall ~20세션 기준 recall 95% CI ≈ ±10~15%p → 베이스라인 vs 파인튜닝 개선폭 유의성 판단 어려움), test 1명 개인 편향, 학습 데이터 33% 손실로 파인튜닝 효과 제한, fall stratification 미보장, 봉인 대상자 일정 지연 시 평가 자체 risk.
  - **재논의 후보:** (a) **3-fold cross subject** — fold마다 1명 test, 학습 3회로 variance↓·데이터 손실 0·CI 폭 ≈ ±5~8%p로 축소. RTX4060 30 epoch 약 30분 가정 시 추가 60분 비용. (b) 2+1 봉인 유지하되 test subject 사전 지정 + 동일 환경/시간대/장비 보장으로 environment confound 차단. (c) 자체수집 양 자체를 240→그 이상으로 확대해 1명 봉인 규모를 키움.
  - **결정 시점:** 자체수집 3명 합류·본격 수집 진입 직전. 그 시점에 (a)/(b)/(c) 중 채택안 결정 후 본 항목 status 갱신.
  - **그 외 항목(running 제외, 증강 train-only, fall 우선순위, 베이스라인 산출 항목)은 분할 방식과 독립이라 그대로 유효.**

### [D-020] SDP z-score 정규화 적용 및 ablation 계획
- **Date:** 2026-05-17
- **Decided by:** user / codex review
- **Content:**
  - 현재 SDP 출력 `(28, 20)`에 window 단위 z-score 정규화를 추가하기로 결정.
  - 1차 구현/적용안은 **A안 Global z-score**: `mean=sdp.mean()`, `std=sdp.std()`, `sdp=(sdp-mean)/(std+eps)`. A안은 ACF lag 간 상대 크기/decay 구조를 보존하면서 개인 간 진폭 편차와 Intel 5300 ↔ ESP32 scale gap을 완화하는 안정적 기본안으로 채택.
  - **B안 Per-lag z-score**: `mean=sdp.mean(axis=0, keepdims=True)`, `std=sdp.std(axis=0, keepdims=True)`. B안은 각 lag별 시간축 변화량을 강조해 SafeSignal의 "낙상 순간 급격한 변화" 설명과 잘 맞지만, high-lag noise 과증폭 및 FAR 증가 가능성이 있어 ablation 후보로 유지.
  - fine-tuning 진입 후 먼저 A안으로 학습한 모델의 평가 지표를 확보한 뒤, **동일 split**에서 B안 전처리로 재학습하여 지표를 비교하고 최종 정규화 방식을 결정.
  - A/B 비교 시 `fall_recall`, `FAR`, `fall_f1`, confusion matrix, 특히 `walking/picking/sit_stand → fall` 오탐을 함께 확인. B안 적용 시 `std_floor`와 clipping(`[-5,5]` 또는 `[-3,3]`) 적용을 검토.
- **Status:** A안 Global z-score 구현 완료 (main `14bfb12` — 2026-05-17). `window_to_model_input()` SDP 직후에 `(sdp - sdp.mean()) / (sdp.std() + 1e-6)` 적용. Alsaify·SafeSignal 양 경로 공통. 후속으로 main `8c85920` (2026-05-17)에서 `rpca_sparse()`에 degenerate 입력 방어(nan/inf → ValueError, `np.std(D) < 1e-8` → zero S 조기 반환) 추가 — constant/zero 윈도우에서 RobustPCA `mu = N/(4·‖D‖₁)`가 inf로 발산하며 발생하던 RuntimeWarning(`divide by zero` / `invalid value in multiply`) 제거. 학습/평가 및 B안 ablation 비교는 pending. final normalization choice pending ablation.
---

## Implementation Status

| Module | Status | Branch | Last updated |
|--------|--------|--------|--------------|
| preprocessing/pipeline.py | done | main | 2026-05-17 |
| pretrained/model.py (CNNGRUAttention) | done | feature/pretrained-model | 2026-05-09 |
| train.py | done | feature/pretrained-model | 2026-05-10 |
| metrics.py (FallMetrics) | done | feature/pretrained-model | 2026-05-09 |
| augment/augment.py | done | feature/pretrained-model | 2026-05-09 |
| model/r_pca.py | done | feature/pretrained-model | 2026-05-09 |
| preprocessing/rpca.py (degenerate 입력 방어 wrapper) | done | main | 2026-05-17 |
| firmware/csi_tx (PS 비활성화 적용) | done | main | 2026-05-12 |
| firmware/csi_rx1 (PS 비활성화 + 디버그 카운터) | done (디버그 빌드) | main | 2026-05-12 |
| firmware/csi_rx2 (PS 비활성화 + 디버그 카운터) | done (디버그 빌드) | main | 2026-05-12 |
| Alsaify 전체 사전학습 (E1+E2) | done (외부 수신·로컬 적용) | main (`model/pretrained/checkpoints/` gitignored) | 2026-05-14 |
| UDP 수신 서버 | pending | - | - |
| WebSocket 서버-Pi4 통신 | pending | - | - |
| 자체 데이터 수집 파이프라인 | done | feature/pretrained-model | 2026-05-11 |
| preprocessing/resample.py (D-018 SafeSignal 100Hz 리샘플) | done | main | 2026-05-13 |
| preprocessing/loader.py SafeSignal 경로 (load_safesignal_csv 등) | done | main | 2026-05-13 |
| preprocessing/pipeline.py SafeSignal 경로 (preprocess_safesignal_file*) | done | main | 2026-05-13 |
| fine-tuning | pending | - | - |
| Pi4 하드웨어 버튼 인터페이스 | pending | - | - |
| E2E 통합 테스트 | pending | - | - |
| inference/ 모듈 (InferenceWorker + FallPredictor + SlidingWindowBuffer) | done | main | 2026-05-11 |
| main 브랜치 통합 (server/dongseok + feature/pretrained-model) | done | main | 2026-05-11 |

---

## Review Notes

### 2026-05-17 — 발표자료 다이어그램 15종 제작 완료

- **작업 범위:** SafeSignal 졸업발표용 설명 다이어그램 전체 세트 제작. 구현 코드 변경 없음.

| # | 제목 | 챕터 | 핵심 내용 |
|---|------|------|-----------|
| 1 | WiFi CSI 원리 | 3 | 이전 세션 제작 |
| 2 | 기술 비교표 | 3 | 이전 세션 제작 |
| 3 | Tx-Rx 삼각형 배치도 | 4 | 벽1 중앙 Tx / 벽2 양측 Rx1·Rx2, 관측 각도 α≠β |
| 4 | UDP 패킷 구조 224B | 4 | 헤더 16B + 페이로드 208B(amplitude×52) |
| 5 | RPCA 분해 M=L+S | 5 | 저랭크 배경 L vs 희소 전경 S 행 스트립 히트맵 |
| 6 | ACF 개념도 | 5 | lag 1~20, 낙상=급강하 / 걷기=주기진동 / 정적=상관유지 |
| 7 | SDP 패턴 3종 비교 | 5 | 낙상·걷기·정적 SDP 히트맵 (시간×lag, 색=ACF z-score) |
| 8 | 전처리 파이프라인 흐름도 | 5 | 외부 W=300 → RPCA → ACF → SDP → (N,1,28,20) shape 변화 |
| 9 | 모델 아키텍처 | 6 | CNN×3 → BiGRU(h=128,양방향) → Attention → FC(6클래스) |
| 10 | Temporal Attention 가중치 시각화 | 6 | 28 스텝 바 차트, 낙상 구간(t=13~16) 집중, α₁₄=0.154 |
| 11 | 도메인 갭 브리징 | 7 | Alsaify(320Hz,90ch) + SafeSignal(100Hz,104ch) → SDP 평균 → (28,20) 동일 shape |
| 12 | 2단계 학습 전략 흐름도 | 7 | 1단계(Alsaify, 전 레이어 lr=1e-3) → 2단계(SafeSignal, CNN동결/BiGRU부분/Attn+FC 고lr) |
| 13 | 데이터 증강 3종 시각화 | 7 | Jittering(σ=0.05) / Scaling(U[0.8,1.2]) / TimeWarping 원본 비교 |
| 14 | Recall-FAR 트레이드오프 곡선 | 8 | ROC 곡선 + MVG 목표 구간 해칭(Recall≥0.85·FAR≤0.15) + Alsaify 결과 포인트(R=0.919, FAR=5.6%) |
| 15 | 시스템 통합 아키텍처 | 10 | ESP32-S3×3 → UDP 5005 → 추론서버 → WebSocket 'FALL_DETECTED' → Pi4 → SOLAPI → 보호자 SMS |

- **다음 단계:** 발표 자료(PPT)에 다이어그램 삽입 / 자체 데이터 수집 및 파인튜닝 진행 (W4 목표)

### 2026-05-14 — 팀원 Alsaify 사전학습 best.pt 수령·적용

- 입력: 사용자 제공 `checkpoints-20260512T010924Z-3-001.zip` (프로젝트 루트). 내용물 — `best.pt` (1,558,504 B), `last.pt` (동일 크기), `best_metrics.json`, `final_metrics.json`, `history.json`, `dataset_cache_e12_w300_s300_tail_ps.npz` (16.5 MB, Alsaify E1+E2 캐시).
- 메트릭(best 기준): `fall_recall=0.91875`, `fall_precision=0.9074`, `fall_f1=0.9130`, `far=0.0222`, `accuracy=0.8101`, `meets_all_targets=true`. final 시점도 `fall_recall=0.909 / F1=0.893`로 D-011 목표 모두 충족. 테스트셋 1,669개 (fall 320·walking 320·sit_stand 308·lying 320·standing 241·picking 160) — 클래스 6개와 `fall_label=0` 모두 [model/pretrained/model.py](https://github.com/LeapSeeker/wifi-csi-fall-detection/blob/main/model/pretrained/model.py) `CLASSES` 와 일치.
- 캐시 파일명 `dataset_cache_e12_w300_s300_tail_ps.npz` 는 [D-006] (E1+E2) + train.py `_make_cache_path(envs=[1,2], window_size=300, stride=300, tail_window=True, pad_short=True)` 명명 규칙과 정합 — 팀원 측이 합의된 학습 설정을 그대로 사용했음을 확인.
- 적용: 기존 `model/pretrained/checkpoints/` 6개 파일을 `model/pretrained/checkpoints_local_backup_20260514/` 로 통째 이동 후, 팀원 6개 파일을 같은 위치에 배치. `server/inference/config.py:MODEL_PATH` 절대 경로(`project_root/model/pretrained/checkpoints/best.pt`)가 그대로 가리키므로 코드 변경 없음. `model/pretrained/checkpoints/` 는 이미 `.gitignore` 등재(line 15)되어 git status 클린.
- 검증: `FallPredictor(device="cpu")` 로 새 best.pt 로드 → classes 튜플 정확히 `('fall','walking','sit_stand','lying','standing','picking')` 매치, 더미 (300, 104) 랜덤 윈도우 `predict()` 호출 시 softmax 합 1.000000 / 클래스 분기 정상 / `is_fall=False` (랜덤 입력이라 의도된 결과). FallPredictor 자체에는 6-class fine-tune 경고 분기가 있으나 이 체크포인트는 6-class라 경고 없이 통과.
- 잔여물 처리(사용자 결정): 원본 zip `checkpoints-20260512T010924Z-3-001.zip` 삭제 완료(팀원 공유 경로에 원본 잔존). 구 로컬 백업 폴더 `model/pretrained/checkpoints_local_backup_20260514/` 는 보존하되 안내 마커 `_DELETE_AFTER_2026-06-04.txt` 추가 — 신 best.pt 운영 안정 확인 후 데모(2026-06-04) 이후 수동 삭제 예정.

### 2026-05-13 — D-018 커밋 origin/main 푸시

- `f25f018 SafeSignal 자체수집 100Hz 리샘플 전처리 추가`를 origin/main에 푸시 (`a589205..f25f018`). 코드/STATE 변경 없음, 로컬 ahead 1 → 동기화 완료.

### 2026-05-13 — Codex review: D-018 SafeSignal 리샘플 구현 검토 및 커밋 완료

- 검토 대상 커밋 (wifi-csi-fall-detection main): `f25f018 SafeSignal 자체수집 100Hz 리샘플 전처리 추가`.
- 검토 결과: blocking issue 없음. SafeSignal 전용 경로가 Alsaify 기존 경로와 분리되어 있고, 공통 후단(`sliding_windows`, `windows_to_model_input`, `window_to_model_input`)만 재사용하는 구조가 D-018/D-013과 정합됨.
- Codex 보완: `resample_to_100hz()`에 `target_hz <= 0` 및 `step_us <= 0` validation 추가, `test_safesignal.py`의 grid count 주석 오타 수정 후 동일 커밋에 포함.
- 검증 재현: 테스트 전용 Python 3.14 의존성(`numpy`, `pandas`, `scipy`, `tqdm`)을 임시 `.codex-test-deps`에 설치해 실행 후 삭제. `SAFESIGNAL_FULL_TEST=1 python -m model.preprocessing.test_safesignal` → `ALL_OK`, `preprocess_safesignal_file_full` inputs `(2, 1, 28, 20)` 확인. `python model/preprocessing/test_pipeline.py` → Alsaify 전처리 경로 PASS, CNN forward만 `torch` 미설치로 skip.
- 잔여 리스크: 실제 fine-tuning 진입 시 `train.py`/캐시 빌더가 SafeSignal CSV 전용 경로를 호출하도록 연결 필요. 실시간 추론은 아직 packet-count 기반 buffer라 70Hz 환경에서 300패킷이 약 4.3초를 커버함 — E2E 단계에서 timestamp-aware resampling buffer 전환 필요.

### 2026-05-13 — D-018 SafeSignal 100Hz 리샘플 전처리 구현

- 적용 브랜치: `main` (코드 커밋은 별도, 이 STATE 업데이트와 분리). Alsaify 경로(`load_csi_csv`, `preprocess_file*`, `preprocess_directory*`)는 변경하지 않고 SafeSignal 전용 경로를 추가만 함.
- 신규 파일: `model/preprocessing/resample.py`, `model/preprocessing/test_safesignal.py`. 기존 파일 확장: `model/preprocessing/loader.py`, `model/preprocessing/pipeline.py`, `model/preprocessing/__init__.py`.
- `resample.py`: `resample_to_100hz(amp, timestamps_us, target_hz=100.0, max_gap_ms=100.0) → ResampleResult`. step_us = round(1e6/target_hz) (100Hz → 10,000us). 동작: stable sort → 동일 timestamp 평균 병합 (`np.add.at` 그룹 평균, drop이 아닌 이유는 같은 시점 두 측정값을 모두 무시하지 않기 위함) → 단조 증가 검증 → 첫/마지막 timestamp 안쪽만 균일 격자 생성 → subcarrier별 `np.interp` 선형 보간. cubic/spline 미사용(overshoot 위험). max_gap_ms 초과 gap은 hard reject 하지 않고 `gap_count`/`max_gap_us`로 metadata만 노출. `ResampleResult` 필드: `amplitude`, `timestamps_us`, `original_count`, `resampled_count`, `original_rate_hz`, `target_hz`, `max_gap_us`, `gap_count`. n_packets<2 케이스는 빈 결과 + nan rate로 안전 반환.
- `loader.py`: 모듈 docstring을 Alsaify+SafeSignal 양쪽 명시. 신규 `SafeSignalMeta(env, subject, activity:str, trial, filename)`, `SafeSignalRaw(amplitude, timestamps_us, rx, meta)`, `parse_safesignal_filename`, `load_safesignal_csv(path, rx="both")`. 파일명 정규식 `E{env}_S{subj}_A_{ACTIVITY}_T{trial}` (activity는 `[A-Z0-9_]+`로 `SIT_STD` 같은 멀티토큰 허용, 문자열 그대로 보존). `rx="rx1"|"rx2"` → (n,52), `rx="both"` → Rx1/Rx2 서브캐리어 concat (n,104) (D-013 정합). 필수 컬럼 누락 시 명확한 ValueError. Alsaify 경로 시그니처/동작 변경 없음.
- `pipeline.py`: `SafeSignalPreprocessResult(windows, meta, resample)`, `SafeSignalModelInputResult(inputs, meta, resample)` dataclass 추가. `preprocess_safesignal_file(...)` = `load_safesignal_csv → resample_to_100hz → sliding_windows`, `preprocess_safesignal_file_full(...)`은 후단에 `windows_to_model_input` 적용. 파라미터: `rx="both"`, `target_hz=100.0`, `max_gap_ms=100.0`, `window_size=WINDOW_SIZE`, `stride=None`, `drop_last=True`, `tail_window=False`, `pad_short=False`, `rpca_max_iter=DEFAULT_MAX_ITER`, `rpca_tol=None`. 5초 수집 데이터에서는 호출자가 `tail_window=True`로 잔여 패킷 보존. Alsaify 함수 4종(`preprocess_file`, `preprocess_file_full`, `preprocess_directory`, `preprocess_directory_full`) 변경 없음. 공통 후단(`sliding_windows`, `windows_to_model_input`, `window_to_model_input`)만 재사용.
- `__init__.py`: 신규 public symbol 8개를 `__all__`에 추가 (load_safesignal_csv, parse_safesignal_filename, SafeSignalMeta, SafeSignalRaw, resample_to_100hz, ResampleResult, preprocess_safesignal_file, preprocess_safesignal_file_full, SafeSignalPreprocessResult, SafeSignalModelInputResult).
- `test_safesignal.py`: 7-case 모두 ALL_OK. (1) parse 성공 + `SIT_STD` 멀티토큰 활성, (2) synthetic 400 packet × 70Hz CSV로 `load_safesignal_csv(rx='both')` shape=(400,104), rx='rx1' shape=(400,52), (3) resample 후 `np.diff(timestamps_us) == 10000` 전부, 외삽 없음 (start≥min, end≤max), original_rate≈70Hz, (4) 중복 + 역순 timestamp 처리(평균 병합 + stable sort), (5) synthetic으로 `preprocess_safesignal_file(tail_window=True)` → windows (2, 300, 104), (6) 실제 CSV `data/raw/E1_S01_A_STAND_T001.csv` (377 패킷, 5.49s, 68.49Hz)로 windows (2, 300, 104) — `gap_count=5`, `max_gap=189782us` 노출(reject 아님), (7) `preprocess_safesignal_file_full` (RPCA 포함) → inputs (2, 1, 28, 20). 기본 SKIP, `SAFESIGNAL_FULL_TEST=1` 환경변수로 활성. test_pipeline.py(Alsaify) 별도 PASS 재확인.
- 결정 정정: D-018 본문은 scipy `interp1d` 선형 보간을 명시했으나 구현은 사용자 지침에 따라 `np.interp` 사용. 결과 동일(둘 다 1차 선형 보간), 의존성 1개 줄어듦. D-018 본문은 별도 결정 갱신 없이 구현 노트로만 남김.
- 범위 외(이번 작업 미포함): `server/inference/buffer.py`의 timestamp-aware resampling buffer 전환. 이번은 오프라인 자체수집 CSV 학습 입력 생성만 가능하게 하는 범위. 실시간 추론 측은 별도 작업.
- 잔여: 자체수집 학습 시 `train.py`/캐시 빌더가 SafeSignal 경로를 호출하도록 추가 작업 필요 — 현재 train.py는 Alsaify `preprocess_directory_full`만 사용. fine-tuning 단계 진입 시 캐시 명명 규칙(`dataset_cache_e*_w*_s*[_tail][_ps].npz`)에 `_safesignal` suffix 또는 별도 빌더 도입 검토.

### 2026-05-12 — 유니캐스트 효과 검증 + 70Hz 천장 진단 → D-018 도입

- **데이터 (post-unicast E1_S01)**:

  | 파일 | duration | pair_rate | rx1 cb-rate | rx2 cb-rate |
  |------|---------:|----------:|------------:|------------:|
  | WALK_T001 | 8.51 s | 67.0 Hz | 68.0 Hz | 67.1 Hz |
  | WALK_T002 | 8.49 s | 76.8 Hz | 77.9 Hz | 77.7 Hz |
  | STAND_T001 (post-unicast) | 5.49 s | 68.7 Hz | 68.9 Hz | 69.9 Hz |

- **펌웨어 STATS 로그 (1초 단위, RX1·RX2 두 노트북 시리얼 동시 캡처)**: 모든 조건에서 `match/cb ≈ 50%`, `sent == match`, `qfull = 0`, `fail = 0`. 큐/UDP/페어링 무손실 확정. ~30% TX 프레임이 RX 라디오 demod 단계 이전에 손실 (펌웨어에서 더 짜낼 여지 없음).
- **RF 변경 실험 (STAND 5초 match 평균)**:
  - 채널 11, 책상 분리 (USB 위치): RX1 64/s, RX2 60/s
  - 채널 11, 밀집 1m + 헬리컬 안테나 정렬: RX1 62/s, RX2 63/s
  - 채널 1, 밀집 1m + 헬리컬 안테나 정렬: RX1 67/s, RX2 68/s
  - 세 조건 모두 변동 ±5/s 안. **거리/안테나/채널 모두 천장의 원인이 아님 확정**.
- **1초 dip 패턴**: 약 5~7초 주기로 cb가 절반 수준(150→70)으로 떨어졌다 즉시 복귀. 채널 11/1 동일 발생 → Windows 모바일 핫스팟 스택의 주기 백그라운드 작업(스캔/관리) 추정. 펌웨어/채널 변경으로 회피 불가.
- **WALK 동작 손실 가설 기각**: STAND 정적 자세도 동일한 ~70Hz 천장. 걷기 동작이 air capture를 추가로 떨어뜨린다는 가설은 데이터로 기각. WALK_T002(77Hz)가 STAND(69Hz)보다 오히려 높은 케이스도 있음(walking의 fade null 평균화 효과로 추정).
- **결론**: 70Hz 천장은 Windows 모바일 핫스팟 스택의 처리 한계. 거리/안테나/채널/PS 모두 해소 불가. 펌웨어 단에서 더 짜낼 여지 없음. **현재 환경에서는 전처리 단계 100Hz 리샘플로 보정(D-018)**, 라우터 확보 시 재평가.
- **broadcast era 데이터 격리**: `data/raw/_archive/broadcast/`로 4개 CSV 이동 (`E1_S01_A_STAND_T001`, `E1_S02_A_WALK_T001~T003`) + `NOTE.txt` 작성. `.gitignore`에 `data/raw/_archive/` 추가. 전처리 글로브(`root.glob("*.csv")`)가 비재귀라 학습 입력에서 자동 제외. count_sessions 카운터도 archive 비포함 (UX 정상).
- **코드 변경**: `collect/collect_main.py` `_run_session` 출력에 `pair_rate` / `capture_ratio` 표시 추가 — 운용 중 천장 변동 즉시 가시화. 펌웨어/페어링/저장 로직/CSV 포맷 변경 없음.
- **잔여**: D-018 리샘플 구현은 보드 없이 가능, 후속 진행. RX1/RX2 STATS 디버그 코드 정리는 라우터 환경 재평가 마치고 진행 권장 (지금 정리하면 라우터 환경에서 다시 추가해야 하므로 한 번에 처리).

### 2026-05-12 — 펌웨어 변경 main 푸시 + gitignore 보강

- 적용 커밋 (wifi-csi-fall-detection main): `dcb117c [수정] WiFi Power Save 비활성화 및 RX 처리량 모니터 카운터 추가`, `b7145fa [수정] gitignore 보강 및 데이터 sanity-check 스크립트 추가`.
- 펌웨어: TX/RX1/RX2 `esp_wifi_set_ps(WIFI_PS_NONE)` 적용분, RX1/RX2 `stats_task` + 5개 카운터 디버그 코드, sdkconfig IDF 5.3.5→5.4.3 자동 재생성분을 main에 반영. 주석은 "DEBUG, 안정화 후 제거"로 표시. WALK 재수집 검증은 별도 진행.
- 운영: `.gitignore`에 `server/logs/`, `.claude/settings.local.json` 추가. 추적되고 있던 `.claude/settings.local.json`은 `git rm --cached`로 추적 해제 (개인 권한 설정, 팀 공유 대상 아님). 분석용 sanity-check 스크립트 `data/raw/_analyze_T002.py`를 분석 도구로 보존 (CSV loss/seq/amp 통계, T001~T002 검증용).

### 2026-05-11 — WALK 세션 페어 변동성 진단 + WIFI_PS_NONE 적용 (D-017)

- **현상**: WALK(8초) 세션의 저장 페어 카운트가 24~33개로 극단적으로 낮음. STAND_T001은 485/500 (97%)로 정상이었던 환경에서 27분 후 발생.
- **데이터 분석 (E1_S02_A_WALK_T001~T003)**:
  - 세 파일 모두 **첫 0.31~0.32초 동안만 페어 정상 캡처** (pair_rate 77~106Hz, SNTP std 0.17~1.82). 그 후 7.7초간 페어 0개.
  - 세션 간 RX1/RX2 seq 증가량: 평균 3~7Hz (정상 100Hz의 3~7%) → TX는 100Hz 송신 중인데 air에는 burst 형태로만 떠 있음.
- **진단 단계**:
  1. **TX 펌웨어 디버그 (1초당 송신 카운트)**: 35초 관측 `TX rate=100 pkt/s`, `fail=0` 완벽 유지 → TX 결백. 이후 TX 디버그 코드 원복.
  2. **RX1 펌웨어 디버그 (단계별 카운터)**: `g_csi_total/g_csi_match/g_csi_qfull/g_udp_sent/g_udp_fail` 5개 카운터 + `stats_task` 추가. 1초마다 STATS 로그 출력.
  3. **RX1 STATS 패턴 (디버그 빌드 첫 관측)**: cb는 50~270/s로 계속 들어오는데 match만 burst→silence 무작위 패턴 (예: 7~8s burst, 30~50s silence, 36s burst, 31s silence). 즉 RX 채널/큐/송신 모두 정상, TX 패킷만 air에서 끊김.
  4. **RX1 부팅 로그 결정적 단서**:
     - `connected with coin, aid = 8, channel 11` — AP "coin"이 채널 11. 코드 `DEFAULT_WIFI_CHANNEL=6`은 STA hint일 뿐 AP가 결정 → 양쪽 다 11로 점프, 일관성 문제 없음.
     - `wifi:pm start, type: 1` — TX/RX 모두 `WIFI_PS_MIN_MODEM` (ESP-IDF 기본값) 활성. modem sleep으로 RF 주기적으로 off.
     - `AP's beacon interval = 102400 us, DTIM period = 3` — AP가 broadcast를 약 307ms마다 forward.
     - `sta ip: 192.168.137.72, gw: 192.168.137.1` — Windows 모바일 핫스팟 (192.168.137.x = Windows ICS 기본 대역).
     - `WiFi 끊김 → 재연결` 로그 0건 — STA 연결 자체는 안정.
- **원인 진단**: Windows 모바일 핫스팟의 broadcast forwarding 제약(DTIM 3 + client isolation 경향) + ESP32 `WIFI_PS_MIN_MODEM` modem sleep 조합. TX `sendto`는 100Hz 정상이지만 AP→air 단계에서 burst로만 띄움. RX는 promiscuous로 sniff하지만 자기 modem sleep 시점에 RF off되어 burst 구간만 캡처.
- **1순위 fix 적용 (이번 세션)**:
  - TX/RX1/RX2 세 펌웨어 모두 `wifi_init()` 끝에 한 줄 추가:
    ```c
    ESP_ERROR_CHECK(esp_wifi_set_ps(WIFI_PS_NONE));
    ESP_LOGI(TAG, "WiFi Power Save 비활성화 (WIFI_PS_NONE)");
    ```
  - 사용자가 3개 보드 모두 빌드/플래시 완료, 부팅 로그에서 `pm start, type: 0` 및 ESP_LOGI 메시지 확인 완료.
  - 효과 검증(WALK 재수집)은 다음 세션에서 진행 예정.
- **다른 발견**:
  - `csi_rx1_main.c:86`의 `wifi_event_group` 변수가 미사용 잔재 (실제는 `s_wifi_event_group` 사용) → 컴파일 warning만 발생, 기능 영향 없음. 진단 마무리 시 함께 정리 검토.
  - 휴대폰 핫스팟도 일반 무선 공유기 대비 broadcast forwarding/client isolation/DTIM 제어 측면에서 제약 있음(특히 iOS). 사용자가 일반 공유기 확보 어려워 휴대폰 핫스팟 + WIFI_PS_NONE으로 진행 결정 (D-017).
- **잔여 디버그 코드 (PS 효과 검증 후 정리)**:
  - `firmware/csi_rx1/main/csi_rx1_main.c`, `firmware/csi_rx2/main/csi_rx2_main.c`: 전역 카운터 5개 + `stats_task` + `wifi_csi_cb`/`udp_send_task` 카운트 증가 코드. **PS 비활성화 효과는 영구 코드, 카운터/stats_task는 임시 디버그**라 한 파일 안에 섞여 있음. 검증 종료 후 카운터만 분리 정리.
  - TX는 1초당 송신 카운트 로그를 이미 원복 완료 (`csi_tx_main.c`).
- **3순위 fix (별도 자문 진행 중)**: TX target `255.255.255.255` (broadcast) → 노트북 LAN IP `192.168.137.1` (unicast)로 변경. 이론적으로 broadcast forwarding 제약 회피 가능, 다만 unicast의 무선 air-time 특성(retry/ACK 패턴)이 broadcast와 달라 RX promiscuous sniff 효과는 실측 필요. 다른 AI 자문 후 결정.

### 2026-05-11 — 추론 파이프라인 구현 (inference/ 모듈, server 연결, D-013~D-015)

- 적용 브랜치: `main` (server/dongseok + feature/pretrained-model 병합 후 추가 구현). 코드 커밋은 별도, 이 STATE 업데이트와 분리.
- 브랜치 통합: `git merge server/dongseok --no-ff`, 이어서 `git merge feature/pretrained-model --no-ff`. 두 병합 모두 충돌 없이 자동 병합 — `server/`는 server/dongseok이, `model/`/`collect/`/`firmware/csi_*`/`data/`는 feature/pretrained-model이 단독 보유. `server/main.py`/`server/requirements.txt`는 feature/pretrained-model 쪽이 0-byte 빈 파일이라 server/dongseok 구현이 그대로 유지됨. main에 server+model 양쪽 코드가 모두 존재.
- 신규 디렉터리 `server/inference/`: `__init__.py`, `config.py`, `buffer.py`, `predictor.py`, `worker.py`, `_selfcheck.py`. 외부 노출은 `InferenceWorker` 만.
- `config.py`: `WINDOW_SIZE=300`, `INFERENCE_STRIDE=100` (D-014 RTX4060 실측 전 기본값), `FALL_CLASS_IDX=0`, `FALL_THRESHOLD=0.5`, `N_SUBCARRIERS_EACH=52`. `MODEL_PATH`는 `Path(__file__).resolve().parents[2] / "model/pretrained/checkpoints/best.pt"` 절대 경로로 계산 (cwd 무관).
- `buffer.SlidingWindowBuffer`: `deque(maxlen=300)`, `add(rx1_amp, rx2_amp)`는 Rx1(52)+Rx2(52) concat row를 append (D-013). `_since_last_predict` 초기값을 `stride` 로 설정해 버퍼가 처음 가득 차는 즉시 첫 trigger 발생, 이후 stride개 이상 추가 시 다시 trigger. `get_window()` 는 `(300, 104) float32`. 잘못된 길이 입력은 False 반환.
- `predictor.FallPredictor`: `torch.load(model_path, map_location=device)` → `CNNGRUAttention.load_state_dict(ckpt["model"])`. `ckpt["classes"]` 길이/순서가 `model.pretrained.model.CLASSES`와 다르면 warning(7-class fine-tuned 모델 대응 여지). `predict(window: (300,104))` → `window_to_model_input` → `(1,1,28,20)` tensor → softmax → `{class, confidence, is_fall, probabilities}`. `is_fall`은 `argmax==FALL_CLASS_IDX AND fall_prob >= FALL_THRESHOLD`. sys.path에 project_root 추가하여 `model.*` import 보장.
- `worker.InferenceWorker`: top-level은 `multiprocessing/queue`만 import (torch/RPCA import는 child process 내부 `_inference_process`에서 수행 — main process가 heavy import 부담 없음, self-check가 torch 없이 동작 가능). `mp.get_context("spawn")` (Windows 안전). `input_queue=ctx.Queue(maxsize=500)`, `put_nowait()` 큐 포화 시 드롭 카운트 로그. `get_result()`는 `queue.Empty` → None. `start()` 전에는 process 없음. child 프로세스는 `project_root`와 `server/`를 sys.path에 prepend 후 `from inference.predictor import FallPredictor` 형식으로 import, 실패 시 `from server.inference.predictor`로 fallback.
- `server/main.py`: top-level에 있던 `RPiConnection`/`FallCooldown`/`PacketMonitor`/`PairingBuffer` 인스턴스 생성과 `pairing_buffer` 전역 생성을 모두 `main()` 함수로 이동. top-level은 None 초기화만 유지. `if __name__ == "__main__":` 첫 줄에 `multiprocessing.freeze_support()`. 콜백(`on_paired`/`on_packet_received`/`on_fall_detected`)은 top-level 정의 유지하되 객체 None 가드 추가. `on_paired()`의 기존 TODO 주석 제거하고 `inference_worker.put(rx1, rx2)` 호출. `on_fall_detected(confidence, seq_num, timestamp_us)` 시그니처로 변경 후 `rpi_connection.send_fall_alert(confidence, seq_num, timestamp_us)`에 그대로 전달. 새 `result_loop()` 스레드(50ms 폴링)가 `inference_worker.get_result()`를 처리: `is_fall=True`면 콜백 호출, `error` 키면 `log_warn`. 기존 cleanup_loop/stats_loop 동작 유지.
- `server/ws_handler/rpi_connection.py`: `send_fall_alert(confidence=0.0, seq_num=0, timestamp_us=0)` 시그니처. 페이로드 = `{"event":"fall_detected","label":"fall","confidence":...,"seq_num":...,"timestamp_us":...}` JSON. `timestamp_us=0`이면 host wall-clock μs로 fallback (가능하면 추론 결과의 rx1 timestamp_us 사용). 기존 `"FALL_DETECTED"` 문자열 송신 제거. `import json`, `import datetime` 추가.
- Pi4 포맷 확정: `class` 키 사용 안 함, `label` 사용 (Python/JSON 소비 측에서 `class`는 예약어 혼동 회피). D-008 본문은 `event/class/confidence/timestamp_us`로 표기되어 있고 context/SHARED.md도 일관되지 않음 — 이번 구현부터 위 JSON으로 통일. test/test_pi4_ws.py도 `if message == "FALL_DETECTED":` 문자열 비교를 `json.loads()` 후 `event=="fall_detected" AND label=="fall"` 확인으로 갱신, confidence/seq/ts 로깅 추가. Pi4 실제 수신 코드는 아직 미구현이므로 서버측 포맷이 reference.
- `server/config/settings.py`: `SUBCARRIER_COUNT`를 64 → 52로 수정 (D-007 LLTF 기준 Rx 단일 패킷). `SUBCARRIER_COUNT_CONCAT=104` 신규 추가 (D-013 Rx1+Rx2 concat 추론 입력). 대시보드/PairingBuffer/PacketMonitor 등 기존 코드가 참조하는 키는 `device_id/seq_num/timestamp_us/n_subcarriers/amplitudes` 그대로 유지됨.
- `server/receiver/udp_receiver.py`: 기존 `HEADER_FORMAT="<BIQH"` (15B) — n_subcarriers를 헤더에서 받는 옛 구조 → D-007 `"<BBbBIQ"` (16B header + 52f amplitude = 224B) 로 교체. `parse_packet()`은 size<224 / magic≠0xAB / device_id∉{RX1,RX2} 모두 None 반환. 반환 dict 키 (device_id/rssi/seq_num/timestamp_us/n_subcarriers=52/amplitudes(list len 52))로 downstream 호환 유지.
- UDP 실기 수신 테스트는 이번 작업에서 수행하지 않음 (py_compile + self-check 수준). 실기 ESP32 패킷 수신 검증은 별도 진행. Implementation Status의 "UDP 수신 서버" 행은 그대로 pending 유지.
- self-check (`server/inference/_selfcheck.py`): 4-case 모두 통과 — (1) D-007 224B 더미 패킷 parse 정상/size·magic·device_id 불일치 None / (2) SlidingWindowBuffer (300,104) shape + stride trigger 정확히 [300,400,500] / (3) MODEL_PATH `Path` 절대 경로 + 파일명 best.pt (존재 확인 안 함) / (4) InferenceWorker start() 없이 put/get_result 예외 없음. `ALL_OK` 출력. self-check는 torch/checkpoint/RPCA를 import하지 않음 — `worker.py` top-level에 predictor import가 없기 때문. _selfcheck.py 시작부에서 script dir(server/inference/)을 sys.path에서 제거해야 `server/inference/config.py` 모듈이 `server/config/` 네임스페이스 패키지를 가리지 않음(이 트릭 없으면 `from config.settings import ...` 에서 `config is not a package` 에러).
- py_compile 검증: 9개 파일 (`server/inference/{__init__,config,buffer,predictor,worker}.py`, `server/main.py`, `server/receiver/udp_receiver.py`, `server/ws_handler/rpi_connection.py`, `server/config/settings.py`) 전부 통과.
- 의존성 추가: `python-dotenv` (server/requirements.txt에 이미 명시되어 있었으나 로컬 환경 미설치 상태였음 → 설치). 신규 의존성 추가는 없음.

### 2026-05-11 — 자체 데이터 수집 파이프라인 (collect/) 구현

- 적용 브랜치: `feature/pretrained-model` (코드 커밋은 별도, 이 STATE 업데이트와 분리).
- 신규 파일: `collect/labels.py`, `collect/beep.py`, `collect/udp.py`, `collect/recorder.py`, `collect/collect_main.py`, `collect/_selfcheck.py`.
- 수집 목표: D-010(240세션)에서 270세션으로 사용자 갱신 — 낙상 9종(앉/서/걷×앞/뒤/옆) ×10 = 90, 비낙상 6종(SIT_STD/LIE/WALK/STAND/RUN/PICK) ×30 = 180. `labels.ACTIVITY_INFO` 단일 source of truth, `total_target_sessions()`로 270 검증.
- 패킷 파싱 기준: STATE.md D-007 + `firmware/csi_rx1/main/csi_rx1_main.c`, `firmware/csi_rx2/main/csi_rx2_main.c`의 `csi_packet_t` (`__attribute__((packed))`)에 정렬. 224B = `<BBbBIQ`(16B header) + `52f`(208B). `parse_packet()`은 size/magic(0xAB)/device_id(0x01|0x02) 모두 검증.
- 페어링: `PairingBuffer`가 Rx1/Rx2 timestamp 기준 50ms 이내 가장 가까운 패킷을 pair로 확정, 200ms 미완성 만료, 버퍼 200개 cap, `threading.Lock` 보호. cleanup은 host wall-clock이 아닌 가장 최근에 본 패킷의 `timestamp_us`를 reference로 삼아 ESP↔host SNTP skew 영향 제거. UDP receiver는 `start_udp_receiver()`로 1회 실행되는 daemon thread 2개(recv, cleanup).
- 녹화 시점: ENTER → `beep_ready()` → 3초 카운트다운 → **여기까지 CSV에 저장하지 않음** → `recorder.start_session()` → 첫 stage 직전 `beep_stage()` → stage/duration 대기 → 마지막 stage 종료 직후 `recorder.stop_session()` → `beep_end()`. 즉 ready/카운트다운 구간은 CSV에서 제외되고 실제 동작 구간만 저장.
- CSV 컬럼 107개 고정 (rssi 미저장): `timestamp_us, seq_rx{1,2}, amp_rx1_0..51, amp_rx2_0..51`. 파일명: `data/raw/E{env}_S{subj:02d}_A_{code}_T{trial:03d}.csv`. trial은 동일 (env, subject, code) 조합 파일 수 + 1 자동 증가.
- 손실률: `(max_seq - min_seq + 1 - unique_seq_count) / (max_seq - min_seq + 1)`을 rx1/rx2 각각 계산해 max 반환. 5% 초과 시 경고 출력 후 저장 여부 사용자 선택.
- CLI 진입점: `python collect/collect_main.py [--port 5005]`. 초기 선택은 env → subject → activity 순(스펙은 env→activity→subject지만 활동 표 완료 수가 정확하려면 subject가 먼저 정해져야 함 — 결과 동일, UX만 조정). 변경 메뉴(1.env / 2.activity / 3.subject / 4.없음 / q.종료) 루프.
- 검증: `python -m py_compile collect/{labels,recorder,udp,beep,collect_main}.py` 통과. `python collect/_selfcheck.py` — 5개 항목(parse_packet, pairing(50ms 통과/60ms 차단/cleanup), loss_rate(0/10%/empty/single), trial 자동 증가 + CSV 컬럼 107개 순서, labels 270세션) 모두 ALL_OK. 손실률 계산은 numpy 없이 stdlib만 사용.

### 2026-05-11 — sanity check 결과 및 펌웨어 수정 (P1~P5)

- T002 CSV (390 Hz): P1 가설 A(NVS 잔존 interval) 유력. TX NVS 로드 제거로 100 Hz 고정 (T2).
- RX MAC 필터 추가로 P1 가설 B(비TX 트래픽 캡처) 동시 차단 (T3).
- P2(rx2 seq 비단조): P1 해결 후 재수집으로 재검증 예정.
- P3(DC null idx 21-30): esp-csi 기본 마스킹으로 추정, 학습 영향 없음.
- P4(T001 빈 파일): recorder.py save_session에서 0행 파일 자동 skip 적용.
- P5(컬럼 수): STATE.md 109→107 정정 완료.
- 기존 수집 데이터(T002 등) 390 Hz 기록으로 전면 무효. P1 수정 후 재수집 필요.
- 의존성: pandas (CSV 저장), winsound (Windows 표준 — 별도 설치 불필요). 외부 설치 추가 없음.
- 문서 불일치 명시: `context/SHARED.md`의 UDP 패킷 구조 표(magic 누락, rssi 누락, subcarrier_num 1B로 표기)는 D-007 및 실제 펌웨어와 다름. 사용자 지침에 따라 이번 작업에서는 SHARED.md 수정하지 않음. `firmware/CLAUDE.md`의 예시 구조체(`csi_packet_t`)도 magic/rssi/reserved 누락으로 펌웨어와 불일치 — 동일 사유로 미수정.
- 범위 외: `data/collection_log.md` 자동 갱신은 사용자 요청에 따라 이번 범위 제외. `server/dongseok` 수신 서버와 동일 UDP 포트 동시 bind 시도하지 않음(독립 실행).
- 잔여: D-010(240세션)이 STATE.md에 남아있지만 collect 구현은 270세션 기준으로 정렬됨 — D-010 본문 갱신 또는 신규 D 결정 발행은 사용자 결정 사항으로 남김.

### 2026-05-10 — build_cache 캐시 파라미터 실적용 동기화 (Codex 검토 반영)

- 적용 커밋: `6d90fb4 [수정] build_cache가 캐시 파라미터를 실제로 적용하도록 동기화` (feature/pretrained-model).
- 문제 (Codex 검토): `_make_cache_path()`는 `CACHE_WINDOW_SIZE/CACHE_STRIDE/CACHE_TAIL_WINDOW/CACHE_PAD_SHORT`로 파일명을 만들지만 `build_cache()` 내부 `preprocess_files_full()` 호출은 `tail_window=True, pad_short=True`만 하드코딩되어 있고 window_size/stride는 함수 기본값(WINDOW_SIZE, None)을 그대로 사용. 결과: `CACHE_STRIDE=100`으로 바꾸면 파일명은 `_s100`이 되지만 실제 전처리는 stride=300으로 돌아 캐시 자동 분리 목적이 깨짐.
- 수정: `build_cache()` 시그니처에 `window_size/stride/tail_window/pad_short` 4개 인자 추가 (기본값은 모듈 상수 `CACHE_*`). 내부 `preprocess_files_full()` 호출에 그대로 전달. 빌드 시작 로그에 적용 파라미터를 함께 출력. `main()`에서 `build_cache()` 호출 시 `_make_cache_path()`에 넘긴 동일한 `CACHE_*` 값을 명시 전달. 미사용 상수 `DEFAULT_CACHE_PATH`도 제거 (전체 레포에 잔여 참조 없음 확인).
- 검증: `python -m py_compile model/pretrained/train.py` 통과. `rg "DEFAULT_CACHE_PATH|dataset_cache_tail_ps"` 결과 없음. `_make_cache_path()` 4 케이스 (envs 정렬 `[2,1]→e12`, stride=None→window_size 표기, flags 조합, stride=100 변경) 전부 통과. `build_cache` 시그니처 `inspect.signature` 확인 — 4개 인자 모두 `CACHE_*` 기본값과 일치.
- 영향 범위: train.py 한 파일. 전처리/모델/best.pt 로직, `--cache_path` 수동 지정, `--rebuild_cache` 동작 모두 보존.

### 2026-05-10 — 캐시 파일명 파라미터 기반 자동 생성

- 적용 커밋: `2782517 [수정] 캐시 파일명 파라미터 기반 자동 생성으로 변경` (feature/pretrained-model). 후속 커밋 `f64889f`는 codex의 학습 종료 시 best.pt 기준 요약 출력 추가를 별도 분리한 것.
- 문제: 기존 캐시 파일명이 `dataset_cache[_e<envs>]_tail_ps.npz`로 고정이라 window_size/stride/tail_window/pad_short가 바뀌어도 같은 캐시를 재사용. 팀원이 학습을 돌리려면 매번 `--rebuild_cache`를 수동으로 붙여야 했음.
- 수정: `model/pretrained/train.py`에 헬퍼 `_make_cache_path(cache_dir, envs, window_size, stride, tail_window, pad_short)` 추가 (상수 블록 아래, build_cache 위). 형식 `dataset_cache_e{envs}_w{ws}_s{stride}[_tail][_ps].npz`. stride=None이면 window_size 값으로 표기. 모듈 상수 `CACHE_WINDOW_SIZE/CACHE_STRIDE/CACHE_TAIL_WINDOW/CACHE_PAD_SHORT`를 source of truth로 두고 `main()`에서 `args.cache_path is None`일 때 이 헬퍼로 경로 생성. 캐시 존재 확인/로드/저장 모두 동일 경로 사용.
- 검증: 사용자 지정 3 케이스 모두 통과 — `(envs=[1,2], w=300, s=300, tail=T, ps=T) → dataset_cache_e12_w300_s300_tail_ps.npz`, `(... tail=T, ps=F) → ...tail.npz`, `(... tail=F, ps=F) → ...npz`.
- 영향 범위: train.py 한 파일. 전처리 코드, 모델 아키텍처, best.pt 저장 로직 변경 없음. `--rebuild_cache` 플래그 유지. 미사용 상수 `DEFAULT_CACHE_PATH`는 스코프 최소화 위해 그대로 둠 (다음 정리 시 제거 권장).
- 후속 호환성: 기존 캐시 파일(`dataset_cache_tail_ps.npz`)은 새 명명 규칙과 매치되지 않으므로 첫 실행 시 자동 재빌드된다. 디스크에서 수동으로 옮기거나 `--cache_path`로 직접 지정하면 재사용 가능.

### 2026-05-10 — Codex update: best.pt 선정 기준 문서/로그 정리

- 배경: `df88ef4`에서 `best.pt` 저장 기준이 fall recall 단독 최고에서 `BEST_RECALL_MIN(0.90)` 이상 epoch 중 fall F1 최고로 변경됨.
- Codex 조치: `model/README.txt`의 `best.pt` 설명을 새 기준으로 갱신하고, `model/pretrained/metrics.py` 사용 예시 주석을 recall 임계값 + F1 비교 기준으로 수정. `model/pretrained/train.py`는 학습 종료 시 선택된 `best.pt`의 recall/F1 기준값을 요약 출력하도록 `best_recall_for_best`를 활용.
- 검증: `python -m py_compile model/pretrained/train.py model/pretrained/metrics.py` 통과.


### 2026-05-10 — best.pt 저장 기준 변경 (recall 임계값 + F1)

- 적용 커밋: `df88ef4 [수정] best.pt 저장 기준 recall 임계값 + F1 복합 기준으로 변경` (feature/pretrained-model).
- 문제: 기존 `train.py`는 `m.fall_recall > best_recall` 단독 기준으로 best.pt를 저장. recall만 높고 FAR이 큰(오탐 많은) 모델이 best로 선정될 수 있어 D-011 성능 목표(Recall≥85%, FAR≤15%, F1≥0.85)와 정렬되지 않음.
- 수정: `model/pretrained/train.py`에 상수 `BEST_RECALL_MIN: float = 0.90` 추가. 학습 루프에서 추적 변수를 `best_f1`, `best_recall_for_best`로 교체하고, `m.fall_recall >= BEST_RECALL_MIN` 통과 에폭 중 `m.fall_f1`이 갱신될 때만 best.pt와 best_metrics.json 저장. 미달 에폭은 last.pt만 갱신하고 안내 메시지 출력. 학습 종료 후 best.pt가 한 번도 저장되지 않은 경우(`best_f1 < 0`) 임계값 조정 안내 WARN 출력.
- 영향 범위: 사전학습 train.py 한 파일. 캐시 파일명, 전처리, 모델 아키텍처 변경 없음.
- 잔여 리스크: BEST_RECALL_MIN=0.90이 Stretch 목표 기준으로 잡혀있어 초기 에폭에서 best.pt가 갱신되지 않을 수 있음. 임계값 미달이 지속되면 상수만 낮춰 재학습 가능 (예: 0.85 = MVG 기준).

### 2026-05-10 — Codex review: pad_short API 및 문서 보완 적용 확인

- 확인 커밋: `7e343d5 [수정] pad_short API 및 문서 보완`.
- 검토 결과: `sliding_windows()`의 `tail_window`, `pad_short`는 `*` 뒤 keyword-only 인자로 유지됨. `preprocess_directory()`에도 `pad_short: bool = False`가 추가되었고 내부 `preprocess_file(..., pad_short=pad_short)`로 정상 전달됨.
- 문서 확인: `sliding_windows()` Returns 설명이 `pad_short=True`, `drop_last=False`, `tail_window=True` 조건별 반환 동작과 일치하도록 보완됨.
- 잔여 리스크: 현재 로컬 시스템 Python에는 `numpy`가 없어 import 기반 런타임 검증은 실패했으며, 코드/시그니처 정적 검토 기준으로 확인함.


### 2026-05-10 — pad_short API 및 문서 보완 (Codex P2/P3 정리)

- 적용 커밋: `7e343d5 [수정] pad_short API 및 문서 보완`.
- `model/preprocessing/pipeline.py`: `preprocess_directory()` 시그니처에 `pad_short: bool = False` 추가하고 내부 `preprocess_file(..., pad_short=pad_short)`로 전달. 윈도우-only 디버깅/분석 경로에서도 300 미만 trial 보정을 적용 가능하도록 노출. 기본값 False 유지로 기존 호출자 호환.
- `model/preprocessing/window.py`: `sliding_windows()` Returns 설명을 실제 동작과 일치하도록 갱신. 기존 "n_packets < window_size 면 (0,…) 반환" 단일 케이스 → pad_short × drop_last 조합별 분기와 tail_window 추가 윈도우 케이스를 명시. 동작 로직과 keyword-only 설계는 변경 없음.
- 검증: `inspect.signature`로 `tail_window`/`pad_short`가 KEYWORD_ONLY 유지, `preprocess_directory(..., pad_short=True)`가 bind OK, 위치 기반 6-인자 호출은 `TypeError` 차단 확인.

### 2026-05-09 — pad_short 옵션 추가 (window_size 미만 trial drop 방지)

- 문제: Alsaify A04(standing) C02 trial은 다운샘플 후 294~299 패킷이라 W=300 미만. 기존 코드는 `n_packets < window_size`이고 `drop_last=True`이면 empty 반환 → 30 subjects × 2 envs 기준 약 240 trial이 조용히 버려짐. 기존 `tail_window`는 `n_packets >= window_size`에서만 동작해 이 문제를 해결하지 못함.
- 수정: `sliding_windows()`에 `pad_short: bool = False` 파라미터 추가. True면 `n_packets < window_size`인 경우 `drop_last`와 무관하게 zero-padding을 뒷부분에 적용해 윈도우 1개 생성. `tail_window`와 독립적으로 동작 (전자는 짧은 trial, 후자는 긴 trial 잔여 처리).
- 영향 범위: `model/preprocessing/window.py`, `model/preprocessing/pipeline.py` (preprocess_file, preprocess_file_full, preprocess_files_full, preprocess_directory_full, _worker_full args tuple), `model/pretrained/train.py` (build_cache에서 `pad_short=True` 추가, 캐시 파일명 `dataset_cache[_e<envs>]_tail_ps.npz`로 변경), `model/preprocessing/test_pipeline.py` (4-case 검증 추가).
- 적용 정책: 사전학습(train.py)만 `pad_short=True`. 기본값 False 유지로 기존 호출자 호환.

### 2026-05-09 — Codex review: preprocess_directory tail_window 옵션 적용 확인

- 확인 커밋: `f4766ac [수정] preprocess_directory tail_window 옵션 추가`.
- 검토 결과: `model/preprocessing/pipeline.py`의 `preprocess_directory()` 시그니처에 `tail_window: bool = False`가 추가되었고, 내부 `preprocess_file(..., tail_window=tail_window)`로 정상 전달됨. 기본값 False 유지로 기존 호출자 호환성도 보존됨.
- 결론: 슬라이딩 윈도우 tail 보정 API 누락 항목은 해결됨.

### 2026-05-09 — Codex review: sliding window tail 보정 API 일관성

- 검토 결과: `sliding_windows(tail_window=True)` 구현과 `train.py` 사전학습 캐시 빌드 경로 적용은 의도대로 동작하는 구조로 확인. 400패킷 trial에서 기존 1개 윈도우만 생성되던 문제는 `amplitude[-300:]` tail 윈도우 추가로 해결됨.
- 추가 발견: `preprocess_directory()`만 `tail_window` 인자를 노출하지 않아, RPCA 이전 윈도우-only 디버깅/분석 경로에서는 새 tail 보정 정책을 적용하지 못할 수 있음. 학습 메인 경로에는 영향 없음.
- 제안 조치: Claude Code에서 `model/preprocessing/pipeline.py`의 `preprocess_directory()` 시그니처에 `tail_window: bool = False`를 추가하고 내부 `preprocess_file(..., tail_window=tail_window)`로 전달. 기본값 False 유지.

### 2026-05-09 — sliding window tail loss 수정

- 문제: Alsaify 320Hz→100Hz 다운샘플링 시 trial당 ~400 패킷이 되는데 W=300, stride=300, drop_last=True 조합에서 윈도우 1개([0:300])만 생성되고 나머지 ~100 패킷이 버려짐.
- 수정: `sliding_windows()`에 `tail_window: bool = False` 파라미터 추가. True면 정방향 슬라이딩 후 잔여 패킷이 있을 때 `amplitude[-window_size:]` 슬라이스 윈도우 1개를 추가 (overlap 발생 가능, drop_last=False의 zero-padding보다 정보 손실 적음).
- 영향 범위: `model/preprocessing/window.py`, `model/preprocessing/pipeline.py` (preprocess_file, preprocess_file_full, preprocess_files_full, preprocess_directory_full, _worker_full args tuple), `model/pretrained/train.py` (build_cache에서 tail_window=True 사용, 캐시 파일명 `dataset_cache[_e<envs>]_tail.npz` 로 변경하여 기존 캐시와 분리), `model/preprocessing/test_pipeline.py` (4번 단계 직후 검증 추가).
- 적용 정책: 사전학습(train.py)만 tail_window=True. 실시간 추론 서버는 이번 수정 범위 아님. 기본값 False 유지로 기존 호출자 호환.

---

## Pending Items

- [x] WIFI_PS_NONE 적용 후 WALK 세션 재수집 → 페어 카운트 안정성 검증 (2026-05-12 완료. 8초 세션 570~652 페어, burst→idle 패턴 소멸. 700~800 목표는 미달이고 ~70Hz 천장 발견 → [D-018]로 처리)
- [x] 3순위 fix(TX broadcast → unicast) 적용 여부 결정 (2026-05-12 적용 + 효과 확정. broadcast 3Hz → unicast ~70Hz)
- [ ] PS 비활성화 효과 검증 종료 후 RX1/RX2 디버그 카운터·stats_task 정리 (`csi_rx1_main.c`, `csi_rx2_main.c`) — 라우터 환경 재평가 마친 뒤 진행 권장 (D-018 후속)
- [x] self-collected 100Hz 리샘플 구현 (2026-05-13 완료. `model/preprocessing/resample.py` 신규 + loader/pipeline 확장. scipy `interp1d` 대신 `np.interp` 사용 — 결과 동일, 의존성 -1. ResampleResult metadata에 gap_count/max_gap_us/original_rate_hz 노출. Alsaify 경로 무변경.)
- [ ] portable router 확보 시 70Hz 천장 해소 가능성 재평가, 리샘플 필요성 재판단 ([D-017]/[D-018] 후속)
- [ ] fine-tuning 진입 전 `train.py`/캐시 빌더에 SafeSignal CSV 전용 경로 연결 (`preprocess_safesignal_file*`)
- [ ] SDP z-score A안(Global) 학습/평가 후, 동일 split에서 B안(Per-lag) 재학습 ablation 수행 및 최종 정규화 방식 결정 ([D-020] 후속. A안 구현 자체는 main `14bfb12`로 2026-05-17 완료)
- [ ] E2E 실시간 추론 단계에서 `server/inference/buffer.py`를 timestamp-aware 100Hz resampling buffer로 전환
- [ ] `wifi_event_group` 미사용 잔재 변수 정리 (`csi_rx1_main.c:86`, RX2 동일)
- [x] Alsaify 전체 사전학습 실행 (2026-05-14 팀원 결과 수령·로컬 적용 — best fall_recall=0.919 / F1=0.913 / FAR=0.022, meets_all_targets=true)
- [x] `preprocess_directory()`에 `tail_window` 옵션 추가 (윈도우-only 디버깅/분석 API 일관성 보완)
- [ ] Sliding window size 실험적 결정 (데이터 수집 후)
- [ ] 자체 수집 계획 최종 구조 확정 (W3 이전)
- [ ] 보호자 알림 상세 시나리오 확정 (동석 담당, SOLAPI vs KakaoTalk 포함)
- [ ] ESP32 3대 배터리 런타임 정량화
- [ ] GitHub 브랜치 전략 확정
- [ ] 포터블 라우터 사용 가능 여부 확인
- [ ] RTX4060에서 window_to_model_input() 단일 윈도우 latency 실측 → stride 최종 확정 (D-014 후속)
- [x] server/dongseok + feature/pretrained-model → main 브랜치 통합
- [x] inference/ 모듈 구현 (InferenceWorker, 슬라이딩 윈도우 버퍼, 결과 큐)

---

## Milestones

| Week | 날짜 | 목표 |
|------|------|------|
| W3 | 2026-05-14 | 전처리 파이프라인 완성, UDP 안정화, 1차 데이터 수집 |
| W4 | 2026-05-21 | 2차 수집 완료 (목표 210세션), fine-tuning MVG 1차 |
| W5 | 2026-05-28 | E2E 통합, 2환경 검증 |
| W6 | 2026-06-04 | Demo |
| W7 | 2026-06-11 | 최종 발표 |



