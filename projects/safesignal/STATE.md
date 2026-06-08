# SafeSignal Project State

_Last updated: 2026-06-09 (데모 머신 RTX4060/i5-13500HX 실시간 추론 병목 해소 Phase 0~4 실행계획 반영; Phase 1 baseline 후 Phase 2 범위 결정; RPCA multiprocessing/max_iter sweep/threshold 운영점 순서 확정) | Updated by: codex_

---

## ▶ NEXT ACTION (2026-06-10 데모 머신 실시간 추론 병목 해소) — Phase 0+1부터 시작

> 데모 머신: 팀원 노트북 RTX4060 / i5-13500HX / 64GB RAM.
> 목표: CPU RPCA 병목으로 인한 stall·pair drop·gap garbage를 제거하고, 같은 운영점으로 replay/라이브 데모를 검증한다.
> 원칙: 모델 재생성 없음(D s42, git f186764). shadow/pre-gate/warm-start 신규 구현은 후순위. 각 Phase 종료 후 보고·멈춤. 충돌, 수동 변경, 품질 저하는 자동 진행하지 말고 보고한다.

### Phase 0) 환경 정합 — 측정 전제

1. `git status --short`, branch, 최근 log로 gate1 최신(`f186764`) 여부 확인.
   - git에 없는 수동 변경 파일이 있으면 멈추고 보고. 덮어쓰지 않는다.
   - 최신이 아니면 `git pull`; 충돌 시 자동 머지 금지, 멈추고 보고.
2. D s42 checkpoint 확인:
   - `model/finetune/checkpoints_item4_balanced/onset_primary_balanced_aug_s42/best_operating.pt`
3. E3/E4 정적/fall CSV 확인: `data/cleaned`.
4. CUDA 확인: `torch.cuda.is_available() == True` 및 GPU명 RTX4060. False면 CUDA PyTorch 설치 필요로 보고.
5. 운영 env 확인:
   - `SAFESIGNAL_FALL_CONSECUTIVE_N=1` (N=2는 낙상 단발 transient를 죽여 폐기)
   - `SAFESIGNAL_ENERGY_GATE_ENABLED=False` (SDP energy gate는 낙상/정적 분리 실패로 폐기)
   - `SAFESIGNAL_INFERENCE_STRIDE=100`
   - `SAFESIGNAL_MODEL_PATH`는 D s42 checkpoint
   - energy gate, shadow/pre-gate가 켜져 있으면 끄고 보고. 새 pre-gate 구현은 하지 않는다.

### Phase 1) 베이스라인 측정 — 최대 분기점

CSV replay로 짧게 측정 후 보고·멈춤.

- warm-up 3회 제외 후 20~30 window 측정.
- `compute_window_energy`/RPCA ms, CNN ms, full predict ms 각각 mean/p50/p95 기록.
- CUDA 측정 시 CNN 전후 `torch.cuda.synchronize()`로 비동기 시간을 보정.
- 같은 replay window set을 이후 max_iter/tol 품질 비교에도 고정한다.

판정:

- RPCA 200에서 p95 <= 1.5s: 구조 변경 최소화, Phase 2 스킵/최소화 가능.
- p95 1.5~3s: CUDA CNN + RPCA max_iter/tol 조정.
- p95 > 3s: max_iter/tol + multiprocessing 풀세트 검토.

### Phase 2) stall 해결 — 2-A 단일 latency, 2-B throughput

#### 2-A. 비병렬 단일 latency p95 <= 1.5s 목표

1. CUDA CNN 사용으로 CNN 병목 제거.
2. RPCA `max_iter` sweep: `200 -> 180 -> 160 -> 150`, 필요 시 `120 -> 100`까지 확장.
   - 선택 기준은 latency 최저가 아니라 replay recall/FAR 유지되는 최속값.
   - 학습은 200 기준이므로 추론 `max_iter` 축소는 분포 차이 리스크를 품질로 확인한다.
3. 부족 시 `tol` 완화 sweep 추가.
   - 현 기본은 `1e-7 * ||D||_F`로 실시간 추론에는 과엄격할 수 있음.
   - `1e-6`, `1e-5` 계열 후보를 같은 window set으로 비교.
4. warm-start는 후순위. 현재 API 없음, 병렬과 충돌, 분포 리스크가 있어 데모 우선 적용 대상 아님.

#### 2-B. multiprocessing으로 throughput >= 1 win/s 및 1s 근접

worker 생성 전 BLAS thread 제한:

```powershell
$env:OMP_NUM_THREADS="1"
$env:MKL_NUM_THREADS="1"
$env:OPENBLAS_NUM_THREADS="1"
$env:NUMEXPR_NUM_THREADS="1"
```

- window 단위 RPCA multiprocessing: workers `2 -> 3 -> 4`만 실측. 5+는 SVD/BLAS 경합 가능성이 커서 비추천.
- 각 window에 `window_id`, `start_ts`, `end_ts`를 붙여 전처리 worker로 전달.
- 결과는 `window_id/end_ts` reorder buffer에서 정렬 후 CNN에 전달.
- bounded queue로 backlog/drop 방지.
- 너무 오래된 out-of-order 결과는 discard하되 로그는 남긴다. N=1 운영이라 알림 순서 의존은 낮지만 로그 해석에는 중요하다.

제외: joint PCA, RPCA 알고리즘 전체 교체, energy/pre-gate 신규 구현.

### Phase 3) threshold 운영점 확정 — 0.5 고정 금지

stall 해결 후 첫 작업.

1. replay fall 60 window의 `fall_conf` 분포 확인.
   - `>0.5` 수, `0.15~0.5` 수, subtype별 약한 구간(SIT_F, S02 등) 확인.
2. threshold sweep: `0.3`, `0.4`, `0.5` 각각 replay recall/FAR/F1 산출.
   - 기존 `0.05`, `0.15` 값은 비교 후보로만 둔다.
3. 선택 기준: demo recall 합격선을 유지하는 가장 높은 threshold(FAR 최소).
   - replay와 라이브 데모가 같은 threshold를 쓰도록 확정한다.

### Phase 4) 라이브 데모 검증

RTX4060 + ESP32 라이브, Phase 3 threshold 사용.

1. 낙상 5회: 4~5/5 감지 목표.
2. 정적 1~2분: 오발 빈도 확인.
3. 부족하면 threshold 미세조정. 최종 기준은 replay 수치와 live 검증을 함께 기록한다.

### 보존 / 산출물

- 각 Phase 산출물(베이스라인, max_iter/tol sweep, conf 분포, threshold 평가, live 결과)을 디스크 저장.
- 코드 변경은 config/env 토글 우선, 변경 전 백업.
- 전체 종료 후 커밋·푸시. `--force` 금지. 대용량 산출물은 git 추적 여부 별도 판단.

### Codex 보강 메모

- Phase 1이 최대 분기점이다. RTX4060에서 RPCA 200 p95가 이미 1.5s 근처면 풀세트 최적화로 들어가지 않는다.
- max_iter 선택은 latency가 아니라 recall/FAR 유지되는 최속값 기준이다.
- multiprocessing 성패는 BLAS thread=1 제한에 달려 있다.
- Phase 1/2 품질 비교는 같은 replay window set으로 수행해야 한다.

---
## ▶ NEXT ACTION (2026-06-09 새 환경 즉시 실행 런북) — 비낙상 포함 balanced 증강

> 새 환경(재부팅 시 C 초기화)에서 "ai-workspace state 참고해서 더미데이터 생성 및 학습 진행해줘"
> 라고 하면 이 런북대로 바로 진행한다. 코드/실행 상세는 wifi-csi-fall-detection 의
> `debug/modeling/diag_out/onset_detector/finalization/HANDOFF.md` balanced 섹션과 동일(중복 기재).
> 설계 잠금(재논의 금지)은 아래 Review Notes 2026-06-08 "비낙상 포함 균형 증강 Q1~Q6" 참조.

### 0) 부트스트랩 (D드라이브 영구, C 초기화 환경)
```powershell
cd D:\Project\LastProject\wifi-csi-fall-detection
. .\tools\env_bootstrap.ps1
git pull   # feature/event-centered-gate1 (balanced 스크립트 커밋 a44b9ac 포함)
```

### 1) preflight — 없으면 스크립트가 fail-fast (스펙 0-1)
- `debug/dummy_gen/out2/dummies_clean400.npz` — git에 없음(대용량 ignore). Drive/백업 복원. lineage(696)/use(301) 정합 확인.
- `debug/modeling/diag_out/onset_detector/item4/eval_windows.pkl` — 없으면 `python debug/modeling/item4_precompute_eval_windows.py` 먼저.
- A/B 가중치 `model/finetune/checkpoints_item4_reg/{fixed,onset_primary}_s{42..46}/best_operating.pt` — gitignore라 학교 PC 로컬에만. config는 balanced(epochs40/wd1e-4/patience8/early_stop8)와 일치 확인됨.

### 2) 실행 순서
```bash
python debug/modeling/item4_train_balanced.py --check-ab-reuse                 # (1) A/B 재사용 가능 여부 리포트
python debug/dummy_gen/balanced/nonfall_dummy_generate.py                      # (2) non-fall 더미(robust gate p97.5)
python debug/modeling/item4_build_balanced_caches.py                          # (3) C/D balanced cache
python debug/modeling/item4_train_balanced.py --smoke                        # (4) smoke(preflight/share tol/counter/1ep)
python debug/modeling/item4_train_balanced.py --policies fixed_balanced_aug onset_primary_balanced_aug \
  --seeds 42 43 44 45 46 --epochs 40 --weight-decay 1e-4 --patience 8 --early-stop-start 8 \
  --ckpt-root checkpoints_item4_balanced                                       # (5) 본학습 C/D
python debug/modeling/item4_balanced_eval.py --require-seeds 5 --ab-root auto # (6) 평가
```
- (1)에서 A/B 재사용 불가로 나오면 (5) 앞에 재학습:
  `python debug/modeling/item4_train_balanced.py --retrain-ab --seeds 42 43 44 45 46 --ckpt-root checkpoints_item4_balanced`
  (이 경우 (6)의 `--ab-root auto` 가 balanced 재학습본을 자동 우선)

### 3) 설계 잠금 요지 (상세 = Review Notes 2026-06-08)
- arm: A=fixed / B=onset_primary / C=fixed_balanced_aug / D=onset_primary_balanced_aug. Primary 대비 = D-B, Secondary = C-A, Interaction 참고.
- effective share arm별 보존(C=A, D=B), tol 1e-3 초과 시 학습 전 fail-fast. (검증: A/B 모두 maxerr <5e-5)
- 더미 train-only, val/test 봉인, threshold/test 튜닝 금지. 더미 online 증강 skip(marker 기반 실측 검증).
- 판정: statistically_supported_improvement / _regression / _mixed / directional_candidate / null (방향 분리).

### 4) 산출물 분류
- git 추적: 신규 스크립트 4개, `debug/dummy_gen/balanced/nonfall_lineage.csv`·`*quality_report*`, `debug/modeling/balanced_aug/*.json|md|csv`, HANDOFF.
- Drive 업로드(ignore): `nonfall_dummies.npz`, `item4_cache_*_balanced_aug.npz`, `model/finetune/checkpoints_item4_balanced/`.

### 실행 중 주의
non-fall 생성기 `nonfall_quality_report` 의 `sanity_max_abs_diff`(재생성 raw→z-SDP vs cache) 확인 — 크면(>1e-2) orig cache RPCA 파라미터 불일치 신호(더미 분포 밖 위험). 기본 warning 진행, 의심 시 `--strict-allclose`.

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
  - Intel 5300, 30 subcarriers × 3 Rx antenna = 90 CSI feature columns, 320Hz, 30 subjects × 5 experiments × 20 trials
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
  - 결정: self-collected CSV는 **전처리 로더 단계에서 timestamp 기반 100Hz 균일 격자 보간 리샘플** 적용. scipy `interp1d` 선형 보간, 캡처 범위 안쪽만 채움(외삽 없음). Alsaify는 기존 pretrained 전처리 경로에서 320Hz → 100Hz downsample을 적용하므로 SafeSignal timestamp 보간 경로는 미적용.
  - 적용 위치: `model/preprocessing/loader.py` (또는 self-collected 전용 로더). 원본 CSV는 그대로 두고 로딩 단계에서만 변환. 실시간 추론 측은 별도 검토 — 페어 incoming 시점에 동일 보간 적용 필요(stride 누적 + 시점 보간).
  - 가시화 보강: `collect/collect_main.py` `_run_session` 출력에 `pair_rate` / `capture_ratio` 추가, 운용 중 천장 변동 즉시 확인.
  - 라우터 확보 시 리샘플 필요성 재평가 — Pending Items에 등록.
- **Status:** confirmed (2026-05-13 구현 완료 — `f25f018 SafeSignal 자체수집 100Hz 리샘플 전처리 추가`)

### [D-019] 자체수집 평가 분할 + 베이스라인 측정 흐름 확정
- **Date:** 2026-05-14
- **Decided by:** user / claude-code
- **Content:**
  - **분할:** [D-021]/[D-022] 반영 후 자체수집은 **3-fold cross-subject**로 평가한다. subject는 피험자를 뜻하며, `S01`/`S02`/`S03`는 피험자 코드다. E4는 test 전용 환경으로 봉인하지 않고 subject 기준으로 train/test에 함께 분배한다.

    | Fold | Train | Test |
    |------|-------|------|
    | Fold 1 | `E2_S02`, `E3_S03`, `E4_S02`, `E4_S03` | `E1_S01`, `E4_S01` |
    | Fold 2 | `E1_S01`, `E3_S03`, `E4_S01`, `E4_S03` | `E2_S02`, `E4_S02` |
    | Fold 3 | `E1_S01`, `E2_S02`, `E4_S01`, `E4_S02` | `E3_S03`, `E4_S03` |
  - 각 fold에서 test subject의 E4 포함 전체 raw 데이터는 학습/증강/모델 선정/하이퍼파라미터 튜닝/threshold 결정에 일체 사용 금지한다. `groups=subject` 기준의 non-overlapping group split으로 취급한다.
  - **흐름:** 3명·4환경 수집 완료 → fold별 봉인 subject로 현 best.pt 베이스라인 평가(6-class 기준) → train split만 증강(5×, D-010/D-022) → fold별 파인튜닝(7-class) → 동일 fold test로 비교 평가. 베이스라인 vs 파인튜닝 개선폭은 6-class 교집합으로 보고하고 7-class 전체 metrics는 보조 지표.
  - **증강 적용 범위:** 학습 파이프라인의 train split 내부에서만 호출. test/val 에는 raw 그대로. `augment/augment.py` 호출 위치가 split 이후인지 코드 점검 항목으로 추가.
  - **running 클래스:** 사전학습 best.pt는 6-class (running 없음, D-006) → 베이스라인 평가에서 자체수집의 running 세션은 제외하고 6-class만 채점. 파인튜닝 모델은 7-class로 학습·평가.
  - **fall 수집 우선순위:** D-011 핵심 지표가 fall_recall이고 fall 세션 부족 시 평가 자체가 무의미 → 수집 일정상 전체 세션을 전부 못 채워도 각 subject·environment의 fall 세션 우선, walking/sit_stand 차순위, picking/lying 최후순위.
  - **베이스라인 평가 산출 항목 (필수):** fold별/subject별 클래스별 confusion matrix, FALL_THRESHOLD sweep(0.3~0.7), macro 평균과 subject별 metrics. 파인튜닝 설계 입력으로 사용.
  - **보조 분석:** E4는 별도 환경 일반화 분석으로도 보고한다. 단, primary split은 3-fold cross-subject이며 E4-only 결과는 subject leakage 가능성이 있으므로 최종 성능의 단독 근거로 쓰지 않는다.
