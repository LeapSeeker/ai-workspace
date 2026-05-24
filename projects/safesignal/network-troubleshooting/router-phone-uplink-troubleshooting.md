# 공유기-휴대폰 업링크 문제 대응 메모

_Last updated: 2026-05-24 | Updated by: codex_

## 현재 수집 네트워크 구조

SafeSignal 수집 현장에서는 노트북이 휴대폰에 직접 이더넷 테더링을 받는 구조가 아니라, 아래 구조로 운용한다.

```text
휴대폰 인터넷
  ↓ 랜선 / USB-C 이더넷 젠더
공유기 coin
  ↓ Wi-Fi
노트북 + ESP32 TX/RX1/RX2
```

따라서 Windows의 `이더넷 2` 어댑터는 노트북 인터넷 경로와 무관할 수 있다. `이더넷 2`가 `169.254.x.x`이고 Default Gateway가 없어도, 노트북이 공유기에 Wi-Fi로 붙어 있다면 수집망 자체와 직접 관련이 없을 수 있다.

운영 중 확인해야 하는 핵심 어댑터는 `Wireless LAN adapter Wi-Fi`이다.

## 정상 기준

`ipconfig`에서 Wi-Fi 항목을 확인한다.

예시:

```text
Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 10.223.127.51
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.223.127.228
```

이 경우 노트북은 공유기 Wi-Fi에 정상 접속한 상태다. ESP32 설정은 Wi-Fi IPv4 주소를 기준으로 맞춘다.

```text
RX1: set_server 10.223.127.51 5005
RX2: set_server 10.223.127.51 5005
TX : set_target 10.223.127.51 5005
```

IPv4는 고정값으로 보면 안 된다. 공유기 DHCP, 재부팅, 업링크 상태, 접속 순서, 장소에 따라 바뀔 수 있다. 수집 시작 전 항상 Wi-Fi IPv4를 다시 확인한다.

## 인터넷 안 됨 현상의 해석

Wi-Fi가 보이고 연결되지만 인터넷이 안 되는 경우, 노트북과 공유기 사이보다 공유기 위쪽 구간을 먼저 의심한다.

```text
노트북/ESP32
  ↓ Wi-Fi coin
공유기
  ↓ 랜선 / 젠더
휴대폰
  ↓ 모바일망
인터넷
```

Wi-Fi 연결이 된다는 것은 대체로 `노트북 ↔ 공유기` 구간은 살아 있다는 뜻이다. 인터넷이 안 되면 주로 아래 구간 문제다.

- 공유기 ↔ 휴대폰 랜선/젠더 접촉 불량
- 휴대폰의 이더넷 공유/테더링 상태 불안정
- 휴대폰 모바일 데이터 또는 통신사 테더링 제한
- 공유기 WAN/DHCP/DNS 상태 이상
- 공유기 업링크 재협상 중 내부 DHCP 대역 변경

## 문제 상황 예시 분석

문제 발생 시 ipconfig 예시:

```text
Ethernet adapter 이더넷 2:
   Autoconfiguration IPv4 Address. . : 169.254.95.197
   Default Gateway . . . . . . . . . :

Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 192.168.0.6
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.0.1
```

해석:

- `이더넷 2`의 `169.254.x.x`는 이 구성에서는 무시 가능하다. 노트북이 휴대폰에 직접 연결된 구조가 아니기 때문이다.
- Wi-Fi의 `192.168.0.6 / gateway 192.168.0.1`은 노트북이 공유기에서 IP를 받았다는 의미다.
- 그러나 이 출력만으로 공유기가 외부 인터넷을 정상 제공한다고 판단할 수는 없다.
- 인터넷이 안 됐다면 `공유기 192.168.0.1 → 휴대폰/상위망 → 인터넷` 구간이 끊겼을 가능성이 높다.

특히 정상 수집 때 `10.223.127.x` 대역이었다가 문제 시 `192.168.0.x` 대역으로 바뀌면, 공유기/업링크 상태 변화로 DHCP 대역이 바뀌었을 가능성을 본다.

## 빠른 진단 명령

문제 발생 시 `ipconfig` 다음에 아래를 실행한다.

```powershell
ping <Wi-Fi Default Gateway>
ping 8.8.8.8
nslookup google.com
```

예시:

```powershell
ping 10.223.127.228
ping 8.8.8.8
nslookup google.com
```

판단 기준:

```text
gateway ping 실패
→ 노트북 ↔ 공유기 Wi-Fi 자체 문제

 gateway ping 성공, 8.8.8.8 실패
→ 공유기까지는 정상, 공유기 위쪽 휴대폰/WAN/인터넷 문제

8.8.8.8 성공, google.com 실패
→ DNS 문제

pair count는 올라가는데 Drive만 실패
→ 수집망은 정상, 인터넷/DNS/Google Drive 문제
```

## 수집 우선 대응 절차

인터넷이 불안정해도 노트북과 ESP32가 같은 Wi-Fi에 있으면 로컬 수집은 가능하다.

