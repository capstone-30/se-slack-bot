# Slack Weather & Schedule Bot 배포 가이드 (DEPLOY)

`main.py` 하나가 Slack 앱 + 스케줄러 + ICS API를 한 프로세스로 띄운다. 배포는 **이 프로세스를 계속 살려 두는 일**이 전부다.
인스턴스는 1개만 — 중복 방지와 날씨 캐시가 프로세스 메모리에 있어 2개 이상 띄우면 브리프가 중복 발송된다.
시스템 계약은 [spec.md](../spec.md), 모듈 구조는 [plan.md](../plan.md#구현-방향), 사용자용 요약은 [README.md](../README.md).

| 대상 | 방식 | 상태 |
|---|---|---|
| Oracle Cloud (Always Free) | systemd + nginx, main 푸시 시 CI가 자동 배포 | 주력 |
| 로컬 | `python main.py` + Socket Mode | 개발 |

## 1. 로컬 실행

설치·실행은 [README 시작하기](../README.md#시작하기)를 따릅니다. `.env`는 `load_dotenv()`가 읽습니다.

로컬에서는 `SLACK_APP_TOKEN`(`xapp-`)을 넣어 Socket Mode로 실행하면 공인 URL이나 터널링 없이 사용할 수 있습니다.

### 환경 변수

| 변수 | 필수 | 기본값 / 용도 |
| --- | --- | --- |
| `SLACK_BOT_TOKEN` | 예 | Slack Bot Token |
| `SLACK_SIGNING_SECRET` | 예 | 요청 서명 검증 |
| `OPENWEATHER_API_KEY` | 예 | OpenWeatherMap API 키 |
| `SLACK_CHANNEL_ID` | 예 | 공용 데일리 브리프 채널 |
| `SLACK_APP_TOKEN` | 아니요 | 설정 시 Socket Mode |
| `NOTIFY_TIME` | 아니요 | 공용 브리프 시각, `07:00` |
| `DEFAULT_CITY` | 아니요 | 공용 브리프 도시, `Seoul` |
| `DATABASE_URL` | 아니요 | 설정 시 PostgreSQL 사용 (`DB_PATH`보다 우선) |
| `DB_PATH` | 아니요 | SQLite 경로, `./data/bot.db` |
| `CALENDAR_ACCESS_TOKEN` | 아니요 | ICS 구독에 사용할 접근 토큰 |
| `GOOGLE_TOKEN_DIR` | 아니요 | OAuth 토큰 경로, `./data/google_tokens` |
| `PORT` | 아니요 | Slack HTTP 포트, `3001` |
| `API_PORT` | 아니요 | ICS API 포트, `3000` |
| `DEBUG` | 아니요 | 설정 시 DEBUG 로그 |

## 2. Slack App 설정

| 항목 | 값 |
|---|---|
| Bot Token Scopes | `chat:write`, `commands`, `im:write` |
| Slash Commands | 쓸 이름을 전부 등록 (`/weather` `/schedule` `/config` `/brief` `/bot-help` + 한글·숫자 별칭) |
| Request URL (HTTP 모드) | `https://<도메인>/slack/events` |
| Socket Mode | `SLACK_APP_TOKEN` 설정 시. Request URL 불필요 |

`im:write`가 없으면 사용자별 데일리 브리프(DM)와 `/config` 확인 메시지가 전부 실패한다. 코드에 별칭을 추가해도 Slack App에 같은 이름을 등록하지 않으면 호출되지 않는다.

## 3. Google Calendar (선택)

OAuth 클라이언트 파일 `credentials.json`을 프로젝트 루트에 두고 **사용자별로 1회** 인증한다. 브라우저가 열리므로 로컬에서 실행한다.

OAuth 동의 화면이 "테스트" 상태면 인증할 Google 계정을 Google Cloud Console의 **테스트 사용자**로 먼저 추가해야 한다. 빠뜨리면 인증 단계에서 액세스 차단 오류가 난다. `SLACK_USER_ID`는 Slack 프로필 → `...` 더보기 → 멤버 ID 복사로 얻는 `U`로 시작하는 값이다.

```bash
python -c "from dotenv import load_dotenv; load_dotenv(); from bot.google_calendar import authorize_user; authorize_user('SLACK_USER_ID')"
```

토큰 저장 위치는 `DATABASE_URL` 유무로 갈린다(SPEC 참고). 파일로 만든 토큰을 PostgreSQL로 옮기려면:

```bash
python scripts/upload_google_token.py <slack_user_id>
```

미인증·조회 실패는 빈 일정으로 처리되고 날씨·시간표 브리프는 그대로 나간다.

## 4. Oracle Cloud

Always Free `VM.Standard.E2.1.Micro` (Ubuntu 22.04, 1 OCPU / 1 GB) 기준. RAM 1GB라 여유가 없으니 필요하면 swap을 잡는다.

### 4.1 네트워크

OCI 콘솔에서 VCN + Public Subnet + Internet Gateway를 만들고 라우트 테이블에 `0.0.0.0/0 → Internet Gateway`를 추가한다. Security List Ingress에 22 / 80 / 443(TCP)을 연다.

### 4.2 서버 준비

```bash
ssh -i <private-key>.key ubuntu@<PUBLIC_IP>

sudo apt update && sudo apt install -y python3.11 python3.11-venv nginx
# OCI 이미지는 iptables 로도 막혀 있다
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo apt install -y iptables-persistent && sudo netfilter-persistent save

git clone <repo-url> ~/slackbot && cd ~/slackbot
python3.11 -m venv venv && venv/bin/pip install -r requirements.txt
cp .env.example .env       # 값 입력
```

### 4.3 systemd

`/etc/systemd/system/slackbot.service`:

```ini
[Unit]
Description=Slack Bot
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/slackbot
ExecStart=/home/ubuntu/slackbot/venv/bin/python main.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now slackbot
```

`load_dotenv()`가 `WorkingDirectory`의 `.env`를 읽으므로 `EnvironmentFile`은 필요 없다.

### 4.4 nginx

`/etc/nginx/sites-available/slackbot`:

```nginx
server {
    listen 80;
    server_name <PUBLIC_IP_OR_DOMAIN>;

    location /slack/ { proxy_pass http://127.0.0.1:3001; }
    location / { proxy_pass http://127.0.0.1:3000; }

    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/slackbot /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

`/slack/`은 Slack 앱(`PORT`, 3001), 나머지는 ICS API(`API_PORT`, 3000)로 간다. Request URL은 `http://<PUBLIC_IP>/slack/events`. 도메인이 있으면 `certbot --nginx`로 HTTPS를 붙인다.

## 5. 자동 배포

main 푸시 → `.github/workflows/ci.yml`의 `test` 잡(3.11/3.12/3.13) 통과 → `deploy` 잡이 SSH로 `git pull` 후 `sudo systemctl restart slackbot`.

필요한 Repository Secrets: `OCI_HOST`, `OCI_USER`, `OCI_SSH_KEY`.

> **주의**: 워크플로의 `pip install`은 PATH의 pip를 쓴다. 위처럼 venv로 설치했다면 워크플로를 `venv/bin/pip`로 바꿔야 의존성이 서비스가 실제로 보는 곳에 설치된다. 그러지 않으면 `requirements.txt`에 새 패키지를 추가한 배포가 `ModuleNotFoundError`로 죽는다.
>
> `sudo systemctl restart`가 비밀번호를 묻지 않으려면 배포 계정에 해당 명령의 NOPASSWD sudoers 설정이 있어야 한다.

## 6. 운영

```bash
sudo systemctl status slackbot
sudo journalctl -u slackbot -f          # 실시간 로그
sudo systemctl restart slackbot
curl -s localhost:3000/health           # {"status":"ok"}
sqlite3 data/bot.db "select slack_user_id,city,notify_time,timezone from user_config"
```

- [ ] `DEBUG=1`을 `.env`에 넣고 재시작하면 DEBUG 로깅(날씨 API 원본 응답 포함)이 켜진다
- [ ] 인스턴스는 항상 1개만 — 2개 이상이면 브리프가 중복 발송된다
- [ ] `.env`, `credentials.json`, `data/`, OAuth 토큰(`token*.json`)은 커밋하지 않는다 (`.gitignore` 적용됨)

## 7. 트러블슈팅

| 증상 | 점검 |
|---|---|
| 부팅 직후 종료 | 필수 env 4개(`SLACK_BOT_TOKEN` `SLACK_SIGNING_SECRET` `OPENWEATHER_API_KEY` `SLACK_CHANNEL_ID`) 또는 `NOTIFY_TIME` 형식. 로그에 이유가 남는다 |
| 커맨드 무응답 (`dispatch_failed`) | Slack App에 그 이름이 등록됐는지, Request URL이 `/slack/events`인지, Socket Mode면 `SLACK_APP_TOKEN`이 맞는지 |
| 공용 브리프만 오고 DM은 안 옴 | `im:write` 스코프, `user_config`에 레코드가 있는지(`/config` 1회 실행 필요) |
| 브리프가 두 번 옴 | 인스턴스가 2개거나 알림 시각 직전에 재시작됨. 중복 방지는 프로세스 메모리다 |
| ICS가 `503` | `CALENDAR_ACCESS_TOKEN` 미설정. `403`이면 쿼리스트링 토큰 불일치 |
| 캘린더 일정만 비어 있음 | 해당 사용자 토큰 미인증·만료. `google_calendar` 로거를 본다. 브리프 자체는 정상 동작 |
| 배포 후 `ModuleNotFoundError` | 5절의 venv/pip 주의사항 |
