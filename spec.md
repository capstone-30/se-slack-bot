# SPEC — SlackBot

## 개요

날씨·강의 시간표·Google Calendar를 지정 시각에 Slack으로 보내는 단일 워크스페이스 개인 봇. 시간표는 슬래시 커맨드로 관리하고 ICS로 구독한다. 사용자 안내는 [README](README.md), 운영은 [DEPLOY](docs/deploy.md)를 따른다.

## 범위

- 포함: 공용 채널·개인 DM 브리프, 날씨·시간표·설정·브리핑·도움말 커맨드, 개인 일정 목록, 브리프 수집 재시도, Google Calendar 읽기, ICS 내보내기. 추가 요구는 시간표 CSV 가져오기/내보내기다.
- 제외: 캘린더 양방향 동기화, 휴강·보강 자동 계산, Slack 모달·버튼 UI, 다중 워크스페이스 OAuth 설치, 전체 날씨 예보 타임라인, DB 마이그레이션 프레임워크, 다중 인스턴스.

## 요구사항

### 커맨드·시간표

- 응답은 모두 ephemeral 텍스트/블록이다. 오류는 `:warning:`으로 표시하고 사용자 입력 오류인 `ValueError` 메시지는 그대로 노출한다.
- 명령어·별칭·인자 계약은 [commands.py](bot/commands.py)의 `*_COMMANDS`, `UPDATE_FIELD_ALIASES`, `DAY_ALIASES`, `_parse_*`가 원본이다. 숫자 `1` 접미 별칭은 중복 명령명 회피용이며 도움말에는 없다. 별칭 추가 시 Slack App에도 등록한다.
- 조회는 `오늘`·`내일`(+1일)·ISO 날짜를 지원하며 그 외 입력은 조용히 오늘로 폴백한다. 추가는 순서 고정·`shlex.split` 파싱(공백 값은 따옴표), 메모는 4번째 이후 토큰을 합친다. 수정의 미등록 필드는 오류이며 삭제는 토큰이 정확히 2개여야 한다.
- 요일은 별칭으로 정규화하고 시각은 제로패딩된 `HH:MM` 및 `start < end`를 검증한다. 수정·삭제는 `slack_user_id + id`로 제한해 다른 사용자·`default` 일정을 보호하며 미일치 시 "찾지 못했습니다" 오류다.
- `/config [도시] [HH:MM] [timezone]`는 인자 없으면 조회, 타임존만 바꿀 때도 앞 두 값을 요구한다. 저장 후 확인 DM 실패는 무시한다. 잘못된 타임존은 `Asia/Seoul`로 폴백한다.
- `/schedule 목록`은 본인 일정만 월~일 순으로 나열하고 수정·삭제에 쓸 ID를 함께 보여준다. CSV 가져오기도 `_validate_day`·`_validate_times`를 적용하며 잘못된 행이 하나라도 있으면 전체 거절한다.

### 브리프·외부 서비스

- 날씨는 필수다. 시간표는 사용자 데이터가 없으면 `default`, Google Calendar는 미인증·실패 시 `[]`로 폴백한다. 정보 수집 실패 시 해당 채널에 `:rotating_light:` 알림을 보내고 `run_brief`는 `False`, 전송 성공 시 `True`를 반환한다. Slack 전송은 지수 백오프로 3회 재시도한다.
- 공용 `daily_brief`는 `Asia/Seoul`의 지정 시각에 `SLACK_CHANNEL_ID`로 `DEFAULT_CITY`·`default` 시간표를 보낸다. 개인 `user_daily_briefs`는 매분 0초에 사용자별 시각·타임존을 확인해 `channel=user_id`로 DM을 보낸다.
- 성공 발송 키 `YYYY-MM-DD HH:MM`을 사용자별로 영속화해 재시작 후 같은 분 중복 발송을 막는다.
- 날씨는 현재 날씨와 첫 예보 구간 `pop`만 사용한다. 캐시 TTL은 1시간이며 `RequestException` 시 만료 캐시라도 반환하고 캐시가 없으면 예외를 올린다. 예보 실패는 `rain_prob=0`이다.
- 날씨 401은 `RequestException`을 상속하지 않는 `InvalidAPIKey`다. 캐시 폴백·수집 재시도 없이 스케줄러까지 전달해 해당 잡을 제거하고, 커맨드에서는 경고로 표시한다. 일시적 수집 오류 재시도의 총 대기는 매분 잡을 위해 60초를 넘지 않아야 한다.
- Google Calendar는 `calendar.readonly`, `primary`, 오늘 00:00~23:59만 조회한다. 만료 토큰은 갱신·저장하고 모든 조회 예외는 `[]`로 반환한다.

### ICS·저장소·기동

- `GET /calendar/{slack_user_id}.ics?token=…`: `CALENDAR_ACCESS_TOKEN` 미설정 503, 불일치 403, 비교는 `secrets.compare_digest`. `/health`도 제공한다.
- ICS는 사용자의 전체 시간표를 `RRULE FREQ=WEEKLY;BYDAY=…`로 내보낸다. DTSTART는 `Asia/Seoul`의 오늘 이후 첫 해당 요일이며 타임존도 고정한다. UID는 `uuid5(NAMESPACE_URL, "slackbot:{user}:{course_id}")`를 유지해 재구독 중복을 막는다.
- DB 계약은 [courses.py](bot/courses.py)의 `init_db`, [config_store.py](bot/config_store.py)의 `CREATE_USER_CONFIG`, [database.py](bot/database.py)의 `CREATE_GOOGLE_TOKENS`가 원본이다. `courses` 소유자 기본값은 `default`, `user_config`·`google_tokens` 키는 `slack_user_id`다.
- 명시적 `db_path` → SQLite, 아니면 `DATABASE_URL` → PostgreSQL, 그 외 `DB_PATH`(기본 `./data/bot.db`)다. 내부 `db_path=None`은 PostgreSQL을 뜻한다. SQL은 `placeholder(db_path)`로 `?`/`%s`, `is_postgres(db_path)`로 PK `RETURNING id`를 분기한다. 커서 직접 사용 대신 정상 종료 커밋·예외 종료 롤백의 `db()`를 사용한다.
- Google OAuth 토큰 원문은 `DATABASE_URL`이 있으면 DB, 아니면 `GOOGLE_TOKEN_DIR` 파일에 저장한다. 이 선택은 import 시 고정되어 런타임 환경변수 변경은 반영하지 않는다. 로컬 인증은 [google_calendar.py](bot/google_calendar.py)의 `authorize_user`와 경로 상수를 따른다.
- 필수 환경변수 누락·`NOTIFY_TIME` 형식 오류는 `sys.exit(1)`, 그 외 기동 실패는 로그 후 진행한다. 필수값·기본값 원본은 [main.py](main.py), 설정 안내는 [환경 변수](docs/deploy.md#환경-변수)다. `courses`가 비면 부팅 시 `default` 샘플 6건을 재삽입하므로 빈 상태가 유지되지 않는다.

## 완료 기준

- 위 계약과 회귀 테스트를 충족하고 CI 검증이 통과한다.
- 개인 목록의 ID를 수정·삭제에 그대로 사용할 수 있다.
- 일시적 `RequestException` 1회 뒤 수집 성공 시 브리프가 전송되고 오류 알림은 없다.
- 내보낸 CSV를 그대로 가져오면 같은 시간표가 되며, 입력 오류 시 부분 저장은 없다.
