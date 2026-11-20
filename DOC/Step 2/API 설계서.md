# API 설계서

**프로젝트** 대치루틴 — 초1~고1 학습·안전 관리 앱
**문서** API 설계서 (API Design Specification)
**버전** v1.2 · **작성일** 2026-11-18 · **개정일** 2026-11-20
**기준 커밋** [`53cdf857102746081d6cbb12eed5a758759f0de4`](https://github.com/SHS5029/Daechi_team4_sorry/tree/53cdf857102746081d6cbb12eed5a758759f0de4) (REST 계약 59개 정상 비교용 스냅샷)

**DB 재현 기준** `9af07b8`. DB 계약·테이블 수·격리 시험은 [DB 설계서 v1.3](./데이터베이스%20스키마%20설계서.md) 기준으로 관리한다.
**작성** 윤기화 (WBS 2.1.4) · **검토** 오병훈 (WBS 7.4.1)
**제출** 11/20 단계 산출물 (S1)

**이번 보완의 핵심** 59개 동작의 응답 필드·성공 예시·오류·상태를 [API 응답 계약 v1.2](./API%20응답%20계약%20v1.2.md)에 별도 색인으로 추가했다. OpenAPI가 아직 선언하지 않은 성공 응답 스키마의 공백과, 코드가 필요한 후속 작업은 §7·§8에 명시했다.

**제품 일정** 12/9 전후 출시 목표. 기능 동결·스토어 제출일은 팀 재확정 대기. 기존 12/13 동결·12/18 스토어 제출·1/3 공개는 이전안이다. 12/18 Auto-Healing과 12/24 대화형 웹 교육 제출 및 1/4 발표는 유지한다.

**검토 자료** [OpenAPI 스냅샷](./openapi-53cdf85.json) · [59개 동작 상세 계약](./API%20상세%20계약%2053cdf85.md) · [59개 응답 계약](./API%20응답%20계약%20v1.2.md) · [DB 설계서](./데이터베이스%20스키마%20설계서.md).

**동기화한 문서** [기획서](./기획서%20v0.5.md) · [기능 명세서](./기능%20명세서.md) · [기술 명세서](./기술%20명세서.md) · [WBS](./WBS.md). 결정·담당·기한은 §8에 모았다.

> **이 문서의 근거.** 아래 사실은 실행 중인 서버와 저장소에서 **직접 추출**했습니다. 손으로 옮겨 적지 않았습니다.
>
> | 근거 | 추출 방법 | 결과 |
> | --- | --- | --- |
> | API 계약 | `GET http://127.0.0.1:8000/openapi.json` | OpenAPI 3.1 · **58 path / 59 operation / 31 schema** |
> | 서버 라우터 | `server/app/api/*.py`, `server/app/admin.py` | 8개 라우터, `prefix="/v1"` |
> | 서술 계약 | `docs/mvp/03-api.md` (47KB, 25개 상세) | 요청·응답·오류·동작 규칙 |
> | 기능별 계약 | `docs/features/{block,location,family-agreement,push}.md` | 1.0 기능 계약(잠금·위치·약속·푸시) |
> | 관리자 계약 | `docs/admin-web/admin-web-screen-spec.md` | 관리자 17개 |
> | 오류 코드 | `server/app/errors.py` | **24개** |
> | 실응답 확인 | 무인증·없는 경로 호출 | `401 unauthorized` / `404 not_found` (문서와 일치) |

---

## 목차

1. 개요와 AI 도우미 도구 · 2. 공통 규약·실패 대응 · 3. 엔드포인트 인벤토리 · 4. 대표 계약 · 5. 화면↔API 매핑 · 6. 역할·권한 매트릭스 · 7. 검증 · 8. 기준 결정과 반영 이력 · 9. 부록

---

## 1. 개요

### 1.1 목적

이 문서는 **앱·웹·관리자·AI 워커가 서버와 주고받는 계약**을 한 곳에 고정한다. 목적은 세 가지다.

1. **화면과 서버를 병렬로 개발한다.** 화면 담당이 서버 구현을 기다리지 않도록 요청·응답의 모양을 먼저 확정한다.
2. **권한과 실패를 계약으로 만든다.** "누가 부를 수 있는가", "실패하면 어떤 코드로 끝나는가"를 API 수준에서 못 박는다.
3. **문서와 구현을 대조한다.** 기준 구현·설계·미실시 시험과 변경 이력은 §7~8에 기록한다.

### 1.2 범위

| 구분 | 범위 | 근거 |
| --- | --- | --- |
| **MVP (11/22 구현 · 11/24 검증)** | 가입·문자 인증·자녀 등록·기기 연결·자료 업로드·문항 생성·검수·커리큘럼·오늘 미션·완료·오답·부모 현황 | 학습 MVP 화면 13개(P-01~08, C-01~05) |
| **제품 1.0 후보** | 가족 약속·잠금 정책·해제 요청/승인·집중 모드·학원 장소·도착 알림·푸시 | `docs/features/*` |
| **AI 도우미 (12/24 산출물)** | 대화 세션·메시지·제안 수락/거절·피드백 | FR-WEB-002 |
| **관리자 (내부)** | 관리자 인증·문제은행·가족 조회·사용량·감사 로그 | WBS 4.4.2 |
| **범위 밖** | 실시간 위치 조회·이동 경로, 성적 순위, 결제 API | 기획서 4.4·FR-PAR-004 |

### 1.3 요약

| 항목 | 값 |
| --- | --- |
| 엔드포인트 | **59개** (경로 58개, `/healthz` 포함) |
| 도메인 묶음 | **11개** |
| 요청 스키마 | **31개** (부록 A) |
| 오류 코드 | **24개** (2.4) |
| 인증 방식 | 4종 — 학부모 JWT · 자녀 익명 세션 · 관리자 세션 쿠키 · 기기 키 |
| 멱등성 | 본문 UUID(6개 API) + `device_event_keys` + 상태 검사 + 부분 유니크 + 중복 완료 차단 |
| 성공 상태 코드 | `200` / `201` / `202` / `204` |
| 응답 계약 | [59개 응답 계약 v1.2](./API%20응답%20계약%20v1.2.md)에 성공 필드·null 허용·주요 오류·상태를 정리 |
| OpenAPI 응답 스키마 | 현재 성공 응답은 `schema:{}`인 경우가 있어, 응답 계약 문서를 검토 기준으로 병행 |
| 연동 검증 | 서버 API 시험은 있으나 4xx·429 실제 부하, 동시 30건 경합, 실기기 E2E는 미실시 |
| 예정·제외 | 관리자 문제은행 일부 정책 검증과 `review` 수락 시 대상 상태 재검사는 예정. `library_set`의 수락 후 실행과 보상(상점·포인트·장비)은 이번 출시·교육 제출 범위에서 제외 |

### 1.4 AI 도우미 도구 계약

[팀 자체 30문항 §2](./BFCL/bfcl-30-questions-v2.md)과 기준 커밋의 `assistant/models.py`, `lookups.py`, `validate.py`, `services/assistant.py`를 대조했다. 모델은 `{reply, proposals[], need[]}`를 반환한다. 서버가 `need[]`의 읽기 도구를 실행하고, `proposals[]`는 검증 후 부모에게 보여줄 카드로 저장한다. 별칭은 문자열(`child1`, `set2`, `lib1`)이며 서버가 가족 범위에서 UUID로 바꾼다. 구조화 스키마의 미사용 필드는 `null`을 보낸다. 아래 필수는 각 도구가 유효하려면 필요한 인자다.

| 도구 | 입력·호출 조건 | 성공 출력 | 실패·범위 |
| --- | --- | --- | --- |
| `generate_set` · 제안 | `set:string` 필수. 가족 세트 상태가 `uploaded` 또는 `failed`, 생성 한도가 남아야 한다 | 수락 시 `{job_id,set_id}` (UUID 문자열), 제안 `accepted`. 작업 접수까지 | 상태 충돌 `409 conflict`, 파일 오류 `422 file_invalid/file_too_large`, 한도 `429 limit_exceeded` |
| `library_set` · 제안 | `child,material:string` 필수, `count:int|null` 10~60. 공개 자료·자녀 학년 범위·한도 검사 | 판단 평가에서는 카드 선택만 채점. 실제 세트 생성 결과는 현재 없음 | 수락은 `409 conflict` + `{reason:"not_available"}`, 카드 `open`. 구현 범위 제외 |
| `review` · 제안 | `set:string` 필수. 제안 검증에서 가족 세트의 `items.pending > 0` | 수락 시 `{set_id}`, 카드 `done`. 앱이 검수 화면을 열고 사람이 검수 | 제안 시 조건 불충족이면 카드 폐기. 기준 코드에서 수락 시 문항 대기 수 재검사와 자동 승인은 수행하지 않음 |
| `curriculum` · 제안 | `child,set:string` 필수, `daily_new_cap:int|null` 3~20(기본 10), `days:int|null` 7~56(기본 14). 미편성 승인 문항 필요 | `{child_id,set_ids,start_date,end_date,daily_new_cap,rest_weekdays,preview}`. 미리보기의 `curriculum_id`는 null. 사용자가 저장할 때 `POST /v1/curricula`에 `proposal_id` 전달 | `404 not_found` 또는 `422 no_approved_items/items_in_active_curriculum/plan_infeasible`. 실행 실패 시 카드 `open` |
| `library_search` · 조회 | `subject:string|null, grade:int|null, query:string|null` 선택. 공개 팀 자료의 과목·학년·제목/단원으로 조회 | `{materials:[{ref,title,subject,grades,units,items,already_given,days_since_published}]}` 최대 5건. 새 결과에 서버가 별칭 부여 | 일치 없음은 `{materials:[]}`. 조회 예외는 상위 요청 오류 처리 |
| `set_units` · 조회 | `set:string` 필수. 가족 세트로 제한 | `{units:[{unit,items,approved}]}` 최대 20건, 거절 문항 제외 | 별칭 없음 `{error:"unknown_ref"}`. 결과 없음은 빈 배열 |
| `unit_history` · 조회 | `child:string` 필수. 해당 가족 자녀의 최근 90일 첫 풀이 | `{units:[{unit,set,answered,first_try_correct_rate}]}` 최대 20건. 세트 별칭은 null 가능 | 별칭 없음 `{error:"unknown_ref"}`. 결과 없음은 빈 배열 |

도구별 현재 상태는 다음과 같다.

| 도구 | 라우트/그래프 | 수락 후 실행 | 현재 판정 | 비고 |
| --- | --- | --- | --- | --- |
| `generate_set` | 구현 | `generation.submit` | 구현 | 세트 상태·소유권은 실행 서비스에서 재검사 |
| `library_set` | 카드 검증만 | 없음 | 제외 | 수락 시 409 `{reason:"not_available"}` |
| `review` | 구현 | 앱의 검수 화면 열기 | 구현·부분 | 수락 시 `set_id`만 확인하며, 대상 세트의 현재 대기 수 재검사는 미구현 |
| `curriculum` | 구현 | `curricula.create(dry_run=True)` | 구현 | 미리보기만 저장하지 않음. 실제 저장은 별도 `POST /v1/curricula` |
| `library_search` | 구현 | 읽기 전용 | 구현 | 최대 5건, 새 결과에 별칭 부여 |
| `set_units` | 구현 | 읽기 전용 | 구현 | 최대 20건, 거절 문항 제외 |
| `unit_history` | 구현 | 읽기 전용 | 구현 | 최근 90일, 최대 20건 |

제안 검증에서 허용되지 않은 대상·상태는 저장하지 않는다. 입력 수치의 범위 처리는 기준 구현의 검증·보정 규칙을 따른다. 읽기 실패 객체는 모델 내부 도구 결과이며 HTTP 오류와 구분한다.

팀 자체 30문항은 답글 최대 600자, 제안 최대 3개, 조회 최대 2회, 모델 호출 최대 3회로 채점한다. 배포 후보 그래프는 조회를 라운드로 계산하고 상위 모델 호출을 별도 예산으로 관리하므로 §2.6.1의 운영 한도와 분리해 기록한다.

예를 들어 읽기 요청은 `need: [{tool:"set_units", set:"set2", subject:null, grade:null, query:null, child:null}]`로 표현한다. 서버가 조회 결과를 모델 입력에 추가한 다음, 마지막 응답의 `proposals[]`를 검증해 화면에 보여준다. `need[]`와 `proposals[]`가 빈 배열이면 도구 없이 안내만 반환한다.

### 1.5 요청부터 수락·실행까지

1. `POST /v1/assistant/messages`에서 보호자 인증, 대화 소유권, 자녀의 가족 범위와 일일 한도를 확인한다. 부모 메시지 UUID로 실행을 확보하고 최근 대화 및 별칭으로 바꾼 현황을 모델에 전달한다.
2. 서버가 조회 결과를 모델에 되돌려주고, 구조화 출력과 제안 대상·상태를 검증한다. 부모 메시지의 `id`는 도우미 답변 `assistant_messages.usage.reply_to`에, 답변 ID는 `assistant_proposals.message_id`에 연결된다.
3. `POST /v1/assistant/proposals/{proposal_id}/accept`에서 보호자 권한·제안 가족 소유권·`open` 상태를 행 잠금으로 확인한다. 생성 접수와 커리큘럼 미리보기는 각 서비스에서 대상 소유권·현재 상태를 재검사한다. 검수 화면 열기에도 대상 소유권·대기 수를 수락 시점에 재확인하도록 보강하고 별도로 시험한다. 거절은 `dismiss`로 남긴다.
4. 생성 시 `assistant_proposals.result.job_id` → `generation_jobs.id`로 진행과 최종 결과를 확인한다. 커리큘럼은 별도 저장에 `proposal_id`를 전달한다. 수락 자체는 별도 HTTP 요청이다. 동일 업무를 추적하는 기준은 `thread_id → 부모 메시지 ID → 답변 ID → proposal_id → job_id`이며, 전체 구간에서 공통 correlation ID를 구조화 로그에 넣는 작업은 예정이다.

### 1.5.1 수락 시 재검증표

| 제안 | 제안 가족·`open` 상태 | 대상 소유권·현재 상태 | 실행 | 현재 상태 |
|---|---|---|---|---|
| `generate_set` | 행 잠금으로 확인 | `generation.submit`이 세트 소유권·상태·파일을 재검사 | 작업 접수 | 구현 |
| `curriculum` | 행 잠금으로 확인 | `curricula.create(dry_run)`이 자녀·세트 소유권·승인 문항을 재검사 | 미리보기 | 구현 |
| `review` | 행 잠금으로 확인 | **현재 재검사하지 않음** | 앱이 검수 화면을 염 | **미구현·후속** |
| `library_set` | 행 잠금으로 확인 | 실행하지 않음 | 409 `not_available` | 제외 |

`review`는 수락 시점에 세트가 이 가족의 것인지, `items.pending > 0`인지 다시 확인해야 한다. 이 항목은 2026-11-20 기준 **미구현**이며, 담당 심환석·기한 11/22·WBS 3.7.9·7.4.1b에 둔다. 수락 경합은 제안 행 잠금까지 구현되어 있고, 동시 최초 요청·고아 인계는 12/31 통합 시험 전 보강한다.

**평가 구분** 팀 자체 30문항 v1.0.0은 `decision_only`이며 문항 JSON 변환과 모델 실행은 원문 기준 미실시다. 가상 조회가 필요한 E 분류도 실제 부모 수락·DB 쓰기·워커 성공과 분리한다. 서비스 시험은 별도 분모로 생성 수락→작업 완료, 검수 화면 연결, 커리큘럼 미리보기→별도 저장, 수락 경합·권한 거부를 기록한다. `library_set`은 실제 생성 성공 시험에 넣지 않고 409 거절 시나리오로 검증한다. BFCL 원본 30문항 점수와도 합산하지 않는다. `review` 재검사는 현재 `미구현`으로 표시하며, 구현 전에는 “검수 화면 연결 성공”과 “검수 상태 재검증 완료”를 같은 것으로 보지 않는다.

---

## 2. 공통 규약

### 2.1 기준 주소와 버전

```
개발  http://127.0.0.1:8000/v1
운영  https://{API_DOMAIN}/v1
```

- 모든 경로는 `/v1` 아래다. `/healthz`만 예외(로드밸런서·배포 확인용).
- 버전은 경로에 둔다. 하위 호환이 깨지는 변경은 `/v2`를 새로 만들고, `/v1`은 6개월 유지한다.
- 매니페스트: `GET /openapi.json` (OpenAPI 3.1). 문서·클라이언트·테스트가 같은 원본을 본다.

### 2.2 인증과 역할

| 주체         | 자격 증명                                                                            | 발급                                                       | 권한 경계                                                                       |
| ---------- | -------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------------------------------------------------------- |
| **학부모**    | `Authorization: Bearer <Supabase access token>`                                  | Supabase Auth (Google·Apple) + `POST /guardian/register` | 자기 가족 전체                                                                    |
| **자녀 기기**  | 같은 헤더, **익명 세션** 토큰                                                              | Supabase 익명 로그인 → `POST /pairing/claim`                  | 연결된 자녀 1명의 미션·문항 읽기, 풀이·이벤트 쓰기                                              |
| **관리자**    | `haru_admin_session`, HttpOnly, Secure(기본 true), SameSite=Strict, Path=/v1/admin | `POST /admin/auth/login`                                 | 절대 8시간, 유휴 20분. 로그아웃 시 세션 만료 및 쿠키 삭제. 별도 레거시 HTML `/admin` 인증은 59개 API 범위 밖 |
| **기기 이벤트** | `Authorization` + `X-Device-Key` 헤더                                              | `POST /devices/event-key`                                | 1.0 위치 이벤트 한정                                                               |

규칙

1. **역할은 토큰이 아니라 DB 조회로 판정한다.** 학부모 여부는 `guardians` 행, 자녀 기기 여부는 `child_devices` 행으로 확인한다.
2. 익명 토큰으로 학부모 API를 부르면 `forbidden`. 학부모 토큰으로 자녀 API(`missions/*`, `wrong-notes/*`, `pairing/claim`)를 불러도 `forbidden`.
3. `register`를 아직 부르지 않은 로그인 사용자의 학부모 요청은 `forbidden`(가족 없음).
4. 익명인데 연결이 없거나 끊겼으면 `not_paired`.
5. **없는 자원과 남의 자원을 구분하지 않는다.** 둘 다 `404 not_found`다. 존재 여부 자체가 정보이기 때문이다.

### 2.3 요청·응답 형식

| 항목   | 규칙                                                             |
| ---- | -------------------------------------------------------------- |
| 본문   | JSON, 필드 이름 `snake_case`                                       |
| 인코딩  | UTF-8                                                          |
| 시각   | ISO 8601 **UTC** (`2026-11-24T08:12:00Z`) — 마이크로초가 붙을 수 있다     |
| 날짜   | `YYYY-MM-DD` (**한국 시간** 기준)                                    |
| 연월   | `YYYY-MM`                                                      |
| "오늘" | **서버가 Asia/Seoul로 계산한다.** 기기 시계를 신뢰하지 않는다                      |
| 빈 값  | `null`은 "값 없음". 빈 배열 `[]`과 구분한다                                |
| ID   | UUID v4. 앱이 만든 ID를 받는 API는 본문 `id`로 받는다(멱등성)                   |
| 오프라인 | 앱이 기기에서 먼저 처리하고 `client_completed_at`·`answered_at`으로 시각을 보정한다 |

### 2.4 오류 형식과 코드

```json
{ "error": { "code": "code_expired", "message": "인증번호가 만료됐습니다.", "details": {} } }
```

- `code`는 **고정 문자열**(프로그램이 판단), `message`는 사용자 노출용(바뀔 수 있음), `details`는 코드별 보조 정보다.
- 입력 형식 오류는 `validation_failed`이고 `details.errors`에 `[{loc, msg}]`가 온다. 경로 ID가 UUID가 아니어도 같다.
- 없는 경로·허용하지 않는 메서드도 `not_found`다.

| HTTP | code                                                                   | 뜻                         |
| ---- | ---------------------------------------------------------------------- | ------------------------- |
| 401  | `unauthorized`                                                         | 토큰 없음·만료                  |
| 403  | `forbidden`                                                            | 역할 불일치                    |
| 403  | `not_paired`                                                           | 자녀 기기 미연결                 |
| 403  | `phone_not_verified`                                                   | 휴대전화 인증 전                 |
| 404  | `not_found`                                                            | 없거나 남의 것(구분하지 않음)         |
| 409  | `conflict`                                                             | 이미 처리됨 (`details.status`) |
| 409  | `mission_expired`                                                      | 지난 날짜의 미션                 |
| 422  | `validation_failed`                                                    | 형식 오류                     |
| 422  | `consent_required`                                                     | 필수 동의 누락                  |
| 422  | `code_invalid` · `code_expired`                                        | 인증번호·연결 코드 오류             |
| 422  | `file_invalid` · `file_too_large`                                      | 업로드 실패 (`details.reason`) |
| 422  | `no_approved_items` · `items_in_active_curriculum` · `plan_infeasible` | 커리큘럼 생성 불가                |
| 422  | `mission_incomplete`                                                   | 미션 미완료                    |
| 429  | `rate_limited`                                                         | 호출 빈도 초과                  |
| 429  | `limit_exceeded`                                                       | 한도 초과(업로드 수·자녀 수·도우미 메시지) |
| 500  | `internal`                                                             | 서버 오류                     |
| 503  | `sms_failed`                                                           | 문자 발송 실패                  |
| 503  | `assistant_off` · `assistant_busy`                                     | 도우미 꺼짐 / 동시 호출 자리 없음      |
| 503  | `geo_failed`                                                           | 장소 검색 제공자 실패              |

> **총 24개.** 초기 기획 문서의 대문자 코드 표기는 2026-11-20에 소문자 구현 코드로 동기화했다. 이후 새 오류를 추가하면 이 표와 `server/app/errors.py`를 같은 변경에서 고친다.

### 2.5 멱등성 — 같은 요청이 두 번 와도 한 번만 처리한다

| 방식                      | 적용 API                                                                                                                            | 동작                                                  |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| **본문 UUID**             | `POST /missions/{id}/complete`, `/missions/sync`, `/assistant/messages`, `/events/place`, `/focus-sessions`, `/devices/event-key` | 앱이 만든 `id`가 이미 있으면 **저장된 결과를 그대로 반환**한다. 새로 만들지 않는다 |
| **`device_event_keys`** | `POST /events/place`                                                                                                              | `(device, key)` 유니크. 재전송은 200을 돌려주고 새 행을 만들지 않는다    |
| **상태 검사**               | `POST /sets/{id}/generate`, `/pairing/claim`, `/unlock-requests/{id}/decision`                                                    | 상태가 맞지 않으면 `conflict`(`details.status`)             |
| **부분 유니크 인덱스**          | `unlock_requests`                                                                                                                 | 자녀당 `pending` 1건만 허용(`unlock_requests_one_pending`) |
| **중복 완료 차단**            | 미션 완료                                                                                                                             | 같은 완료 요청 2회 → `attempts` 기록 1건 (인수조건 14)            |

- 공통 `Idempotency-Key` 헤더를 요구하지 않는다. 각 경로의 본문 UUID·기기 키·상태 검사·부분 유니크를 적용한다.
- AI 도우미 메시지는 같은 `id` 재전송 시 **모델을 다시 부르지 않고** 저장된 두 말을 돌려준다. 답이 아직 없으면 `conflict`.

### 2.6 한도와 레이트리밋

| 대상                     | 한도                                                              | 오류                                | 근거                                                               |
| ---------------------- | --------------------------------------------------------------- | --------------------------------- | ---------------------------------------------------------------- |
| 자료 업로드                 | 가정당 **하루 5건**                                                   | `limit_exceeded` (`max_uploads`)  | `LIMIT_UPLOADS_PER_DAY`                                          |
| 업로드 크기                 | PDF **30쪽**·**20MB**, 사진 **10장**(장당 5MB), CSV 50행               | `file_too_large`                  | `LIMIT_*`                                                        |
| AI 도우미                 | 가정당 **하루 30 메시지**                                               | `limit_exceeded` (`max_messages`) | `usage_counters.messages`                                        |
| 동시 생성                  | 워커 1개 큐(`FOR UPDATE SKIP LOCKED`)                               | 대기(202)                           | 기획서 5.2                                                          |
| 도우미 동시 호출              | `ASSIST_MAX_PARALLEL` 코드 기본 4(프로세스별), 슬롯 대기 최대 10초              | `503 assistant_busy`              | `app/config.py`, `services/assistant.py`. 운영 환경의 덮어쓰기·프로세스 수 미확인 |
| `/guardian/phone/send` | 같은 번호 재발송 30초, 번호당 5회/일·사용자당 10회/일. 인증번호 3분, 오입력 5회             | `rate_limited` 또는 코드 오류           | `services/guardians.py`                                          |
| `/geo/search`          | 사용자당 30회/분(프로세스 메모리)                                            | `rate_limited`                    | `services/geo.py`                                                |
| `/pairing/claim`       | 익명 사용자당 실패 5회/10분. 연결 코드 10분 유효                                 | `429 rate_limited` 또는 코드 오류       | `services/pairing.py`                                            |
| `/unlock-requests`     | 일일 5회, 거절 후 10분 대기, 대기 요청 30분                                   | `rate_limited` 또는 상태 충돌           | `services/block.py`                                              |
| `/admin/auth/login`    | IP당 10회/5분, 계정 실패 5회면 15분 잠금                                    | `rate_limited` 또는 로그인 오류          | `services/admin_auth.py`                                         |
| 관리자 목록                 | `limit` 기본 50, 최소 1. 감사·문항 목록 최대 200, 가족·가정별 사용량 최대 100         | `422 validation_failed`           | OpenAPI의 라우트별 쿼리 계약                                              |
| 그 밖의 경로                | 아래 §2.6.0의 공통 상한과 경로별 예외를 적용한다. 코드에 아직 없는 값은 `문서 결정·구현 예정`으로 표시 | `rate_limited` | `API 설계서 §2.6.0`, 운영 게이트웨이 제한 별도 확인 |

> 한도 확인과 한도 증가 사이에 모델 호출이 있어 **동시 요청 시 한도를 소폭 넘을 수 있다.** 이는 알려진 한계이며, 엄격한 상한이 필요하면 예약 방식으로 바꾼다.

### 2.6.0 경로별 호출·동시성 상한

값은 **MVP 기준 결정값**이다. `구현`은 기준 커밋 코드에 있음, `예정`은 이 문서의 결정을 코드·게이트웨이에 아직 반영하지 않았음을 뜻한다.

| 경로 묶음 | 호출 상한 | 동시성·대기 | 상태 |
|---|---|---|---|
| 인증된 읽기: `/assistant/thread`, `/jobs/{id}`, `/geo/search` 외 업무 조회 | 사용자당 **120회/분**, burst 30 | 요청 동시 8, 10초 큐 | 예정(현재 코드는 경로별 제한 없음) |
| 자녀 기기 읽기·쓰기: `/missions/today`, `/missions/ahead`, `/wrong-notes/practice` | 기기당 **120회/분**, burst 30 | 요청 동시 8 | 예정 |
| 학부모 쓰기: `/children`, `/pairing/codes`, `/curricula`, `/curricula/{id}/stop`, 세트 삭제 | 사용자당 **60회/분**, burst 20 | 요청 동시 4 | 예정 |
| 자녀 기기 이벤트: `/events/place`, `/devices/status`, `/places/changed` | 기기당 **120회/분** (`/devices/status`는 12회/시간, `/places/changed`는 6회/분) | 요청 동시 8 | 예정 |
| AI 도우미: `/assistant/messages` | 가정당 **30회/일**, 사용자당 120회/시간 | API 프로세스당 **4개 동시**, 슬롯 대기 **10초** 뒤 `assistant_busy` | 구현(동시 4·10초), 운영 프로세스 1개 기준 |
| 생성 접수: `/sets/{id}/generate` | 가정당 **5건/일**, 같은 세트 재시도 2회 | 워커 1개 큐, 202 대기 | 구현 |
| 위치 검색: `/geo/search` | 사용자당 **30회/분** | 요청 동시 4 | 구현(프로세스 메모리) |
| 문자: `/guardian/phone/send` | 번호당 **5회/일**, 사용자당 **10회/일**, 재발송 30초 | 동시 2 | 구현 |
| 연결: `/pairing/claim` | 익명 사용자당 실패 **5회/10분** | 동시 2 | 구현(프로세스 메모리) |
| 잠금 요청: `/unlock-requests` | 자녀당 **5회/일**, 거절 후 10분, 대기 30분 | 동시 2 | 구현 |
| 관리자 로그인: `/admin/auth/login` | IP당 **10회/5분**, 계정 5회 실패 시 15분 잠금 | 동시 2 | 구현 |
| 관리자 조회·변경 | 관리자당 **120회/분**(조회), **20회/분**(변경), burst 20 | 동시 4 | 예정 |
| 운영: `/healthz` | IP당 **60회/분** | 동시 4 | 예정 |
| 제안 수락: `/assistant/proposals/{id}/accept` | 사용자당 **30회/분** | 제안 행 잠금, 동시 1건 | 구현(행 잠금), 부하 시험 예정 |

예정 값의 담당은 윤기화, 적용 기한은 **11/22 코드 반영, 12/31 부하 시험**이다. 코드에 들어가기 전에는 운영 환경에서 이 수치가 적용된다고 주장하지 않는다.

### 2.6.1 실패 처리와 재시도

| 상황                  | 감지·현재 기준                                                                                                                                | 재시도·대체·종료                                                                                                                   |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| 도우미 포화·꺼짐           | 슬롯 10초 대기 뒤 `503 assistant_busy`, 비활성 상태는 `503 assistant_off`                                                                           | 포화 시 기준 서버는 부모 메시지 보류 행을 정리한다. 클라이언트 설계는 5초 후 같은 ID로 1회 재전송, 재실패 시 사용자에게 알리고 종료. 꺼짐은 자동 재시도 0회                              |
| 도우미 제공자 실패·출력 검증 실패 | 기본 모델 호출당 30초·그래프 기본 호출 최대 3회. 설정 시 상위 모델 호출당 60초·최대 2회, 조회 라운드 최대 2회. 그래프 예산과 별도로 어댑터 내부에서 기본 모델은 호출·스키마 각 1회, 상위 모델은 호출 0회·스키마 1회 재시도 | 상위 모델의 응답도 검증한다. 소진 시 `fallback:true` 고정 안내, 유효 제안 없음, 일일 메시지 수 차감 없음. 이용자가 검수 화면 또는 담당자에게 직접 넘기도록 안내하는 절차는 설계 항목이다         |
| 생성 작업 실패·중단         | `202` 접수 후 작업 상태 조회. 파이프라인 단계 경계 전체 예산 20분, 생성·검수 모델 호출 기본 180초, 입력 분석은 쪽·사진 수에 따라 별도. 워커의 `running` 25분 경과 감지, 단일 워커 재시작 시 고아 작업 복구    | 중단 작업은 실행 횟수 2회 미만일 때만 재큐잉(최대 1회 재실행), 소진 시 `failed/timeout`. 실패 상태의 재접수는 사용자 선택. 재실패나 근거·정답 검증 실패는 담당 검수자로 이관하며 자동 승인하지 않음 |
| 일일 사용량 초과           | 가정 키와 KST 날짜로 확인. 도우미 모델 호출 또는 생성 작업 큐 등록 전에 `429 limit_exceeded`                                                                       | 자동 재시도 0회. KST 다음 날 또는 정책에 따른 한도 변경 후 새 요청. 동시 요청의 한도 예약은 보강 계획                                                             |

동일 ID의 저장된 답은 재사용한다. 실행 중 같은 ID는 `409 conflict`이며, 기준 코드의 1분 고아 판정은 모델 처리시간과 겹칠 수 있어 재호출 위험이 남아 있다. 클라이언트는 409 시 새 ID로 자동 제출하지 않고 대화 조회로 상태를 확인한다. 두 최초 동시 요청과 고아 인계는 DB 슬롯 예약·유일성 및 실행 소유권 검증을 추가한 뒤 경합 시험으로 확인한다. 생성 접수는 세트 행 잠금과 상태 검사, 제안 수락은 제안 행 잠금과 상태 검사에 기반한다. 200 고정 안내는 `안내`, 다시 검증되어 실제 사용 가능한 결과를 저장한 경우에만 `복구 성공`으로 기록한다. 응답·작업 ID와 모델·문항 버전·지연·비용·검증 결과를 남기고, 비용을 실측하지 않았으면 미측정으로 표기한다.

### 2.7 성공 상태 코드

| 코드    | 쓰임                  | 예                                                                                          |
| ----- | ------------------- | ------------------------------------------------------------------------------------------ |
| `200` | 조회·갱신·이미 존재하는 결과 반환 | `GET /jobs/{id}`, `POST /missions/today`                                                   |
| `201` | 새 자원 생성             | `POST /children`, `/curricula`, `/pairing/codes`                                           |
| `202` | 비동기 접수(워커가 처리)      | `POST /sets/{id}/generate`                                                                 |
| `204` | 본문 없는 성공            | `DELETE /children/{id}` · `/sets/{id}` · `/assistant/threads/{id}`, `POST /focus-sessions` |

### 2.8 목록·페이지네이션

- **관리자 API만** `limit`·`offset`을 쓴다(`/admin/bank/items`, `/admin/families`, `/admin/usage/families`). 감사 로그는 `limit`·`before_id`로 조회한다. 실제 제한은 §2.6과 OpenAPI 참조.
- 앱 API는 목록이 **구조적으로 작게 유지**되도록 설계했다(오늘 미션·자녀 수·주간 리포트). 페이지네이션 대신 **상한과 필터**를 둔다.
- 정렬은 서버가 정한다(최신순 또는 계획일 순). 클라이언트는 정렬을 요청하지 않는다.

---

## 3. 엔드포인트 인벤토리

[OpenAPI 스냅샷](./openapi-53cdf85.json)에서 추출한 **58 path · 59 operation**이다. 아래 목록의 API는 `구현(라우트 등록)` 상태이며 동작·운영 QA 통과는 §7에서 별도로 판정한다. [동작별 상세 계약](./API%20상세%20계약%2053cdf85.md)에는 요청 필드의 타입·필수·null·제한, 성공/오류 계약 출처와 기준 핸들러를 붙였다. [59개 응답 계약](./API%20응답%20계약%20v1.2.md)에는 성공 필드·null 허용·주요 오류·상태를 한 표로 정리했다. 인증은 §2.2를 참조한다.

> 59개 operation의 개별 상태는 §3의 그룹 상태와 [API 응답 계약](./API%20응답%20계약%20v1.2.md)의 행 상태를 함께 본다. `리뷰` 수락 재검사, 부하 시험, 공통 correlation ID는 아직 완료가 아니다.

| API 목록 그룹            | 상태                   | 요구사항                          | 관련 DB·서비스 경계                                                                                    |
| -------------------- | -------------------- | ----------------------------- | ----------------------------------------------------------------------------------------------- |
| 1. 인증·가족             | 구현                   | FR-ACC-001                    | `guardians`, `families`, `phone_verifications`                                                  |
| 2. 자녀 관리             | 구현                   | FR-ACC-002, FR-LOCK-001       | `children`, `consents`, `block_policies`                                                        |
| 3. 기기 연결·상태          | 구현                   | FR-PAIR-001, FR-LOCK-001      | `pairing_codes`, `child_devices`, `device_event_keys`, `device_status`                          |
| 4. 학습 자료·문항 생성       | 구현                   | FR-SET-001~003, FR-AI-001~005 | `study_sets`, `generation_jobs`, `items`, `usage_counters`                                      |
| 5. 커리큘럼·미션·오답        | 구현                   | FR-CUR-001, FR-QUIZ-001       | `curricula`, `curriculum_items`, `daily_missions`, `mission_items`, `attempts`, `review_states` |
| 6. AI 도우미            | 구현(7개 라우트), 웹 배포 예정  | FR-WEB-002, FR-AI-004~005     | `assistant_threads`, `assistant_messages`, `assistant_proposals`, `usage_counters`              |
| 7. 가족 약속·잠금·집중       | 구현, 실기기·배포 상태 별도     | FR-LOCK-001~004               | `consents`, `agreement_status`(뷰), `block_policies`, `focus_sessions`, `unlock_requests`        |
| 8. 장소·위치             | 구현                   | FR-LOC-001~002                | `places`, `place_events`, `device_event_keys`, 외부 주소 검색                                         |
| 9. 알림                | 구현                   | FR-LOCK-003, FR-LOC-001       | `push_tokens`, `child_devices`                                                                  |
| 10. 관리자              | 구현(라우트), 일부 정책 시험 예정 | FR-BANK-001~002               | `admin_users`, `admin_sessions`, `audit_logs`, `items`, `bank_sources`, `usage_counters`        |
| 11. 운영               | 구현                   | 가용성 검증                        | `/healthz`의 응답은 앱 프로세스 상태. DB 확인 없음                                                             |
| 팀 자료 수락 실행·상점·포인트·장비 | 이번 범위 제외             | 후속 범위 검토                      | 신규 REST·DB 변경 없음                                                                                |

#### 1. 인증·가족 — 3개

| #   | 메서드    | 경로                          | 요청                | 성공 응답 | 동작        |
| --- | ------ | --------------------------- | ----------------- | ----- | --------- |
| 1   | `POST` | `/v1/guardian/phone/send`   | `PhoneSendBody`   | 204   | 인증번호 발송   |
| 2   | `POST` | `/v1/guardian/phone/verify` | `PhoneVerifyBody` | 200   | 인증번호 확인   |
| 3   | `POST` | `/v1/guardian/register`     | —                 | 200   | 보호자·가족 생성 |

#### 2. 자녀 관리 — 4개

| #   | 메서드      | 경로                                       | 요청           | 성공 응답 | 동작            |
| --- | -------- | ---------------------------------------- | ------------ | ----- | ------------- |
| 1   | `POST`   | `/v1/children`                           | `ChildBody`  | 201   | 자녀 등록 + 동의 저장 |
| 2   | `DELETE` | `/v1/children/{child_id}`                | —            | 204   | 자녀 삭제(연결 해제)  |
| 3   | `POST`   | `/v1/children/{child_id}/block-policy`   | `PolicyBody` | 200   | 잠금 정책 저장      |
| 4   | `POST`   | `/v1/children/{child_id}/block-reselect` | —            | 200   | 차단 앱 재선택 열기   |

#### 3. 기기 연결·상태 — 5개

| #   | 메서드    | 경로                      | 요청                 | 성공 응답 | 동작              |
| --- | ------ | ----------------------- | ------------------ | ----- | --------------- |
| 1   | `POST` | `/v1/devices/event-key` | —                  | 200   | 기기 이벤트 키 발급     |
| 2   | `POST` | `/v1/devices/status`    | `DeviceStatusBody` | 204   | 기기 권한·상태 보고     |
| 3   | `POST` | `/v1/pairing/claim`     | `ClaimBody`        | 200   | 연결 코드 사용(기기 연결) |
| 4   | `POST` | `/v1/pairing/codes`     | `PairingCodeBody`  | 201   | 연결 코드 발급        |
| 5   | `POST` | `/v1/pairing/release`   | —                  | 204   | 기기 연결 해제        |

#### 4. 학습 자료·문항 생성 — 3개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `GET` | `/v1/jobs/{job_id}` | — | 200 | 생성 작업 진행률 |
| 2 | `DELETE` | `/v1/sets/{set_id}` | — | 204 | 세트 삭제 |
| 3 | `POST` | `/v1/sets/{set_id}/generate` | — | 202 | 문항 생성 접수 |

#### 5. 커리큘럼·미션·오답 — 7개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/curricula` | `CurriculumBody` · 쿼리 `dry_run`(boolean) | 201 | 커리큘럼 생성(+미리보기) |
| 2 | `POST` | `/v1/curricula/{curriculum_id}/stop` | — | 204 | 커리큘럼 중지 |
| 3 | `POST` | `/v1/missions/ahead` | — | 200 | 다음 날 미션 미리 받기 |
| 4 | `POST` | `/v1/missions/sync` | `SyncBody` | 200 | 오프라인 기록 동기화 |
| 5 | `POST` | `/v1/missions/today` | — | 200 | 오늘 미션 조회·생성 |
| 6 | `POST` | `/v1/missions/{mission_id}/complete` | — | 200 | 미션 완료·채점 |
| 7 | `POST` | `/v1/wrong-notes/practice` | — | 200 | 오답 재풀이 세트 |

#### 6. AI 도우미 — 7개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/assistant/messages` | `AssistantMessageBody` | 200 | 도우미 메시지 전송 |
| 2 | `POST` | `/v1/assistant/messages/{message_id}/feedback` | `AssistantFeedbackBody` | 204 | 답변 신고 |
| 3 | `POST` | `/v1/assistant/proposals/{proposal_id}/accept` | — | 200 | 제안 수락 |
| 4 | `POST` | `/v1/assistant/proposals/{proposal_id}/dismiss` | — | 204 | 제안 거절 |
| 5 | `GET` | `/v1/assistant/thread` | — | 200 | 도우미 대화 조회 |
| 6 | `POST` | `/v1/assistant/threads` | — | 201 | 새 대화 시작 |
| 7 | `DELETE` | `/v1/assistant/threads/{thread_id}` | — | 204 | 대화 삭제 |

#### 7. 가족 약속·잠금·집중 — 7개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/agreements` | `AgreeBody` | 200 | 가족 약속 동의 |
| 2 | `POST` | `/v1/agreements/confirm` | `ConfirmBody` | 200 | 약속 확인 |
| 3 | `POST` | `/v1/agreements/revoke` | `RevokeBody` | 200 | 동의 철회 |
| 4 | `POST` | `/v1/focus-sessions` | `FocusBody` | 204 | 집중 모드 기록 |
| 5 | `POST` | `/v1/unlock-grants` | `GrantBody` | 201 | 1회성 해제 부여 |
| 6 | `POST` | `/v1/unlock-requests` | — | 201 | 잠금 해제 요청 |
| 7 | `POST` | `/v1/unlock-requests/{request_id}/decision` | `DecisionBody` | 200 | 해제 승인·거절 |

#### 8. 장소·위치 — 3개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/events/place` | `PlaceEventBody` · 헤더 `x-device-key` | 200 | 장소 이벤트 수신 |
| 2 | `GET` | `/v1/geo/search` | 쿼리 `q`(string) | 200 | 장소 검색 |
| 3 | `POST` | `/v1/places/changed` | — | 204 | 등록 장소 변경 통지 |

#### 9. 알림 — 2개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/push-tokens` | `PushTokenBody` | 204 | 푸시 토큰 등록 |
| 2 | `POST` | `/v1/push-tokens/remove` | `RemoveTokenBody` | 204 | 푸시 토큰 삭제 |

#### 10. 관리자 — 17개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `GET` | `/v1/admin/audit-logs` | 쿼리 `action`(string) `admin_email`(string) `target_type`(string) `target_id`(string) `since`(string) `until`(string) `limit`(integer) `before_id`(string) | 200 | 감사 로그 조회 |
| 2 | `GET` | `/v1/admin/audit-logs/location-disclosures` | 쿼리 `days`(integer) | 200 | 위치 제공 기록 조회 |
| 3 | `POST` | `/v1/admin/auth/login` | `LoginBody` | 200 | 관리자 로그인 |
| 4 | `POST` | `/v1/admin/auth/logout` | — | 204 | 관리자 로그아웃 |
| 5 | `GET` | `/v1/admin/bank/items` | 쿼리 `set_id`(string) `type`(string) `unit`(string) `q`(string) `limit`(integer) `offset`(integer) | 200 | 문제은행 문항 목록 |
| 6 | `GET` | `/v1/admin/bank/items/{item_id}` | — | 200 | 문항 상세 |
| 7 | `GET` | `/v1/admin/bank/packs` | — | 200 | 기본팩 목록 |
| 8 | `PUT` | `/v1/admin/bank/packs/{pack_id}/source` | `PackSourceBody` | 200 | 팩 출처 연결 |
| 9 | `POST` | `/v1/admin/bank/packs/{pack_id}/status` | `PackStatusBody` | 200 | 팩 상태 변경 |
| 10 | `GET` | `/v1/admin/bank/sources` | — | 200 | 출처 목록 |
| 11 | `POST` | `/v1/admin/bank/sources` | `SourceCreate` | 201 | 출처 등록 |
| 12 | `PATCH` | `/v1/admin/bank/sources/{source_id}` | `SourcePatch` | 200 | 출처 수정 |
| 13 | `GET` | `/v1/admin/families` | 쿼리 `q`(string) `limit`(integer) `offset`(integer) | 200 | 가족 목록 |
| 14 | `GET` | `/v1/admin/families/{family_id}` | — | 200 | 가족 상세 |
| 15 | `GET` | `/v1/admin/me` | — | 200 | 관리자 세션 확인 |
| 16 | `GET` | `/v1/admin/usage/families` | 쿼리 `month`(string) `limit`(integer) `offset`(integer) | 200 | 가정별 사용량 |
| 17 | `GET` | `/v1/admin/usage/summary` | 쿼리 `days`(integer) | 200 | 사용량 요약 |

#### 11. 운영 — 1개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `GET` | `/healthz` | — | 200 | 서버 상태 확인 |

---

## 4. 대표 계약

전체 25개 상세 계약은 `docs/mvp/03-api.md`에 있고, 여기서는 **화면이 직접 의존하는 6개**만 요약한다.

### 4.1 `POST /v1/children` — 자녀 등록 + 동의

| 항목 | 내용 |
| --- | --- |
| 호출 | 학부모 (JWT) |
| 화면 | P-03 자녀 등록·동의 |
| 멱등성 | 상태 검사 없음. 중복 등록은 자녀 수 한도로 막는다 |

```json
// 요청
{ "nickname": "별명", "birth_ym": "2017-03", "grade": 3,
  "consents": [{ "type": "privacy", "doc_version": "2026-11-01" },
               { "type": "ai_notice", "doc_version": "2026-11-01" }] }

// 201
{ "child_id": "…", "consent_ids": ["…", "…"], "sms_sent": true }
```

- `nickname`·`birth_ym`·`grade`·`consents` 모두 필수. **이메일·전화번호 필드는 존재하지 않는다**(설계상 금지).
- 동의는 `consents`에 **주체·문서 버전·시각**으로 저장하고, 같은 시각에 부모 휴대전화로 확인 문자를 보낸다(개인정보 보호법 시행령 제17조의2 근거).
- 오류: `validation_failed`, `consent_required`, `phone_not_verified`, `forbidden`, `limit_exceeded`(가정당 자녀 수).

### 4.2 `POST /v1/sets/{set_id}/generate` → `GET /v1/jobs/{job_id}` — 문항 생성

| 항목 | 내용 |
| --- | --- |
| 호출 | 학부모 |
| 화면 | P-05a 업로드 → P-05b 진행률 |
| 선행 | 앱이 `study_sets` 행을 만들고 Storage `sources/{family_id}/{set_id}/`에 파일을 올린 뒤 호출한다 |
| 멱등성 | 세트 상태가 `uploaded`·`failed`일 때만 접수. `generating`이면 `conflict` |

```json
// 요청: 본문 없음
// 202
{ "job_id": "…" }
```

```json
// GET /v1/jobs/{job_id} → 200
{ "id": "…", "set_id": "…", "status": "running", "step": "verify",
  "progress": { "concepts": 30, "generated": 90, "passed": 0, "dropped": 0 },
  "error_code": null }
```

| 필드 | 값 |
| --- | --- |
| `status` | `queued` · `running` · `succeeded` · `failed` |
| `step` | `parse` · `plan` · `generate` · `verify` · `save` |
| `progress` | 개념 경로 `{concepts, generated, passed, dropped}` / 문제지 경로 `{questions, skipped, generated, passed, flagged, dropped}` |
| `error_code` | 실패 시 원인 코드. 사용자 문구는 `04-ai-pipeline.md` 7장 |

- 서버가 접수 전에 검사한다: 가족 소유·상태, PDF 쪽수/용량, CSV 헤더·행수, 사진 순서(`01.jpg`부터 연속)·형식(JPEG `FF D8`), 하루 5건 한도.
- **검사에서 떨어진 요청은 한도를 쓰지 않는다.**
- 워커가 작업을 집은 뒤 **같은 검사를 다시** 한다. 접수 후 원본이 바뀌면 `unreadable`로 끝난다.
- 앱은 Realtime 구독 + **5초 폴링**을 함께 쓴다(놓친 변경 대비).

### 4.3 `POST /v1/curricula` — 커리큘럼 배분

| 항목 | 내용 |
| --- | --- |
| 호출 | 학부모 |
| 화면 | P-07 커리큘럼 설정 (미리보기 `?dry_run=true`) |
| 배분 | **규칙 기반** — 총 문항 ÷ 학습일, 요일 가중치. LLM은 순서·묶음만 제안 |

```json
// 요청
{ "child_id": "…", "title": "과학 3단원", "set_ids": ["…"],
  "start_date": "2026-11-24", "end_date": "2026-12-07",
  "daily_new_cap": 12, "rest_weekdays": [6] }

// 201 (dry_run=true 이면 200, curriculum_id=null)
{ "curriculum_id": "…", "total_items": 96, "study_days": 12, "per_day": 8 }
```

- `rest_weekdays`: 0=월 … 6=일.
- 오류: `no_approved_items`, `items_in_active_curriculum`(다른 활성 커리큘럼이 선점), `plan_infeasible`(`details.needed_per_day`), `not_found`(자녀·세트가 내 가족/기본팩이 아님), `validation_failed`(기간·학습일 0).
- `set_ids`는 비어 있지 않고 겹치지 않아야 하며, `daily_new_cap`은 1~50, `title`은 1~60자.

### 4.4 `POST /v1/missions/today` · `/{id}/complete` · `/missions/sync` — 자녀 학습

| 항목 | 내용 |
| --- | --- |
| 호출 | 자녀 기기 |
| 화면 | C-03 오늘 미션 · C-04 채점 · (오프라인) 동기화 |
| 멱등성 | `today`는 여러 번 불러도 같은 미션. 완료는 2회 요청해도 기록 1건 |

```json
// POST /missions/today → 200
{ "missions": [
  { "curriculum": { "id": "…", "title": "기본 영단어 100" },
    "mission": { "id": "…", "mission_date": "2026-11-24", "status": "open",
      "items": [ { "item_id": "…", "kind": "review", "seq": 1,
                   "type": "multiple_choice", "prompt": "…", "answer": "…",
                   "accepted_answers": [], "choices": ["…"], "explanation": "…", "unit": "…" } ] } } ] }
```

```json
// POST /missions/{id}/complete → 200
{ "mission_id": "…", "correct": 20, "total": 20,
  "unlock_granted": true, "next_review_at": "2026-11-25" }
```

- 완료 조건은 **전부 풀이 + 틀린 문항 1회 재풀이**. 미완료면 `422 mission_incomplete`.
- 오프라인에서 만든 기록은 `POST /missions/sync`로 **한 번만** 올린다(요청 본문 `days[]`).
- 틀린 문항은 간격 반복 5단계(1·2·4·7·15일)의 **1단계로 복귀**한다.

### 4.5 `POST /v1/assistant/messages` — AI 도우미(대화형 웹·앱)

| 항목 | 내용 |
| --- | --- |
| 호출 | 학부모 |
| 화면 | CW-01~03, 부모 앱 AI 도우미 |
| 멱등성 | 앱이 만든 `id`. 같은 `id` 재전송 시 모델을 다시 부르지 않는다 |

```json
// 요청
{ "id": "앱이 만든 UUID", "thread_id": "…", "text": "이번 주 뭘 공부시킬까요?", "child_id": null }

// 200
{ "messages": [ { "role": "user", "…": "…" }, { "role": "assistant", "…": "…" } ],
  "messages_left_today": 26 }
```

- 도우미가 꺼져 있으면 `assistant_off`. `text`는 1~500자. 남의 대화·자녀면 `not_found`.
- 동시 호출 자리가 10초 내 나지 않으면 `assistant_busy`(부모의 말은 지워지므로 같은 `id`로 재전송).
- 12/24 웹의 인수 기준은 근거 또는 근거 부족 고지다. 이 기준의 메시지 REST 응답이 별도 `page`·`quote`를 보장한다고 보지 않는다. 모델에 보내는 자녀 별칭과 개인정보 처리도 웹 통합 시험에서 확인한다.
- 파이프라인 실패 시 `fallback: true`로 정해진 문장을 돌려주고, **메시지 한도를 차감하지 않는다.**

### 4.6 `POST /v1/events/place` — 장소 이벤트(1.0)

| 항목 | 내용 |
| --- | --- |
| 호출 | 자녀 기기 (`Authorization` + `X-Device-Key`) |
| 화면 | C-10 알림 기록 · P-12 도착 기록 |
| 멱등성 | 본문 `id` + `device_event_keys` |

```json
// 요청
{ "id": "…", "place_id": "…", "kind": "enter", "source": "geofence",
  "accuracy_m": 35, "occurred_at": "2026-11-24T09:05:00Z" }
```

- 서버가 저장하는 것은 **장소 ID·종류·시각·정확도**뿐이다. **좌표 원본과 이동 경로는 받지 않는다.**
- 판정(반경 10분 체류)은 **기기에서** 끝내고, 서버는 결과만 받는다.
- 알림을 보낼 때마다 `location_disclosure_logs`에 제공 사실을 남긴다(위치정보법 제3자 제공 통지 근거).

---

## 5. 화면 ↔ API 매핑

### 5.1 학습 MVP (13개 논리 화면 / 15개 상태)

| 화면 | 기능 ID | 호출 API |
| --- | --- | --- |
| P-01 로그인 | FR-ACC-001 | Supabase Auth (직접) → `POST /guardian/register` |
| P-02a 문자 인증 | FR-COM-001 | `POST /guardian/phone/send` → `/verify` |
| P-03 자녀 등록 | FR-ACC-002 | `POST /children` |
| P-04 연결 코드 | FR-PAIR-001 | `POST /pairing/codes` |
| P-05a 업로드 | FR-SET-001·DOC-001 | Storage 직접 업로드 → `POST /sets/{id}/generate` |
| P-05b 진행률 | FR-SET-002 | `GET /jobs/{id}` + Realtime |
| P-06 검수 | FR-SET-003 | Supabase 직접 읽기/수정(`items` 승인·수정·삭제) |
| P-07 커리큘럼 | FR-CUR-001 | `POST /curricula?dry_run=true` → `POST /curricula` |
| P-02b 홈(오늘 현황) | FR-PAR-001 | Supabase 뷰 `mission_progress` 직접 조회 |
| P-08 정오답 | FR-QUIZ-002 | Supabase 직접 조회(`attempts`·뷰 `wrong_note_items`) + `POST /wrong-notes/practice` |
| C-01 시작 | FR-ACC-001 | Supabase 익명 로그인 |
| C-02 연결 코드 입력 | FR-PAIR-001 | `POST /pairing/claim` |
| C-03 오늘 미션 | FR-CUR-001·QUIZ-001 | `POST /missions/today` |
| C-04 채점·해설 | FR-QUIZ-001 | `POST /missions/{id}/complete` · `/missions/sync` |
| C-05 오답 노트 | FR-QUIZ-002 | 뷰 `wrong_note_items` + `POST /wrong-notes/practice` |

> **설계 원칙:** RLS로 막을 수 있는 읽기·쓰기는 **앱이 Supabase에 직접** 한다. 비밀 키·LLM·외부 발송이 필요한 일만 FastAPI를 거친다. 그래서 위 표에 "직접"이 섞여 있다.

### 5.2 1.0 · 관리자 · 웹

| 화면 | 호출 API |
| --- | --- |
| P-09 잠금 정책 / C-07 가족 약속 | `POST /agreements` · `/confirm` · `/revoke`, `POST /children/{id}/block-policy` |
| P-10 해제 승인 / C-09 요청 | `POST /unlock-requests` → `POST /unlock-requests/{id}/decision` |
| C-06 차단·C-08 퀴즈 게이트 | 기기 내부 판정(서버 호출 없음) + `POST /unlock-grants` |
| C-16~18 집중 모드 | `POST /focus-sessions` |
| P-11 장소 설정 | `GET /geo/search` → Supabase 직접 INSERT(`places`) → `POST /places/changed`(기기 통지) |
| P-12 도착 기록 / C-10 | 자녀: `POST /events/place`·`/devices/status` / 부모: Supabase 직접 조회 |
| 알림 등록 | `POST /push-tokens` · `/push-tokens/remove` |
| A-01~04 관리자 | `/admin/auth/*`, `/admin/bank/*`, `/admin/families`, `/admin/usage/*`, `/admin/audit-logs*` |
| W-01~03 열람형 웹 | Supabase 직접(SELECT 뷰 `mission_progress`·`curriculum_progress`) |
| CW-01~03 대화형 웹 | `GET /assistant/thread`, `POST /assistant/threads`, `/assistant/messages`, `/assistant/proposals/*` |

---

## 6. 역할·권한 매트릭스

| 영역 | 학부모 | 자녀 기기 | 관리자 | 비인증 |
| --- | --- | --- | --- | --- |
| 가입·문자 인증 | ✔ | — (익명) | — | ✔ (인증 전 단계) |
| 자녀 등록·삭제 | ✔ 자기 가족 | ✘ | 읽기(감사 기록) | ✘ |
| 연결 코드 발급 | ✔ | ✘ | ✘ | ✘ |
| 연결 코드 사용 | ✘ | ✔ | ✘ | ✘ |
| 자료 업로드·생성 | ✔ 자기 가족 | ✘ | ✘ | ✘ |
| 문항 검수·승인 | ✔ | ✘ | 문제은행만 | ✘ |
| 커리큘럼 생성·중지 | ✔ (14세 미만 자녀) | ✘ (14세 이상은 본인) | ✘ | ✘ |
| 오늘 미션 조회·완료 | 읽기만 | ✔ 자기 것 | ✘ | ✘ |
| 오답 노트 | 읽기 | ✔ | ✘ | ✘ |
| 가족 약속·잠금 정책 | 설정 | 동의·조회 | ✘ | ✘ |
| 해제 요청 | 승인 | 요청 | ✘ | ✘ |
| 장소·도착 기록 | 설정·열람 | 이벤트 전송 | ✘ | ✘ |
| AI 도우미 | ✔ 자기 가족 | ✘ | ✘ | ✘ (익명 샘플 세션은 별도) |
| 문제은행·사용량·감사 | ✘ | ✘ | ✔ | ✘ |
| 타 가족 데이터 | ✘ (RLS) | ✘ (RLS) | 부분(감사 기록 필수) | ✘ |

> 이 표는 **API 검사 + DB RLS 두 겹**으로 강제된다. API만 막고 RLS를 열어두면 앱이 Supabase에 직접 붙는 순간 뚫리기 때문에, `supabase/tests/01_rls_guardian.test.sql`·`02_rls_device.test.sql`로 교차 접근 0건을 검증한다.

---

## 7. 검증

| 검증 | 방법 | 결과 |
| --- | --- | --- |
| 계약 추출 | `GET /openapi.json` | 58 path · 59 operation · 31 schema **일치** |
| 오류 형식 | 무인증 `POST /v1/missions/today` | `401 {"error":{"code":"unauthorized",…}}` **일치** |
| 없는 경로 | `GET /v1/nope` | `404 {"error":{"code":"not_found",…}}` **일치** |
| 권한 격리 | pgTAP `01`·`02` | 타 가족 접근 **0건** |
| 서버 테스트 | `uv run pytest` (96개 테스트 파일) | 2026-11-16 기준 **649건 통과** (기존 기록 인용) |
| 서버 테스트(최신 로컬 확인) | `uv run --no-sync pytest tests/unit tests/assistant -q` | **685 통과, 1 실패**. 실패 1건은 Windows 경로 구분자 기대값 문제로 확인. 전체 API·pgTAP 재실행은 아님 |
| 앱 정적 검사 | `flutter analyze` · `flutter test` | CI 기준. 2026-11-20 로컬에서 학부모 619·자녀 733·공용 400(+4 skip)·관리자 15 통과 확인 |
| **재확인(11/18 pull 후)** | `2e25cb0` → `53cdf85` 13커밋 반영 후 재추출 | **58 path · 59 operation · 31 schema 동일**. `server/app/api/*`·`errors.py` 변경 없음 |
| 응답 계약 | [59개 응답 계약 v1.2](./API%20응답%20계약%20v1.2.md) | 59개 성공 필드·주요 오류·상태 정리. OpenAPI 성공 스키마는 여전히 `schema:{}` |
| **미검증** | 4xx·429 실제 부하 측정, 동시 30건 경합, `review` 수락 상태 재검사 | **미실시** — 12/31 통합 테스트에서 보강 |

> **한계를 먼저 적는다.** `limit_exceeded`·`assistant_busy`는 코드와 단위 테스트로만 확인했고, 실제 동시 부하로 재현하지 않았다. 제출 문서에 "검증 완료"로 쓰지 않는다.

OpenAPI는 기준 커밋에서 재추출했다. 11/16의 `649건 통과`·CI 통과는 기존 기록의 인용이며 원본 실행 로그는 확인하지 못했다. 도우미 30건 동시 요청과 한도 경합, 폴백의 실제 복구 재시험은 **미실시**다. 잠금 실기기 검증 결과, iOS 배포 권한 승인, 스토어 제출 상태를 각각 기록한다.

| 검증 항목 | 상태와 근거 |
| --- | --- |
| 계약 재추출 | `53cdf85` 앱에서 OpenAPI를 생성해 58경로·59동작·31스키마 및 인벤토리 59건 대조 완료. 외부 API·모델·DB 호출 없이 추출 |
| 서버·앱 시험 원본 | 기존 시험은 인용. 실제 실행 조회는 [구현 저장소 Actions](https://github.com/SHS5029/Daechi_team4_sorry/actions)에 접근한 뒤 커밋과 실행 번호를 연결한다 |
| 잠금 실기기 | [11/18 2차 체크리스트](./대치루틴%20앱%20잠금%20확인%20-%20팀원용%202차%20%2811-18%29.md) 기준 항목 존재. 2차 전체 최종 통과 근거는 미확인 |
| iOS 배포 권한 | 기술 명세서 기준 신청 후 회신 대기. 승인 결과는 이 문서에서 확인되지 않음 |
| 스토어 제출 | 새 제출일 재확정 대기. 제출 영수증·심사 결과는 확인되지 않음 |

---

## 8. 계약 기준과 변경 이력

| 확정한 문서 기준 | 확인·후속 담당 | 기한 | 반영 위치 |
| --- | --- | --- | --- |
| 대화 REST는 `/v1/assistant/*`의 threads·messages·proposals. 세트 선택·근거 표시는 12/24 웹 인수 시험에서 확인 | 심환석·차해든 | 11/20 문서, 12/24 웹 | 기획 §4.7·7.3, 기능 FR-WEB-002, 기술 §4.7·6, WBS 3.7.9 |
| 소문자 오류 24종, 공통 오류 봉투, 입력 검증 422 | 윤기화 | 11/20 | API §2.4, 기능 FR-SET-001, 기술 §6.2 |
| REST 기준은 58경로·59동작·31스키마, 기획·기술은 요약 | 윤기화 | 11/20 | API §3, 기획 §7.3, 기술 §6.1 |
| 멱등성은 본문 UUID·기기 키·상태 검사·부분 유니크 등 경로별 방식 | 윤기화 | 11/20 문서, 12/31 경합 검증 | API §2.5, 기획 §7.3, 기술 §4·6, 기능 DB 매핑, DB §13 |
| 생성은 `queued/running/succeeded/failed`와 `step`. 자녀 인증은 익명 토큰과 활성 기기 검사. 관리자 목록은 지정 경로에서 페이지네이션 | 윤기화 | 11/20 | API §2·4, 기술 §6 |
| 문항 근거는 AI 생성 문항에 적용. 기본팩은 출처 레지스트리로 추적 | 심환석·윤기화 | 11/20 | 기능 FR-AI-001, DB §13 |
| 대치루틴 표시명과 12/9 전후 출시 목표. 이전 12/13 동결·12/18 스토어 제출·1/3 공개는 이력으로 보존. 교육 마감은 유지 | 오병훈·최강현 | 11/20 제출 전 새 동결·스토어 제출일 확인 목표 | 여섯 문서의 일정, 기획 §9, WBS §5 |
| 팀 자료 수락 실행·보상은 이번 출시·필수 교육 범위 제외. 기본 학습과 필수 제출 우선 | 오병훈·심환석 | 11/20 문서, 후속 검토일 미정 | API §1.4·3, 기획·기능·기술·WBS |
| 미선언 응답 필드·예시와 원본 검증 로그 보강 | 윤기화 | 11/20 문서, 12/31 통합 시험 | [API 응답 계약 v1.2](./API%20응답%20계약%20v1.2.md), API §7 |
| 59개 동작의 성공 필드·정상/오류 예시·상태를 응답 계약에 색인 | 윤기화 | **11/20 완료** | API 응답 계약 §1~§9 |
| `review` 수락 시 대상 세트 소유권·`items.pending > 0` 재검사와 경합 시험 | 심환석 | **11/22 구현, 12/31 경합 시험** | API §1.5.1, 3.7.9, WBS 7.4.1b |
| 메시지·제안·작업 공통 correlation ID를 구조화 로그에 연결 | 심환석·윤기화 | **11/22 로그 계약, 12/31 통합 시험** | API §1.5, 3.7.9, WBS 7.4.1c |
| API·화면 흐름·기능·기술·WBS의 경로·오류·앱명·버전 참조 통일 | 오병훈 | **11/20 완료** | 여섯 문서의 변경 이력 |
| 경로별 호출·동시성 상한을 코드/게이트웨이에 반영 | 윤기화·최강현 | **11/22 코드, 12/31 부하 시험** | API §2.6.0, WBS 7.4.1d |

**v1.1 · 2026-11-19.** 도구 7종, 수락·추적, 분류와 운영 제한, OpenAPI를 추가했다. 이 REST 스냅샷 비교 기준은 `53cdf857102746081d6cbb12eed5a758759f0de4`이다. 로컬 HEAD `9af07b8`에는 이후 서버 변경이 있으므로 최신 배포 결과는 배포 커밋에서 재검증한다. DB 제출 기준 재현은 `9af07b8`에서 별도 완료했다.

**v1.2 · 2026-11-20.** 59개 동작의 응답 계약을 별도 문서로 만들고, `review` 수락 재검사·공통 correlation ID·경로별 한도·문서 동기화 작업을 담당·기한과 함께 §8에 추가했다. `review` 재검사는 문서상 후속 작업이며, 현재 구현 완료로 표시하지 않는다. OpenAPI 성공 응답 스키마 공백은 응답 계약 문서로 보완하고, FastAPI 응답 모델 선언은 12/31 전 보강 항목으로 남긴다.

**C-13 변경.** 자녀 앱의 두 자리 수 곱셈 확인 제거. 가족 약속 동의·철회와 `/v1/agreements*` 계약 유지.

---

## 9. 부록

### 부록 A. 요청 스키마 (31개)

| 스키마 | 필드 | 타입 | 필수 |
| --- | --- | --- | --- |
| `AgreeBody` | `types` | array | ✔ |
|  | `doc_version` | string | ✔ |
| `AssistantFeedbackBody` | `feedback` | string | ✔ |
| `AssistantMessageBody` | `id` | string | ✔ |
|  | `thread_id` | string | ✔ |
|  | `text` | string | ✔ |
|  | `child_id` | string / null |  |
| `ChildBody` | `nickname` | string | ✔ |
|  | `birth_ym` | string | ✔ |
|  | `grade` | integer | ✔ |
|  | `consents` | array | ✔ |
| `ClaimBody` | `code` |  | ✔ |
|  | `platform` | string | ✔ |
|  | `device_name` | string / null |  |
| `ConfirmBody` | `child_id` | string | ✔ |
|  | `types` | array | ✔ |
| `ConsentIn` | `type` | string | ✔ |
|  | `doc_version` | string | ✔ |
| `CurriculumBody` | `child_id` | string | ✔ |
|  | `title` | string | ✔ |
|  | `set_ids` | array | ✔ |
|  | `start_date` | string | ✔ |
|  | `end_date` | string | ✔ |
|  | `daily_new_cap` | integer | ✔ |
|  | `rest_weekdays` | array |  |
|  | `proposal_id` | string / null |  |
| `DecisionBody` | `minutes` | integer / null |  |
|  | `rest_of_day` | boolean / null |  |
|  | `decision` | string | ✔ |
| `DeviceStatusBody` | `location_permission` | string / null |  |
|  | `notifications_enabled` | boolean / null |  |
|  | `block_permission` | string / null |  |
|  | `block_auth` | string / null |  |
|  | `blocked_apps_count` | integer / null |  |
|  | `app_version` | string / null |  |
| `FocusBody` | `id` | string | ✔ |
|  | `minutes` | integer | ✔ |
|  | `started_at` | string | ✔ |
|  | `ended_at` | string / null |  |
|  | `result` | string / null |  |
| `GrantBody` | `minutes` | integer / null |  |
|  | `rest_of_day` | boolean / null |  |
|  | `child_id` | string | ✔ |
| `HTTPValidationError` | `detail` | array |  |
| `LoginBody` | `email` | string | ✔ |
|  | `password` | string | ✔ |
| `PackSourceBody` | `source_id` | string / null | ✔ |
| `PackStatusBody` | `status` | string | ✔ |
|  | `reason` | string / null |  |
| `PairingCodeBody` | `child_id` | string | ✔ |
| `PhoneSendBody` | `phone` | string | ✔ |
| `PhoneVerifyBody` | `phone` | string | ✔ |
|  | `code` | string | ✔ |
| `PlaceEventBody` | `id` | string | ✔ |
|  | `place_id` | string | ✔ |
|  | `kind` | string | ✔ |
|  | `source` | string | ✔ |
|  | `accuracy_m` | number / null |  |
|  | `occurred_at` | string | ✔ |
| `PolicyBody` | `mission_enabled` | boolean | ✔ |
|  | `schedule_enabled` | boolean | ✔ |
|  | `windows` | array | ✔ |
| `PushTokenBody` | `token` | string | ✔ |
|  | `app` | string | ✔ |
|  | `platform` | string | ✔ |
| `RemoveTokenBody` | `token` | string | ✔ |
| `RevokeBody` | `child_id` | string / null |  |
|  | `types` | array | ✔ |
| `SourceCreate` | `source_url` | string / null |  |
|  | `license` | string / null |  |
|  | `collected_at` | string / null |  |
|  | `terms_ok` | boolean / null |  |
|  | `ai_input_ok` | boolean / null |  |
|  | `derivative_ok` | boolean / null |  |
|  | `distribution_scope` | string / null |  |
|  | `note` | string / null |  |
|  | `title` | string | ✔ |
| `SourcePatch` | `source_url` | string / null |  |
|  | `license` | string / null |  |
|  | `collected_at` | string / null |  |
|  | `terms_ok` | boolean / null |  |
|  | `ai_input_ok` | boolean / null |  |
|  | `derivative_ok` | boolean / null |  |
|  | `distribution_scope` | string / null |  |
|  | `note` | string / null |  |
|  | `title` | string / null |  |
| `SyncAttemptIn` | `id` | string | ✔ |
|  | `item_id` | string | ✔ |
|  | `is_correct` | boolean | ✔ |
|  | `is_retry` | boolean |  |
|  | `answered_at` | string | ✔ |
|  | `response` | string / null |  |
| `SyncBody` | `days` | array | ✔ |
| `SyncDayIn` | `date` | string | ✔ |
|  | `curriculum_id` | string | ✔ |
|  | `mission_id` | string / null |  |
|  | `items` | array | ✔ |
|  | `attempts` | array |  |
|  | `completed_at` | string / null |  |
| `SyncItemIn` | `item_id` | string | ✔ |
|  | `kind` | string | ✔ |
|  | `seq` | integer | ✔ |
| `ValidationError` | `loc` | array | ✔ |
|  | `msg` | string | ✔ |
|  | `type` | string | ✔ |
|  | `input` |  |  |
|  | `ctx` | object |  |

### 부록 B. 호출 예시

```bash
# 1) 자녀 등록 (학부모 토큰)
curl -X POST http://127.0.0.1:8000/v1/children \
  -H "Authorization: Bearer $PARENT_JWT" -H "Content-Type: application/json" \
  -d '{"nickname":"별명","birth_ym":"2017-03","grade":3,
       "consents":[{"type":"privacy","doc_version":"2026-11-01"}]}'

# 2) 오늘 미션 (자녀 기기 토큰)
curl -X POST http://127.0.0.1:8000/v1/missions/today \
  -H "Authorization: Bearer $CHILD_JWT"

# 3) 진행률 (5초 폴링)
curl http://127.0.0.1:8000/v1/jobs/$JOB_ID -H "Authorization: Bearer $PARENT_JWT"

# 4) 오류 형식 확인 (무인증)
curl -i -X POST http://127.0.0.1:8000/v1/missions/today
# HTTP/1.1 401 ... {"error":{"code":"unauthorized","message":"로그인이 필요합니다.","details":{}}}
```

### 부록 C. 용어

| 용어 | 뜻 |
| --- | --- |
| 세트(`study_set`) | 학부모가 올린 학습 자료 1건. 문항의 묶음 단위 |
| 생성 작업(`generation_job`) | 파싱→계획→생성→검수→저장을 도는 비동기 작업 |
| 간격 반복 | 1·2·4·7·15일 5단계 상자. 틀리면 1단계로 복귀 |
| 이관 | 자동 판정을 포기하고 사람(부모)에게 넘기는 것 |
| fail-closed | 판정할 수 없으면 **안전한 쪽(차단·이관)**으로 끝내는 원칙 |
| 멱등성 | 같은 요청이 여러 번 와도 결과가 한 번만 반영되는 성질 |
| RLS | Postgres 행 수준 보안. 가족·자녀 단위 격리의 실제 집행 지점 |
| DLQ | 처리 실패를 버리지 않고 쌓아두는 대기열 |