1. `ipconfig`에서 Wi-Fi IPv4 확인
2. RX1/RX2 `show_config`로 서버 IP 확인
3. TX `show_config`로 target IP 확인
4. 다르면 현재 Wi-Fi IPv4로 변경
5. 서버 실행 후 dashboard pair count 증가 확인
6. Drive 업로드가 실패하면 로컬 CSV 저장을 우선하고 나중에 수동 업로드

예시:

```text
Wi-Fi IPv4가 192.168.0.6이면:
RX1: set_server 192.168.0.6 5005
RX2: set_server 192.168.0.6 5005
TX : set_target 192.168.0.6 5005
```

```text
Wi-Fi IPv4가 10.223.127.51이면:
RX1: set_server 10.223.127.51 5005
RX2: set_server 10.223.127.51 5005
TX : set_target 10.223.127.51 5005
```

## 휴대폰/공유기 업링크 복구 절차

Wi-Fi는 연결되지만 인터넷이 안 되고, `gateway ping 성공 + 8.8.8.8 실패`라면 아래 순서로 복구한다.

1. 휴대폰 화면 잠금 해제
2. 휴대폰 모바일 데이터 상태 확인
3. 휴대폰과 공유기 사이 랜선/USB-C 이더넷 젠더 재결합
4. 휴대폰의 이더넷 공유/테더링 기능 OFF → ON
5. 공유기 WAN 상태 LED 확인
6. 공유기 전원 재인가 후 Wi-Fi 재접속
7. `ipconfig`로 Wi-Fi IPv4/Gateway 재확인
8. ESP32 서버 IP가 바뀐 Wi-Fi IPv4와 일치하는지 확인

## 증상별 대처 방법

### 1. Wi-Fi는 연결되지만 인터넷만 안 됨

증상:

```text
Wi-Fi SSID coin 연결됨
ipconfig의 Wi-Fi IPv4/Gateway 존재
웹/Drive/rclone 연결 실패
```

대처:

1. Wi-Fi Gateway까지 통신되는지 확인한다.

```powershell
ping <Wi-Fi Default Gateway>
```

2. Gateway ping이 성공하면 공유기 위쪽 문제로 보고 휴대폰/공유기 업링크를 재연결한다.

```text
휴대폰 화면 잠금 해제
모바일 데이터 ON 확인
휴대폰 ↔ 공유기 랜선/젠더 분리 후 재결합
휴대폰 이더넷 공유/테더링 OFF → ON
공유기 WAN LED 확인
```

3. 인터넷 경로를 확인한다.

```powershell
ping 8.8.8.8
```

4. `ping 8.8.8.8`은 되는데 웹/Drive만 안 되면 DNS 문제로 보고 DNS를 임시 변경한다.

Windows Wi-Fi 설정에서 DNS를 수동으로 바꾼다.

```text
기본 DNS: 8.8.8.8
보조 DNS: 1.1.1.1
```

또는 관리자 PowerShell에서 Wi-Fi DNS만 임시 지정한다.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses 8.8.8.8,1.1.1.1
```

성공 판정:

```powershell
nslookup google.com
```

복구 후 DNS를 자동으로 되돌리려면:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ResetServerAddresses
```

### 2. 수집 대시보드 pair count가 안 올라감

증상:

```text
서버는 실행됨
대시보드는 열림
pair count가 0이거나 증가하지 않음
```

대처:

1. 노트북 Wi-Fi IPv4를 확인한다.

```powershell
ipconfig
```

2. RX1/RX2/TX의 저장된 대상 IP를 확인한다.

```text
RX1: show_config
RX2: show_config
TX : show_config
```

3. 노트북 Wi-Fi IPv4와 다르면 ESP32 설정을 갱신한다.

```text
RX1: set_server <노트북 Wi-Fi IPv4> 5005
RX2: set_server <노트북 Wi-Fi IPv4> 5005
TX : set_target <노트북 Wi-Fi IPv4> 5005
```

4. 각 보드를 재시작한다.

```text
restart
```

5. 서버를 다시 열고 dashboard에서 pair count가 증가하는지 확인한다.

성공 판정:

```text
Dashboard pair count 증가
RX1/RX2 패킷 손실률 패널 값 갱신
최근 페어링 기록 표시
```

계속 실패하면 Windows 방화벽에서 UDP 5005 수신이 막혔을 가능성을 본다. 임시 확인용으로 개인/공용 네트워크 방화벽을 잠깐 끄고 pair count가 올라오는지 확인한다. 데모 운영 전에는 방화벽을 계속 끄기보다 Python/포트 5005 허용 규칙으로 정리한다.

### 3. IP 대역이 바뀜

증상:

```text
정상 때: 10.223.127.51
문제 때: 192.168.0.6
같은 SSID coin인데 IPv4 대역이 바뀜
```

해석:

공유기 DHCP 또는 업링크 상태가 바뀌면서 노트북이 다른 대역을 받은 것이다. 이 자체가 항상 오류는 아니지만, ESP32가 이전 IP로 계속 송신하면 수집이 끊긴다.

대처:

