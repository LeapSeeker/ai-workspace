# SafeSignal Project State

_Last updated: 2026-05-11 | Updated by: claude-code_

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

---

## Implementation Status

| Module | Status | Branch | Last updated |
|--------|--------|--------|--------------|
| preprocessing/pipeline.py | done | feature/pretrained-model | 2026-05-09 |
| pretrained/model.py (CNNGRUAttention) | done | feature/pretrained-model | 2026-05-09 |
| train.py | done | feature/pretrained-model | 2026-05-10 |
| metrics.py (FallMetrics) | done | feature/pretrained-model | 2026-05-09 |
| augment/augment.py | done | feature/pretrained-model | 2026-05-09 |
| model/r_pca.py | done | feature/pretrained-model | 2026-05-09 |
| firmware/csi_tx (ESP-IDF, 주화 작성) | ingested | feature/pretrained-model | 2026-05-09 |
| firmware/csi_rx1 (ESP-IDF, 주화 작성) | ingested | feature/pretrained-model | 2026-05-09 |
| firmware/csi_rx2 (ESP-IDF, 주화 작성) | ingested | feature/pretrained-model | 2026-05-09 |
| Alsaify 전체 사전학습 (E1+E2, RTX4060) | pending | - | - |
| UDP 수신 서버 | pending | - | - |
| WebSocket 서버-Pi4 통신 | pending | - | - |
| 자체 데이터 수집 파이프라인 | done | feature/pretrained-model | 2026-05-11 |
| fine-tuning | pending | - | - |
| Pi4 하드웨어 버튼 인터페이스 | pending | - | - |
| E2E 통합 테스트 | pending | - | - |

---

## Review Notes

### 2026-05-11 — 자체 데이터 수집 파이프라인 (collect/) 구현

- 적용 브랜치: `feature/pretrained-model` (코드 커밋은 별도, 이 STATE 업데이트와 분리).
- 신규 파일: `collect/labels.py`, `collect/beep.py`, `collect/udp.py`, `collect/recorder.py`, `collect/collect_main.py`, `collect/_selfcheck.py`.
- 수집 목표: D-010(240세션)에서 270세션으로 사용자 갱신 — 낙상 9종(앉/서/걷×앞/뒤/옆) ×10 = 90, 비낙상 6종(SIT_STD/LIE/WALK/STAND/RUN/PICK) ×30 = 180. `labels.ACTIVITY_INFO` 단일 source of truth, `total_target_sessions()`로 270 검증.
- 패킷 파싱 기준: STATE.md D-007 + `firmware/csi_rx1/main/csi_rx1_main.c`, `firmware/csi_rx2/main/csi_rx2_main.c`의 `csi_packet_t` (`__attribute__((packed))`)에 정렬. 224B = `<BBbBIQ`(16B header) + `52f`(208B). `parse_packet()`은 size/magic(0xAB)/device_id(0x01|0x02) 모두 검증.
- 페어링: `PairingBuffer`가 Rx1/Rx2 timestamp 기준 50ms 이내 가장 가까운 패킷을 pair로 확정, 200ms 미완성 만료, 버퍼 200개 cap, `threading.Lock` 보호. cleanup은 host wall-clock이 아닌 가장 최근에 본 패킷의 `timestamp_us`를 reference로 삼아 ESP↔host SNTP skew 영향 제거. UDP receiver는 `start_udp_receiver()`로 1회 실행되는 daemon thread 2개(recv, cleanup).
- 녹화 시점: ENTER → `beep_ready()` → 3초 카운트다운 → **여기까지 CSV에 저장하지 않음** → `recorder.start_session()` → 첫 stage 직전 `beep_stage()` → stage/duration 대기 → 마지막 stage 종료 직후 `recorder.stop_session()` → `beep_end()`. 즉 ready/카운트다운 구간은 CSV에서 제외되고 실제 동작 구간만 저장.
- CSV 컬럼 109개 고정: `timestamp_us, seq_rx{1,2}, rssi_rx{1,2}, amp_rx1_0..51, amp_rx2_0..51`. 파일명: `data/raw/E{env}_S{subj:02d}_A_{code}_T{trial:03d}.csv`. trial은 동일 (env, subject, code) 조합 파일 수 + 1 자동 증가.
- 손실률: `(max_seq - min_seq + 1 - unique_seq_count) / (max_seq - min_seq + 1)`을 rx1/rx2 각각 계산해 max 반환. 5% 초과 시 경고 출력 후 저장 여부 사용자 선택.
- CLI 진입점: `python collect/collect_main.py [--port 5005]`. 초기 선택은 env → subject → activity 순(스펙은 env→activity→subject지만 활동 표 완료 수가 정확하려면 subject가 먼저 정해져야 함 — 결과 동일, UX만 조정). 변경 메뉴(1.env / 2.activity / 3.subject / 4.없음 / q.종료) 루프.
- 검증: `python -m py_compile collect/{labels,recorder,udp,beep,collect_main}.py` 통과. `python collect/_selfcheck.py` — 5개 항목(parse_packet, pairing(50ms 통과/60ms 차단/cleanup), loss_rate(0/10%/empty/single), trial 자동 증가 + CSV 컬럼 109개 순서, labels 270세션) 모두 ALL_OK. 손실률 계산은 numpy 없이 stdlib만 사용.
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

- [ ] Alsaify 전체 사전학습 실행 (RTX4060 서버 수령 후)
- [x] `preprocess_directory()`에 `tail_window` 옵션 추가 (윈도우-only 디버깅/분석 API 일관성 보완)
- [ ] Sliding window size 실험적 결정 (데이터 수집 후)
- [ ] 자체 수집 계획 최종 구조 확정 (W3 이전)
- [ ] 보호자 알림 상세 시나리오 확정 (동석 담당, SOLAPI vs KakaoTalk 포함)
- [ ] ESP32 3대 배터리 런타임 정량화
- [ ] GitHub 브랜치 전략 확정
- [ ] 포터블 라우터 사용 가능 여부 확인

---

## Milestones

| Week | 날짜 | 목표 |
|------|------|------|
| W3 | 2026-05-14 | 전처리 파이프라인 완성, UDP 안정화, 1차 데이터 수집 |
| W4 | 2026-05-21 | 2차 수집 완료 (목표 210세션), fine-tuning MVG 1차 |
| W5 | 2026-05-28 | E2E 통합, 2환경 검증 |
| W6 | 2026-06-04 | Demo |
| W7 | 2026-06-11 | 최종 발표 |









