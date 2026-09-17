# PLAN — SlackBot

## 저장소 구조

- 코드·테스트·CI·문서는 하나의 Git 저장소에서 관리한다. 기존 코드 저장소 `SE-SlackBot/main`을 기준으로 코드 저장소의 커밋 이력을 보존하고, 문서 저장소는 파일만 통합한다.
- 실행 코드(`main.py`, `bot/`), 테스트(`tests/`), 의존성 파일과 `.github/workflows/`는 루트에 둔다. 요구사항·계획은 루트 `spec.md`·`plan.md`에, 배포·소개 자료는 `docs/`에 두며, 공통 작업 지침은 루트 `AGENTS.md`에서 관리한다.
- 프로젝트 소개·설치·사용·테스트·문서 안내는 루트 `README.md` 하나로 통합한다. 소개 이미지는 `docs/assets/`에 유지한다.
- 설치·실행·테스트·배포 명령은 저장소 루트를 기준으로 하고, 내부 문서와 코드 참조는 상대 경로를 사용한다.

## 구현 방향

- [main.py](main.py) 한 프로세스에서 Slack Bolt(메인 스레드), APScheduler `BackgroundScheduler`, FastAPI ICS(daemon thread + uvicorn)를 실행한다. `SLACK_APP_TOKEN`이 있으면 Socket Mode, 없으면 HTTP `PORT`(기본 3001), ICS는 `API_PORT`(기본 3000)다.
- 부팅 순서는 env·알림 시각 검증 → DB 초기화·샘플 처리 → 커맨드 등록 → 스케줄러 → API 스레드 → Slack 앱이다.
- [commands.py](bot/commands.py) → [scheduler.py](bot/scheduler.py) → 데이터·외부 서비스·메시지·전송 모듈의 단방향 의존을 유지한다. 모듈별 책임은 [bot](bot)의 파일명과 공개 함수를 따른다.
- 브리프 수집 실패는 `scheduler.COLLECT_BACKOFF`(2·5초)로 2회 재시도하고, `InvalidAPIKey`는 재시도 없이 위로 올린다.
- 날씨 캐시는 프로세스 메모리, 개인 브리프는 매분 폴링, 중복 방지는 기존 `user_config.settings_json.last_brief`(사용자당 1건, 정리 없음)를 사용한다. 배포 절차는 [DEPLOY](docs/deploy.md)를 따른다.

## 단계별 계획

1. 시간표 CSV: 입출력 경로를 결정한 뒤 `courses.py`에 대량 삽입, 표준 라이브러리 `csv`로 직렬화를 구현한다.

## 검증 전략

실행 명령은 [README 테스트](README.md#테스트)를 따른다.

- [CI](.github/workflows/ci.yml)는 Python 3.11·3.12·3.13에서 `pip check` → `compileall` → `pytest`를 실행한다. 변경 모듈에 대응하는 [tests](tests)에서 [완료 기준](spec.md#완료-기준)을 검증한다.
- 스키마 변경 시 `CREATE TABLE IF NOT EXISTS`만으로 기존 DB 컬럼이 추가되지 않으므로 수동 `ALTER TABLE` 절차를 PR에 포함한다.
- `ponytail:` 주석은 명시된 확장 조건이 성립할 때만 재검토한다.

## 리스크 및 미결정

- CSV 경로 미정: ICS 옆 API 또는 슬래시 커맨드 붙여넣기. Slack 파일 업로드는 추가 스코프가 필요하다.
- 인스턴스는 1개 전제다. 늘리려면 날씨 캐시를 DB로 옮기고 `last_brief`의 read-modify-write 경합을 먼저 없앤다.
- 개인 DM은 `im:write`가 없으면 실패한다. 날씨 401로 제거된 잡은 키 수정 후 프로세스를 재시작해야 재등록된다.
- FastAPI 데몬 스레드는 Slack 메인 스레드 종료 시 함께 종료된다. `/health`는 Slack 앱 상태를 보증하지 않는다.
- ICS 날짜를 서버 로컬 날짜로 계산하면 UTC 환경에서 첫 회차가 한 주 어긋날 수 있다. SPEC의 날짜·UID 계약을 유지한다.