- **Ref:** [scikit-learn GroupKFold](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GroupKFold.html), [scikit-learn cross-validation user guide](https://scikit-learn.org/stable/modules/cross_validation.html), [statsmodels Wilson proportion CI](https://www.statsmodels.org/stable/generated/statsmodels.stats.proportion.proportion_confint.html)
- **Status:** confirmed (2026-05-18 재검토 완료 — 3-fold cross-subject 채택, 봉인 대상 S01/S02/S03 fold별 명시)

### [D-020] SDP z-score 정규화 적용 및 ablation 계획
- **Date:** 2026-05-17
- **Decided by:** user / codex review
- **Content:**
  - 현재 SDP 출력 `(28, 20)`에 window 단위 z-score 정규화를 추가하기로 결정.
  - 1차 구현/적용안은 **A안 Global z-score**: `mean=sdp.mean()`, `std=sdp.std()`, `sdp=(sdp-mean)/(std+eps)`. A안은 ACF lag 간 상대 크기/decay 구조를 보존하면서 개인 간 진폭 편차와 Intel 5300 ↔ ESP32 scale gap을 완화하는 안정적 기본안으로 채택.
  - **B안 Per-lag z-score**: `mean=sdp.mean(axis=0, keepdims=True)`, `std=sdp.std(axis=0, keepdims=True)`. B안은 각 lag별 시간축 변화량을 강조해 SafeSignal의 "낙상 순간 급격한 변화" 설명과 잘 맞지만, high-lag noise 과증폭 및 FAR 증가 가능성이 있어 ablation 후보로 유지.
  - fine-tuning 진입 후 먼저 A안으로 학습한 모델의 평가 지표를 확보한 뒤, **동일 split**에서 B안 전처리로 재학습하여 지표를 비교하고 최종 정규화 방식을 결정.
  - A/B 비교 시 `fall_recall`, `FAR`, `fall_f1`, confusion matrix, 특히 `walking/picking/sit_stand → fall` 오탐을 함께 확인. B안 적용 시 `std_floor`와 clipping(`[-5,5]` 또는 `[-3,3]`) 적용을 검토.
- **Status:** A안 Global z-score 구현 완료 및 최종 유지 확정 (main `14bfb12` — 2026-05-17). `window_to_model_input()` SDP 직후에 `(sdp - sdp.mean()) / (sdp.std() + 1e-6)` 적용. Alsaify·SafeSignal 양 경로 공통. 후속 main `8c85920` (2026-05-17)에서 `rpca_sparse()` degenerate 입력 방어 추가. 2026-06-03 정식 트랙1 ablation(`debug/modeling/diag_out/track1_formal_comparison.json`)에서 B안 per-lag는 A안 대비 recall -0.030, FAR +0.016, F1 -0.033으로 net negative라 기각. 최종 정규화는 A안 global z-score 유지.

### [D-021] 자체수집 환경 구성 및 피험자 코드 확정
- **Date:** 2026-05-18
- **Decided by:** user / codex
- **Content:**
  - 자체수집 환경:
    - E1: 주화 개인 주거 환경 — S01 단독 수집
    - E2: 진규 개인 주거 환경 — S02 단독 수집
    - E3: 동석 개인 주거 환경 — S03 단독 수집
    - E4: 학교 실습 장소 — S01·S02·S03 3명 함께 수집
  - 피험자 코드:
    - S01 = 주화
    - S02 = 진규
    - S03 = 동석
  - 펌웨어 NVS IP 수정 완료:
    - TX(COM15): target `10.197.192.132:5000` → `192.168.137.1:5005`
    - RX1(COM11): server `10.197.192.132:5005` → `192.168.137.1:5005`
    - RX2(COM14): server `10.197.192.132:5005` → `192.168.137.1:5005`
    - 수정 방법: `idf.py monitor` 상태에서 `set_target` / `set_server` 명령으로 NVS 갱신
    - 검증: UDP 패킷 수신 정상 확인 완료 (`magic=0xab`, 224B, RX1·RX2 양쪽)
- **Ref:** [D-007] UDP 패킷 구조, [collect/labels.py](https://github.com/LeapSeeker/wifi-csi-fall-detection/blob/main/collect/labels.py)
- **Status:** confirmed

### [D-022] 자체수집 전체 세션 수 재산정
- **Date:** 2026-05-18
- **Decided by:** user / codex
- **Content:**
  - 코드/서버의 기존 수집 목표를 유지한다. `collect/labels.py` 구현 기준은 env/subject 조합 1개당 낙상 9종×10회=90세션, 비낙상 6종×30회=180세션, 총 270세션이다.
  - 이미 팀원이 서버 표시 기준으로 수집을 진행했으므로, 수집 중간에 목표 횟수를 줄이지 않는다.
  - 최종 자체수집 목표: **1,620세션**.
    - E1: S01 270세션
    - E2: S02 270세션
    - E3: S03 270세션
    - E4: S01 270세션 + S02 270세션 + S03 270세션 = 810세션
  - 전체 클래스 합계:
    - fall: 540세션
    - walking / sit_stand / lying / standing / running / picking: 각 180세션
    - non-fall 합계: 1,080세션
  - env/subject 조합별 270세션 구성:
    - 낙상: `FALL_SIT_F`, `FALL_SIT_B`, `FALL_SIT_S`, `FALL_STD_F`, `FALL_STD_B`, `FALL_STD_S`, `FALL_WALK_F`, `FALL_WALK_B`, `FALL_WALK_S` 각 10회 = 90세션
    - 비낙상: `SIT_STD`, `LIE`, `WALK`, `STAND`, `RUN`, `PICK` 각 30회 = 180세션
  - 이 구조는 D-019의 3-fold cross-subject 평가를 유지한다. 같은 subject가 private 환경과 E4 환경을 모두 가지므로 subject 일반화와 environment 일반화를 분리해서 보조 분석할 수 있다.
- **Ref:** [collect/labels.py](https://github.com/LeapSeeker/wifi-csi-fall-detection/blob/main/collect/labels.py), [scikit-learn GroupKFold](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.GroupKFold.html), [scikit-learn cross-validation user guide](https://scikit-learn.org/stable/modules/cross_validation.html)
- **Status:** confirmed

### [D-023] SafeSignal fine-tuning 데이터 축소 및 학습/검증 기준 확정
- **Date:** 2026-05-19
- **Decided by:** user / claude-ai / codex
- **Content:**
  - **Class 구성:** fine-tuning은 `running`을 포함한 7-class로 진행한다. 최종 순서는 `fall`, `walking`, `sit_stand`, `lying`, `standing`, `running`, `picking`으로 둔다.
  - **pretrained head 이식:** 현재 `best.pt`는 6-class이며 `classifier.1.weight` shape은 `(6, 256)`, class 목록은 `fall`, `walking`, `sit_stand`, `lying`, `standing`, `picking`이다. 7-class head로 교체할 때 기존 `picking` weight를 새 index 6으로 옮기고, 새 `running` row(index 5)는 새로 초기화한다. 단순 앞 6행 복사는 `picking` weight가 `running` 자리에 들어가므로 금지한다.
  - **fall 세션 축소:** 수집 부담과 class imbalance 완화를 위해 side fall activity 3종(`FALL_SIT_S`, `FALL_STD_S`, `FALL_WALK_S`)은 fine-tuning 기본 수집/학습/평가 범위에서 제외한다. 신규 수집은 `FALL_*_F`, `FALL_*_B` 6종 × 10회 = fall 60세션, non-fall 6종 × 30회 = 180세션, env/subject 조합당 총 240세션으로 진행한다.
  - **fine-tuning 수집 규모:** 대상 env-subject 조합은 6개(`E1-S01`, `E2-S02`, `E3-S03`, `E4-S01`, `E4-S02`, `E4-S03`)로 둔다. 조합당 240 원본 세션이므로 SafeSignal 원본 총합은 1,440세션이다. 증강 5배 기준의 최대 학습 샘플 규모는 7,200개이나, 이는 수집 세션 수가 아니라 train split에만 증강을 적용했을 때의 학습 샘플 규모로 표현한다. validation/test는 raw-only 유지한다.
  - **방향 변이:** side fall을 별도 activity로 수집하지 않는 대신 forward/backward 낙상은 방향을 유지한 범위에서 제한적 각도 변이를 포함한다. 권장: 각 fall activity 10회 중 중앙 4회 + 좌대각 3회 + 우대각 3회, 대략 ±30도 이내. 90도에 가까운 실제 side fall은 F/B label에 섞지 않는다.
  - **기존 side CSV 처리:** 이미 수집된 `FALL_*_S` CSV가 있으면 기본 train/validation/test에서 제외하고, 제외 파일 수를 로그에 출력한다.
  - **Loss:** baseline은 unweighted `CrossEntropyLoss`. ablation은 fall class custom weight 1.2 / 1.5를 비교한다. inverse-frequency weighted CE는 7-class 기준에서 fall weight를 낮춰 fall recall 우선 전략과 충돌하므로 사용하지 않는다. fall weight 2.0 이상은 FAR 증가 위험으로 기본 후보에서 제외한다.
  - **Backbone freeze:** 이 항목의 partial freeze 기본안은 [D-027]로 대체됨. 최종 fine-tuning 정책은 combined training + full unfreeze + 5 epoch warmup이다.
  - **Threshold 선택:** fold별 test 최적화는 test leakage로 간주하고 금지한다. train split 내부 validation prediction을 3-fold 전체에서 pooled하여 global threshold 1개를 선택한 뒤, 모든 fold test에 동일 threshold를 고정 적용한다.
  - **Threshold selection rule:** ① `fall_recall ≥ 0.90` and `FAR ≤ 0.10` 만족 threshold 중 `fall_f1` 최대, ② 없으면 공식 목표 `fall_recall ≥ 0.85` and `FAR ≤ 0.15` 만족 threshold 중 `fall_f1` 최대, ③ 없으면 recall 우선 및 FAR 초과폭 최소 threshold 선택 후 별도 표시한다.
  - **Checkpoint:** `last.pt`, `best_val_loss.pt`, `best_operating.pt`를 저장한다. 최종 배포/보고 기준은 `best_operating.pt`이며, `best_fall_recall.pt`는 저장하더라도 참고용으로만 취급한다.
  - **Pass/Fail:** 공식 통과 기준은 3-fold mean 기준 `fall_recall ≥ 0.85`, `mean FAR ≤ 0.15`, `fall_f1 ≥ 0.85`. stretch 기준은 `fall_recall ≥ 0.90`, `mean FAR ≤ 0.10`.
  - **Fold flag:** 평균 기준과 별도로 fold outlier를 flag로 보고한다. `WARN_FAR_FOLD`: fold FAR > 0.15, `HIGH_RISK_FAR_FOLD`: fold FAR > 0.20, `WARN_RECALL_FOLD`: fold recall < 0.85, `HIGH_RISK_RECALL_FOLD`: fold recall < 0.75.
  - **재수집 후보:** HIGH_RISK flag가 발생하거나 `running/picking/sit_stand/walking → fall` 오탐이 여러 fold 또는 동일 subject의 private/E4 환경에서 반복되면 해당 subject/action 재수집 또는 추가 수집 후보로 판단한다. 단일 fold의 WARN 수준 이탈은 전체 실패가 아니라 subject/environment 특성 주석으로 처리한다.
  - **검증기 출력:** global threshold, mean metrics + pass/fail, fold별 metrics/flag, confusion matrix, 반복 오탐 패턴, excluded side-fall file count를 출력한다.
- **Ref:** [D-019] 3-fold cross-subject 평가, [D-022] 기존 자체수집 목표, `collect/labels.py`, `model/pretrained/checkpoints/best.pt`
- **Status:** confirmed

### [D-024] Fine-tuning HPO 적용 방식 확정
- **Date:** 2026-05-21
- **Decided by:** user / claude-ai / codex
- **Content:**
  - fine-tuning 하이퍼파라미터 자동 탐색은 **Optuna TPESampler + PatientPruner 기반 Bayesian Optimization**으로 확정한다.
  - 적용 타이밍은 manual baseline 1회 실행 후 분기한다. 결과가 목표에 근접하면 manual ablation으로 마무리하고, 목표에 미달하면 Optuna 20~30 trials를 즉시 실행한다. 다만 일정 지연을 막기 위해 지금 단계에서 `run_training()` 결과 반환, HPO mode 플래그, objective wrapper 최소 구조는 준비한다.
  - 1차 search space: `source_ratio={0.50,0.55,0.60,0.65}`, `hard_weight=1.10~1.50`, `attention_lr={1e-4,3e-4,5e-4}`, `head_lr={5e-4,1e-3}`.
  - 2차 search space: `fall_weight={1.0,1.2,1.5}`, `backbone_lr={3e-5,1e-4}`. `threshold`는 HPO 탐색 대상이 아니며 각 trial 내부 validation sweep으로 선택한다.
  - Objective는 accuracy를 제외하고 `fall_f1`, recall 부족 penalty, FAR 초과 penalty를 사용한다. 공식 목표(`recall >= 0.85`, `FAR <= 0.15`)를 만족하는 후보를 살리고, stretch 목표(`recall >= 0.90`, `FAR <= 0.10`)에는 bonus를 부여한다.
  - Pruning은 warmup 5 epoch 동안 금지하고 epoch 10 이후부터 PatientPruner를 적용한다. HPO 과정에서 sealed test fold는 절대 사용하지 않고, best trial 확정 후 딱 1회만 sealed test를 실행한다.
- **Status:** confirmed

### [D-025] 패킷 동기화 및 100Hz 선형보간 품질 관리 보완
- **Date:** 2026-05-21
- **Decided by:** user / codex review
- **Content:**
  - 현재 Rx1/Rx2 페어링은 `seq_num`이 아니라 `timestamp_us` 기반 nearest match로 수행되며, 한쪽 패킷 손실 시 seq가 어긋날 수 있으므로 이 방향은 유지한다.
  - 현재 허용 오차 `PAIR_TOLERANCE_US=50ms`는 70Hz 기준 패킷 간격(약 14ms)에 비해 넓다. 실제 수집 후 `abs(rx1_ts - rx2_ts)` 분포(p95/p99)를 확인하고, 안정적이면 20ms 안팎으로 축소하는 것을 검토한다.
  - 현재 CSV row의 `timestamp_us`는 Rx1 timestamp만 저장한다. 학습/추론 일관성은 유지되지만 Rx2 timestamp와 pair delay를 사후 검증하기 어렵기 때문에 후속으로 `pair_dt_us`, 가능하면 `timestamp_rx1_us`/`timestamp_rx2_us`를 수집 로그 또는 CSV metadata에 남긴다.
  - 오프라인 전처리의 100Hz 선형보간은 timestamp 정렬, 중복 timestamp 평균, 외삽 금지로 구현되어 있어 기본 방향은 타당하다. 단, `max_gap_ms`는 현재 hard reject가 아니라 metadata 집계용이므로 큰 timestamp gap이 있는 window를 warning/skip 처리하는 정책을 추가한다.
  - 실시간 추론 버퍼도 같은 100Hz 선형보간 가정을 사용하지만 `max_gap_ms` 설정을 실제로 사용하지 않는다. offline preprocessing과 realtime inference의 gap 처리 정책을 맞추는 것을 후속 구현 대상으로 둔다.
- **Status:** confirmed

### [D-026] 공유기/채널 고정 효과와 WiFi 간섭 한계 정리
- **Date:** 2026-05-21
- **Decided by:** user / codex
- **Content:**
  - 공유기 사용은 핫스팟 대비 채널 고정, 연결 안정성, 수집 조건 재현성 개선에는 도움이 된다.
  - 다만 공유기를 사용해도 해당 채널을 독점하는 것은 아니므로 주변 AP의 동일 채널/인접 채널 사용, 신호 세기, 다중경로 변화로 인한 WiFi 간섭은 남는다. 이는 CSI 기반 시스템의 구조적 한계로 취급한다.
  - 패킷 손실률은 1~2%면 거의 문제 없고, 10~15%는 100Hz 보간과 window 기반 모델로 사용 가능할 가능성이 있으나 완전한 상태는 아니다. 평균 손실률보다 낙상 순간 근처의 연속 gap, timestamp gap, pair delay 분포가 더 중요하다.
  - 라우터 확보 시 리샘플 필요성 자체를 폐기하기보다, 채널 고정 후 pair rate, loss rate, max gap, pair delay 분포가 얼마나 개선되는지 재평가한다.
- **Status:** confirmed

### [D-027] Fine-tuning freeze 정책 정정: full unfreeze + warmup 확정
- **Date:** 2026-05-21
- **Decided by:** user / claude-ai / codex
- **Content:**
  - [D-023]의 partial freeze 기본안은 이후 combined training 논의 결과로 대체한다. 최종 fine-tuning 정책은 **combined training + full unfreeze + warmup**으로 확정한다.
  - 이유: Alsaify pretrained model이 validation 지표는 높았지만 SafeSignal 실제 환경에서 sitting/static 상황의 fall false positive가 반복되었고, 이는 단순 classifier head 문제가 아니라 도메인 특성 차이로 인한 backbone feature/temporal representation mismatch 가능성이 크다. 따라서 attention/head만 학습하는 partial freeze보다 backbone까지 낮은 learning rate로 적응시키는 full unfreeze가 목적에 더 맞다.
  - warmup은 5 epoch 고정으로 둔다. warmup 동안 backbone lr은 `1e-5`, 이후 `1e-4`로 증가한다. attention lr은 `3e-4`, head lr은 `1e-3`을 기본값으로 둔다.
  - warmup 5 epoch 동안 early stopping/pruning은 비활성화한다. early stopping 감시는 epoch 10부터 시작한다.
  - partial freeze는 기본 정책이 아니라 ablation 또는 fallback 후보로만 남긴다.
- **Status:** confirmed

### [D-028] Fine-tuning 최종 수집 규모 기준 정리
- **Date:** 2026-05-21
- **Decided by:** user / codex
- **Content:**
  - fine-tuning 기준 SafeSignal 원본 수집 규모는 **1,440세션**으로 둔다.
  - 계산 기준: env-subject 조합 1개당 240세션, 총 6개 env-subject 조합 = 1,440세션.
  - 환경 구성은 4개 환경이며, 이 중 3개 개인 환경은 각 1명씩 수집하고, 나머지 1개 공용 환경(E4)은 3명이 모두 수집한다. 따라서 조합은 `E1-S01`, `E2-S02`, `E3-S03`, `E4-S01`, `E4-S02`, `E4-S03`이다.
  - 240세션 구성은 side fall 제외 기준이며, fall 60세션 + non-fall 180세션이다.
  - 증강 5배 기준 7,200은 원본 세션 수가 아니라 train split에만 증강을 적용했을 때의 최대 학습 샘플 규모로 표현한다.
  - 단, 수집 일정 또는 환경 문제로 일부 개인 환경 수집이 제외될 수 있으므로 1,440세션은 최종 목표 기준이며 실제 학습 규모는 일정에 따라 변경될 수 있다.
  - D-022의 1,620세션/270세션 구조는 side fall을 포함한 이전 전체 수집 기준으로 남기되, fine-tuning 기본 학습/검증/보고 기준은 본 결정(D-028)의 1,440세션/240세션 구조를 따른다.
- **Status:** confirmed

### [D-029] HPO 목표 기준 정렬
- **Date:** 2026-05-21
- **Decided by:** user / codex
- **Content:**
  - HPO objective와 trial selection은 fine-tuning 최종 기준을 따른다.
  - 공식 목표는 `fall_recall >= 0.85`, `FAR <= 0.15`, `fall_f1 >= 0.85`이며, stretch 목표는 `fall_recall >= 0.90`, `FAR <= 0.10`으로 둔다.
  - Optuna trial 내부에서는 threshold를 탐색 파라미터로 두지 않고 validation threshold sweep 결과 metrics를 objective에 넘긴다.
  - sealed test는 HPO 과정에서 사용하지 않으며, best trial 확정 후 최종 1회만 적용한다.
  - 코드 구현 시 이전 skeleton의 operating/stretch 명명 또는 더 엄격한 임계값이 남아 있으면 본 결정과 D-024 기준으로 정렬한다.
- **Status:** confirmed


### [D-030] 2026-06-04 데모 primary class policy를 pretrained6로 확정
- **Date:** 2026-06-02
- **Decided by:** user / codex / claude-code
- **Content:** 데모 primary는 `running`을 제외한 6-class `pretrained6`(`fall`, `walking`, `sit_stand`, `lying`, `standing`, `picking`)로 확정한다. 7-class `finetune7`은 reference-only로 유지한다. 근거는 (1) Alsaify 사전학습 best.pt가 6-class이며 running 클래스가 공개 source에 없고, (2) 2026-06-02 CPU 30epoch within-subject 비교에서 6-class가 7-class보다 fall recall/F1이 높았으며, (3) 7-class run에서 `running→fall`이 false-positive share 약 0.244로 최대 FAR 기여원이었다는 점이다.
- **Ref:** `docs/CODEX_HANDOFF_2026-06-02.md` (commit `237c93f`, "Key Decisions" 및 6-class vs 7-class 표), `model/finetune/train.py` commit `0b25a49` (`--class_policy pretrained6`, 6-class strict load 경로), `debug/modeling/derive_pretrained6_cache.py` commit `9e3064d`.
- **Status:** confirmed (demo primary; [D-019] cross-subject 정식 학술 평가는 별도 유지)
### [D-031] event-centered windowing 설계 7개 항목 확정
- **Date:** 2026-06-05
- **Decided by:** user / claude-ai / codex review
- **Content:**
  - **Scope:** D-020 per-lag ablation 기각 및 step2 후처리 sweep 한계 확인 이후, post-fall 정적 단서 의존을 줄이고 transient supervision을 강화하기 위한 event-centered windowing/labeling 트랙을 설계로 확정했다. 본 결정은 구현 전 설계 결정이며, 실제 코드/성능 판단은 현재 `wifi-csi-fall-detection` 저장소 기준으로 검증한다.
  - **Q-좌표계:** 자체수집 fall 세션은 100Hz 리샘플 후 `resampled_count` 기준으로 정규화한다. `resampled_count < 550` 세션은 제외하고, `>= 550`은 `[:550]`로 절단한 뒤 Stage beep 구간 `[0:50]`, `[150:200]`, `[400:450]`을 제거해 clean400을 만든다. clean400은 `original[50:150] + original[200:400] + original[450:550]`이며, clean 좌표계는 `pre[0:100] + fall[100:300] + post[300:400]`, 명목 onset은 clean frame 100이다. raw CSV 행 수 기준 필터링은 금지한다. 폐기된 더미 데이터의 onset=160 및 명목 original 200 값은 변환 기준점으로 쓰지 않고, 우리 자체수집 기준으로 새 onset metadata를 생성한다.
  - **Q-onset:** onset은 자동 후보 생성 + 선택적 수동 검수 방식으로 확정한다. 1차 자동 신호는 full-session RPCA sparse frame energy의 sustained rise-onset이며, peak 자체나 절대 peak height threshold를 onset/fall 판별 기준으로 쓰지 않는다. 기본 방향은 clean400 frame energy 5-frame smoothing, baseline clean[0:80~100] median/MAD, 탐색 clean[60:220], `baseline_median + k*MAD` 초과가 M frame 지속되는 첫 지점이다. CUSUM/change-point는 2차 후보로만 사용한다. 저장 필드는 `onset_frame_clean`, `onset_frame_original`, `peak_frame_clean`, `source`(`auto_reviewed`/`manual`/`manual_corrected`), `confidence`, 원시 confidence 구성요소(`rise_strength`, `rise_slope`, `peak_distance`, `topk_spread`, `baseline_noise`), `needs_review`, `notes`로 한다. `needs_review` 조건은 rise 범위 밖, peak가 clean[100:300] 밖, confidence 낮음, top-k 흩어짐, rise_strength 낮음, baseline noise 높음, `resampled_count < 550`, 약점 subtype(FALL_WALK_B 등), clean frame 60 미만의 walking-pre 오인 후보를 포함한다. 통과 confidence threshold는 train/val 분포로만 결정한다.
  - **Q-평가:** primary는 event-level 세션 단위로 고정한다. 기존 `debug/modeling/diag_event_sweep.py` 정의를 유지해 fall 세션은 session fire 시 TP, 아니면 FN, non-fall 세션은 1회 이상 fire 시 FP=1, 다중 fire도 세션당 1로 계산한다. `event_FAR = FP / nonfall_sessions`이며 fire 위치는 primary pass/fail에 반영하지 않는다. 비교는 두 층으로 분리한다: baseline-axis는 `threshold_min=0.1`, `N=2`, `margin=on_m0.2`, `stride=50`, `tail_window=False`를 고정해 모델만 비교하고 STATE의 event_recall 0.6528/FAR 0.1111과 직접 대조한다. operating-point는 STATE 60-config grid를 val split에서 재-sweep하고 sealed test에는 `selected_by_val`과 `baseline_fixed_config`를 각 1회만 적용한다. cooldown은 track B 실시간 추론 설계로 이관한다. 보조 지표로 forward/tail window 진단과 timely-detection/latency(`time_origin=onset_frame_clean`, `fire_time_frame=N번째 positive window end_frame`, `latency_s`, `early_fire`, `timely_3s`, `timely_4s`, `late_tp`, first-fire latency median/p90)를 기록하되 primary pass/fail과 섞지 않는다.
  - **Q1 윈도우 정렬:** 모델 입력 window size는 D-004와 동일하게 300 frame으로 유지한다. 정렬 ablation은 fixed `clean[50:350]`과 onset 기준 `[onset-50:onset+250]`을 비교한다. post 분량은 적음/많음 ablation으로 마지막에 확인한다. multi-window shift는 단일 crop 이득 확인 전에는 넣지 않는다.
  - **Q2 라벨 재정의:** post-only window 처리 방식은 A) fall에서 제외, B) 기존 `lying` 클래스로 전환을 ablation한다. transient 포함 판정은 window와 fall interval 겹침 비율의 빡빡/느슨 두 수준을 비교하되, 구체 %는 onset/window manifest의 train/val 겹침 분포를 보고 결정한다. 임의 50% 같은 근거 없는 수치는 사용하지 않는다. 6-class pretrained6 정책을 유지하며, post-only→lying은 기존 class를 재사용하므로 head 구조와 best.pt strict load에 영향을 주지 않는다.
  - **Q3 비낙상 처리:** event-centered 변경은 fall window 정의에만 적용하고 non-fall 세션은 기존 sliding window 정책을 유지한다. non-fall window 수를 물리적으로 제한하지 않고 sampler로만 균형을 잡는다. 기존 WeightedRandomSampler의 source-ratio 정규화와 SafeSignal hard non-fall class weight 구조를 사용한다. post-only→lying 선택 시 lying weight를 선제 조정하지 않고, class별 raw count와 effective sampled mass(`sum(weights[y==class & source==safesignal])`)를 run report에 남긴다. source_ratio는 baseline-axis ablation에서 0.60/0.40으로 유지하고, source_ratio sweep은 operating optimization으로 분리한다.
  - **Q4 윈도우 수 확보:** multi-window는 조건부 보조 수단으로만 둔다. 발동 신호는 train fall window 수가 baseline의 50~60% 미만, effective fall mass 과소, 단일 crop이 `late_tp`는 줄였지만 recall이 부족한 경우를 함께 본다. 단일 crop이 recall/FAR/late_tp를 모두 개선하면 fall count가 줄어도 shift는 보류한다. shift 폭은 onset/window manifest의 train/val 분포로 결정한다. multi-window 사용 시 class별 raw count/effective sampled mass, session별 window count/mass p50/p90/max, fall session당 shift 수 분포를 추가 로깅하고, session over-representation이 실제 확인될 때만 session-normalized weighting을 검토한다.
  - **실험 순서:** ① Q1 정렬 비교(기본값: post-only 제외, overlap 느슨, post 적음) → ② Q2 post-only 처리 비교 → ③ Q2 overlap 빡빡/느슨 비교 → ④ Q1 post 분량 비교. 고정값 오염 방지를 위해 정렬 승자가 onset이면 post-only 승자 설정에서 fixed 정렬을 1회 재확인하고, post-only 승자가 제외이면 overlap 승자 설정에서 lying 전환을 1회 재확인한다. 효과 차이가 작거나 부호가 흔들리면 `alignment × post_only` 2x2 추가 확인을 허용한다.
  - **교차 점검 / 선행 게이트:** 구현 전 `beep concat artifact sanity`를 수행한다. clean400 좌표계는 유지하되, beep 제거 concat이 RPCA/ACF/SDP 접합부 artifact를 크게 만들면 원본 continuous crop/peak-centered crop 우선 여부를 재논의한다. onset 기준 arm은 `needs_review`가 해결된 onset만 사용한다. 미해결 세션은 원칙적으로 수동 해결 후 사용하고, 제외가 불가피하면 train/val/test 전체에서 제외 수를 보고한다. onset/window manifest는 전체 파일 metadata로 생성 가능하나, overlap threshold/shift 폭/needs_review confidence threshold 결정은 split-scoped train/val 통계로만 수행하고 test는 봉인한다.
  - **Implementation inputs:** 다음 구현 순서는 onset detector → onset/window manifest 생성기 → event-centered cache builder → `train.py` 분기/runner → ablation execution/report 순서로 둔다. cache lineage에는 `manifest_id`, `align`, `post_policy`, `overlap_policy`, `label_policy`, `post_amount`, `class_policy`를 포함한다.
  - **선행 게이트 1 결과 (2026-06-05, beep concat artifact sanity → clean400_concat 유지 확정):** 위 `교차 점검 / 선행 게이트`가 예고한 검증을 finetuned_baseline6(`model/finetune/checkpoints_compare6_cpu/best_operating.pt`, dict이면 `ckpt["model"]`, `classes[0]=="fall"`, output dim 6) 기준 read-only로 수행했고, **clean400_concat(=`concat_main`) 유지**로 닫혔다. 근거 3층:
    - **(1) 접합부/row occlusion:** WALK_B에서 `concat_main` fall_prob가 `continuous_center`보다 높고 그 신호가 concat-local frame 50(=명목 onset=splice) row에 의존(E2_S02 attention 60.8%가 boundary rows, z-SDP boundary-row occlusion 시 0.924→0.137). 단 frame 50은 splice이자 fall onset이라 occlusion만으로 artifact/onset 분리 불가.
    - **(2) splice-smoothing probe (결정타):** raw amplitude 레벨에서 splice junction을 smoothing(선형 crossfade + 이동평균 2방식)한 뒤 정식 RPCA→ACF→SDP→모델을 재실행. boundary_50 불연속을 크게 줄여도(E2_S02 amp_jump 0.694→0.033) fall_prob 생존(0.924→0.918, 2방식·전 width 일치). 즉 모델 fall 출력의 주 원인은 합성 splice edge가 아니라 실제 onset/motion transition 보존 → splice artifact 우려 해소.
    - **(3) WALK_B session-level recall 정량화:** WALK_B 59세션(60 후보 중 `resampled_count<550` 1 skip), thr0.1 운영선 기준 session recall = `concat_main` 0.644(38/59) vs `continuous_center` 0.492(29/59), **recall_gain = +0.153**. 4분할 both=25 / concat_only=13 / center_only=4 / neither=17(합 59). env/subject 6개 중 5개 concat 우세·동률, center 순이득 환경 0개(이득이 한 subject에 쏠리지 않음).
  - **선행 게이트 1 유보 (닫힌 결정과 분리해 기록):** (a) 보조 후보 `continuous_fall_post`(orig[200:500])가 raw recall 0.797로 더 높으나 **fall-only 데이터라 FAR 불명 + post-fall 의존 위험** → Gate 1 번복 근거 아님. 닫힌 것은 `concat_main` vs `continuous_center`이고 `continuous_fall_post`는 기각이 아니라 열어둔 탐색 항목. (b) `concat_main` 절대 recall 0.644는 목표 0.85 미달 — 좌표계는 토대이지 목표 달성이 아님(모델/Q-onset/증강 별도 과제). (c) E4_S01은 두 crop 다 2/10으로 저조 → 해당 env/subject 신호 품질 별도 점검.
  - **선행 게이트 1 코드 영향/산출물:** 전부 read-only 진단 — 동결 파일(`pipeline.py`/`rpca.py`/`acf.py`/`sdp.py`/학습·추론) 무수정, 신규 스크립트만 `debug/modeling/`에 추가(`gate1_beep_concat_artifact.py`, `gate1b_beep_concat_artifact.py`, `gate1b_walkb_occlusion.py`, `gate1b_walkb_splice_smoothing.py`, `gate1b_walkb_detection_rate.py`). 수치/그림은 `debug/modeling/diag_out/beep_concat_artifact/`(gate1·gate1b·walkb occlusion/smoothing/detection_rate) 하위. 재현·참조용.
  - **[Gate 2 detector 기준 확정 / final onset_manifest는 pending] onset detector probe (2026-06-05):** 360 fall 세션(train 230/val 57/test 72 봉인, excluded<550 11) probe로 자동 onset detector 기준 확정. **확정된 것은 detector 기준이며, onset annotation 전체(final onset_manifest)는 needs_review/manual 처리 후 별도 확정.**
    - **base detector 확정:** search=nominal original[190:350], k_mad=3.0, sustain_frames=5, smooth_frames=5. 근거: nominal k3/s5가 clean valid 최고(251/349=72%), hard% 최저(29%). k3.5/s5는 too_early 줄지만 rise_not_found 증가로 손해. 우선순위(조기오인 최소 > param 안정 > miss 감소).
    - **broad 격하 = diagnostic only:** broad-only 1.1%, plot 확인상 broad가 stage2 beep tail/search 경계 노이즈를 가짜 onset으로 주입(SIT_B T005 rise=352 노이즈 crossing). nominal rise_not_found는 broad rescue 없이 needs_review/manual로.
    - **soft threshold set A (10/90 분위수, train+val 확정):** rise_strength<1.136, rise_slope<0.016, confidence_ref<0.522, baseline_noise_ratio>1.206, baseline_mad>0.079, param_sensitivity>51, topk_spread_high(>10) / topk_spread_extreme(>45 보조 로그). soft_warning_count>=2 → needs_review. 3분위수(10/90·15/85·20/80) 비교 후 budget 최소 위해 10/90 채택.
    - **onset 분포 (provisional):** 자동 rise_frame_clean median ~131(p25 114/p75 157), param 무관 안정(129~137). peak_frame_clean median 182(원본 282 충격점). 명목 onset clean 100보다 ~31프레임 늦음 → 항목 4(onset-aligned crop) 필요성 강화. **131은 provisional — needs_review/manual 처리 후 final onset_manifest의 확정 onset 분포로 re-alignment point 재계산.**
    - **needs_review 총량 43.6%(set A):** hard 33.4%(바닥, soft로 못 줄임) + soft 증분. 억지로 budget(35%) 맞추면 위험한 자동 onset 통과 → label 품질 우선으로 수용. 비-WALK 24~44%는 대부분 진짜 평탄/노이즈라 검수 정당.
    - 산출물: `debug/modeling/diag_out/onset_detector/` (onset_probe_manifest_long, session_summary, soft_threshold_compare, plots/plots_soft). read-only, 동결 파일 무수정, train/val만(test 봉인).
  - **[Gate 2 open] WALK onset baseline contamination — 구조적 미해결:** FALL_WALK는 낙상 전 pre 구간이 걷기라 baseline window original[50:150]이 보행 motion 오염 → threshold 폭등(plot T007 noise 3.06, threshold 0.223) → search 구간 상대적 평탄 → rise_not_found/too_early/weak candidate 증가. WALK_B hard 55.8%, WALK_F hard 42.6% (vs STD_B 12%, SIT_F 22%). soft threshold로 해결 불가. 현재 처리: WALK_B/F review_priority=high + 높은 수동검수 수용. future options(발표 후): ① WALK 전용 baseline window ② WALK onset용 다른 신호 ③ 수동검수 수용(현재 채택). ①②는 detector에 subtype 임의 규칙 주입 위험이라 보류. 근본은 데이터/도메인(걷기↔낙상 신호 분리 난이도) — 게이트 1 WALK_B 최약, 기존 FALL_WALK_B 최약과 일관.
  - **[onset_manifest v1_auto_reviewed — clean subset, final 아님] (2026-06-05):** base detector(nominal/k3/s5, soft set A)로 360 fall 세션 분류. v1은 auto_reviewed clean subset이며 final onset_manifest 아님 — pending_manual은 진규 검수 후 v2에서 확정.
    - **분류:** auto_reviewed 214(59.4%, usable_for_training) / pending_manual 135(37.5%) / excluded 11(<550). auto_exclude_candidate 0 (rise_not_found 세션도 peak_over_baseline≥1.10(최약 1.13)이라 자동 제외 0건, 전부 진규 검수로 — 보수 기준대로 억지 자동제외 없음).
    - **subtype 편향 (WALK 과소대표):** SIT_B 58%/SIT_F 70%/STD_B 77%/STD_F 70% vs WALK_B 37%(22/60)/WALK_F 45%(27/58). WALK가 auto_reviewed에서 크게 빠짐 — plot 재확인상 걷기 energy가 baseline[50:150] 점령→threshold 폭등→search 평탄(게이트 2 WALK baseline contamination 구조 재확인).
    - **provisional onset median (auto_reviewed only, clean frame):** overall 132.5(~131 일관). by_subtype WALK_B 117(최조기)·SIT_B 130·SIT_F 132·STD_F 135.5·STD_B 142·WALK_F 145. **provisional — auto_reviewed만이라 "쉽게 잡힌" 세션 치우침(특히 WALK_B 117), v2에서 재계산 필수.**
    - **항목 4 영향:** auto_reviewed 214로 항목 4 crop alignment provisional 착수 가능. 단 WALK 과소대표로 깨끗한 비-WALK 과대표 → onset-aligned 효과가 실제보다 좋게 보일 위험. 항목 4는 (a) 전체/subtype별 분리 평가 (b) WALK는 v2 후 재평가 단서 필수.
    - **priority high pending_manual:** 115(train/val 98, WALK 62/115=54%). 진규 직접 검수 진행. 산출물: priority_review_queue.csv(정렬·reason·top-k·추천, 전 추천에 "Recommendation only, 진규 승인 필요" 명시, pending onset 전부 null) + plots_priority/(98).
    - 산출물: `debug/modeling/diag_out/onset_detector/finalization/` (manifest_v1_auto_reviewed.csv, summary, priority_review_queue.csv, plots_priority/). read-only, train/val 기준(test 봉인), 동결 파일 무수정. **Claude Code는 pending_manual onset 미확정 — 진규 검수 후 v2.**
  - **[검수 도구 v2 — 인터랙티브 곡선 차트 (2026-06-06, push `d79e26c`)]:** PNG 고정 이미지 → JS canvas 인터랙티브 차트로 재구현(검수자 피드백). sparse-energy 곡선이 디스크에 없어(manifest_v1 실행 중 메모리에서만 렌더 후 소멸) `export_energy_curves.py`(= `gate2_onset_manifest_v1.recompute_energy` 동일 계산: load→resample→rpca_sparse→mean|·|→5smooth)로 98세션 재계산→`energy_curves.json`(478KB, 98/98) 빌드시 임베드. 반영 6개: ① x축 눈금 3단계(1/10/100) ② hover frame·energy 툴팁 + 클릭→수정입력 ③ 수정 모달 실시간 '내 선택' 초록 세로선 ④ 색약 접근성(구간 빗금패턴+글자라벨, 파랑/주황/초록 팔레트, strip 색+글자) ⑤ 에너지 곡선 강조(굵은 파랑)·thr/auto 보조(흐림) ⑥ rise_not_found 안내 + SEARCH 밖 onset 지정 허용(경고만). 기존 기능·Export(csv/json) 스키마·localStorage `_v1` 유지. read-only(데이터·plot png·동결파일·manifest 무수정, 도구 HTML+신규 파생물만). 곡선은 HTML 임베드라 file:// 더블클릭으로 표시(서버 불필요). **진규 시각검수 완료 — 6개 요구사항 반영 확인.** 검수 판정 자체는 진규가 도구로 진행 → `review_decisions.csv` Export → v2 manifest.
### [D-031 추가] onset_manifest v2 수동 검수 완료 및 항목4 alignment 설계 확정
- **Date:** 2026-06-06
- **Decided by:** user / codex / claude-ai
- **Content:**
  - 진규가 onset_manifest v1 pending manual 115개 검수를 완료했고, `manifest_v2_manual_augmented` 기준으로 crop 방식별 usable flag를 확정한다.
  - fixed 정책은 `usable_for_fixed=True`인 349/360 세션을 유지한다. `onset_unusable` 세션은 fixed crop에서 제외하지 않으며, `data_invalid` 11개만 전체 정책에서 제외한다.
  - onset-aligned 정책은 `usable_for_onset_aligned=True`이고 crop 범위가 clean400 안에 들어오는 세션만 사용한다. 수동 onset이 있어도 crop이 clean400 밖이면 clamp하지 않고 `crop_out_of_bounds`로 onset-aligned coverage에서 별도 집계한다.
  - median fallback/imputation은 사용하지 않는다. 제외 세션의 onset은 null로 유지한다.
  - 항목4 메인 비교는 non-WALK pooled paired comparison으로 수행한다. 같은 세션에서 `fixed=clean[50:350]`와 `onset_primary=clean[onset-50:onset+250]`를 비교한다. 두 crop은 모두 onset 기준 `-50:+250` 구조이며, 차이는 명목 onset 100 정렬과 실제 수동 onset 정렬뿐이므로 순수 alignment effect를 검증한다.
  - optional post amount ablation으로 `onset_reduced=clean[onset-100:onset+200]`를 생성한다. 메인 결론이 아니라 post-fall 정적 단서 의존 감소 확인용 보조 실험이다.
  - single-crop 실험이므로 overlap 정책은 `not_applicable`로 기록한다. multi-window stride/overlap은 후속 확장 시 별도 결정한다.
  - 보고는 paired alignment effect와 coverage/inclusion rate를 분리한다. WALK는 fixed 주력, onset-aligned는 pooled exploratory로 보고하며 inclusion/exclusion/crop_out_of_bounds rate를 반드시 명시한다.
  - 주요 지표는 event recall, forward recall, late_tp, FAR, first-fire latency median/p90, timely_3s/timely_4s이며 F1은 보조 지표로 둔다. 통계는 paired bootstrap CI, fire/no-fire McNemar, 단일 비율 Wilson CI를 사용한다.
  - test split은 설계, threshold, crop 통계 결정에 사용하지 않고 봉인한다.
  - **검수 발견 (정성, 항목4 해석 근거):**
    - WALK는 onset-aligned 부적합 (구조적). 검수 제외 사유 1위는 walking_residual(6개)이 아니라 no_clear_transient(43개) — WALK는 "걷기가 가려서"가 아니라 "낙상 transient 신호 자체가 약하거나 묻혀서" onset을 못 잡음. 게이트 1·2부터 5번째 일관 확인.
    - SIT는 완만한 낙상: 앉다 스르륵 무너져 신호가 STD(급격한 단발)와 달리 완만하게 솟음. 급격한 단발 없어도 baseline 위로 완만히 확실히 솟으면 낙상으로 판정. 낙상 종류별 신호 형태 차이.
    - 환경/피험자 품질은 세션 단위 판정 (일괄 분류 금지): E4_S01/E1_S01은 세션마다 깨끗/시끄러움 편차 큼. low_quality_env_subject 9개는 전부 noise>1.206 세션(E2_S02 4, E4_S03 3, E4_S02 2), E1_S01/E4_S01 미포함 — 검수 초기 "E4_S01 저조" 인상은 환경 일괄 단정 오류로 정정. drift형 출렁은 noise_ratio로 안 잡히니 육안 병행.
  - **v2 usable N:** usable_for_fixed 349/360, usable_for_onset_aligned 264/360 (auto 214 + manual 50). non-WALK pooled train+val 148 → 항목4 메인 paired 비교 가능. subtype onset-aligned 전부 20+ (WALK_F 39/WALK_B 37/SIT_F 46/SIT_B 40/STD_F 51/STD_B 51).
  - **검수 분포:** 115개(high 98 + normal 17) = 확정 50(modify 43/approve 7) / 제외 65. onset_status: auto_reviewed 214, manual_corrected 50, onset_unusable 65, pending_manual 20, data_invalid 11.
  - **final onset median (clean, imputation 없음):** overall 134.0 (manual 150.0/auto 132.5). by_subtype WALK_B 120.0 / SIT_B 130.5 / SIT_F 132.5 / WALK_F 136.5 / STD_F 142.0 / STD_B 144.0. by_split train 130.0 / val 143.0 / test 134.0.
- **Status:** confirmed
- **Status:** Gate 1·2 완료, onset_manifest v2 확정(검수 115개 반영). 항목4 alignment 설계 확정 → Gate 3 cache builder 진행 예정. Pending: cache builder(fixed/onset_primary/onset_reduced), 항목4 paired 비교(non-WALK pooled 메인), event-level 평가.

### [D-031 추가] 항목4 학습/평가 최종 결과 (onset 정렬 검증)
- **Date:** 2026-06-06
- **Decided by:** user / codex / claude-ai
- **Content:**
  - **onset 정렬 검증:** sealed test(non-WALK pooled paired, N=27)에서 onset_primary는 fixed와 **동일 recall**(Δ−0.015, 비유의)에 **FAR 유의 감소**(Δ−0.139 [95%CI −0.191, −0.091]). McNemar p=0.727. → onset 정렬의 가치 = "같은 recall에서 FAR 유의 감소(오발화 억제)"로 검증.
  - **운영점(val recall–FAR frontier, onset_primary, 정규화):** FAR≤0.15 → recall 0.77 / recall 85% → FAR ~0.20 / max-F1 0.71(recall 0.89 / FAR 0.22). onset_primary가 운영곡선 전 구간에서 fixed 우위(FAR≤0.15: 0.767 vs 0.665).
  - **목표 대비:** recall≥85% AND FAR≤15% **동시 미달**(recall 85%는 FAR ~20%에서 달성). F1≥0.85는 event-level 구조상(비낙상:낙상=180:27) 도달 불가(최대 ~0.72).
  - **최적화 레버 결과:** 정규화(weight_decay 1e-4, early-stop↑)로 과적합 완화 → frontier +0.02~0.03. threshold grid 확장(0.30→0.70)·N{1,2,3}·margin{..0.3} 등 post-processing은 **무변화(한계 도달)**. 체크포인트 선택(best_operating vs best_val_loss) 민감도 ~0.05.
  - **post_amount ablation:** onset_primary(post250) > onset_reduced(post200) — FAR/F1 모두 primary 우위. fixed_baseline 모드: onset은 자기 threshold 필요(고정 적용 시 recall도 유의 감소) → threshold 최적화와 alignment 효과 분리 확인.
  - **병목 = 모델/데이터 capacity.** post-processing·정규화는 한계까지 소진. ROI 최고 후속 레버 = **multi-window**(onset 주변 다중 crop): onset_primary는 train fall 114개(fixed 223의 절반)인데도 이김 → 데이터 늘리면 frontier 상향 여지(단 D-031 단일 crop 우선 원칙으로 현재 보류).
  - **산출물:** Gate3 cache(fixed/onset_primary/onset_reduced) + crop_index, item4 학습 cache·eval_windows.pkl, ckpt(checkpoints_item4 / _reg, best_operating·best_val_loss ×15 seed, 로컬), item4_eval_report.md/json. 스크립트(build_gate3_cache/item4_build_policy_cache/item4_train/item4_precompute_eval_windows/item4_event_eval) 추적. read-only(원본·동결파일·manifest 무수정). 학습 seed 42–46 고정 재현 가능.
- **Status:** confirmed — onset 정렬 가치(FAR 억제) 검증. 절대 목표 미달(병목=capacity), 후속 multi-window 후보.

### [D-031 추가] 더미 직접생성 + 4-arm 증강 학습 결과 (within-subject in-domain)
- **Date:** 2026-06-06
- **Decided by:** user / codex / claude-ai
- **Content:**
  - **더미 생성:** 팀원 6/7.py 로직을 clean400 좌표로 이식(난수 factor/scale/crop_offset/snr 전부 lineage 저장 — 팀원 원본은 미저장이라 onset 역산 불가했음). slow는 onset 추적 약점(delta median 17 vs normal 5.5/fast 9.0)이라 onset-aligned에서 제외. 2차 normal+fast ×6 → onset 교차검증(expected 해석 vs detected Gate2) 통과 **use 301**(delta median2/p90 6, splice 새 artifact 1%, baseline_noise 감소).
  - **이중 증강 발견·수정:** train.py 온라인 증강(jitter/scale/timewarp/noise)이 더미에도 적용돼 이중 증강 → **더미만 온라인 증강 skip**(filename 마커), 원본 유지. (없었으면 C/D 무효였음.)
  - **4-arm(A fixed_orig / B onset_orig / C fixed_aug / D onset_aug) ×5seed, sealed test non-WALK paired N=27:**
    - 비통제: **B→D ΔFAR +0.110 [+0.070,+0.151] 유의(FAR 악화)**, Δrecall +0.008 ns. 단 더미가 fall만 늘려 class 불균형(fall:non 0.078→0.259, fall mass 2.9배).
    - **(B) class-ratio 통제(fall effective mass 원본 고정)**: **B→D ΔFAR +0.028 [−0.014,+0.071] 유의차 미검출**, 결합 (D−C)−(B−A) +0.118→+0.026(ns), Δrecall −0.044 ns. → **FAR 악화는 class 불균형 artifact 확정. 통제 후 더미 효과 = null(중립).**
  - **결론:** 현재 **in-domain offline 더미(fall만)는 onset-aligned 성능을 강화하지 못함(중립, N=27 under-powered).** 개선하려면 **비낙상 포함 균형 증강** 또는 multi-window/신규 세션. 결론 범위 = within-subject in-domain (새 subject/env 일반화 아님).
  - **핸드오프:** HANDOFF.md(repo root) — 산출물 분류(git 경량 / Drive 대용량 수동 / 제외), 이어받기 절차. Drive 자동업로드 불가(MCP payload + 보안 classifier 차단) → 사용자 수동.
  - **산출물:** 스크립트(dummy_clean400_lib/dummy_generate/dummy_validate, item4_build_arm_caches/item4_arms_eval), out2/lineage.csv, arms_results(_ctrl) report/json, item4_cache_*_aug.npz(로컬), ckpt checkpoints_item4_arms(_ctrl)(로컬·재학습 가능). read-only(원본·동결·manifest 무수정).
- **Status:** confirmed — 더미 효과 null(중립). FAR 악화는 class artifact. 후속=균형 증강.

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
| Alsaify 전체 사전학습 (E1+E2, ACF lag=1..20) | done (lag1..20 재학습) | main (`model/pretrained/checkpoints/` gitignored) | 2026-05-18 |
| UDP 수신 서버 (`receiver/udp_receiver.py` + `server/main.py::start_receivers`) | done (통합 서버에서 UDP 수신 시작) | main | 2026-05-11 |
| WebSocket 서버-Pi4 통신 (`ws_handler/rpi_connection.py`, `server/main.py::RPiConnection`) | done (Pi4 outbound WebSocket 수신/낙상 알림 전송 경로 구현) | main | 2026-05-11 |
| 자체 데이터 수집 파이프라인 (collect/, D-023/D-028 240세션·side fall 제외 정렬) | done | codex/finetune-train-skeleton `4adfbff` | 2026-05-22 |
| server 수집 경로 (collect_manager.py invalid activity_code 가드) | done | codex/finetune-train-skeleton `4adfbff` | 2026-05-22 |
| 수집 pair_dt/품질 기록·리포트 (CSV 110컬럼, collect/quality.py, CLI/서버UI/check_csv_quality, on_paired 카운터 정합) | done (report-only, PAIR_TOLERANCE_US 미변경) | codex/finetune-train-skeleton `48ff88e` | 2026-05-22 |
| 대시보드 품질 박스 항목별 색상 + 종합 판정(저장/검토/폐기) 배지 (index.html) | done (report-only, 저장 차단 미연결) | codex/finetune-train-skeleton `5657f57` | 2026-05-22 |
| preprocessing/resample.py (D-018 SafeSignal 100Hz 리샘플) | done | main | 2026-05-13 |
| preprocessing/loader.py SafeSignal 경로 (load_safesignal_csv 등) | done | main | 2026-05-13 |
| preprocessing/pipeline.py SafeSignal 경로 (preprocess_safesignal_file*) | done | main | 2026-05-13 |
| fine-tuning (train.py: combined training + full unfreeze + warmup, on-the-fly 증강 연결, `--class_policy finetune7/pretrained6`, `--split cross_subject/within_subject`, `--threshold_min`) | in progress (main 반영; HPO/gap-quality hooks 여전히 pending) | main `0b25a49` | 2026-06-01 |
| verify_aug_gate.py (TrainAugmentDataset 증강 적용/Alsaify pass-through 검증 게이트) | done | main `9e3064d` | 2026-06-01 |
| debug/modeling/derive_pretrained6_cache.py (7-class cache → running 제외 6-class 파생) | done | main `9e3064d` | 2026-06-01 |
| debug/modeling/eval_zeroshot_by_subject.py (subject별 zero-shot 진단) | done (실측 결과 미기록) | main `0b25a49` | 2026-06-01 |
| debug/modeling/build_rawsdp_cache.py (per-lag D-020용 SafeSignal raw SDP cache builder) | done (`origin/exp/perlag-cache` `0db4c26`; local `safesignal_e1234_finetune7_rawsdp.npz` 확인, X=(3581,1,28,20), full_zscore max_abs_diff=2.38e-7) | origin/exp/perlag-cache | 2026-06-03 |
| debug/modeling/build_alsaify_rawsdp_cache.py (D-020 트랙1 Alsaify raw SDP cache builder) | done (commit `79a8f20`; pipeline.py 등 동결 파일 미수정) | main | 2026-06-03 |
| debug/modeling/track2_probe_make_caches.py / track2_probe_analyze.py (트랙2 provisional mixed-normalization probe 로컬 스크립트) | local untracked (A/B 3-seed 완료; `track2_probe_comparison.json` 기록) | local | 2026-06-03 |
| debug/modeling/track1_formal_make_caches.py / track1_formal_analyze.py (D-020 트랙1 정식 ablation 로컬 스크립트) | local untracked (A=both global, B=both per-lag 3-seed 완료; `track1_formal_comparison.json` 기준 per-lag 기각) | local | 2026-06-03 |
| Pi4 하드웨어 버튼 인터페이스 | pending (rpi4/는 문서/가이드 중심, 실제 버튼 I/O 구현 미확인) | - | - |
| E2E 통합 테스트 | pending | - | - |
| inference/ 모듈 (InferenceWorker + FallPredictor + SlidingWindowBuffer) | done | main | 2026-05-11 |
| main 브랜치 통합 (server/dongseok + feature/pretrained-model) | done | main | 2026-05-11 |
| tools/augment_inspector.py (증강 파라미터 검토용 시각화) | done | main | 2026-05-18 |
| preprocessing/acf.py lag 정책 (lag0 제외, lag=1..20) | done | main `49f92be` | 2026-05-18 |
| collect/drive_upload.py (rclone 기반 Google Drive 자동 업로드) | done (운영 확인: `gdrive:SafeSignal_Dataset`, `E*/S*` 자동 분류) | main | 2026-05-22 |
| debug/preprocessing/analyze_sdp_energy.py (정식 pipeline 기준 z-score 전 SDP energy 분석) | done (분석 전용, 추론 코드 무변경) | codex/no-motion-baseline | 2026-05-25 |
| debug/modeling/build_onset_review_tool.py + export_energy_curves.py (onset 검수 도구 v2 — 인터랙티브 곡선 차트) | done (read-only; 곡선 임베드 98/98, 검수자 피드백 6개 반영, 진규 시각검수 완료) | feature/event-centered-gate1 `d79e26c` | 2026-06-06 |
| 항목4 onset-aligned / 더미 4-arm 실험 스크립트 (`build_gate3_cache`, `item4_*`, `dummy_*`) | done (onset_primary는 fixed 대비 같은 recall에서 FAR 유의 감소; 더미 fall-only 효과는 class-ratio 통제 후 null) | feature/event-centered-gate1 `2b07f8e` | 2026-06-06 |
| debug/dummy_gen/balanced/nonfall_dummy_generate.py + item4_build_balanced_caches/item4_train_balanced/item4_balanced_eval.py (비낙상 포함 balanced 증강 실험) | 작성·정적검증 완료(compile/effective-share maxerr<5e-5/window_starts), 2026-06-09 학교 PC 실행 대기 | feature/event-centered-gate1 `a44b9ac` | 2026-06-08 |

---

## Review Notes

### 2026-06-08 — 추론 파이프라인 데모 정비 (졸업 직결) (claude-code / codex)

- **배경:** 서버 추론 경로에 임시 track1 모델(`checkpoints_track1_formal_global_s43/best_operating.pt`), threshold 0.5, 기존 서버 판정식 불일치, energy gate 없음 상태가 남아 있었고, 정적·다인 환경에서 fall 오발이 확인됐다. 원인은 도메인 갭(Alsaify/track1 mismatch) + 환경 노이즈로 본다. 데모 환경은 통제 예정이나, 통제 후에도 정적 오발이 지속되면 졸업 직결 리스크다.
- **핵심 정합성 버그:** 학습 평가의 `predict_with_fall_threshold()`는 `fall_prob >= threshold`이면 argmax와 무관하게 fall로 강제하지만, 기존 서버 판정은 사실상 `argmax == fall AND fall_confidence >= threshold`였다. 따라서 항목4/후속 평가에서 고른 threshold가 서버에서 같은 의미로 동작하지 않았다. `server/inference/predictor.py`는 `raw_is_fall = fall_conf >= self.threshold`로 정렬해 argmax 조건을 제거했다. 이 변경은 기본 동작을 의도적으로 바꾸는 버그 수정이며 토글 뒤에 숨기지 않는다.
- **fall alert confidence 정렬:** 판정식 정렬 후에는 argmax class가 non-fall이어도 `is_fall=True`가 가능하므로, `server/main.py::handle_inference_result()`에서 Pi4/SMS fall alert confidence는 argmax confidence가 아니라 `fall_confidence`를 우선 전달하도록 정렬했다.
- **다층 방어:** fine-tuned 모델 교체(`SAFESIGNAL_MODEL_PATH`), 평가 운영점 threshold 주입(`SAFESIGNAL_FALL_THRESHOLD`), SDP energy gate(`SAFESIGNAL_ENERGY_GATE_ENABLED/THRESHOLD/METRIC`), N consecutive(`SAFESIGNAL_FALL_CONSECUTIVE_N`), 환경 통제를 함께 사용한다. 기본값은 기존 E2E 보존 방향으로 유지한다(model path=track1 임시 모델, threshold=0.5, energy gate off, N=1, stride=100 유지).
- **energy helper 단일화:** `server/inference/energy.py::compute_window_energy()`를 새로 두고 predictor와 calibration 도구가 같은 helper를 import한다. 정의는 `debug/preprocessing/analyze_sdp_energy.py`와 같은 z-score 직전 SDP metric이다(`rpca_sparse -> stacked_doppler_profile -> sdp_mean_abs/sdp_fro/sdp_std/sdp_max_abs/sparse_ratio/raw_std/raw_delta_mean`). 동일 window 기준 helper/predictor/analyze 경로 maxdiff 0을 검증했다.
- **energy gate / calibration:** energy gate는 실제 데모 경로인 `FallPredictor.predict()` 내부, 모델 호출 전에 적용된다. low-energy skip 시 `class=non_fall_energy_gate`, `confidence=0.0`, `fall_confidence=0.0`, `raw_is_fall=False`, `is_fall=False`로 반환하고 skip/pass counter를 result에 담는다. `debug/inference/calibrate_energy_gate.py`를 추가해 오늘 정적 측정 1차와 데모 당일 시연장 정적 측정 2차를 같은 절차로 돌릴 수 있게 했다. 기본 후보는 static p99 * 1.2이며, `--fall-patterns "*FALL*.csv"`로 기존 fall energy p5와 비교해 threshold가 fall p5보다 낮은지 확인한다.
- **replay / 검증 도구:** `debug/inference/replay_static_csv.py`는 `SlidingWindowBuffer + FallPredictor` 실제 서버 추론 경로로 `NO_MOTION/STAND/LIE` CSV를 replay해 raw fall/final fall/energy skip 수를 확인한다. calibration 도구는 모델 추론 없이 energy 분포 p50/p90/p95/p99/max와 PowerShell env 설정 예시를 출력한다.
- **검증 완료:** py_compile, `server/inference/_selfcheck.py` ALL_OK, checkpoint strict load(classes=`fall,walking,sit_stand,lying,standing,picking`, fall index `(0,)`), 전처리 동일성(maxdiff 0), 판정식 smoke(argmax walking이어도 fall_confidence>=threshold면 raw_is_fall True), alert confidence smoke, energy gate 강제 skip, N=2 env 반영, replay smoke, calibration smoke를 통과했다.
- **stride / 평가-데모 간극:** 서버 stride는 100으로 유지한다. 학습/항목4 평가는 event-level 및 stride 50 조건을 포함할 수 있으므로, 데모 기대치는 서버 조건(stride 100, 선택한 threshold, energy gate, N setting) replay/라이브로 재측정해야 한다. 평가 수치를 데모 기대치로 그대로 쓰지 않는다.
- **합격 기준:** 정적 CSV replay 0 발화 -> E3 라이브 정적 5~10분 무발화 -> fall 리허설 정량 합격선(N회 중 X회, 권장 5회 중 4회 이상 등 사용자 확정) 통과. 정적 0건만으로는 충분하지 않고, fall이 잡히는지도 반드시 함께 본다. energy threshold는 E3 baseline 기반으로 최종 조정한다.
- **백업/롤백:** 코드 수정 전 `backup/inference-demo-prechange` 브랜치와 `debug/inference/pre_claude_backup/` patch/status/bak 파일을 만들었고, backup 디렉터리는 `.git/info/exclude`로 커밋 제외했다. 코드 repo 커밋 대상은 추론 정비 파일만이며, `debug/modeling/onset_sparse_energy_diag.py`, backup, smoke json, `__pycache__`는 제외한다.


### 2026-06-08 — 항목4 후속 후보 triage + 비낙상 포함 균형 증강 실험 계획 (claude-ai / codex)

- **배경:** 항목4 onset 정렬 검증(D-031: FAR 유의 감소 Delta -0.139 [-0.191,-0.091], recall 비유의 Delta -0.015, 절대목표 R>=0.85 & FAR<=0.15 미달, 병목=모델/데이터 capacity)과 더미 4-arm(fall-only 더미의 FAR 악화는 class-ratio artifact, fall mass 통제 후 더미 효과 null, within-subject in-domain, N=27 under-powered) 이후 재시도 가치 있는 후보를 triage했다.
- **후보 triage:**
  - 후보2 multi-window onset crop = 성능 레버 관점의 최고 ROI 후보(deferred next lever)이나, 2026-06-09 즉시 실행 우선순위는 후보1 balanced 증강으로 둔다. 근거: onset_primary가 train fall 114개(fixed 223의 51%)인데도 FAR frontier 우위 → 정렬 이득 확인, 데이터량 병목. D-031 Q4 발동 조건(train fall window가 baseline 50~60% 미만) 충족. 단 D-031 단일 crop 우선 원칙으로 보류였던 카드이며 설계 결정 6항목 잠금 대기.
  - 후보1 비낙상 포함 균형 증강 = 내일(2026-06-09) 학교 도착 직후 실행 우선순위. 기대치는 큰 frontier 상승이 아니라 artifact 교정/중립 또는 소폭 개선 확인으로 둔다. 이유: train.py `TrainAugmentDataset`가 이미 SafeSignal train 전 샘플(fall/non-fall, label 무관, source==SafeSignal 조건만)에 on-the-fly 증강을 적용 중이라 오프라인 균형 더미의 추가 다양성이 일부 중복될 수 있다. 그래도 fall-only artifact를 제거한 4-arm 재평가로 유의미한 변화 여부를 확인한다.
  - 후보3 Q2 post-only/overlap label ablation = multi-window 이후 후순위. 단일 onset crop(`clean[onset-50:onset+250]`)은 정의상 항상 onset을 포함하므로 post-only window가 거의 없어 multi-window/sliding 체제에서만 의미가 있다. post_amount ablation(post250>post200)과 forward/tail 진단(tail recall 0.833 >> forward 0.542)이 post 제거 시 recall 손상 위험을 시사하므로 진단용 ablation으로만 둔다.
- **코드 사실 확인 (Codex 검증 완료):** `model/finetune/train.py` `TrainAugmentDataset.__getitem__()`은 label을 보지 않고 `self.augment and meta.get("source") == SOURCE_SAFESIGNAL` 조건만 사용한다. 따라서 SafeSignal train subset이면 fall/non-fall 전부 on-the-fly 증강 대상이고, Alsaify는 pass-through다. `debug/modeling/item4_train.py` monkeypatch도 동일하게 SafeSignal 원본 전부 증강, 더미만 filename marker로 online augmentation skip한다. `train.py` 1097줄 "Currently a no-op" 주석은 stale이며 차후 코드 수정 시 정정 후보다.
- **비낙상 균형 증강 사전검증 Q1 (Codex, 파일 기준):** 현재 `debug/dummy_gen/dummy_clean400_lib.py`, `dummy_generate.py`, `dummy_sanity.py`, `dummy_validate.py`는 fall/onset 전용이다. `dummy_generate.py::select_origins()`는 `train` + `non-WALK fall` + `onset_status in (auto_reviewed, manual_corrected)` + `onset_frame_clean` 존재 조건으로 origin을 고르고, `_worker()`는 `expected_onset()`과 `detect_onset_clean()`의 onset_delta로 use/pending/exclude를 판정한다. `dummy_validate.py`도 onset_delta/baseline_noise/crop_oob 검증을 전제로 한다. 따라서 non-fall 클래스 생성 경로는 현재 없으며, D-031 Q3 원칙상 non-fall은 onset-aligned가 아니라 기존 sliding-window 정책을 유지하는 별도 생성/cache builder가 필요하다. 구현 시 `item4_cache_fixed.npz`/`item4_cache_onset_primary.npz`의 `(1,28,20)` z-SDP를 직접 perturb하지 않는다. cache는 split/origin registry로만 쓰고, 원본 CSV/clean data에서 기존 sliding raw window `(300,104)`를 복원한 뒤 raw perturbation(`factor/scale/crop_offset/snr/time_warp`)을 적용하고 `window_to_model_input()`으로 z-SDP를 재생성한다. fixed와 onset_primary의 non-fall cache는 파일 기준 검증 결과 row count 2323, metadata(`y/filename/subject/env/activity/trial/split_assignment`) 및 `X`가 완전 동일(max_abs_diff=0.0)이므로, balanced v1의 non-fall dummy set은 arm별로 따로 만들지 않고 하나의 train-only set을 C/D에 공유한다. 추정이 아니라 현재 파일 기준 결론이다.
- **비낙상 균형 증강 Q2 잠금 (Codex/Claude 합의, 파일 기준):** 통제 단위는 raw count/session count/window count가 아니라 `WeightedRandomSampler`가 쓰는 class별 effective sampled mass(`sum(weights[y==c & source==safesignal])`)로 확정한다. `model/finetune/train.py::build_sample_weights()`는 source quota를 먼저 고정해 SafeSignal mass=source_ratio(항목4 `item4_train.py` 기준 0.60), Alsaify mass=0.40으로 만들고, hard class weight는 SafeSignal source 내부 class mix만 바꾼다. baseline 실측(더미 없음, seed42 frozen split, pretrained6, hard_weight=1.30/source_ratio=0.60) 결과 fixed train SafeSignal effective share는 `fall 0.1157 / walking 0.2030 / sit_stand 0.2233 / lying 0.1858 / standing 0.1183 / picking 0.1538`, onset_primary는 `fall 0.0627 / walking 0.2152 / sit_stand 0.2367 / lying 0.1969 / standing 0.1254 / picking 0.1630`이다. 따라서 균형 증강 통제 기준은 단일 전역 baseline이 아니라 arm별 baseline이다: C(fixed_aug)는 A(fixed_orig)의 effective share를, D(onset_aug)는 B(onset_orig)의 effective share를 각각 보존한다. 공통 비율로 맞추면 fixed↔onset의 fall share 차이(223 vs 114 train fall window, effective share 0.1157 vs 0.0627)를 지워 alignment 효과를 confound한다. 이전 fall-mass 통제는 fall-only 증강의 불균형을 되돌려 "더미 자체가 해로운가"를 묻는 실험이고, 이번 균형 증강은 arm별 class effective ratio를 보존한 채 추가 coverage/diversity가 도움 되는지 묻는 별도 실험이다. 구현은 생성량 exact matching이 아니라 균일/상한 배수 생성 후 balanced 실험용 wrapper의 arm별 target-share `class_weight_fn`으로 effective share를 맞추는 방식(b안)으로 잠근다. 기본 `build_sample_weights()`는 무수정(read-only)으로 두고 wrapper만 얹는다. "전체 mass 증가"가 아니라 `relative effective share 유지, epoch sample count는 cache size에 따라 증가`(`WeightedRandomSampler num_samples=len(sample_weights)`)로 표현한다. 실행 전 판정 기준도 잠근다: `statistically supported`는 paired 95% CI가 0을 제외할 때, `directional candidate`는 5 seed 중 4개 이상 같은 방향이며 (`Delta FAR <= -0.05` 또는 `Delta recall >= +0.03`)을 만족하되, 반대 지표가 크게 악화되지 않을 때로 제한한다(Delta FAR 개선 후보는 `Delta recall >= -0.05`, Delta recall 개선 후보는 `Delta FAR <= +0.05`). 그 외는 null(중립)로 보고한다. directional candidate는 확정 결론이 아니라 multi-window 위에 얹어볼 신호로만 해석한다. 원본 non-fall은 매 epoch on-the-fly 증강을 받지만 더미는 marker로 online 증강 skip되는 frozen 샘플이므로, null이면 in-domain 더미 다양성 자체가 무의미한지 on-the-fly가 이미 포화시킨 것인지 구분 불가하다는 caveat를 결론에 명시한다. 최종 report에는 A/B/C/D arm별 effective share와 target 대비 absolute error를 출력하고, tolerance(권장 <=1e-3)를 넘으면 성능 평가 전에 fail-fast한다. 또한 더미 online 증강 skip 여부, X/y/source/is_augmented/origin 메타 일관성, 더미 total mass <= 원본 동급 상한 준수 여부를 출력한다. 더미 mass 상한은 sampler weight가 아니라 raw generated/used window count 기준으로 관리한다.
- **비낙상 균형 증강 Q3 잠금 (Codex/Claude 합의, 파일 기준):** `debug/dummy_gen/dummy_clean400_lib.py`의 speed profile은 fall 전용 onset 로직이 아니라 `_time_warp()` factor 범위(`fast 0.75~0.90`, `normal 0.95~1.05`, `slow 1.10~1.30`)이며 clean400 전체 시계열 time-warp다. 따라서 비낙상에 기술적으로 적용 가능하지만, non-fall sliding window는 시간 구조 자체가 label 의미이므로 적용 의미와 위험은 클래스별로 다르다. fall에서 slow를 제외한 근거는 onset 추적 불안정(delta median 17)이고 비낙상에는 onset이 없으므로 그 근거를 일반화하지 않는다. 다만 WALK는 보행 주기, SIT_STD는 transition timing, STAND/LIE는 정적 구간, PICK은 짧은 동작 타이밍이 label 의미라 slow time-warp가 artifact/confound를 만들 수 있다. 따라서 1차 balanced non-fall dummy v1은 `normal+fast`만으로 잠그고, slow는 별도 확장 ablation v2(optional)로 둔다. v1에 slow를 섞지 않는 이유는 slow artifact와 balanced 효과를 분리하기 위해서다. 해석 가드: v1의 slow 제외는 fall 조건 일치 + 안전한 1차 실행을 위한 선택이지 "비낙상 slow가 나쁘다"는 입증이 아니다. v1이 null이어도 slow였다면 달랐을 가능성을 배제하지 못하므로 v2는 열린 후보로 남긴다. non-fall 품질 게이트는 onset 지표(`expected_onset`, `onset_delta`, `rise_not_found`)를 쓰지 않고, `window_to_model_input()` 이후 z-SDP feature를 같은 class 원본 train window 분포와 비교한다. 최소 게이트는 finite check, class별 robust center/MAD distance p95 또는 p97.5 이내, raw clean/window amplitude mean ratio `[0.5, 2.0]`, transform params 기록이다. 게이트 임계, slow 포함 여부, 더미 배수/상한은 train-only로 결정한다. threshold/config 선택만 val을 사용하고 test는 봉인한다. 원본·동결·manifest·기존 `dummy_validate.py`는 수정하지 않고 비낙상 검증은 신규 wrapper/script로 처리한다.
- **비낙상 균형 증강 Q4 잠금 (leakage / 이중증강 가드, Codex/Claude 합의, 파일 기준):** 비낙상 더미 origin은 `manifest_v2_manual_augmented.csv`가 아니라 `debug/modeling/diag_out/onset_detector/item4/item4_cache_fixed.npz` 또는 동일 canonical split_map의 `split_assignment == "train" AND y != 0` row 기준으로 선택한다. 현 item4 구조에서 fall split은 manifest_v2가 authoritative이지만, non-fall split은 `model/finetune/cache/safesignal_e1234_pretrained6.npz`에 canonical within_subject seed42 splitter를 적용해 만든 split_map 기준이다(`debug/modeling/item4_build_policy_cache.py`). 따라서 non-fall 경로에서 manifest_v2 split을 참조하면 split source를 혼동하는 설계 오류다. 비낙상 더미는 새 split을 하지 않고 모두 train으로 append하며, val/test origin 사용은 즉시 leakage로 간주해 assert 실패시킨다. leakage/group key는 window가 아니라 `origin_filename`이다. 비낙상은 한 origin session에서 sliding window가 여러 장 나오므로, window 단위 split이나 window 단위 group key를 쓰면 같은 세션 파생 window가 train/val/test로 흩어져 원본↔파생뿐 아니라 파생끼리도 누수된다. 다만 filename만으로는 같은 세션의 정확한 원본 window를 복원할 수 없으므로 lineage 재현 key로 `origin_cache_row_idx`와 raw sliding 기준 `origin_window_start`도 반드시 기록한다. 필수 lineage/meta 컬럼은 `aug_filename`, `origin_cache`, `origin_cache_row_idx`, `origin_filename`, `origin_group_key`, `origin_activity`, `origin_y`, `origin_split`, `origin_subject`, `origin_env`, `origin_trial`, `origin_window_start`, `transform_profile`, `transform_params`, `aug_status`, `reject_or_pending_reason`, `is_augmented=True`, `source`(dummy 식별 가능 값)이다. fall lineage와 공통 컬럼은 맞추되, onset 전용 컬럼(`expected_onset`, `onset_delta`, `rise_not_found`)을 non-fall에 억지로 의미 부여하지 않는다. online augmentation skip은 코드 단정만으로 닫지 않고 런타임 카운터로 검증한다. 현재 `debug/modeling/item4_train.py` monkeypatch는 `is_augmented`가 아니라 filename marker(`AUGMENT_FILENAME_MARKERS`, 예: `__aug` → `_AUG` match)를 보고 skip하므로, non-fall 더미 filename도 반드시 `__aug`/`_AUG` marker를 포함해야 한다. 주의: vanilla `model/finetune/train.py` 경로만 쓰면 `SafeSignalDataset.from_npz()`가 npz의 `source` 컬럼을 무시하고 모든 row를 `safesignal`로 덮어쓰며, `TrainAugmentDataset.__getitem__()`은 `meta["source"] == "safesignal"`만 보고 online augmentation을 적용한다. 따라서 더미 source를 `safesignal_dummy`로 넣는 것만으로는 frozen skip이 보장되지 않고, 오히려 sampler는 unknown source를 지원하지 않는다. balanced 실험은 반드시 별도 `item4_train_balanced.py`에서 raw-only assertion no-op + frozen split + filename-marker online-skip monkeypatch를 함께 적용해야 한다. run report에는 `dummy_total`, `dummy_marker_matched`, `dummy_online_aug_applied`, `is_augmented_online_aug_applied`를 출력하고 기대값은 dummy/is_augmented online augmentation applied 0건이다. 더미 생성·품질 gate·split/group/marker 결정은 train-only 기준으로 수행하고 test는 봉인한다. 원본·동결 파일·manifest·기존 `train.py`는 수정하지 않고, 신규 builder/train wrapper/report 산출물만 추가한다.
- **비낙상 균형 증강 Q5 잠금 (평가 arm / 통계, Codex/Claude 합의):** balanced 증강 arm 이름은 기존 fall-only 비통제 `fixed_aug`/`onset_primary_aug`와 혼용하지 않는다. A=`fixed`, B=`onset_primary`, C=`fixed_balanced_aug`, D=`onset_primary_balanced_aug`로 잠그고 파일/cache/ckpt/report도 `*_balanced_aug` 네이밍으로 분리한다. Primary contrast는 onset arm 내부 `D - B`(onset_primary_balanced_aug vs onset_primary)이며 Delta FAR/Delta recall을 paired로 본다. Secondary는 `C - A`이며 단순 sanity가 아니라 D-B 해석의 대조군이다. balanced 증강이 fixed에서도 나쁘면 D 개선을 onset 맥락으로 귀속하기 어렵기 때문이다. Interaction `(D-B) - (C-A)`는 참고 지표로만 남기며, 이전 fall-only 비통제 실험처럼 interaction을 primary로 두지 않는다. 이전 fall-only 4-arm은 class artifact 검정이고, 이번 balanced 실험은 arm 내부 직접 대비로 "arm별 effective ratio를 보존한 추가 coverage/diversity가 도움 되는가"를 묻는 별도 실험이다. 일정상 A/B는 항목4 기존 결과를 조건부 재사용하고 C/D만 학습한다. 단 재사용 조건은 `item4_cache_fixed.npz`, `item4_cache_onset_primary.npz`, `split_assignment`, seeds 42~46, pretrained checkpoint, train args(`source_ratio=0.60`, `hard_weight=1.30`, `weight_decay=1e-4`, `patience=8`, `early_stop_start=8`, `epochs=40` 등), eval windows/event eval grid, checkpoint root가 모두 동일해야 한다. 동일성은 checksum 또는 config diff로 report에 남기며, 보증 실패 시 A/B도 재학습한다. 평가 출력은 세 종류로 잠근다: primary table은 `selected_by_val` 기준 D vs B, secondary table은 `fixed_baseline` 기준 D vs B, frontier table은 FAR<=0.15 / <=0.20 / <=0.30 / max-F1이다. val-selected 운영점은 항목4 구조와 동일하게 허용하되 threshold/config는 val에서만 고르고 sealed test에는 1회만 적용한다. 통계는 seed별 방향성을 보존한다. seed별 Delta FAR/Delta recall을 출력해 directional candidate를 판정하고, 95% CI는 기존 item4 session bootstrap 방식과 맞추며, 추가로 5-seed mean±std를 보고한다. seed pooled bootstrap 하나로만 닫지 않는다. 판정 기준은 Q2와 동일하다: `statistically supported`는 paired 95% CI가 0을 제외할 때, `directional candidate`는 5 seed 중 4개 이상 같은 방향이며 (Delta FAR <= -0.05 또는 Delta recall >= +0.03)을 만족하되 반대 지표가 크게 악화되지 않을 때로 제한한다(Delta FAR 개선 후보는 Delta recall >= -0.05, Delta recall 개선 후보는 Delta FAR <= +0.05). 그 외는 null이다. test 봉인, train-only 결정, 원본·동결·manifest·기존 train.py 무수정 원칙을 유지하고 신규 builder/train_balanced wrapper/report만 추가한다.
- **비낙상 균형 증강 Q6 잠금 (산출물 위치 / git·Drive·ignore 분류, Codex/Claude 합의, 파일 기준):** 분류 기준은 용량뿐 아니라 재현 가능성이다. 재현 불가능하거나 결과 해석·leakage 검증에 필요한 경량 산출물은 git 추적 필수이고, 재현 가능한 대용량 cache/checkpoint는 로컬 + Drive 수동 업로드 대상으로 둔다. 실제 `.gitignore` 확인 결과 `debug/modeling/diag_out/` 전체가 ignore되어 있으므로, balanced lineage/report/checksum을 `debug/modeling/diag_out/onset_detector/item4/balanced/` 아래에 두면 추적 필수 증거가 ignore된다. 따라서 추적 필수 산출물은 ignore되지 않는 별도 경량 디렉터리에 둔다. 잠정 위치는 `debug/dummy_gen/balanced/` for non-fall dummy lineage/report/checksum, `debug/modeling/balanced_aug/` for balanced eval report/config diff/checksum/frontier json/md/csv이다. 신규 스크립트는 `debug/dummy_gen/dummy_generate_nonfall_balanced.py`(또는 동등 명칭), `debug/modeling/item4_build_balanced_arm_caches.py`, `debug/modeling/item4_train_balanced.py`, `debug/modeling/item4_balanced_eval.py`처럼 기존 스크립트 옆에 둔다. git 경량 추적 필수: 신규 스크립트, `debug/dummy_gen/balanced/lineage.csv` 또는 `nonfall_lineage.csv`, leakage/quality summary json/md, A/B reuse checksum/config diff, balanced eval report md/json, frontier table, effective-share report, online-skip counter report, HANDOFF.md balanced 섹션. Drive 대용량 수동 업로드: `debug/modeling/diag_out/onset_detector/item4/item4_cache_*_balanced_aug.npz`, 필요 시 balanced dummy npz, `model/finetune/checkpoints_item4_balanced*/` checkpoint(.pt). ignore/재생성 가능: Q2 실측 또는 builder로 재생성 가능한 `debug/modeling/diag_out/onset_detector/item4/item4_cache_*.npz`, `*_balanced_aug.npz`, eval window pkl, checkpoint 디렉터리. 구현 시 `.gitignore`에는 balanced 대용량만 명시적으로 추가한다(예: `debug/dummy_gen/balanced/*.npz`, `debug/dummy_gen/balanced/validation/` 등). `*.csv` 같은 광범위 ignore는 추가 금지이며, lineage csv가 `git check-ignore`에 걸리지 않는지 CI/수동 확인한다. 기존 `model/finetune/checkpoints_*/`는 checkpoint ignore에 이미 걸리고, `debug/modeling/diag_out/`는 cache류 ignore에 이미 걸린다. `HANDOFF.md`에는 balanced 실험 분류와 이어받기 절차를 별도 섹션으로 추가한다. 원본·동결 파일·manifest·기존 train.py는 수정하지 않는 read-only 원칙을 유지한다.
- **자체 검토 보강 (Codex, 2026-06-08):** Q1~Q6 설계는 큰 방향에서 문제 없으나 실행 전 fail-fast 항목을 추가해 잠근다. (1) non-fall dummy는 z-SDP 직접 변형이 아니라 raw sliding window perturbation 후 `window_to_model_input()` 재생성. (2) `fixed`/`onset_primary` non-fall cache 동일성은 파일 기준으로 검증 완료(row 2323, metadata equal, `X` max_abs_diff=0.0)했으므로 하나의 non-fall dummy set을 C/D에 공유. (3) dummy origin/gate/배수/상한은 train-only, threshold/config만 val-selected, sealed test는 1회. (4) A/B/C/D effective share target error는 성능표보다 먼저 보고하고 tolerance 초과 시 실패 처리. (5) directional candidate는 한 지표 개선만 보지 않고 반대 지표 악화 가드를 함께 적용. (6) 더미 mass 상한은 raw window count 기준. (7) class별 품질 gate 통과 수가 매우 적은 클래스는 sampler share가 맞아도 diversity contribution이 작다고 별도 표시. (8) lineage에는 `origin_window_start`를 포함해 raw window 재현성을 보장. (9) 본 학습 전 1seed/1epoch smoke test로 online augmentation applied 0건, effective share match, val/test dummy 0건, A/B reuse checksum/config diff, C/D eval 로딩을 확인한다.
- **multi-window 설계 결정 목록 (잠금 대기):**
  1. shift set & 폭 근거: 후보 `{[-70:+230],[-50:+250],[-30:+270]}` onset±20 jitter 3장. 폭은 train/val(non-WALK pooled, test 제외) onset 분위수와 세션별 auto/manual onset 불일치 분포로 정당화한다(D-031 Q4).
  2. session-normalized weighting: 3장 세션의 1장 세션 과대표 방지. WeightedRandomSampler(source-ratio 정규화 + hard non-fall weight) 통합 지점과 로그(class raw count, effective sampled mass, 세션별 window count p50/p90/max, fall 세션당 shift 수 분포)를 설계한다.
  3. crop_out_of_bounds: clamp 금지, OOB window 제외, coverage 분리 집계(D-031 v2). center [-50:+250]는 usable_for_onset_aligned라 in-bounds 보장, -70 pre/+270 post 끝만 OOB 가능.
  4. non-fall 불변: multi-window는 fall window에만 적용하고 non-fall은 기존 sliding + sampler 균형을 유지한다(D-031 Q3).
  5. 비교 arm/평가: single onset_primary(baseline-axis) vs multi-window, non-WALK pooled paired sealed test N=27, event-level, baseline-fixed config, 폭/weighting은 train/val로만 결정하고 test는 봉인한다.
  6. 후보1과 분리: multi-window 단독 확인 후 필요 시 균형 증강을 적층한다.
- **Status:** triage 확정. 실행 우선순위는 2026-06-09 비낙상 더미 생성 및 균형 증강 4-arm 재평가다. multi-window는 전략상 ROI가 높은 다음 레버로 설계 1번부터 순차 잠금한다.

### 2026-06-08 — D-031 항목4 onset-aligned + 더미 4-arm 결과 정정 (Codex)

- **근거:** `wifi-csi-fall-detection` 원격 `origin/feature/event-centered-gate1` 최신 `HANDOFF.md` 및 STATE D-031 질의응답 내용을 대조했다. 현재 로컬 코드 브랜치는 원격보다 22커밋 뒤처져 있어 로컬 파일 목록만 보면 항목4 스크립트가 누락된 것처럼 보일 수 있다.
- **onset 정렬 결론:** sealed test non-WALK pooled paired 비교(N=27 fall, 180 non-fall)에서 `onset_primary`는 `fixed` 대비 recall 차이는 비유의(Delta -0.015, McNemar p=0.727)이나 FAR는 유의하게 감소(Delta -0.139, 95% CI [-0.191, -0.091])했다. 결론은 "같은 recall에서 FAR를 줄이는 alignment effect"로 한정한다. 목표 recall>=0.85 AND FAR<=0.15 동시 달성은 실패했고, event-level 180:27 불균형 때문에 F1>=0.85는 구조적으로 주장하지 않는다.
- **더미 4-arm 결론:** fall-only 더미를 붙인 비통제 비교에서는 FAR 악화가 보였으나, fall effective mass를 원본 수준으로 고정한 class-ratio 통제 후 B->D Delta FAR는 +0.028 [-0.014, +0.071]로 유의차가 없다. 따라서 "더미가 onset FAR을 악화"가 아니라 "fall-only 증강으로 생긴 class 불균형 artifact"가 맞다. 현 더미 자체 효과는 null(중립)로 기록한다.
- **다음 후보 정정:** D-020 per-lag는 정식 트랙1 ablation에서 net negative로 기각됐고 최종 정규화는 global z-score 유지다. 최신 후속 개선 후보는 (1) 비낙상 포함 균형 증강, (2) multi-window 조건부 보조, (3) 신규 세션 수집 순으로 본다. multi-window는 STATE D-031 Q4처럼 train fall window/effective mass 부족과 recall 부족 신호가 함께 있을 때만 발동하는 보조 수단이며, 단독 1순위로 표현하지 않는다.
- **코드/산출물 위치:** 스크립트는 `debug/modeling/{build_gate3_cache,item4_build_policy_cache,item4_train,item4_precompute_eval_windows,item4_event_eval,item4_build_arm_caches,item4_arms_eval,sanity_gate_item4}.py`, `debug/dummy_gen/{dummy_clean400_lib,dummy_sanity,dummy_generate,dummy_validate}.py`에 있다. 대용량 cache/checkpoint/dummy npz는 git이 아니라 Drive 수동 업로드 대상으로 둔다.


### 2026-06-04 — 더미데이터 수령 전 event-centered / beep timing / RPCA 조건 정리 (Codex)

- **목적:** 팀원 더미데이터를 받은 뒤 바로 검증·학습 판단을 이어갈 수 있도록, 현재까지 확인된 운영점/전처리/시간축 이슈와 필수 preflight를 정리한다. 실제 코드 판단의 진실 공급원은 `wifi-csi-fall-detection` repo이며, 이 항목은 진행 상태 기록이다.
- **Run A / threshold 상태:** fw1.5 Run A는 `best_operating` selector가 threshold=0.20을 골라 s43 recall 0.736/FAR 0.094/F1 0.721로 보였지만, split 재현 assert 통과 후 고정 threshold sweep에서 s43 `fw1.5 @ thr0.05`는 recall 0.799/FAR 0.144/F1 0.706으로 baseline s43(0.778/0.150/0.687)을 모두 개선했다. 단 이는 sealed test sweep에서 운영점을 고른 결과라 test-set tuning caveat가 있으며, 발표용 공식 수치로 쓰려면 threshold를 사전에 고정한 3-seed 재현이 필요하다. 이후 사용자/Claude 보고상 thr0.05는 3-seed 평균 FAR이 0.169로 FAR≤0.15 조건을 넘겼고, thr0.10/0.15 fixed fallback 평가가 후속 과제로 잡혔다(STATE 작성 시점 Codex가 산출물 직접 검증하지 않음).
- **증강 위치 / RPCA 조건:** 현재 train 증강은 전처리·cache 생성 단계가 아니라 `TrainAugmentDataset` 계열의 train-time/on-the-fly tensor augmentation이다. CSV→resample/window/RPCA/ACF/SDP/z-score→cache 이후 train subset에만 적용되는 구조로 이해해야 한다. 반면 RPCA max_iter=30/200은 전처리 조건이다. 기존 정식 raw SDP/no-motion/track1 계열은 RPCA max_iter=200 기준으로 기록되어 있으므로, 팀원 더미데이터 결과가 RPCA 30회 제한으로 나온 경우에는 “더미데이터 효과”와 “전처리 조건 변경 효과”가 섞인다. 공식 비교 전에는 RPCA 200 재현 또는 RPCA 30을 새 파이프라인 조건으로 명시하고 train/val/test/realtime 모두 같은 조건으로 맞춰야 한다.
- **4초 프로토콜과 실제 5.5초 녹화:** `collect/labels.py`의 fall stage는 pre 1s + fall 2s + post 1s지만, `collect_main.py`/`server/collect_manager.py`는 recording 시작 후 각 stage마다 `beep_stage()` 0.5s blocking을 실행한 뒤 sleep한다. ready beep/countdown은 녹화 전, end beep은 녹화 후다. 따라서 정상 fall CSV는 대략 5.5초이며 타임라인은 stage1 beep 0.0~0.5s, pre 0.5~1.5s, stage2 beep 1.5~2.0s, fall sleep 2.0~4.0s, stage3 beep 4.0~4.5s, post 4.5~5.5s다. 기존 `FALL_RANGE_S=(1.0,3.0)`/frame 100~300 가정은 beep 포함 실측 시간축과 어긋난다. 실제 event-centered 후보 구간은 frame 150~400(stage2 beep 시작~fall sleep 끝), frame 200~400(fall sleep only), 또는 불확실성 흡수를 위한 frame 150~450 broad search로 재검토해야 한다.
- **beep 제거 가능성과 리스크:** resample 후 정상 550±5 frame 세션에 대해 `[0:50]`, `[150:200]`, `[400:450]`을 제거하면 400 frame(pre 100/fall 200/post 100)으로 복원할 수 있다. 그러나 제거 후 concat은 시간 연속성을 끊기 때문에 RPCA→ACF→SDP에서 접합부 artifact가 생길 수 있다. event-centered 구현 전에 read-only 진단으로 원본 continuous crop(예: 150:450/200:500)과 beep-removed concat(400 frame에서 50:350 등)의 SDP cosine/mean abs diff/energy curve corr/peak 위치/모델 확률을 비교해야 한다. 접합 artifact가 크면 concat 대신 원본 continuous window crop 또는 peak-centered crop을 우선한다.
- **더미데이터 수령 후 preflight:** 팀원 더미데이터가 “4초”라고 해도 처리 방식은 생성 경로에 따라 다르다. (1) 5.5초 beep 포함 원본을 4초로 압축했다면 beep도 압축되어 고정 1.5초 제거는 잘못이다. (2) beep 제거 후 clean 4초를 변형했다면 더미는 그대로 쓰고 실측만 clean 4초로 맞추는 것이 맞다. (3) 실제 파일이 5초대라면 실측과 같은 처리 후보가 필요하다. 수령 즉시 frame count 분포(~400/~550/가변), timestamp/duration, 원본-더미 origin 매핑, beep/low-motion 흔적, RPCA max_iter, split leakage 여부(원본과 파생 더미가 train/test로 갈라지지 않도록 group split)를 확인한다.
- **다음 실행 순서 제안:** ① 실측 5.5초 데이터에서 beep 제거 concat ACF/SDP artifact 진단(read-only), ② 더미데이터 파일 수령 후 시간 구조/RPCA/split leakage preflight, ③ 양쪽 시간 구조를 동일 기준으로 맞춘 cache builder 설계, ④ session/origin 단위 split을 먼저 고정한 뒤 peak-centered single/multi-window를 생성한다. 더미 또는 multi-window를 만든 뒤 파일 단위 random split을 하면 leakage 위험이 크므로 금지한다.
### 2026-06-03 — D-020 per-lag 트랙2/트랙1 결과 및 최종 정규화 선택 (Codex)

- **검증 입력:** `debug/modeling/diag_out/track2_probe_comparison.json`, `track2_probe_cache_summary.json`, `track1_formal_comparison.json`, `track1_formal_cache_summary.json`을 직접 확인했다. worktree에는 로컬 실행 스크립트 `debug/modeling/track1_formal_make_caches.py`, `track1_formal_analyze.py`, `track2_probe_make_caches.py`, `track2_probe_analyze.py` 4개가 untracked이며, main 커밋 이력은 2026-05-29 이후 `721718e`→`79a8f20`까지 기존 정리와 동일하다.
- **트랙2 PROVISIONAL mixed-normalization probe:** SafeSignal만 per-lag, Alsaify는 global cache 고정이라 D-020 정식 결론용이 아니다. control gate는 seed42 A(global) R=0.7153/FAR=0.1609로 baseline reference R=0.781/FAR=0.134 대비 recall -0.0657, FAR +0.0269라 경고 band(±0.05~0.10)다. 3-seed 평균은 A(global) R=0.7384±0.0590/FAR=0.1444±0.0190/F1=0.6688±0.0280, B(per-lag mixed) R=0.7269±0.0579/FAR=0.1451±0.0165/F1=0.6614±0.0457. standing→fall은 평균 -0.0509로 좋아 보이나 seed44에서는 악화되어 부호 불일치이고, walking→fall은 +0.0586으로 3-seed 모두 악화했다. 해석: 약한 mixed signal이며 단독 기각/채택 근거가 아니다.
- **트랙1 D-020 정식 ablation:** SafeSignal+Alsaify 모두 같은 raw/order/HP에서 A=both global, B=both per-lag로 비교했다. cache summary 기준 SafeSignal A-global은 기존 finetune7 global과 allclose true(max_abs_diff=2.3841858e-7), Alsaify A-global self-consistent z-score는 ok(`z_mean_absmax=1.5412058e-7`), per-lag clipped_ratio는 SafeSignal 0.00187, Alsaify 0.00324(`STD_FLOOR=1e-4`, `EPS=1e-6`, `CLIP=3.0`)다. 6-class 파생 후 shapes/counts는 SafeSignal 3041 windows(`fall 718 / walking 540 / sit_stand 524 / lying 539 / standing 360 / picking 360`), Alsaify 8326 windows(`fall 1600 / walking 1600 / sit_stand 1515 / lying 1600 / standing 1211 / picking 800`)로 라벨 0..5 범위가 유지됐다.
- **트랙1 결과와 결론:** A(both global)는 R=0.7384±0.0378/FAR=0.1437±0.0311/F1=0.6708±0.0186, B(both per-lag)는 R=0.7083±0.0547/FAR=0.1594±0.0430/F1=0.6374±0.0143이다. paired delta는 recall -0.0301(부호 불일치), FAR +0.0157(3-seed 일관 악화), F1 -0.0334(3-seed 일관 악화). standing→fall은 평균 -0.0880이나 seed43 무변동으로 일관성이 없고, walking→fall +0.0617 및 picking→fall +0.0509는 일관 악화다. 따라서 D-020 B안 per-lag는 정식 ablation 기준 기각하고, 최종 정규화는 A안 global z-score를 유지한다.
- **다음 방향:** event-level sweep과 forward/tail 진단상 후처리만으로 recall 0.85/FAR 0.15 동시 달성이 어렵고, per-lag도 net negative였다. 다음 개선 후보는 post-fall 정적 단서 의존을 줄이고 transient supervision을 강화하는 event-centered windowing/labeling이지만, 아직 결정·구현·실험된 항목은 아니므로 Pending 탐색 과제로만 둔다.

### 2026-06-03 — 2026-05-29 이후 모델 진단/트랙2 probe 진행분 정리 (Codex)

- **대조 범위:** `wifi-csi-fall-detection`의 2026-05-29 이후 커밋은 `721718e`, `0245445`, `d7c3aaa`, `0b61298`, `9e3064d`, `0b25a49`, `237c93f`, `0772a74`, `d291c87`, `6b7705e`, `79a8f20` 순서로 확인했다. 현재 `HEAD == origin/main == 79a8f20`이고 worktree에는 로컬 per-lag 실행 스크립트 4개(`track1_formal_*`, `track2_probe_*`)가 untracked다.
- **데이터/학습 준비 통합(2026-06-01):** `721718e`와 merge `0b61298`는 데이터 품질 검사와 학습 준비 도구를 main에 통합했고, `0245445`는 ESP32-S3 CPU 클럭 240MHz 적용을 포함한다. 이 내용은 기존 `2026-06-01 — 최종 데이터/학습 이관 준비 상태 및 main 통합 기록`에 이미 반영되어 있어 중복 추가하지 않았다.
- **on-the-fly 증강 + within_subject/pretrained6 경로(2026-06-01):** `9e3064d`는 `TrainAugmentDataset` 실제 증강 연결과 `verify_aug_gate.py` 검증 게이트를 추가했고, `0b25a49`는 `--class_policy pretrained6`, `--split within_subject`, `--threshold_min` 경로를 추가했다. 기존 Review Notes에 구현 내용은 반영돼 있으나, 데모 primary 6-class 확정은 Decisions Log에 없어서 [D-030]으로 추가했다.
- **6-class/7-class CPU 비교 및 병목:** `docs/CODEX_HANDOFF_2026-06-02.md`(commit `237c93f`) 기준 CPU-only 30epoch within-subject 비교는 6-class `pretrained6` R=0.736/F1=0.660/FAR=0.152, 7-class `finetune7` R=0.646/F1=0.583/FAR=0.143이다. 7-class false-positive share는 running 약 0.244로 기록되어 6-class demo-primary 결정의 근거가 됐다. 현재 로컬 `checkpoints_compare6_cpu/within_subject_test_report.json`도 threshold=0.1, R=0.736111, F1=0.660436, FAR=0.152361, `standing→fall` FP share=0.309859를 확인했다. 사용자 기준선의 `checkpoints_compare6_cpu best_operating R=0.781/F1=0.698/FAR=0.134`는 현재 로컬 `checkpoints_compare6_cpu` 산출물과 일치하지 않으며, 비슷한 수치는 2026-06-03 track2 A안 seed42의 별도 run에서 R=0.715/F1=0.640/FAR=0.161로도 재현되지 않았다. 따라서 STATE에는 산출물 기준 0.736/0.660/0.152를 기록한다.
- **zero-shot subject 진단:** `debug/modeling/eval_zeroshot_by_subject.py`(commit `0b25a49`)는 stdout-only라 기존 산출 파일은 없었다. Codex가 `.codex-test-venv` CPU로 재실행해 확인한 결과, Alsaify `best.pt`를 SafeSignal `safesignal_e1234_pretrained6.npz`에 fine-tuning 없이 적용하면 threshold 0.30에서 S01 R=0.146/FAR=0.028/F1=0.236, S02 R=0.076/FAR=0.021/F1=0.132, S03 R=0.246/FAR=0.026/F1=0.370이다. threshold sweep 0.30~0.70에서 best-recall도 모두 threshold 0.30이었다. 결론은 도메인 갭(Alsaify Intel 5300 ↔ ESP32 SafeSignal)과 피험자 3명 데이터 한계가 recall 천장의 주원인이라는 해석을 지지하며, 파이프라인 자체 결함으로 단정할 근거는 없다.
- **Step1 fall window 희석 진단:** `d291c87`는 `debug/modeling/diag_fall_window_dilution.py`를 추가했고, 산출물은 `debug/modeling/diag_out/fall_window_dilution_summary.csv` 등이다. CSV 144 rows(6 subtypes × 12 files × forward/tail) 기준 `n_frames` mean≈552라 tail window는 대체로 세션 후반 `[2.5s:5.5s]`에 걸린다. 전체 평균으로 forward `energy_in_fall_ratio`=0.7531, `attn_in_fall_range`=0.9167, `peak_in_fall_rate`=0.9167이고, tail은 각각 0.2537, 0.0972, 0.5833이다. 즉 tail은 완전한 junk는 아니지만 대부분 post-fall stillness를 담아 weak-label transient dilution을 만든다.
- **Step2 event-level sweep/forward-tail 진단:** `0772a74`는 `debug/modeling/diag_event_sweep.py`를 추가했다. 2026-06-03 기존 STATE Review Notes의 step2 수치(event_recall 최대 0.6528 @ FAR 0.1111, forward/tail threshold=0.2 recall 0.5417/0.8333, tail-only rescue=31)는 이미 반영되어 있어 중복하지 않는다. 단 `237c93f` handoff 문서에 남은 Step3-a(tail down-weight) 제안은 step2 결과 이후 stale이며, 현재 Pending/Review Notes는 tail down-weight 폐기 방향으로 정정한다.
- **checkpoint metadata fix:** `6b7705e`는 `model/finetune/train.py`의 `save_checkpoint()`에 `class_policy`와 정책별 `classes` 저장을 보강하고, 빈 batch 평가 시 output shape를 모델 output dim 기준으로 반환하도록 수정했다. 이는 6-class/7-class checkpoint 혼동 방지용이며 [D-030] 및 step2 checkpoint 검증과 연결된다.
- **D-020 per-lag 트랙 구조:** 트랙1은 Alsaify+SafeSignal 모두 raw SDP에서 동일 정규화로 비교하는 정식 ablation이고, 트랙2는 SafeSignal만 per-lag, Alsaify는 기존 global z-score cache로 고정하는 `PROVISIONAL mixed-normalization probe`다. 트랙2는 positive면 트랙1 투자 가치가 있다는 신호로 보고, null/negative는 mixed-normalization + global-pretrained warm-start mismatch 핸디캡 때문에 단독 기각 근거로 쓰지 않는다. per-lag 수식은 per-window self-normalization이며 4D X=(N,1,28,20)에서 axis=2(시간 28) 축약, lag 20개 독립, `STD_FLOOR=1e-4`, `EPS=1e-6`, `CLIP=3.0`로 고정한다. train split 통계 fit 방식은 사용하지 않으므로 split 이전 cache 생성이 val/test 통계 누수가 아니다. 단 D-020의 리스크(RPCA와 역할 중복으로 증분 제한, high-lag noise 과증폭으로 FAR 증가)는 유지한다.
- **SafeSignal raw SDP cache:** `origin/exp/perlag-cache`의 `debug/modeling/build_rawsdp_cache.py`(`0db4c26`)로 생성된 `model/finetune/cache/safesignal_e1234_finetune7_rawsdp.npz`가 현재 로컬에 존재한다. 확인 결과 `X=(3581,1,28,20)`, `normalization=none_raw_sdp`, `classes=finetune7`, `is_augmented` 전부 False, full provenance(`source/env/filename/activity/trial/within_file_index`) 포함. `debug/modeling/diag_out/track2_probe_cache_summary.json` 기준 A안 raw→global allclose vs 기존 finetune7 X는 true, max_abs_diff=2.3841858e-7이다.
- **Alsaify raw SDP cache:** `79a8f20`은 `debug/modeling/build_alsaify_rawsdp_cache.py`를 추가했다. 로컬 dry-run 로그(`debug/modeling/diag_out/alsaify_rawsdp_build.log`)는 5파일/10윈도우 raw std≈0.05475, self-consistent global z-score 후 mean≈0/std≈1 PASS를 기록했다. 현재 로컬에는 팀원 PC 본 빌드 산출물로 보이는 `model/pretrained/checkpoints/dataset_cache_e12_w300_s300_lag1_20_tail_ps_rawsdp.npz`가 있으며, 확인 결과 `X=(8326,1,28,20)`, `normalization=none_raw_sdp`, raw X.std=0.046756, class counts `{fall:1600, walking:1600, sit_stand:1515, lying:1600, standing:1211, picking:800}`로 기존 z-score Alsaify cache와 분포가 일치한다. 다만 사용자 기준선의 build time 145.5min은 현재 로컬 로그에는 없어서 STATE에 확정 수치로 쓰지 않는다.
- **Track2 provisional probe 상태 정정:** 2026-06-03 후속 실행으로 B안까지 완료되어 `track2_probe_comparison.json`이 생성됐다. 이 이전 기록의 “B안 미완료” 상태는 stale이며, 상세 수치와 해석은 위 `D-020 per-lag 트랙2/트랙1 결과` 엔트리를 기준으로 한다.
- **구현 상태 stale 정정:** `server/main.py`는 `start_receivers()`로 UDP 수신을 시작하고, `RPiConnection` WebSocket 서버, `send_fall_sms()` SOLAPI 알림, `InferenceWorker`를 통합한다. `server/inference/buffer.py`는 `_resample_uniform()`과 `np.interp` 기반 timestamp-aware 100Hz realtime resampling을 이미 구현했다. 따라서 Implementation Status와 Pending Items의 UDP/WebSocket/realtime resampling pending 표현을 코드 기준으로 정정한다. 단 Pi4 하드웨어 버튼 인터페이스, E2E 실기 통합 테스트, non-fall 저장 방식, HPO 구현, 3-fold pooled threshold selector, gap-quality 저장 차단, stride latency 실측은 계속 pending이다.

### 2026-06-03 — step2 event-level sweep 재검증 및 per-lag raw SDP cache 확인 (Codex)

- **검증 입력:** `debug/modeling/diag_out/event_sweep_results.csv`, `forward_tail_split_diag.csv`, `step2_run.log`, `diag_event_sweep.py`, `model/finetune/cache/safesignal_e1234_finetune7.npz` 및 `.summary.json`, `origin/exp/perlag-cache:debug/modeling/build_rawsdp_cache.py`를 확인했다. 최초 기록 시점에는 raw SDP cache/handoff flag를 확인하지 못했으나, 2026-06-03 후속 대조에서 `model/finetune/cache/safesignal_e1234_finetune7_rawsdp.npz`와 `track2_probe_cache_summary.json`을 확인했고 raw→global allclose max_abs_diff=2.3841858e-7로 정정한다. `D:\handoff` flag 자체는 현재 파일시스템에서 확인하지 못했으므로, 완료 근거는 로컬 `.npz`/summary 산출물 기준이다.
- **step2 split/checkpoint sanity:** `step2_run.log` 기준 within-subject split은 seed=42, val=0.2, test=0.2 재실행으로 total=1259/train=800/val=207/test=252 세션, held-out fall=72/non-fall=180(RUN 제외 pretrained6). 학습 cache 동일 2-window sanity에서 fall window recall @0.1 = 0.736(106/144)로 `within_subject_test_report.json`의 fall_recall=0.736과 일치한다고 로그에 기록되어 split·checkpoint 재현은 일관된다. checkpoint는 `model/finetune/checkpoints_compare6_cpu/best_operating.pt`이며 로그가 CPU 30epoch fallback으로 명시하므로 절대 성능 수치가 아니라 후처리 sweep 방법검증용이다. 본판 `within_6class_recall_aug`/GPU 학습 checkpoint 재검증은 이번 산출물에 없으므로 다음 과제로 유지한다.
- **event-level sweep 결과:** `event_sweep_results.csv`는 60 config(5 thresholds × 2 N × 3 margin × 2 stride). FAR≤0.15 후보는 18개이며, event_recall 최대 운영점은 `threshold_min=0.1`, `N=2`, `margin=on_m0.2`, `stride=50`에서 event_recall=0.6528, event_FAR=0.1111, event_F1=0.6763, TP=47/FP=20/FN=25/TN=160이다. 동일 metric tie로 threshold 0.15/0.2/0.25/0.3도 같은 수치다. recall≥0.85 config는 27개이나 최저 FAR가 0.2944라 FAR≤0.15 목표와 양립하지 않는다. N=1/margin off는 recall 0.9167~0.9861이지만 FAR 0.3556~0.6833으로 사용 불가다. 결론: threshold/N/margin 후처리만으로 D-011 recall 0.85/FAR 0.15 동시 달성은 어렵고 모델 자체 개선이 필요하다. N=2는 특히 stride=100에서 spike형 단일-window 낙상 recall을 크게 깎는다.
- **forward/tail 진단:** `forward_tail_split_diag.csv` 기준 threshold=0.2에서 forward recall=0.5417(39/72, FN=33, non-fall FP=30/286), tail recall=0.8333(60/72, FN=12, non-fall FP=20/180), tail-only rescue=31, forward-only=10이다. threshold=0.3에서는 forward recall=0.4722, tail recall=0.7917, tail-only rescue=32, forward-only=9다. 기존 step1의 tail post-fall attention 해석과 맞게 모델은 낙상 순간보다 낙상 후 바닥 정적(post-fall)을 강한 fall 단서로 쓰는 것으로 보인다. 따라서 step3-a(tail down-weight/제외)는 recall 파괴 위험이 커서 남은 카드에서 제외한다.
- **per-lag raw SDP cache 상태:** 로컬 finetune7 cache는 `X=(3581,1,28,20)`, `y=(3581,)`, distinct filename=1439, class_counts=`fall 718 / walking 540 / sit_stand 524 / lying 539 / standing 360 / running 540 / picking 360`, `files_seen=1440`, `files_used=1439`, `skipped={"empty_windows":1}`로 확인했다. `origin/exp/perlag-cache`의 `build_rawsdp_cache.py`는 이 filename 목록을 입력 소스로 쓰고 디렉터리 glob을 쓰지 않으며, raw SDP를 `normalization="none_raw_sdp"`로 저장하고 provenance(`filename/within_file_index/y/subject/env/activity/trial/classes/class_policy`)를 동봉하도록 설계되어 있다. 2026-06-03 후속 대조 기준 `safesignal_e1234_finetune7_rawsdp.npz`가 로컬에 존재하고, `track2_probe_cache_summary.json`에서 A안 raw→global allclose max_abs_diff=2.3841858e-7을 확인했다.
- **다음 과제 정정:** 트랙2와 트랙1은 2026-06-03 후속 실행으로 완료됐고, D-020 정식 결론은 per-lag 기각/global 유지다. 남은 모델 개선 후보는 본판 checkpoint로 step2 forward/tail 패턴 재검증, 그리고 event-centered windowing/labeling 탐색이다.

### 2026-06-01 — fine-tuning train.py: within_subject 데모 평가 모드 + pretrained6 6-class 경로 + on-the-fly 증강 연결 (진규)

> 작성: claude-code (2026-06-02 반영). 어제 진규가 `feature/finetune` 작업으로 진행 후 main에 push한 코드. CODEX는 아직 모르던 상태라 여기 기록한다. 커밋: `9e3064d SafeSignal 온더플라이 증강 구현 및 검증 추가`, `0b25a49 feat: within_subject 평가 모드 + 6-class(pretrained6) 경로 + threshold_min 인자화`. 둘 다 `origin/main`에 반영됨.

- **변경 파일:** `model/finetune/train.py` (+~520/-50, 두 커밋 합산), 신규 `verify_aug_gate.py`, `debug/modeling/derive_pretrained6_cache.py`, `debug/modeling/eval_zeroshot_by_subject.py`, `.gitignore` 정리.
- **on-the-fly 증강 연결 (D-010/D-023 후속, 기존 pending이던 "TrainAugmentDataset 실제 증강 연결" 완료):**
  - `TrainAugmentDataset`가 SafeSignal 샘플에 jittering/scaling/time_warping/noise_scale **중 하나**를 `(seed, idx, epoch)` 기반 결정론적으로 적용. Alsaify 샘플은 증강 없이 통과(pass-through).
  - **b안(on-the-fly)** 방식 — 매 epoch `set_epoch()`로 증강이 갱신됨. train split 내부에서만 적용되며 val/test는 raw 유지 ([D-019]/[D-023] 원칙 유지).
  - `verify_aug_gate.py`: 증강이 의도대로 적용/미적용되는지(특히 Alsaify pass-through, 결정론성) 검증하는 게이트 스크립트.
- **pretrained6 6-class 경로 (running 제외, [D-006] best.pt 직접 평가용):**
  - `--class_policy {finetune7, pretrained6}` 신규. `finetune7`=기존 7-class 경로 **무변경**, `pretrained6`=running 제외 6-class로 현재 `best.pt`를 **strict 로드**(head migration 우회). 6-class 전용 함수/criterion/오탐분포 함수를 7-class와 평행하게 복제했고 기존 7-class 경로는 건드리지 않음.
  - `debug/modeling/derive_pretrained6_cache.py`: 7-class cache에서 running 제외 6-class cache를 파생 생성.
  - `eval_zeroshot_by_subject.py`: subject별 zero-shot(파인튜닝 전 best.pt) 진단 스크립트.
- **within_subject 데모 평가 모드 (⚠️ 정책 주의):**
  - `--split {cross_subject, within_subject}` 신규. `cross_subject`=기존 [D-019] 3-fold 경로 **무변경**(`--fold` 사용). `within_subject`=세션(filename) 단위 subject×class 층화 분할, `--fold` 무시, `--test_ratio`로 (subject,class)별 held-out 비율 지정.
  - 코드 주석/CLI help에 **"demo-only"**로 명시됨. `within_subject_test_report.json` 별도 출력. **[D-019]의 cross-subject가 여전히 primary 평가 기준이며 within_subject는 데모/진단용 보조 경로**다 — 공식 성능 근거로 쓰지 않는다.
  - within_subject 리포트의 pass/fail은 [D-011] 공식 기준(recall≥0.85/FAR≤0.15/F1≥0.85)을 별도 판정 키로 사용.
- **threshold_min 인자화:** `select_global_threshold(..., threshold_min=0.30)` + `--threshold_min` CLI. sweep 범위 `[threshold_min, 0.70]` step 0.05. 기본값 0.30으로 기존 7-class 동작 보존.
- **CODEX 참고 / 미해결:** ① within_subject를 정식 보고 기준에 편입할지 여부는 미결정 — 현재는 데모 전용. 필요 시 D-XXX로 승격 논의. ② pretrained6 zero-shot 베이스라인은 2026-06-03 Codex가 CPU로 재실행해 STATE에 기록 완료. ③ HPO 후속/gap-quality hook은 여전히 pending(아래 Pending Items 유지).

### 2026-06-01 — 최종 데이터/학습 이관 준비 상태 및 main 통합 기록

- 코드 상태: `wifi-csi-fall-detection`의 `codex/no-motion-baseline` 작업 브랜치를 `main`에 병합하고 원격 `origin/main`으로 push 완료. 최종 main 병합 커밋은 `0b61298 [병합] 데이터 품질 검사와 학습 준비 도구 통합`. 포함된 주요 커밋은 `721718e [수정] 데이터 품질 검사와 학습 준비 도구 정리`, `0245445 [수정] ESP32-S3 CPU 클럭 240MHz 적용`.
- 데이터 상태: 로컬 `data/raw` 최종 CSV는 1,447개 기준으로 정리됨. 기본 7-class 학습 대상은 1,440개이며, E3 `NO_MOTION` 7개는 `finetune7` cache 생성에서 제외된다. `data/cleaned`는 timestamp 정렬본 1,447개로 재생성 완료했고, 전체 품질 검사 결과는 총 1,447개 / OK 1,419 / WARN 28 / RECOLLECT 0 / ERROR 0.
- Drive 상태: Google Drive `SafeSignal_Dataset`에는 raw 데이터가 정리되어 있고, `cleaned`는 zip 상태로 저장되어 있음. 학습 artifact는 `SafeSignal_Dataset/SafeSignal_Training_Artifacts/`에 별도 보관하는 계획이며 대상 파일은 `safesignal_e1234_finetune7.npz`, `safesignal_e1234_finetune7.summary.json`, `best.pt`, `dataset_cache_e12_w300_s300_lag1_20_tail_ps.npz`, `requirements.lock.txt`.
- cache 상태: 로컬 cache 원본 경로는 `model/finetune/cache/safesignal_e1234_finetune7.npz` 및 `.summary.json`. 생성 결과는 `files_seen=1440`, `files_used=1439`, `windows=3581`, `skipped={"empty_windows": 1}`. 제외된 파일은 `E4_S02_A_FALL_SIT_F_T005.csv`이며, CSV 품질은 통과하지만 rows=157로 window_size=300 미만이라 학습 window를 만들지 못함. 학교에서 재수집 가능하면 해당 파일만 교체 후 clean-csv/build-cache 재실행, 불가능하면 현재 cache로 학습 진행 가능.
- USB/이동 준비: 코드 이관용 zip은 `C:\Project\LastProject\wifi-csi-fall-detection\.local\transfer\safesignal_code_transfer_20260601_020407.zip`에 생성 완료(약 53.5MB). USB에는 이 zip을 복사했고, 대용량 데이터는 Drive 기준으로 복구하는 전략. 다른 PC에서는 프로젝트 압축 해제 → 새 `.venv` 생성 → `pip install -r requirements.lock.txt` → cache shape 및 CUDA 확인 후 학습 실행.
- 내일 다른 PC 체크리스트: ① `cleaned.zip` 압축 해제 경로가 `data/cleaned/*.csv` 형태인지 확인, ② artifact 5개가 지정 경로에 있는지 확인, ③ `python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"`, ④ `python -c "import numpy as np; d=np.load('model/finetune/cache/safesignal_e1234_finetune7.npz', allow_pickle=True); print(d['X'].shape, d['y'].shape)"` 기대값 `(3581, 1, 28, 20) (3581,)`, ⑤ 가능하면 `E4_S02_A_FALL_SIT_F_T005.csv` 재수집 후 cache를 재생성.

### 2026-05-22 — 대시보드 수집 품질 박스 항목별 색상 + 종합 판정 배지 ([D-025] 후속, report-only)

- 범위: `server/dashboard/templates/index.html` 단일 파일. 커밋 `5657f57` (codex/finetune-train-skeleton, +90/-7). report-only 유지 — 저장 차단/RECOLLECT 미연결, `PAIR_TOLERANCE_US` 등 수집 로직 무변경.
- 항목별 색상: `qualColor(us, {warn,bad})` 헬퍼 추가(us→ms, warn/bad threshold별 `#e6edf3`/`#d29922`/`#ff4d4d`, null→`#8b949e`). pair_rate(85/95Hz), capture_ratio(0.90~1.10/0.95~1.05), pair_dt p50·p95·p99·max, ts_gap p95·max 각 항목 개별 색상.
- 종합 판정 배지(`#qVerdict`): 손실+페어링 종합. 🔴 손실≥20%|p50>10ms|gap_p95>30ms → 폐기·재수집, 🟡 손실≥5%|p50>5ms|gap_p95>15ms → 검토 후 저장, 🟢 그 외 저장 권장. 판정 사유 함께 표시, 수집 로그에도 1줄 기록. 지표별 단일 버킷 분류 + loss/p50/gap null·NaN 방어.
- 판정 기준 주의: 오프라인 `check_csv_quality.py`는 RECOLLECT를 손실률만으로 판정(pair_dt/gap report-only)하나, 대시보드 배지는 사용자 선택으로 손실+페어링 종합(엄격) 기준 채택. pair_dt/gap을 판정에 반영하는 점이 오프라인 도구와 다름. 기존 자체수집 55개(110컬럼)는 p50~7ms·gap_p95~20ms로 대부분 🟡(검토 후 저장)로 분류됨 — 🔴/🟢 거의 없음. 변별력 부족 시 노랑 경계(p50 5→7, gap_p95 15→20) 상향 검토 여지.

### 2026-05-22 — pair_dt/수집 품질 기록·리포트 구현 + 실시간 카운터 정합 ([D-025] 후속)

- 범위: D-025 후속 중 "pair delay 사후 검증 정보 기록 + report-only 리포트"를 구현. 코드 커밋 `48ff88e` (codex/finetune-train-skeleton, 10 files). Codex 검토 후속 정리 4건 포함.
- CSV 스키마: `collect/recorder.py` CSV_COLUMNS 107→110, 맨 끝에 `timestamp_rx1_us`/`timestamp_rx2_us`/`pair_dt_us` append. `timestamp_us`는 호환성 위해 Rx1 timestamp 의미 그대로 유지. `pair_dt_us = abs(rx1.ts - rx2.ts)`. 서버/CLI 모두 `SessionRecorder.add_pair` 단일 경로 경유 → 양쪽 110컬럼 저장.
- 품질 helper: `collect/quality.py` 신규. `summarize_session(buf)` → pair_count/duration/pair_rate/capture_ratio/loss + pair_dt p50·p95·p99·max + ts_gap p95·max. row 0/1, legacy 107컬럼(=pair_dt 없음) 안전(None). loss는 `SessionRecorder.calculate_loss_rate` 재사용으로 기준 일치.
- 표시(report-only): CLI(`collect_main.py`)는 저장 질문 직전 출력. 서버(`collect_manager.py`)는 `collect_finished` payload에 품질 필드 추가, 대시보드(`index.html`)는 저장/폐기 버튼 위 품질 박스 + null-safe 렌더(`N/A`). `check_csv_quality.py`는 `EXPECTED_COLS` 컬럼수 검증을 필수 컬럼명 검증으로 전환(107/110 모두 OK) + pair_dt/gap 통계 출력. MIN_ROWS 주석 100Hz 정정.
- 실시간 카운터 정합: `server/main.py::on_paired`에서 `collect_manager.is_recording`을 `is_collecting` 로컬로 1회만 읽어 수집/추론 분기 기준 고정, `add_pair`를 `update_pair`보다 먼저 호출하여 `collect_pair_count` emit 값이 현재 페어 수와 일치(기존 1개 지연 해소). 저장 CSV pair 수/`pair_update`/`collect_pair_count` 정합.
- 문서: `loader.py`/`test_safesignal.py` docstring을 107/110 호환 설명으로 최신화(로직 무변경, 이름 기반 선택이라 추가 컬럼 무시).
- 비범위 준수: `PAIR_TOLERANCE_US` 미변경, pair_dt/gap은 저장 차단/RECOLLECT 미연결, `SIDE_FALL_MARKERS` 미수정, 펌웨어 IP/포트 미변경, loader 동작 무변경.
- 검증(`.codex-test-venv` python): `collect/_selfcheck.py` ALL_OK(110컬럼·pair delay·quality summary 테스트 포함). `py_compile` 10파일 OK. `test_safesignal.py` ALL_OK. 카운터 스모크 — `add_pair` 동기 증가로 emit 값 [1..5] 1개 지연 없음 + on_paired 정적 순서/단일 is_recording 확인. 107/110 CSV loader 양쪽 (n,104), `check_csv_quality` 혼재 폴더 둘 다 status OK(107=pair_dt N/A, 110=p50/95/99/max).

### 2026-05-22 — collect 자체수집 기준 240세션/side fall 제외 코드 정렬 및 전수 리뷰

- 범위: D-023/D-028(side fall 3종 제외, env-subject 조합당 240세션) 결정을 collect/server/문서 코드에 반영하고 전체 영향 리뷰. 코드 커밋 `4adfbff` (codex/finetune-train-skeleton, 7 files).
- 변경: `collect/labels.py`(270→240, 낙상 9종→6종, `FALL_SIT_S/STD_S/WALK_S` 제거), `collect/_selfcheck.py`(240·6종·side fall 미포함 검증), `collect/collect_main.py`(docstring), `server/collect_manager.py`(`start_session`에 `activity_code not in ACTIVITY_INFO` 방어 가드 추가), `debug/data_collect/check_csv_quality.py`(MIN_ROWS 죽은 `FALL_*_S` 키 제거), `model/CLAUDE.md`·`data/README.txt`(240세션 설명 일치화).
- 안전장치 유지: `model/finetune/train.py`의 `SIDE_FALL_MARKERS`는 기존 side fall CSV를 fine-tuning train/val/test에서 제외하는 안전장치이므로 미수정. `keep` 마스크가 X/y/subjects/envs/filenames 전부에 적용됨을 재확인.
- 리뷰 결과: 기능 버그 없음. collect CLI(`ACTIVITY_ORDER` 동적 순회)·서버 라우트(`/collect/labels`,`/collect/counts` 동적)·대시보드 UI(`renderActivityTable`가 `/collect/labels` 응답 기반 렌더, 270/9종/side fall 하드코딩 없음) 모두 라벨 단일 소스를 따름. 제거된 `FALL_*_S` 요청은 서버 가드가 `{ok:False}`로 차단(KeyError 없음). 저장소 전역에 잔존 270/9종 stale 참조 없음(주석·train.py 안전장치만).
- 검증: `python -m collect._selfcheck` → ALL_OK(240 sessions, side fall excluded). `total_target_sessions()==240`, fall 6종, 비낙상 6종 target=30 확인. `CollectManager().start_session('FALL_SIT_S',...)` → `{ok:False, error:'알 수 없는 activity_code'}`, counts 키 12개. `check_csv_quality.py` import OK(MIN_ROWS 12키, `_S` 없음). `app.py` AST 파싱 OK(flask 미설치로 런타임 라우트 구동은 미수행).

### 2026-05-20 — combined training fine-tuning train.py 골격 구현 및 검토

- 검토/구현 범위: `model/finetune/train.py` 신규 골격 작성 및 Claude Code 후속 수정 결과를 Codex가 재검토. 현재 파일은 `codex/finetune-train-skeleton` 브랜치에 커밋 및 push 완료됨.
- 데이터/학습 정책 반영: Alsaify + SafeSignal combined training, 7-class(`fall/walking/sit_stand/lying/standing/running/picking`), Alsaify 6-class `picking` label 5 → fine-tuning index 6 remap, SafeSignal-only `running` index 5, side-fall(`FALL_*_S`) filename 기반 학습 제외 로그.
- 모델 정책 반영: pretrained 6-class head → 7-class head migration 구현(`new[0:5]=old[0:5]`, `new[5]` re-init, `new[6]=old[5]`), combined training 기준 full unfreeze + 5 epoch backbone warmup 구조 유지. D-027에 따라 full unfreeze + 5 epoch warmup이 최종 정책으로 정렬 완료.
- Sampler 정책 반영: `WeightedRandomSampler`에 source ratio와 SafeSignal hard non-fall class weight 적용. 후속 수정으로 class weight가 source mass를 침범하지 않도록 source별 raw weight 합 정규화 적용 완료(`source_ratio=0.60`이면 SafeSignal weight 합 0.60, Alsaify 0.40 유지).
- 증강 leakage 방지: SafeSignal 공식 cache는 raw-only 전제로 두고, `is_augmented=True`, non-empty `augment_type`, filename 증강 marker(`_AUG`, `AUGMENT`, `JITTER`, `SCALING`, `TIMEWARP`, `NOISE`)가 감지되면 즉시 에러. 증강은 split 이후 `TrainAugmentDataset`에서 train subset에만 적용하는 구조로 고정하되 실제 증강 구현은 TODO. SafeSignal 7,200은 원본 수집 세션 수가 아니라 원본 1,440세션 기준 train 증강 5배 적용 시 가능한 최대 학습 샘플 규모로 표기한다.
- Validation/Test 구조: SafeSignal primary validation, Alsaify+SafeSignal auxiliary validation, SafeSignal sealed 3-fold cross-subject test 구조 반영. 현재 단일 fold runner이며 3-fold pooled threshold 및 mean pass/fail 집계 드라이버는 후속 작업.
- Checkpoint/threshold 정책: `last.pt`, `best_val_loss.pt`, `best_operating.pt` 저장, threshold sweep 0.30~0.70(step 0.05), sealed test 전 `best_operating.pt` reload. checkpoint args 내 `Path`는 PyTorch 2.6+ `weights_only=True` reload 문제를 피하도록 string 직렬화.
- 검증 결과(Claude Code 실행 보고 + Codex 확인): `python -m py_compile model/finetune/train.py` 통과. synthetic sampler mass test, Alsaify remap, head migration, raw-only/augmented cache guard, dummy dry-run 1 epoch CPU 검증 통과 보고. Codex도 py_compile 및 핵심 구현 라인 재확인 완료.
- 남은 TODO: SafeSignal CSV→raw window cache builder, Alsaify pretrained 전처리 경로 기반 fine-tuning cache builder, `TrainAugmentDataset` 실제 증강 연결, 최종 eval/report 확장, 3-fold pooled global threshold selector 및 mean pass/fail 집계, full unfreeze + warmup 정책은 D-027로 확정. GRU warmup 포함 여부는 코드 설명/optimizer group 정리 시 확인.

### 2026-05-19 — Pi4 프로토콜/실시간 추론 버퍼 검증 및 SMS 실기 확인

- 검증 주체: Claude Code 실행 결과를 사용자 공유, Codex가 STATE에 반영.
- 의존성: `.venv` 기준 주요 패키지 import 검증에서 `solapi`만 미설치였으나, 이후 사용자가 `solapi` 설치 및 import 확인 완료.
- 문법 검증: 서버/추론/WebSocket/protocol/test/collect 관련 Python 파일 21개 `py_compile` 통과.
- self-check: `python server/inference/_selfcheck.py` → `ALL_OK` (6/6), `python collect/_selfcheck.py` → `ALL_OK` (8/8).
- Pi4 payload 검증: `server/protocol/pi4_messages.py`의 `build_fall_alert_payload()`가 기존 포맷 `{event:"fall_detected", label:"fall", confidence, seq_num, timestamp_us}`를 그대로 생성하며 JSON 직렬화 정상.
- 실시간 추론 버퍼 검증: 70Hz raw timestamp 입력에서 첫 trigger가 211 packets 지점에 발생(`<300`), `get_window()` shape `(300,104)`, `window_timestamp_us != 0`, 첫 raw timestamp 기준 윈도우 끝까지 정확히 3.000s로 확인. packet-count 300개 대기 문제가 해결됨.
- 낙상 라벨 매핑 검증: pretrained 6-class 기준 `fall_class_indices=(0,)`, fine-tuned 7-class 시뮬레이션 기준 `fall_class_indices=(0,1,2)`로 `FALL_LABELS` 기반 다중 낙상 라벨 판정 정상.
- 모델 로드 검증: 실제 `model/pretrained/checkpoints/best.pt`를 CPU에서 warning 없이 로드했고 dummy `predict()` 결과 softmax 합이 1.000000으로 확인됨.
- 서버 smoke/E2E 검증: `python server/main.py` background 실행 시 UDP 5005, WebSocket 8765, Flask 8080 bind OK, InferenceWorker spawn OK. Pi4 시뮬레이터 WebSocket 연결 후 `/trigger_fall` 호출로 `{"event":"fall_detected","label":"fall","confidence":0.0,"seq_num":0,"timestamp_us":...}` 수신 확인. `timestamp_us=0` 입력 시 서버 현재 Unix us fallback 정상(D-016).
- SMS 실기 검증: 사용자가 실제 SMS 발송 테스트를 수행했고 문제 없이 동작함을 확인.
- 남은 선택 사항: `ws_handler/rpi_connection.py`, `notification/sms.py`의 일부 `print()` 로그는 백그라운드 리다이렉트 시 buffering될 수 있어 logger 전환 시 가시성 개선 가능. 기능 결함은 아님.
- 현 상태 판단: 서버 낙상 알림 기본 흐름(추론 결과 처리 → Pi4 JSON → SMS)은 데모 기본 흐름 기준 검증 완료. 실제 Raspberry Pi 4 장치 수신/오디오/버튼 흐름은 별도 장치/브랜치에서 확인 필요.

### 2026-05-18 — Pi4 낙상 JSON 단일 소스화 및 일반 행동 저장 지점 분리

- 배경: 교수님 피드백에 따라 향후 `stand`/`walk`/`sit`/`lying` 등 일반 행동 패턴을 서버에 축적하고 1주일 단위 생활 패턴 감소 분석에 활용할 가능성을 검토. 단, Pi4 실시간 WebSocket 전송은 현재 낙상 알림 전용으로 유지하기로 정리.
- 결정된 흐름: 낙상(`is_fall=True`)은 기존처럼 Pi4로 JSON 알림 전송 + SMS/cooldown 적용. 낙상이 아닌 일반 행동 추론 결과는 Pi4로 실시간 전송하지 않고, 추후 서버 로컬 저장소에 기록하여 주간 분석에 사용.
- 코드 반영: `server/protocol/pi4_messages.py` 신규 추가. `PI4_EVENT_FALL_DETECTED="fall_detected"`, `PI4_LABEL_FALL="fall"`, `build_fall_alert_payload()`를 서버 측 Pi4 payload 단일 source of truth로 둠.
- 코드 반영: `server/ws_handler/rpi_connection.py`는 직접 payload dict를 만들지 않고 `build_fall_alert_payload()`를 사용하도록 변경. 기존 JSON 포맷 `{event,label,confidence,seq_num,timestamp_us}`는 유지.
- 코드 반영: `server/test/test_pi4_ws.py`는 `"fall_detected"`/`"fall"` 문자열 하드코딩 대신 `protocol.pi4_messages` 상수를 import해 판정.
- 코드 반영: `server/main.py`에 `handle_inference_result(result)`와 `record_activity_result(result)` placeholder 추가. 현재는 fall이면 기존 `on_fall_detected()` 흐름으로 보내고, non-fall은 추후 저장 지점으로 분리만 해둠. 저장 형식(JSONL/CSV/SQLite 등)은 미정.
- 문서 반영: `context/SHARED.md`에 Pi4 WebSocket은 현재 낙상 알림 전용이며 실제 payload 생성 기준은 `server/protocol/pi4_messages.py`라고 명시. 일반 행동 결과는 서버 저장 후보이며 1주일 단위 생활 패턴 분석에 활용 예정, 저장 형식은 미정으로 기록.
- 검증: `python -m py_compile server/main.py server/ws_handler/rpi_connection.py server/test/test_pi4_ws.py server/protocol/__init__.py server/protocol/pi4_messages.py` 통과. `git diff --check` 통과. Payload builder 출력이 기존 포맷과 동일함을 확인.
- 잔여 리스크: 현재 로컬 Python에는 `websockets` 패키지가 설치되어 있지 않아 실제 import/e2e 실행 검증은 미수행. `server/requirements.txt`에는 이미 `websockets`가 포함되어 있음. Pi4 실제 수신부는 별도 브랜치(`feature/rpi4-pipeline`)에서 동시 확인 필요.

### 2026-05-18 — ACF lag0 제외 및 Google Drive 업로드 구현 진행

- **ACF/SDP 변경 완료 및 GitHub main 반영:** `model/preprocessing/acf.py`가 더 이상 lag=0 상수열(`rho_0=1`)을 반환하지 않고, shape `(28,20)`은 유지한 채 lag=1..20을 반환하도록 변경. `model/pretrained/train.py` 캐시 파일명에 `_lag1_20` 포함하여 기존 lag0 캐시 재사용을 방지. 관련 디버그/테스트/inspector 라벨도 `lag (1~20)`으로 갱신. 검증: `py_compile`, `debug/preprocessing/check_acf.py`, `debug/preprocessing/check_sdp.py`, `augment_inspector.py` synthetic/실제 WALK CSV 실행 통과. 커밋 `49f92be [수정] ACF lag0 제외 전처리 적용` 을 `origin/main`에 push 완료.
- **데이터 재수집 필요 없음:** lag 정책 변경은 raw CSV 이후 전처리 단계 변경이므로 기존 자체수집 CSV는 재사용 가능. 단, 기존 전처리 캐시 및 기존 `best.pt`는 lag0 기준이므로 재사용 금지. 사전학습 캐시 재생성 및 사전학습 재실행 필요.
- **Google Drive 업로드 구현 진행:** `collect/drive_upload.py` 신규 추가, `collect/collect_main.py`에서 `SessionRecorder.save_session()` 성공 후 `upload_file_async(path)` 호출하도록 로컬 구현. rclone 기반이며 환경변수 `SAFESIGNAL_DRIVE_UPLOAD=1`, `SAFESIGNAL_DRIVE_REMOTE=gdrive:SafeSignal/data/raw` 설정 시 업로드. 기본값은 비활성화라 기존 수집 동작 유지. 업로드 실패해도 로컬 CSV는 유지되고 `data/upload_log.md`에 성공/실패 기록. `.gitignore`에 `data/upload_log.md` 추가. 검증: `py_compile` 통과, 업로드 비활성 기본 경로 확인, remote 미설정 실패 로그 확인.
- **주의:** Google Drive 업로드 변경은 아직 GitHub에 push하지 않은 로컬 작업트리 상태. rclone 설치 및 `rclone config`로 Google Drive remote 설정이 팀 환경에서 별도 필요.

### 2026-05-18 — tools/augment_inspector.py 추가 (증강 파라미터 검토용 시각화)

- **작업 범위:** SafeSignal 증강 4기법(jittering / scaling / time_warping / noise_scale)의 파라미터 강도를 단일 대표 윈도우 기준으로 사전 점검하는 inspector 스크립트. 학습용 전체 데이터셋 증강/저장 도구가 아니며, 실제 전체 증강은 추후 fine-tuning 파이프라인의 train split에서만 수행한다.
- **파일:** `tools/augment_inspector.py` (신규). 기존 `model/augment/augment.py` 및 `model/preprocessing/pipeline.py`는 import만 사용, 수정 없음.
- **재현성:** `np.random.SeedSequence(seed).spawn(4)`로 기법별 child seed 생성 → `jitter_rng / scaling_rng / warp_rng / noise_scale_rng` 독립 RNG 전달. 호출 순서 변경에도 안정적. 동일 seed 두 번 실행 시 3개 PNG 해시 완전 일치 확인(stats_summary.txt는 synthetic CSV 절대경로 한 줄만 다름).
- **CLI:** `python tools/augment_inspector.py <csv_or_dir> [--out OUT] [--seed 42]` / `python tools/augment_inspector.py --synthetic` (path 없이 실행 가능, 합성 CSV는 출력 디렉터리에 저장).
- **출력물 (tools/inspect_output/ 기본):**
  - `sdp_distribution.png` — 전체 윈도우 z-score SDP 값 히스토그램 + mean/std/min/max/p5/p95
  - `augmentation_heatmaps.png` — 2×3 subplot, 원본 + 4기법, 컬러바 범위는 원본 기준 고정
  - `augmentation_diff_heatmaps.png` — 2×2 subplot, `augmented - original` diff, 대칭 컬러바
  - `stats_summary.txt` — 입력 경로/seed/총 윈도우 수/resample metadata(`original_rate_hz`, `gap_count`, `max_gap_us`)/기법별 max·mean|Δ|·std 변화율(%)/jittering σ vs 원본 SDP std 비율 코멘트
- **운영 가드:**
  - matplotlib 백엔드 `Agg` 강제 (`matplotlib.use("Agg")`) — 헤드리스 환경 안전
  - `PROJECT_ROOT`를 `sys.path` prepend → 임의 cwd 에서도 `model.*` import 가능
  - 경로 미존재 / 디렉터리에 CSV 없음 / 윈도우 수 0 / 필수 패키지 미설치 → 명확한 에러 메시지 + 비정상 종료 코드
  - 합성 CSV는 RPCA degenerate 방어를 통과하도록 정현파 + 작은 노이즈 포함 (constant 입력 회피)
- **합성 데이터 sanity (seed=42):**
  - jittering: `max|Δ|≈0.20`, std 변화 +0.15% — σ=0.05가 원본 z-score std≈1 대비 5% 수준, 합리적 범위
  - scaling(0.8~1.2): `max|Δ|≈0.057`, std 변화 -1.30% — z-score 후 ±20% 배율은 형태 보존, 에너지만 조정
  - time_warping: `max|Δ|≈0.092`, std 변화 -0.03% — 매핑이 [min, max] 안에서 동작, 분포 안정
  - noise_scale: `max|Δ|≈0.72`, std 변화 -14.79% — 두 효과 결합으로 변형이 가장 큼
- **검증:** `python -m py_compile`, `--synthetic`, `data/raw` 실 CSV 3종 케이스 모두 OK.
- **주의:** 이 inspector의 적정성 코멘트는 빠른 sanity check 수준이다. 최종 파라미터 적정성은 train split 성능 + FAR 실측으로 확정해야 한다.
- **2026-05-18 리뷰 반영 (follow-up):**
  - 의존성 누락(numpy/matplotlib/pandas) ImportError 처리에서 `raise` 제거 → `sys.exit(1)`로 변경. 사용자용 CLI 도구라 traceback 노출 없이 안내 메시지만 stderr 출력하고 종료 코드 1로 종료.
  - `stats_summary.txt`의 scaling 적정성 판단을 고정 문구에서 실측 기반 분기로 교체. `original_range`, `max|Δ|/original_range`, `mean|Δ|`, `std_change(%)` 출력 + 4단계 분기(`range<=1e-8` / `ratio<0.10` / `0.10≤ratio≤0.35` / `ratio>0.35`).
  - 실 CSV `data/raw/E1_S01_A_STAND_T001.csv` 기준: `original_range=4.7759`, `max|Δ|/range=1.16%` → "scaling 영향이 약함" 분기. inspector의 z-score SDP에서 0.8~1.2 scaling은 다양성 기여가 제한적이며, 학습 단계에서 효과/대안 검토 필요.

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
- 2026-05-18 후속: `checkpoints_local_backup_20260514/` 폴더 통째 삭제(진규 5/4 로컬본, recall=0.737/f1=0.776 미달, 마커 명시 삭제 예정). lag0 baseline 백업은 `checkpoints_backup_2026-05-18/`(팀원 5/11 운영본, recall=0.919/f1=0.913)이 유지됨.

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
- UDP 수신 서버 구현 자체는 이후 `server/main.py::start_receivers()` 통합 기준으로 Implementation Status를 done으로 정정했다. 다만 ESP32 실기 패킷 수신 리허설은 별도 검증 항목으로 남는다.
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

### 2026-05-21 — HPO 및 패킷 동기화/보간 보완 검토

- HPO 방향은 Optuna TPE + PatientPruner로 확정. `run_training()` 반환 구조, HPO mode, objective wrapper는 다음 fine-tuning 구현 단계에서 추가 필요. HPO는 sealed test를 사용하지 않고 SafeSignal primary validation metrics만 사용한다.
- 패킷 동기화는 timestamp 기반 nearest-match를 유지하되, pair quality 검증을 위해 `pair_dt_us` 기록과 pair delay 분포(p95/p99) 산출이 필요하다.
- 오프라인/실시간 100Hz 선형보간 정책은 방향이 일관되지만, realtime buffer에서 `max_gap_ms`가 실제 skip/warn 조건으로 쓰이지 않는 차이가 있다. 큰 timestamp gap은 보간으로 메우기보다 품질 경고 또는 window 제외 후보로 다룬다.
- 공유기 사용은 핫스팟 대비 채널 고정과 재현성 개선에는 유효하지만 주변 AP와의 채널 공유/간섭은 제거하지 못한다. 수집 품질 평가는 평균 loss rate뿐 아니라 연속 gap, timestamp gap, pair delay를 함께 본다.

### 2026-05-21 — freeze 정책 정정

- D-023에 남아 있던 partial freeze 기본안은 최신 결정과 불일치한다. 현재 최종 정책은 combined training + full unfreeze + 5 epoch warmup이다.
- 다음 코드 작업에서는 `model/finetune/train.py`의 full unfreeze + warmup 구현을 유지하고, 문서/주석/CLI 설명만 이 결정과 일치하도록 정리한다.

### 2026-05-21 — STATE 충돌 문구 정정

- D-006의 Alsaify 구조 표현을 `90 subcarriers`에서 `30 subcarriers × 3 Rx antenna = 90 CSI feature columns`로 정정했다.
- D-018의 `Alsaify는 이미 100Hz` 표현을 기존 pretrained 전처리 경로에서 `320Hz → 100Hz downsample`을 적용한다는 표현으로 정정했다.
- D-023의 partial freeze 문구는 D-027로 대체됨을 명시했다. 최종 정책은 combined training + full unfreeze + 5 epoch warmup이다.
- 2026-05-20 fine-tuning Review Notes의 `train.py untracked` 표현을 `codex/finetune-train-skeleton` 브랜치 commit/push 완료로 정정했다.
- 같은 Review Notes의 `partial-freeze 정책 정렬 필요` 문구를 D-027 기준 정렬 완료로 정정했다.

### 2026-05-21 — fine-tuning 수집 규모와 HPO 기준 정리

- fine-tuning 최종 원본 수집 규모는 env-subject 조합 6개 × 240세션 = 1,440세션으로 확정했다. 4개 환경 중 E4에서 3명이 모두 수집하므로 총 조합은 `E1-S01`, `E2-S02`, `E3-S03`, `E4-S01`, `E4-S02`, `E4-S03`이다.
- 1,440세션은 side fall 제외 기준이며, 일정/환경 문제로 실제 수집 규모는 변동될 수 있다. D-022의 1,620세션은 side fall 포함 과거 전체 수집 기준으로 구분한다.
- HPO는 fine-tuning 최종 기준을 따른다. 공식 목표는 recall 0.85/FAR 0.15/F1 0.85, stretch는 recall 0.90/FAR 0.10이며, sealed test는 HPO에 사용하지 않는다.

### 2026-05-25 — no_motion baseline / fall energy 비교 / 수집 운영 정책 업데이트

- 코드 커밋 현황(`wifi-csi-fall-detection`, `codex/no-motion-baseline`):
  - `0bae9c4 [분석] no_motion calibration baseline 생성 추가`: `tools/safesignal_debug.py calibrate --env E?`로 환경별 no_motion baseline JSON 생성.
  - `eebaab0 [분석] no_motion baseline 요약 검증 명령 추가`: `baseline-summary --env E? --strict`로 baseline schema/핵심 값 검증.
  - `9a1a7e8 [분석] baseline 대비 SDP energy 비교 명령 추가`: `compare-energy`로 no_motion baseline p95/p99와 대상 activity/fall p1/p5 비교.
  - `305abeb [수정] 수집 서버 추론 비활성화 옵션 추가`: `SAFESIGNAL_DISABLE_INFERENCE=1`이면 `InferenceWorker`를 생성/start하지 않음. Git Bash `train` alias 등록 완료.
  - `6b9041e [수정] 낙상 수집 stage를 event 중심으로 조정`: fall 6종 stage를 4초 프로토콜로 변경.
- E2 no_motion baseline 정식 결과(`data/calibration/E2_no_motion_baseline.json`, RPCA max_iter=200):
  - source: 2 files / 596 windows, `rpca_max_iter=200`, strict 검증 통과.
  - `sdp_mean_abs`: p50=0.02783, p95=0.04062, p99=0.04352, min=0.02387, max=0.04444.
  - `sdp_std`: p50=0.03103, p95=0.06065, p99=0.06725.
  - `sparse_ratio`: p50=0.01831, p95=0.01888, p99=0.02057.
  - quality: original_rate_hz p50=96.36Hz, max_gap_ms p99=122.74ms, gap_count max=2, pair_dt_p99_us p50≈40.9ms.
- E1 fall vs E2 no_motion 정식 비교(`compare-energy --env E2 --target-env E1 --target-class fall`, RPCA max_iter=200):
  - 대상: 90 files / 299 windows.
  - `sdp_mean_abs`: no_motion p99=0.04352, fall p5=0.02152, target_ratio_le_nm_p99=0.84615.
  - `sdp_std`: no_motion p99=0.06725, fall p5=0.01839, target_ratio_le_nm_p99=0.85619.
  - `sparse_ratio`: no_motion p99=0.02057, fall p5=0.01713, target_ratio_le_nm_p99=0.28428.
  - 결론: z-score 전 SDP energy만으로 no_motion과 fall 전체 세션 window가 분리되지 않는다. Energy hard skip/gate는 금지. Energy는 metadata/log 또는 후속 보조 조건 후보로만 유지.
- 수집 운영 정책 업데이트:
  - 수집 서버는 Git Bash `train` alias로 실행한다. 동작: repo 이동 → `SAFESIGNAL_DISABLE_INFERENCE=1` → Drive 업로드 env 설정 → `rclone lsd gdrive:` 확인 → `python server/main.py`.
  - 서버 로그에 `[Inference] disabled by SAFESIGNAL_DISABLE_INFERENCE=1`가 떠야 하며, 수집 중 `[InferenceWorker] input_queue full` 로그가 나오면 안 된다.
  - 데이터 수집 품질 기준은 학습 데이터 관리용이다. 실시간 추론에서 pair_dt/ts_gap을 hard reject로 쓰지 않는다.
  - no_motion/fall energy 비교 결과에 따라 오탐 해결의 우선순위는 energy threshold가 아니라 `no_motion class 학습 + fall window 정제 + 품질 metadata`로 이동.
- fall 수집 stage 확정(2026-05-25):
  - `FALL_SIT_F/B`: 앉은 상태 대기 1초 → 낙상 2초 → 낙상 후 정지 1초.
  - `FALL_STD_F/B`: 선 상태 대기 1초 → 낙상 2초 → 낙상 후 정지 1초.
  - `FALL_WALK_F/B`: 1~2걸음 걷기 1초 → 낙상 2초 → 낙상 후 정지 1초.
  - fall 세션 수/target은 유지하고 duration만 5초에서 4초로 변경. 낙상 후 일어나는 동작은 포함하지 않는다.
- 네트워크/장비 운영 메모:
  - 휴대폰/인터넷 uplink 랜선은 공유기 WAN/Internet 포트에 연결한다. LAN 1~4 포트가 아니다.
  - Windows Wi-Fi IPv4가 바뀌면 RX1/RX2는 `set_server <IPv4> 5005`, TX는 `set_target <IPv4> 5005`로 재설정한다.
  - RX1/RX2 NTP 동기화는 모니터에서 확인 가능하나, 장기적으로는 별도 ESP32 status heartbeat 패킷을 추가해 대시보드에서 확인하는 것이 필요하다.

## Pending Items

- [ ] 일반 행동 추론 결과 저장 방식 결정 및 구현 (JSONL/CSV/SQLite 중 선택). Pi4 실시간 전송은 낙상 알림 전용으로 유지하고, non-fall 결과는 서버에 축적하여 1주일 단위 생활 패턴 감소 분석에 활용.

- [x] WIFI_PS_NONE 적용 후 WALK 세션 재수집 → 페어 카운트 안정성 검증 (2026-05-12 완료. 8초 세션 570~652 페어, burst→idle 패턴 소멸. 700~800 목표는 미달이고 ~70Hz 천장 발견 → [D-018]로 처리)
- [x] 3순위 fix(TX broadcast → unicast) 적용 여부 결정 (2026-05-12 적용 + 효과 확정. broadcast 3Hz → unicast ~70Hz)
- [ ] PS 비활성화 효과 검증 종료 후 RX1/RX2 디버그 카운터·stats_task 정리 (`csi_rx1_main.c`, `csi_rx2_main.c`) — 라우터 환경 재평가 마친 뒤 진행 권장 (D-018 후속)
- [x] self-collected 100Hz 리샘플 구현 (2026-05-13 완료. `model/preprocessing/resample.py` 신규 + loader/pipeline 확장. scipy `interp1d` 대신 `np.interp` 사용 — 결과 동일, 의존성 -1. ResampleResult metadata에 gap_count/max_gap_us/original_rate_hz 노출. Alsaify 경로 무변경.)
- [ ] portable router 확보 시 70Hz 천장 해소 가능성 재평가, 리샘플 필요성 재판단 ([D-017]/[D-018] 후속)
- [ ] fine-tuning 후속 구현: ~~SafeSignal CSV→raw window cache builder~~ (finetune7/global cache 및 D-020 raw SDP cache 생성 완료), Alsaify fine-tuning cache builder/정식 raw SDP cache 운용 검증, ~~TrainAugmentDataset 실제 증강 연결~~ (2026-06-01 완료 — b안 on-the-fly, `(seed,idx,epoch)` 결정론적, Alsaify pass-through, `verify_aug_gate.py` 검증. main `0b25a49`/`9e3064d`), 최종 eval/report 확장, 3-fold pooled global threshold selector 및 mean pass/fail 집계, full unfreeze + warmup 정책 반영 확인 (2026-05-20 골격 구현, 2026-06-01 main 반영)
- [x] SDP z-score 후속(D-020): per-lag는 train-split-stat이 아니라 per-window self-normalization으로 probe/ablation했다(axis=2, std_floor=1e-4, eps=1e-6, clip=±3). SafeSignal raw SDP cache와 Alsaify raw SDP cache를 확인한 뒤 트랙2 `PROVISIONAL mixed-normalization probe`와 트랙1 정식 ablation(Alsaify+SafeSignal 모두 동일 정규화)을 2026-06-03 완료했다. 정식 트랙1 결과는 A(global) R=0.738±0.038/FAR=0.144±0.031/F1=0.671±0.019, B(per-lag) R=0.708±0.055/FAR=0.159±0.043/F1=0.637±0.014로 B안 net negative. 최종 정규화는 global z-score 유지, per-lag 기각. step3-a(tail down-weight/제외)는 forward/tail 진단상 tail-only rescue 31~32세션으로 recall 파괴 위험이 커 후보에서 제외. ([D-020] 후속. A안 구현 자체는 main `14bfb12`로 2026-05-17 완료)
- [ ] event-centered windowing/labeling 탐색: D-020 per-lag가 기각되고 step2 후처리 sweep도 recall 0.85/FAR 0.15 동시 달성에 실패했으므로, post-fall 정적 단서 의존을 줄이고 transient supervision을 강화하는 window/label 설계를 다음 후보로 검토한다. 아직 결정·구현·실험된 항목은 아니며, 2026-06-11 발표 전 우선순위와 실험 범위 확정 필요.
- [x] beep 제거 후보 검증: 5.5초 fall 세션에서 stage beep frame `[0:50]`, `[150:200]`, `[400:450]` 제거 후 concat한 400-frame 시퀀스가 RPCA→ACF→SDP에서 접합부 artifact를 만들지 않는지 원본 continuous crop과 비교한다. (2026-06-05 완료 — [D-031] 선행 게이트 1. splice-smoothing probe로 splice edge가 모델 fall 출력의 주 원인이 아님을 확인(E2_S02 amp_jump 0.694→0.033인데 fall 0.924→0.918 생존) → **clean400_concat 유지 확정**. continuous_center 전환 안 함(WALK_B recall_gain +0.153, concat 0.644 vs center 0.492). 보조 `continuous_fall_post` recall 0.797은 FAR 불명이라 열린 탐색 항목으로만 보존. 산출물 `debug/modeling/diag_out/beep_concat_artifact/`.)
- [ ] 더미데이터 수령 후 preflight: frame count/duration 분포, 4초 생성 경로(beep 제거 후 변형인지 5.5초 압축인지), RPCA max_iter(30 vs 200), origin/session 매핑, train/test leakage 여부를 확인한다. 더미가 4초라는 이유만으로 1.5초 beep 제거를 추가 적용하지 않는다.
- [x] ACF lag=1..20 기준으로 Alsaify 사전학습 캐시 재생성 및 `best.pt` 재학습 (`_lag1_20` 캐시 사용) (2026-05-18 완료. `dataset_cache_e12_w300_s300_lag1_20_tail_ps.npz` 기반 30epoch 재학습 — best=epoch7: fall_recall=0.922 / fall_f1=0.902 / FAR=0.029 / acc=0.791, meets_all_targets=true. lag0 포함 버전(recall=0.919/f1=0.913/FAR=0.022/acc=0.810) 대비 recall +0.003, f1 -0.011, FAR +0.007, acc -0.019 — 거의 동등. D-011 stretch(recall≥0.90) 충족)
- [x] Google Drive 자동 업로드용 rclone 설치/설정 절차 확인 (2026-05-22: 수집 노트북 Bash 기준 `SAFESIGNAL_RCLONE_BIN`, `SAFESIGNAL_DRIVE_UPLOAD=1`, `SAFESIGNAL_DRIVE_REMOTE=gdrive:SafeSignal_Dataset`로 Drive 업로드 확인. 팀원 PC는 rclone remote 이름/실행 경로만 별도 확인)
- [x] E2E 실시간 추론 단계에서 `server/inference/buffer.py`를 timestamp-aware 100Hz resampling buffer로 전환 (코드상 `_resample_uniform()` + `np.interp`, Rx1 timestamp 기준 grid timestamp 보존 확인. 단 max_gap_ms hard skip/warn 정책은 아직 report/정책 후속)
- [ ] `wifi_event_group` 미사용 잔재 변수 정리 (`csi_rx1_main.c:86`, RX2 동일)
- [x] Alsaify 전체 사전학습 실행 (2026-05-14 팀원 결과 수령·로컬 적용 — best fall_recall=0.919 / F1=0.913 / FAR=0.022, meets_all_targets=true)
- [x] `preprocess_directory()`에 `tail_window` 옵션 추가 (윈도우-only 디버깅/분석 API 일관성 보완)
- [ ] Sliding window size 실험적 결정 (데이터 수집 후)
- [x] fine-tuning 자체수집 최종 기준 확정 (2026-05-21 완료. [D-028] 기준 — 4개 환경, 6개 env-subject 조합, 조합당 240세션, 총 1,440세션 목표. 일정/환경 문제로 변동 가능)
- [ ] 보호자 알림 상세 시나리오 확정 (동석 담당, SOLAPI vs KakaoTalk 포함)
- [ ] E2E 통합 테스트(실장치/시연 루프): UDP 수신/WebSocket/SMS/InferenceWorker 코드는 통합돼 있으나, 실제 Pi4 장치·오디오·버튼·SMS 포함 end-to-end 리허설 결과는 미기록
- [ ] ESP32 3대 배터리 런타임 정량화
- [ ] GitHub 브랜치 전략 확정
- [ ] 포터블 라우터 사용 가능 여부 확인
- [ ] RTX4060에서 window_to_model_input() 단일 윈도우 latency 실측 → stride 최종 확정 (D-014 후속)
- [x] server/dongseok + feature/pretrained-model → main 브랜치 통합
- [x] inference/ 모듈 구현 (InferenceWorker, 슬라이딩 윈도우 버퍼, 결과 큐)

- [ ] HPO 후속 구현: `run_training()` 결과 객체 반환, `--hpo` 모드, Optuna objective wrapper, PatientPruner 연결, trial별 checkpoint 최소화, best trial 1회 sealed test 실행 구조 추가 ([D-024]/[D-029] 후속, fine-tuning 최종 기준 사용)
- [ ] `--split within_subject` 데모 평가 모드의 위상 결정: 현재 코드/STATE상 demo-only 보조 경로이며 [D-019] cross-subject가 primary. 정식 보고 기준 편입 여부는 미결정 — 필요 시 D-XXX로 승격 논의 (2026-06-01 train.py 추가, main `0b25a49`)
- [x] pretrained6(running 제외 6-class, best.pt strict) zero-shot 베이스라인 실측 및 STATE 기록 (2026-06-03 Codex CPU 재실행): thr0.30 subject recall S01=0.146 / S02=0.076 / S03=0.246, FAR 0.028/0.021/0.026. 도메인 갭+소수 피험자 데이터 한계 해석 보강
- [ ] 본판 `within_6class_recall_aug`/GPU 학습 checkpoint로 step2 event-level 후처리 sweep 및 forward/tail 분리 진단 재검증 (2026-06-03 CPU 30epoch fallback 결과는 방법검증용으로 기록됨)
- [x] 패킷 동기화 품질 기록/리포트 (2026-05-22 완료, 코드 커밋 `48ff88e`): CSV 110컬럼 확장으로 `timestamp_rx1_us`/`timestamp_rx2_us`/`pair_dt_us` 기록(`timestamp_us`는 Rx1 의미 유지). `collect/quality.py` 공통 helper로 pair_dt p50/p95/p99/max + ts_gap p95/max 산출. CLI(저장 질문 직전)·서버 대시보드(저장/폐기 전 품질 박스)·`check_csv_quality.py`(필수 컬럼명 검증으로 107/110 호환 + pair_dt/gap 출력)에 report-only 표시. loader는 이름 기반 선택이라 107/110 모두 (n,104)로 무변경 동작.
- [ ] 패킷 동기화 품질 후속 ([D-025] 잔여): ① `PAIR_TOLERANCE_US`는 아직 미변경 — 실측 pair_dt/gap 분포(p95/p99) 확보 후 재조정 판단. ② pair_dt/gap threshold·WARN/RECOLLECT 기준은 분포 확보 후 결정(현재 report-only, 저장 차단 미연결). ③ offline/realtime `max_gap_ms` skip/warn 정책 정렬은 후속.
- [ ] 공유기/채널 고정 환경에서 pair rate, loss rate, max timestamp gap, pair delay 분포 재측정 후 핫스팟 대비 개선폭과 리샘플 필요성 재평가 ([D-026] 후속)
- [x] 정식 pipeline 기준 z-score 전 SDP energy 분석 스크립트 추가 (2026-05-25, 코드 커밋 별도. `debug/preprocessing/analyze_sdp_energy.py` — load_safesignal_csv→resample_to_100hz→sliding_windows→rpca_sparse→stacked_doppler_profile까지 정식 경로 재현, z-score 직전 energy(sdp_mean_abs/fro/std/max_abs/sparse_ratio/raw_std/raw_delta_mean) + pair_dt/gap metadata 산출, 활동별 p50/p95/p99/min/max + no_motion p95/p99 초과비율, CSV/JSON 저장. E2 NO_MOTION 2파일 빠른 실행 검증 완료. 탐색용(max_iter=30, 300frame 균등샘플링) 대비 정식 경로에서 sdp_mean_abs 절대값이 다르게 나옴(p50 ≈0.027 vs 탐색 0.019) — 절대 threshold는 fall p5 확보 후 재산정 필요)
- [ ] 실시간 추론에 z-score 전 SDP energy metadata 노출 (FALSE_POSITIVE_NOTES §7.10 후속, no_motion gate 사전작업): `window_to_model_input()`(model/preprocessing/pipeline.py)에 z-score 전 SDP를 옵션 반환하도록 확장 → `FallPredictor.predict()`(server/inference/predictor.py)가 sdp energy를 result dict에 포함 → `worker._inference_process`(server/inference/worker.py) result 통과 → `server/main.py` gate 판정에서 사용. buffer(server/inference/buffer.py)에는 gap/pair_dt quality metadata 보존 추가. fall 데이터 energy p5 확인 전 hard skip 금지(D-025/노트 §8.5).

---

- [ ] event-centered windowing 구현 게이트: **(1) beep 제거 concat artifact sanity — 통과(2026-06-05, [D-031] 선행 게이트 1: clean400_concat 유지 확정. splice-smoothing으로 splice artifact 우려 해소, WALK_B recall_gain +0.153, both=25/concat_only=13/center_only=4/neither=17. read-only 진단, 코드 무수정).** 잔여 — **게이트 2 (detector 기준 확정 2026-06-05, final onset_manifest pending):** (2) onset detector probe로 base 기준 확정(nominal[190:350]/k3/s5/smooth5, broad diagnostic-only, soft set A) — onset_probe_manifest + **manifest_v1_auto_reviewed(clean subset: auto_reviewed 214 usable/pending_manual 135/excluded 11, auto_exclude_candidate 0) 생성됨**, **final onset_manifest는 pending_manual 135(WALK 68) 진규 검수 후 v2에서 확정 pending**, (3) `needs_review` onset 수동 해결 또는 split 전체 제외 — **priority_review_queue 115(train/val 98, WALK 62)+plots_priority 98 준비됨, 진규 검수 진행(pending)**, (4) soft threshold set A(10/90) train/val 확정 — overlap threshold·shift 폭·정렬점(~131 provisional, final manifest 후 재계산)은 항목 4·5 정책 후 pending. WALK baseline contamination 미해결(수동검수 수용) · **게이트 3:** (5) event-centered cache builder 및 train/eval runner에서 baseline-axis와 selected-by-val 결과를 분리 기록. 기존 D-004(300-frame), D-013(104dim), D-018(100Hz resample), D-020(global z-score), D-030(pretrained6 demo primary)와 정합 유지.

## Milestones

| Week | 날짜 | 목표 |
|------|------|------|
| W3 | 2026-05-14 | 전처리 파이프라인 완성, UDP 안정화, 1차 데이터 수집 |
| W4 | 2026-05-21 | 2차 수집 완료 (목표 210세션), fine-tuning MVG 1차 |
| W5 | 2026-05-28 | E2E 통합, 2환경 검증 |
| W6 | 2026-06-04 | Demo |
| W7 | 2026-06-11 | 최종 발표 |












### 2026-05-26 — E2 NO_MOTION 정렬 후 품질 재평가 및 재수집 예정

- 발견: E2 `NO_MOTION` 10개 파일을 timestamp 정렬본(`data/cleaned`) 기준으로 재평가한 결과, timestamp reversal은 0으로 해소됐으나 6개 파일이 재수집 후보로 남음.
- 원인 분석:
  - `T005`, `T008`: 정렬 후에도 `gap max >= 150ms` 유지 (`T005` 약 214.3ms, `T008` 약 189.1ms) → 실제 수집 공백 가능성.
  - `T001`, `T003`, `T009`, `T010`: `pair_dt p95 >= 25ms` 근접/초과 (`25.3~26.1ms`) → Rx1/Rx2 동시성 품질 미흡.
  - `T002`, `T004`, `T006`, `T007`: 현 기준에서는 유지 가능.
- 조치 계획: `NO_MOTION`은 baseline/calibration 용도라 일반 행동보다 엄격히 관리한다. 수집 전 기존 E2 `NO_MOTION` 파일은 삭제 또는 격리 후 재수집한다. 가능하면 `PAIR_TOLERANCE_US=25ms` 적용 후 E2 `NO_MOTION` 10개 전체를 다시 수집한다.
- 주의: 원본 삭제 전 필요한 경우 `data/raw` 원본과 `data/cleaned` 정렬본의 백업/격리 위치를 명확히 한다.

### 2026-05-26 — E4 수집 완료 후 모델/threshold 진행 정책

- E4 본수집 완료 기준: `E4_S01`, `E4_S02`, `E4_S03` 모두 목표 trial을 채우는 것을 기준으로 한다. `NO_MOTION`은 전체 행동 수집 완료 후 마지막에 수행한다.
- 수집 중 품질 경고 처리: 수집 중에는 pair/gap 경고를 즉시 폐기 기준으로 쓰지 않고 1차 수집 완료를 우선한다. 단, row 수가 크게 부족하거나 capture/pair rate가 지속적으로 낮거나 장치 위치/안테나/동작 수행이 명확히 잘못된 경우는 현장에서 즉시 재수집한다.
- 재수집 판단: 수집 블록 완료 후 `clean-csv`로 timestamp 정렬본을 만든 뒤, 정렬본 기준 `loss rate`, `absolute gap p95/max`, `pair_dt p95`로 재수집 후보를 확정한다. `timestamp reversal`은 저장 순서 artifact이므로 재수집 기준에서 제외한다.
- E2 처리: E2는 현재 제외하지 않는다. E2 현장에 다시 가서 테스트를 진행한 뒤, 기존 E2 데이터 유지/부분 재수집/전체 재수집 계획을 결정한다.
- NO_MOTION 정책: `NO_MOTION`은 class로 유지한다. 추론 hard-gate로 바로 사용하지 않고, 우선 데이터 품질/분포 분석 및 no_motion class 학습에 활용한다. hard-gate 도입 여부는 fall 데이터 패턴 분석 이후 별도 논의한다.
- 낙상 판단 정책: 낙상 탐지 기준은 현재 수집된 비낙상/부분 낙상 그래프만으로 확정하지 않는다. 본 낙상 데이터 수집 후 SDP/RPCA/rolling std 등에서 `낙상 전 움직임 → 낙상 순간 spike → 낙상 후 정지` 패턴이 실제로 분리되는지 분석한 뒤 결정한다.
- threshold 정책:
  - 공식 성능 threshold는 subject-separated fold validation 기준으로 정한다.
  - demo operating threshold는 공식 threshold를 기본값으로 하되, 실제 시연 후보 환경(E4/E1)에서 sanity check 후 소폭 보정할 수 있다.
  - 현재 raw 수집 데이터가 적으므로 E4/E1 데이터를 threshold 전용 validation/test로 크게 분리하지 않는다. 가능한 raw 데이터는 학습/평가 후보로 보존하고, 증강은 train split에만 적용한다.
  - 시연 후보 환경에서 시간이 남으면 낙상 동작을 각 유형별로 추가 10회 수집하는 방안을 검토한다. 이 추가 수집분은 모델 재학습 또는 demo threshold sanity check에 활용할 수 있다.
- 다음 액션: E4 본수집 완료 후 NO_MOTION 수집 전 시간 여유를 확인하고, 추가 낙상 수집 여부를 결정한다.













