#🕵️ DOTAS v1.0
Dark-web OSINT Threat Alert System

다크웹(.onion) 및 공개 출처(OSINT)에서 특정 키워드·인디케이터(IOC)를 탐지하고,
이를 CSV 저장 + Telegram 알림으로 전달하는 경량 위협 인텔리전스 시스템입니다.

📌 기능 개요

DOTAS v1.0은 다음과 같은 모듈로 구성됩니다:

1. Dark Web 크롤링 (Tor 사용)

.onion 주소 접근

Tor SOCKS5h 이용하여 DNS 요청까지 Tor 내부에서 처리

2. OSINT 수집

GitHub 기반 CTI 인덱스 (deepdarkCTI 등)

텍스트 기반 페이지 자동 파싱

3. 인디케이터(IOC) 자동 추출

이메일 / 도메인 정규식 기반 탐지

키워드 기반 필터링 (“password”, “admin”, 조직 도메인 등)

4. 중복 알림 방지 (IOC Persistence)

seen_indicators.txt에 기록

이미 본 인디케이터는 다시 알림 X (실제 SOC 운영 구조 반영)

5. CSV 저장 + Telegram 실시간 알림

findings.csv에 결과 저장

Telegram Bot API를 이용한 경고 알림

🔧 설치 방법
1) 환경 준비
sudo apt update
sudo apt install tor
sudo service tor start

2) Python 설치 패키지
pip install requests beautifulsoup4 urllib3

3) Telegram 설정

BotFather → 봇 생성 → Bot Token 발급

@userinfobot → Chat ID 확인

다음 코드 수정:

TELEGRAM_TOKEN = "YOUR_BOT_TOKEN"
CHAT_ID = "YOUR_CHAT_ID"

4) 실행
python3 dotas_v1.py

📁 파일 구조
dotas_v1.py
findings.csv              # 인디케이터 결과 저장
seen_indicators.txt        # 중복 알림 방지 목록
README.md

⚙️ 시스템 동작 흐름 (Architecture)
            +------------------+
            |   DARK WEB       | -- Tor --> (requests via socks5h)
            |   (.onion)       |
            +------------------+
                      |
                      v
            +------------------+
            |   Collector      |
            | (fetch_url)      |
            +------------------+
                      |
            +--------------------------+
            |   Analyzer               |
            | - BeautifulSoup cleaner  |
            | - Keyword filter         |
            | - Email/Domain extract   |
            +--------------------------+
                      |
          +-----------+--------------+
          |                          |
          v                          v
+-------------------+       +---------------------+
|  CSV Logger       |       | Telegram Alert Bot  |
| findings.csv      |       | send_telegram()     |
+-------------------+       +---------------------+
                  |
                  v
        +------------------------+
        |   seen_indicators.txt  |
        | (중복 알림 방지 DB)     |
        +------------------------+

⚠️ 주요 기술 난관 & 트러블슈팅 과정

이 프로젝트의 핵심 난관은 크게 3가지였다.
각 문제와 해결 과정을 정리했다.

1️⃣ Tor 프록시 문제 — socks5 vs socks5h
❌ 문제

처음에는 socks5://127.0.0.1:9050 으로 요청을 보냈지만,
.onion 사이트 접근 시 다음 오류가 발생했다:

NameResolutionError: DuckDuckGo...onion could not be resolved


이는 DNS를 로컬에서 resolve하려고 시도했기 때문이다.

✅ 해결

→ socks5h:// 를 사용해야 Tor 내부에서 DNS 해석을 수행한다.

PROXIES = {
    "http": "socks5h://127.0.0.1:9050",
    "https": "socks5h://127.0.0.1:9050"
}


socks5 : DNS 로컬에서 → 오류

socks5h : DNS까지 Tor가 처리 → 정상 접속

2️⃣ 동기 I/O (requests) + 비동기(async) 혼용 문제
❌ 문제

처음에는 이렇게 작성했다:

async def main():
    while True:
        alert_msg = check_darkweb()   # 동기 함수 호출
        await send_alert(alert_msg)
        await asyncio.sleep(60)


check_darkweb() 내부가 blocking I/O이므로
async 루프가 제대로 동작하지 않음 → 전체 loop가 멈짐.

✅ 해결

동기 HTTP 요청(heavy blocking)을
비동기 루프 내에서 쓰레드로 오프로드:

alert_msg = await asyncio.to_thread(check_darkweb)


→ 이벤트 루프는 계속 살아 있음
→ 텔레그램 알림은 async로, HTTP는 동기로 최적 분리

3️⃣ 텔레그램 알림이 반복 전송되는 문제
❌ 문제

수집 대상 페이지에 동일 키워드가 계속 존재하면
60초마다 같은 알림을 계속 보내는 문제 발생.

예:

[경고] bitcoin 발견
[경고] bitcoin 발견
[경고] bitcoin 발견
...

✅ 해결

중복을 기록하는 로직 추가:

HISTORY_FILE = "seen_indicators.txt"

def is_new_indicator(value):
    return value not in seen_list


→ 한번 본 이메일/도메인은 파일에 기록되고
→ 다음 사이클에서는 알림 제외

이로써 SOC 운영 수준의 알림 피로도 해결(De-duplication) 구현.

4️⃣ HTML 파싱 정확도 향상
❌ 문제

키워드가 HTML 태그 안에 있어서 제대로 검색 안 됨:

<span>pri<b>va</b>cy</span>

✅ 해결

BeautifulSoup으로 정규화 후 텍스트만 사용:

soup = BeautifulSoup(raw, "html.parser")
text = soup.get_text(separator="\n")


→ 모든 태그 제거
→ 키워드 필터 & IOC 탐지 정확도 상승

5️⃣ SSL 인증 문제 — verify=False 필요

일부 DarkWeb 프록시나 .onion 서비스는
정상적인 SSL 체인이 없어서 인증서 오류 발생:

ssl.SSLError: CERTIFICATE_VERIFY_FAILED

해결
requests.get(..., verify=False)


SSL 경고 끄기:

urllib3.disable_warnings(...)

🚀 최종 실행 예시 로그
>> [DOTAS] 다크웹 & OSINT 위협 모니터링 시스템 가동

[Cycle 시작] 2025-11-29 13:00:00
[*] 소스 처리 시작: DuckDuckGo Onion
[+] Fetch 성공: ...onion/ (size=117030)
[탐지] DuckDuckGo Onion에서 domain 발견: privacy.com
[*] Telegram 전송 완료

[Cycle 종료] 대기 300초...
