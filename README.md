# 🕵️ DOTAS v2.0  
## Dark-web OSINT Threat Alert System

![Python](https://img.shields.io/badge/Python-3.9%2B-3776AB?style=flat&logo=python&logoColor=white)
![Network](https://img.shields.io/badge/Network-Tor%20Onion-7D4698?style=flat&logo=tor-browser&logoColor=white)
![Focus](https://img.shields.io/badge/Focus-Threat%20Intelligence-red?style=flat)
![Platform](https://img.shields.io/badge/Platform-Kali%20Linux-blue?style=flat&logo=linux)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

> **DOTAS는 다크웹(.onion) 및 OSINT 소스에서 의심스러운 IOC(Indicator of Compromise)를 자동 수집하고,  
> 이를 CSV 저장 + Telegram 실시간 경고로 전달하는 경량 위협 인텔리전스 자동화 시스템입니다.**

---

# 📌 주요 기능 (Features)

### ✔ 1. Dark Web Crawler (Tor 기반)
- `.onion` 주소 접근  
- Tor SOCKS5h 사용 → **DNS 요청까지 Tor 내부에서 처리 (완전 익명성)**  
- 웹 마켓/포럼 페이지 자동 수집

### ✔ 2. OSINT Collector
- deepdarkCTI · GitHub 레포지토리  
- 클리어웹(HTTPS) OSINT 자료 자동 수집

### ✔ 3. IOC 자동 분석
- 이메일 / 도메인 Regex 매칭  
- 키워드 기반 필터링  
- HTML → Plain text 정규화

### ✔ 4. 중복 알림 방지 (De-duplication)
- IOC는 `seen_indicators.txt`에 기록  
- 이미 알림 보낸 데이터는 재전송 없음

### ✔ 5. Telegram 실시간 알림
- Bot API 사용  
- SOC 환경처럼 **즉시 경고(Incident Alert)** 전송  
- CSV 기록 (`findings.csv`) 자동 업데이트

---

# ⚙ 시스템 아키텍처 (Architecture Diagram)

```text
            +------------------+
            |   DARK WEB       |
            |   (.onion)       |
            +------------------+
                      |
        (Tor SOCKS5h Proxy: 127.0.0.1:9050)
                      |
                      v
            +------------------+
            |   Collector      |
            | (fetch_url)      |
            +------------------+
                      |
            +--------------------------+
            |      Analyzer            |
            | - HTML -> Text clean     |
            | - Keyword filter         |
            | - Email/Domain extract   |
            +--------------------------+
                      |
        +-------------+--------------+
        |                            |
        v                            v
+-------------------+      +-----------------------+
|   CSV Logger      |      |  Telegram Alert Bot   |
|  findings.csv     |      |  send_telegram()      |
+-------------------+      +-----------------------+
                  |
                  v
         +-----------------------+
         |  seen_indicators.txt  |
         |  (Duplicate Filter)   |
         +-----------------------+
```

---

# 🛡 Threat Model (위협 모델)

DOTAS는 다음과 같은 보안 위협 탐지를 목표로 설계됨:

| 위협 유형 | 설명 |
|----------|------|
| **개인정보 유출** | 이메일, 계정, 도메인 유출 여부 탐지 |
| **기업 내부 정보 노출** | 회사명/직급/문서 키워드 기반 탐지 |
| **랜섬웨어 협박 페이지** | gang 사이트 게시물 자동 모니터링 |
| **가짜 채용·스피어피싱 페이지** | 키워드 기반 조기 탐지 |
| **자격 증명(credential) 유출** | password/admin 등의 흔적 식별 |

---

# 📁 디렉토리 구조 (Project Structure)

```text
DOTAS/
│── dotas_v2.py           # 메인 실행 파일
│── findings.csv          # 수집된 IOC 로그
│── seen_indicators.txt   # 중복 알림 방지 DB
│── README.md             # 설명 문서
└── requirements.txt      # 패키지 목록
```

---

# 🧩 Requirements (환경 요구사항)

```
Python 3.9+
Tor service (0.4.x 이상)
Kali Linux 또는 Linux 계열 OS
Stable Internet + SOCKS Proxy 가능 환경
```

필수 패키지:

```
requests
beautifulsoup4
pysocks
urllib3
python-telegram-bot
```

설치:

```bash
pip install -r requirements.txt
```

---

# 🔧 설치 및 실행 방법

### 1) Tor 설치 및 실행

```bash
sudo apt update
sudo apt install tor -y
sudo service tor start
```

### 2) 코드 설정

```python
TELEGRAM_TOKEN = "YOUR_BOT_TOKEN"
CHAT_ID = "YOUR_CHAT_ID"
```

### 3) 실행

```bash
python3 dotas_v2.py
```

---

# 📌 실행 로그 예시 (Example Output)

```
>> [DOTAS] Dark-web & OSINT Threat Monitor Started.

[Cycle] 2025-11-30 13:00
[*] Source: DuckDuckGo Onion
[+] Fetch 성공 (size=117030)
[탐지] domain 발견: privacy.com
[*] Telegram 전송 완료
------------------------------------
```

---

# 🩺 Troubleshooting (트러블슈팅)

### 1️⃣ `socks5`는 왜 안 되고 `socks5h`가 필요한가?
- `socks5` → DNS를 로컬이 직접 해석 → `.onion` 불가  
- **`socks5h` → DNS도 Tor 내부에서 처리 → 필수**

---

### 2️⃣ 비동기 루프가 멈춤 (async + requests 충돌)
`requests`는 synchronous blocking → `asyncio` 루프 멈춤  
✔ 해결:  
```python
await asyncio.to_thread(check_darkweb)
```
스레드로 오프로드하여 비동기 루프 보존.

---

### 3️⃣ 중복 알림 폭주 (Alert Fatigue)
✔ 해결: `seen_indicators.txt`에 IOC 저장  
✔ 이미 본 IOC는 절대 재전송 안 함

---

### 4️⃣ HTML 태그 때문에 키워드 탐지 실패
✔ 해결:  
```python
BeautifulSoup(html, "html.parser").get_text()
```
→ 완전한 plain text 환경에서 regex 적용

---

### 5️⃣ SSL 인증서 오류 (`CERTIFICATE_VERIFY_FAILED`)
✔ 해결:
```python
verify=False
urllib3.disable_warnings()
```

---

# 📄 License (MIT)

본 프로젝트는 MIT 라이센스로 배포됩니다.  
자유롭게 수정/확장/재사용 가능합니다.

---

# ⚠️ Legal Disclaimer

> **본 도구는 교육·연구·디지털 포렌식 학습용으로 제작되었습니다.**
>
> 허가받지 않은 시스템/네트워크/다크웹 자원에 대한 무단 접근, 스크래핑, 스캐닝, 크롤링은  
> 대한민국 「정보통신망법」 및 국제 사이버 범죄 관련 법에 따라 처벌될 수 있습니다.
>
> 모든 테스트는 **본인이 소유한 자산 또는 허가받은 환경에서만** 수행해야 합니다.  
> 개발자는 본 도구를 오남용하여 발생하는 어떤 법적 책임도 지지 않습니다.

---

# 🎉 DOTAS v2.0 — Complete.

