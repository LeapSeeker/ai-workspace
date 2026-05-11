# SafeSignal Project State

_Last updated: 2026-05-11 (WIFI_PS_NONE 적용 + 페어 변동성 진단) | Updated by: claude-code_

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
- **Status:** confirmed (PS 비활성화 적용·검증 대기), pending (unicast 전환 결정)

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
| firmware/csi_tx (PS 비활성화 적용) | done | main (워킹 트리) | 2026-05-11 |
| firmware/csi_rx1 (PS 비활성화 + 디버그 카운터) | done (디버그 빌드) | main (워킹 트리) | 2026-05-11 |
| firmware/csi_rx2 (PS 비활성화 + 디버그 카운터) | done (디버그 빌드) | main (워킹 트리) | 2026-05-11 |
| Alsaify 전체 사전학습 (E1+E2, RTX4060) | pending | - | - |
| UDP 수신 서버 | pending | - | - |
| WebSocket 서버-Pi4 통신 | pending | - | - |
| 자체 데이터 수집 파이프라인 | done | feature/pretrained-model | 2026-05-11 |
| fine-tuning | pending | - | - |
| Pi4 하드웨어 버튼 인터페이스 | pending | - | - |
| E2E 통합 테스트 | pending | - | - |
| inference/ 모듈 (InferenceWorker + FallPredictor + SlidingWindowBuffer) | done | main | 2026-05-11 |
| main 브랜치 통합 (server/dongseok + feature/pretrained-model) | done | main | 2026-05-11 |

---

## Review Notes

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

- [ ] WIFI_PS_NONE 적용 후 WALK 세션 재수집 → 페어 카운트 안정성 검증 (목표: 8초 세션 700~800 페어, burst→idle 패턴 소멸)
- [ ] 3순위 fix(TX broadcast → unicast) 적용 여부 결정 — 다른 AI 자문 결과 반영
- [ ] PS 비활성화 효과 검증 종료 후 RX1/RX2 디버그 카운터·stats_task 정리 (`csi_rx1_main.c`, `csi_rx2_main.c`)
- [ ] `wifi_event_group` 미사용 잔재 변수 정리 (`csi_rx1_main.c:86`, RX2 동일)
- [ ] Alsaify 전체 사전학습 실행 (RTX4060 서버 수령 후)
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





