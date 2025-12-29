# Prometheus 시간 동기화 문제 해결

## 환경

| 항목 | 내용 |
|------|------|
| OS | Debian Ubuntu (VM 환경) |
| Prometheus | systemd service로 배포 (Docker 아님) |
| 네트워크 | 폐쇄망 (내부 NTP 서버 없음) |

---

## 문제 발생

### 증상

Prometheus UI에서 다음 경고 메시지 발생:

```
Warning: Error fetching server time: Detected 73 seconds time difference 
between your browser and the server. Prometheus relies on accurate time 
and time drift might cause unexpected query results.
```

### 초기 진단 시도

```bash
# 시간 동기화 상태 확인 시도
timedatectl show-timesync --all
```

**결과:** `Failed to parse bus message: No route to host`

→ 폐쇄망 환경이라 외부 NTP 서버 연결 실패

---

## 원인 분석

### 1. 폐쇄망 환경의 한계

- 기본 NTP 설정은 외부 서버(pool.ntp.org)에 연결 시도
- 폐쇄망에서는 외부 NTP 서버 접근 불가
- 내부 NTP 서버도 구축되어 있지 않음

### 2. 시간 동기화 실패

- VM 환경의 하드웨어 클럭은 정확도가 낮음
- NTP 동기화 없이 시간이 점점 틀어짐
- 서버 시간과 실제 시간 간 약 1분 이상 차이 발생

### 3. Prometheus의 시간 의존성

- Prometheus는 시계열 데이터베이스
- 메트릭 수집 시 타임스탬프 기반으로 저장
- 서버 시간이 틀어지면 데이터 정합성 문제 발생

---

## NTP 개념 정리

### NTP란?

NTP(Network Time Protocol)는 네트워크로 연결된 컴퓨터들의 시간을 동기화하기 위한 프로토콜이다. HTTP나 HTTPS처럼 "프로토콜"이며, 이를 구현한 소프트웨어가 여러 개 있다.

### 프로토콜 vs 구현체

| 프로토콜 | 구현체 (소프트웨어) |
|----------|---------------------|
| HTTP/HTTPS | nginx, apache, caddy |
| NTP | ntpd, chrony, systemd-timesyncd |
| DNS | bind, dnsmasq |

### 계층 구조 (Stratum)

```
Stratum 0  →  원자시계, GPS 시계 (물리적 장치)
    ↓
Stratum 1  →  Stratum 0에 직접 연결된 서버 (Primary Time Server)
    ↓
Stratum 2  →  공용 NTP 서버 (pool.ntp.org 등)
    ↓
Stratum 3+ →  하위 계층 서버들
```

### ntpd vs chrony

| 구분 | ntpd | chrony |
|------|------|--------|
| 특징 | 전통적, 안정적 | 최신, 빠른 동기화 |
| 장점 | 레거시 호환 | VM/컨테이너 환경에 적합 |
| 권장 환경 | 안정적인 네트워크 | 불안정한 환경, VM |

**VM 환경에서는 chrony가 더 적합하다.**

---

## 해결 방법

### 1단계: 기존 ntp 제거 및 chrony 설치

```bash
# ntp 제거
sudo systemctl stop ntp
sudo systemctl disable ntp
sudo apt remove ntp

# chrony 설치
sudo apt install chrony
```

### 2단계: chrony 설정 (Prometheus 서버)

폐쇄망에서는 외부 NTP 서버 대신 로컬 클럭을 기준으로 사용한다.

```bash
sudo nano /etc/chrony/chrony.conf
```

```conf
# 기존 외부 서버 설정 주석 처리
# pool ntp.ubuntu.com iburst
# pool 0.ubuntu.pool.ntp.org iburst

# 로컬 클럭을 시간 소스로 사용
local stratum 10

# 내부 네트워크에서 접근 허용
allow 192.168.93.0/24

# 또는 개별 IP 허용
# allow 192.168.93.11
# allow 192.168.93.12
```

```bash
sudo systemctl restart chronyd
sudo systemctl enable chronyd
```

### 3단계: 수동 시간 보정

`local stratum 10`은 현재 서버 시간을 기준으로 삼는 설정이므로, 이미 틀어진 시간을 먼저 보정해야 한다.

```bash
# chrony 일시 중지
sudo systemctl stop chronyd

# 현재 정확한 시간으로 수동 설정
sudo timedatectl set-time "2024-12-30 16:35:00"

# 하드웨어 클럭에도 반영
sudo hwclock --systohc

# chrony 재시작
sudo systemctl restart chronyd

# 확인
date
chronyc tracking
```

**또는 chrony 명령어로 강제 조정:**

```bash
sudo chronyc makestep
```

### 4단계: Prometheus 재시작

```bash
sudo systemctl restart prometheus
```

### 5단계: 확인

- `date` 명령어로 서버 시간과 브라우저 시간 비교
- Prometheus UI 새로고침하여 경고 메시지 사라졌는지 확인

---

## 타겟 서버 설정 (선택)

Prometheus가 모니터링하는 다른 서버들도 시간 동기화가 필요하다면:

### 아키텍처

```
┌─────────────────────────────────────────┐
│   Prometheus 서버 (192.168.93.10)       │
│   ├─ Prometheus                         │
│   └─ chrony (시간 기준 서버 역할)       │
│       - local stratum 10                │
│       - allow 192.168.93.0/24           │
└─────────────────────────────────────────┘
                    ▲
                    │ NTP (UDP 123)
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
┌─────────────┐           ┌─────────────┐
│  타겟 서버  │           │  타겟 서버  │
│  .93.11     │           │  .93.12     │
│  exporter   │           │  exporter   │
│  chrony     │           │  chrony     │
│  (클라이언트)│           │  (클라이언트)│
└─────────────┘           └─────────────┘
```

### 타겟 서버 chrony 설정

```bash
sudo apt install chrony
sudo nano /etc/chrony/chrony.conf
```

```conf
# 기존 pool/server 주석 처리
# pool ntp.ubuntu.com iburst

# Prometheus 서버를 NTP 서버로 지정
server 192.168.93.10 iburst
```

```bash
sudo systemctl restart chronyd
```

### 동기화 확인

```bash
# 소스 확인 (* 표시 = 동기화됨)
chronyc sources

# 상세 상태
chronyc tracking
```

---

## 요약

| 단계 | 작업 |
|------|------|
| 1 | chrony 설치 (VM 환경에 적합) |
| 2 | chrony.conf에 `local stratum 10` 설정 (폐쇄망용) |
| 3 | 수동으로 시간 보정 (`timedatectl set-time` 또는 `chronyc makestep`) |
| 4 | Prometheus 재시작 |
| 5 | UI에서 경고 메시지 해결 확인 |

---

## 참고 명령어

```bash
# 시간 확인
date
timedatectl

# chrony 상태
sudo systemctl status chronyd
chronyc sources
chronyc tracking

# 강제 시간 동기화
sudo chronyc makestep

# Prometheus 재시작
sudo systemctl restart prometheus
```