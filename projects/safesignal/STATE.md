# SafeSignal Project State

_Last updated: 2026-05-09 | Updated by: claude-code_

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
| train.py | done | feature/pretrained-model | 2026-05-09 |
| metrics.py (FallMetrics) | done | feature/pretrained-model | 2026-05-09 |
| augment/augment.py | done | feature/pretrained-model | 2026-05-09 |
| model/r_pca.py | done | feature/pretrained-model | 2026-05-09 |
| firmware/csi_tx (ESP-IDF, 주화 작성) | ingested | feature/pretrained-model | 2026-05-09 |
| firmware/csi_rx1 (ESP-IDF, 주화 작성) | ingested | feature/pretrained-model | 2026-05-09 |
| firmware/csi_rx2 (ESP-IDF, 주화 작성) | ingested | feature/pretrained-model | 2026-05-09 |
| Alsaify 전체 사전학습 (E1+E2, RTX4060) | pending | - | - |
| UDP 수신 서버 | pending | - | - |
| WebSocket 서버-Pi4 통신 | pending | - | - |
| 자체 데이터 수집 파이프라인 | pending | - | - |
| fine-tuning | pending | - | - |
| Pi4 하드웨어 버튼 인터페이스 | pending | - | - |
| E2E 통합 테스트 | pending | - | - |

---

## Review Notes

### 2026-05-09 — sliding window tail loss 수정

- 문제: Alsaify 320Hz→100Hz 다운샘플링 시 trial당 ~400 패킷이 되는데 W=300, stride=300, drop_last=True 조합에서 윈도우 1개([0:300])만 생성되고 나머지 ~100 패킷이 버려짐.
- 수정: `sliding_windows()`에 `tail_window: bool = False` 파라미터 추가. True면 정방향 슬라이딩 후 잔여 패킷이 있을 때 `amplitude[-window_size:]` 슬라이스 윈도우 1개를 추가 (overlap 발생 가능, drop_last=False의 zero-padding보다 정보 손실 적음).
- 영향 범위: `model/preprocessing/window.py`, `model/preprocessing/pipeline.py` (preprocess_file, preprocess_file_full, preprocess_files_full, preprocess_directory_full, _worker_full args tuple), `model/pretrained/train.py` (build_cache에서 tail_window=True 사용, 캐시 파일명 `dataset_cache[_e<envs>]_tail.npz` 로 변경하여 기존 캐시와 분리), `model/preprocessing/test_pipeline.py` (4번 단계 직후 검증 추가).
- 적용 정책: 사전학습(train.py)만 tail_window=True. 실시간 추론 서버는 이번 수정 범위 아님. 기본값 False 유지로 기존 호출자 호환.

---

## Pending Items

- [ ] Alsaify 전체 사전학습 실행 (RTX4060 서버 수령 후)
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