1. 인터넷이 필요한 상황이면 gateway/8.8.8.8/nslookup 순서로 외부 연결을 확인한다.
2. 수집이 우선이면 현재 Wi-Fi IPv4를 기준으로 ESP32 IP만 다시 맞춘다.
3. 대역이 자주 바뀌면 공유기 관리자 페이지에서 노트북 Wi-Fi MAC에 DHCP 예약을 건다.

예방책:

```text
수집 시작 전 체크리스트에 "ipconfig Wi-Fi IPv4 확인" 추가
ESP32 show_config 확인 후 서버 IP 불일치 시 즉시 갱신
가능하면 공유기 DHCP 예약으로 노트북 IP 고정
```

### 4. Drive 업로드만 실패함

증상:

```text
pair count 정상
CSV 로컬 저장 정상
rclone/Google Drive 업로드 실패
```

대처:

1. 수집을 멈추지 않는다. 로컬 CSV가 source of truth다.
2. 수집 종료 후 로컬 파일 존재를 확인한다.

```bash
ls -lh data/raw/E*_S*_A_*_T*.csv
```

3. 인터넷 복구 후 수동 업로드한다.

```bash
./.local/rclone/rclone-v1.74.1-windows-amd64/rclone.exe copyto \
  data/raw/<파일명>.csv \
  gdrive:SafeSignal_Dataset/E<환경>/S<대상자>/<파일명>.csv
```

4. 업로드 확인:

```bash
./.local/rclone/rclone-v1.74.1-windows-amd64/rclone.exe ls gdrive:SafeSignal_Dataset/E<환경>/S<대상자>
```

성공 판정:

```text
로컬 CSV 파일명과 Drive CSV 파일명이 동일
Drive 대상 폴더에 파일 크기가 표시됨
```

### 5. 5분 수집 종료 후 저장 버튼이 안 열림

증상:

```text
수집 시간이 끝난 것 같은데 저장 버튼 비활성화
폐기 버튼만 활성화
품질 박스가 안 뜸
```

대처:

1. 폐기를 누르지 않는다.
2. 서버 상태를 확인한다.

```powershell
Invoke-RestMethod http://127.0.0.1:8080/collect/status
```

3. `is_recording=false`이고 `pair_count`가 충분하면, 브라우저가 완료 이벤트를 놓친 상태일 수 있다. 저장 API를 직접 호출한다.

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8080/collect/stop -ContentType 'application/json' -Body '{"save":true}'
```

4. 로컬 파일이 생겼는지 확인한다.

```powershell
Get-ChildItem data\raw | Sort-Object LastWriteTime -Descending | Select-Object -First 5 Name,Length,LastWriteTime
```

성공 판정:

```text
E?_S??_A_<ACTIVITY>_T???.csv 생성
파일 크기가 0보다 큼
```

주의:

저장 API가 오래 걸려 타임아웃돼도 CSV가 실제로 저장됐을 수 있다. 타임아웃 직후에는 먼저 `data/raw`를 확인한다.

### 6. 공유기/휴대폰 업링크가 반복적으로 불안정함

현장 임시 대처:

```text
휴대폰 화면 켜둠
휴대폰 절전 모드 해제
모바일 데이터 고정, Wi-Fi 자동 전환 OFF
랜선/젠더를 움직이지 않게 테이프로 고정
공유기 전원 어댑터 안정화
수집 중에는 공유기/휴대폰 위치 변경 금지
```

운영 개선책:

```text
공유기 DHCP 예약으로 노트북 IP 고정
ESP32 서버 IP 변경 절차를 수집 체크리스트에 포함
Drive 업로드 실패를 수집 실패로 보지 않도록 로컬 저장 우선 운영
가능하면 휴대폰 업링크 대신 안정적인 유선 WAN 또는 별도 LTE 라우터 사용
```

## 현장용 1분 복구 루틴

인터넷/수집이 갑자기 이상하면 아래 순서만 따른다.

```text
1. ipconfig에서 Wi-Fi IPv4/Gateway 확인
2. gateway ping
3. dashboard pair count 확인
4. pair count 안 오르면 ESP32 show_config와 Wi-Fi IPv4 일치 확인
5. IP 불일치면 RX1/RX2 set_server, TX set_target 갱신 후 restart
6. pair count는 오르는데 Drive만 안 되면 로컬 저장 후 나중에 업로드
7. gateway는 되는데 8.8.8.8이 안 되면 휴대폰-공유기 업링크 재결합
8. 8.8.8.8은 되는데 google.com이 안 되면 DNS를 8.8.8.8/1.1.1.1로 임시 변경
```
## 운영 원칙

- `이더넷 2` 상태보다 `Wi-Fi IPv4 / Default Gateway`를 우선 확인한다.
- 수집 시작 전 ESP32의 저장된 서버 IP와 노트북 Wi-Fi IPv4가 같은지 반드시 확인한다.
- 인터넷 불안정과 수집 불안정을 분리해서 판단한다.
- 로컬 CSV 저장이 최우선이다. Drive 업로드 실패는 나중에 수동 복구 가능하다.
- 장기적으로는 공유기 DHCP 예약으로 노트북 Wi-Fi MAC에 고정 IP를 부여하는 것이 가장 안정적이다.

