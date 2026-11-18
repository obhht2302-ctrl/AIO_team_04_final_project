# API 설계서

**프로젝트** 하루(Haru) — 초1~고1 학습·안전 관리 앱
**문서** API 설계서 (API Design Specification)
**버전** v1.0 · **작성일** 2026-11-18
**기준 커밋** `53cdf85` (branch `main`, 2026-11-18 · 재확인)
**작성** 윤기화 (WBS 2.1.4) · **검토** 오병훈 (WBS 7.4.1)
**제출** 11/20 단계 산출물 (S1)

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

1. 개요 · 2. 공통 규약 · 3. 엔드포인트 인벤토리 · 4. 대표 계약 · 5. 화면↔API 매핑 · 6. 역할·권한 매트릭스 · 7. 검증 · 8. 5개 기획 문서와 다른 지점 · 9. 부록(A 스키마 · B 호출 예시 · C 용어)

---

## 1. 개요

### 1.1 목적

이 문서는 **앱·웹·관리자·AI 워커가 서버와 주고받는 계약**을 한 곳에 고정한다. 목적은 세 가지다.

1. **화면과 서버를 병렬로 개발한다.** 화면 담당이 서버 구현을 기다리지 않도록 요청·응답의 모양을 먼저 확정한다.
2. **권한과 실패를 계약으로 만든다.** "누가 부를 수 있는가", "실패하면 어떤 코드로 끝나는가"를 API 수준에서 못 박는다.
3. **문서와 구현이 어긋나지 않게 한다.** 이 문서의 모든 표는 코드·실행 서버에서 추출했고, 8장에 **5개 기획 문서와 다른 지점**을 따로 적었다.

### 1.2 범위

| 구분 | 범위 | 근거 |
| --- | --- | --- |
| **MVP (11/22 구현 · 11/24 검증)** | 가입·문자 인증·자녀 등록·기기 연결·자료 업로드·문항 생성·검수·커리큘럼·오늘 미션·완료·오답·부모 현황 | 학습 MVP 화면 13개(P-01~08, C-01~05) |
| **1.0 (12/13 동결)** | 가족 약속·잠금 정책·해제 요청/승인·집중 모드·학원 장소·도착 알림·푸시 | `docs/features/*` |
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
| 멱등성 | 본문 UUID(4개 API) + `device_event_keys` + 중복 완료 차단 |
| 성공 상태 코드 | `200` / `201` / `202` / `204` |
| 미구현(문서만) | `GET /v1/admin/bank/*` 일부 정책, 팀 자료 창고 수락(`library_set`) |

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

| 주체 | 자격 증명 | 발급 | 권한 경계 |
| --- | --- | --- | --- |
| **학부모** | `Authorization: Bearer <Supabase access token>` | Supabase Auth (Google·Apple) + `POST /guardian/register` | 자기 가족 전체 |
| **자녀 기기** | 같은 헤더, **익명 세션** 토큰 | Supabase 익명 로그인 → `POST /pairing/claim` | 연결된 자녀 1명의 미션·문항 읽기, 풀이·이벤트 쓰기 |
| **관리자** | `haru_admin_session` (HttpOnly 쿠키) | `POST /admin/auth/login` | 문제은행·가족·사용량·감사 로그. 자녀 실명·좌표 원본 불가 |
| **기기 이벤트** | `Authorization` + `X-Device-Key` 헤더 | `POST /devices/event-key` | 1.0 위치 이벤트 한정 |

규칙

1. **역할은 토큰이 아니라 DB 조회로 판정한다.** 학부모 여부는 `guardians` 행, 자녀 기기 여부는 `child_devices` 행으로 확인한다.
2. 익명 토큰으로 학부모 API를 부르면 `forbidden`. 학부모 토큰으로 자녀 API(`missions/*`, `wrong-notes/*`, `pairing/claim`)를 불러도 `forbidden`.
3. `register`를 아직 부르지 않은 로그인 사용자의 학부모 요청은 `forbidden`(가족 없음).
4. 익명인데 연결이 없거나 끊겼으면 `not_paired`.
5. **없는 자원과 남의 자원을 구분하지 않는다.** 둘 다 `404 not_found`다. 존재 여부 자체가 정보이기 때문이다.

### 2.3 요청·응답 형식

| 항목 | 규칙 |
| --- | --- |
| 본문 | JSON, 필드 이름 `snake_case` |
| 인코딩 | UTF-8 |
| 시각 | ISO 8601 **UTC** (`2026-11-24T08:12:00Z`) — 마이크로초가 붙을 수 있다 |
| 날짜 | `YYYY-MM-DD` (**한국 시간** 기준) |
| 연월 | `YYYY-MM` |
| "오늘" | **서버가 Asia/Seoul로 계산한다.** 기기 시계를 신뢰하지 않는다 |
| 빈 값 | `null`은 "값 없음". 빈 배열 `[]`과 구분한다 |
| ID | UUID v4. 앱이 만든 ID를 받는 API는 본문 `id`로 받는다(멱등성) |
| 오프라인 | 앱이 기기에서 먼저 처리하고 `client_completed_at`·`answered_at`으로 시각을 보정한다 |

### 2.4 오류 형식과 코드

```json
{ "error": { "code": "code_expired", "message": "인증번호가 만료됐습니다.", "details": {} } }
```

- `code`는 **고정 문자열**(프로그램이 판단), `message`는 사용자 노출용(바뀔 수 있음), `details`는 코드별 보조 정보다.
- 입력 형식 오류는 `validation_failed`이고 `details.errors`에 `[{loc, msg}]`가 온다. 경로 ID가 UUID가 아니어도 같다.
- 없는 경로·허용하지 않는 메서드도 `not_found`다.

| HTTP | code | 뜻 |
| --- | --- | --- |
| 401 | `unauthorized` | 토큰 없음·만료 |
| 403 | `forbidden` | 역할 불일치 |
| 403 | `not_paired` | 자녀 기기 미연결 |
| 403 | `phone_not_verified` | 휴대전화 인증 전 |
| 404 | `not_found` | 없거나 남의 것(구분하지 않음) |
| 409 | `conflict` | 이미 처리됨 (`details.status`) |
| 409 | `mission_expired` | 지난 날짜의 미션 |
| 422 | `validation_failed` | 형식 오류 |
| 422 | `consent_required` | 필수 동의 누락 |
| 422 | `code_invalid` · `code_expired` | 인증번호·연결 코드 오류 |
| 422 | `file_invalid` · `file_too_large` | 업로드 실패 (`details.reason`) |
| 422 | `no_approved_items` · `items_in_active_curriculum` · `plan_infeasible` | 커리큘럼 생성 불가 |
| 422 | `mission_incomplete` | 미션 미완료 |
| 429 | `rate_limited` | 호출 빈도 초과 |
| 429 | `limit_exceeded` | 한도 초과(업로드 수·자녀 수·도우미 메시지) |
| 500 | `internal` | 서버 오류 |
| 503 | `sms_failed` | 문자 발송 실패 |
| 503 | `assistant_off` · `assistant_busy` | 도우미 꺼짐 / 동시 호출 자리 없음 |
| 503 | `geo_failed` | 장소 검색 제공자 실패 |

> **총 24개.** 5개 기획 문서(기술 명세서 6.2)는 이 중 6개를 **대문자**(`INVALID_INPUT` 등)로 적고 있어 실제와 다르다. 8장 참고.

### 2.5 멱등성 — 같은 요청이 두 번 와도 한 번만 처리한다

| 방식 | 적용 API | 동작 |
| --- | --- | --- |
| **본문 UUID** | `POST /missions/{id}/complete`, `/missions/sync`, `/assistant/messages`, `/events/place`, `/focus-sessions`, `/devices/event-key` | 앱이 만든 `id`가 이미 있으면 **저장된 결과를 그대로 반환**한다. 새로 만들지 않는다 |
| **`device_event_keys`** | `POST /events/place` | `(device, key)` 유니크. 재전송은 200을 돌려주고 새 행을 만들지 않는다 |
| **상태 검사** | `POST /sets/{id}/generate`, `/pairing/claim`, `/unlock-requests/{id}/decision` | 상태가 맞지 않으면 `conflict`(`details.status`) |
| **부분 유니크 인덱스** | `unlock_requests` | 자녀당 `pending` 1건만 허용(`unlock_requests_one_pending`) |
| **중복 완료 차단** | 미션 완료 | 같은 완료 요청 2회 → `attempts` 기록 1건 (인수조건 14) |

- `Idempotency-Key` 헤더는 **5개 기획 문서가 "모든 쓰기 API"라고 적었지만, 구현은 위 4가지 방식**이다. 헤더를 쓰지 않는 API는 본문 `id`나 상태 검사로 막는다(8장 5번).
- AI 도우미 메시지는 같은 `id` 재전송 시 **모델을 다시 부르지 않고** 저장된 두 말을 돌려준다. 답이 아직 없으면 `conflict`.

### 2.6 한도와 레이트리밋

| 대상 | 한도 | 오류 | 근거 |
| --- | --- | --- | --- |
| 자료 업로드 | 가정당 **하루 5건** | `limit_exceeded` (`max_uploads`) | `LIMIT_UPLOADS_PER_DAY` |
| 업로드 크기 | PDF **30쪽**·**20MB**, 사진 **10장**(장당 5MB), CSV 50행 | `file_too_large` | `LIMIT_*` |
| AI 도우미 | 가정당 **하루 30 메시지** | `limit_exceeded` (`max_messages`) | `usage_counters.messages` |
| 동시 생성 | 워커 1개 큐(`FOR UPDATE SKIP LOCKED`) | 대기(202) | 기획서 5.2 |
| 도우미 동시 호출 | `ASSIST_MAX_PARALLEL`, 10초 내 자리 없으면 | `assistant_busy` | `18-ai-assistant.md` |
| 일반 호출 | 경로별 상한 | `rate_limited` | `core` 레이트리밋 |

> 한도 확인과 한도 증가 사이에 모델 호출이 있어 **동시 요청 시 한도를 소폭 넘을 수 있다.** 이는 알려진 한계이며, 엄격한 상한이 필요하면 예약 방식으로 바꾼다.

### 2.7 성공 상태 코드

| 코드 | 쓰임 | 예 |
| --- | --- | --- |
| `200` | 조회·갱신·이미 존재하는 결과 반환 | `GET /jobs/{id}`, `POST /missions/today` |
| `201` | 새 자원 생성 | `POST /children`, `/curricula`, `/pairing/codes` |
| `202` | 비동기 접수(워커가 처리) | `POST /sets/{id}/generate` |
| `204` | 본문 없는 성공 | `DELETE /children/{id}` · `/sets/{id}` · `/assistant/threads/{id}`, `POST /focus-sessions` |

### 2.8 목록·페이지네이션

- **관리자 API만** `limit`·`offset`을 쓴다(`/admin/families`, `/admin/usage/families`). 기본값은 서버 설정을 따른다.
- 앱 API는 목록이 **구조적으로 작게 유지**되도록 설계했다(오늘 미션·자녀 수·주간 리포트). 페이지네이션 대신 **상한과 필터**를 둔다.
- 정렬은 서버가 정한다(최신순 또는 계획일 순). 클라이언트는 정렬을 요청하지 않는다.

---

## 3. 엔드포인트 인벤토리

`openapi.json`에서 추출한 **59개 operation**을 도메인별로 묶었다. `요청` 열의 `Body` 이름은 부록 A의 스키마와 대응한다. `authorization` 헤더는 모든 `/v1` 경로에 공통이므로 표에서 생략했다.

#### 1. 인증·가족 — 3개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/guardian/phone/send` | `PhoneSendBody` | 204 | 인증번호 발송 |
| 2 | `POST` | `/v1/guardian/phone/verify` | `PhoneVerifyBody` | 200 | 인증번호 확인 |
| 3 | `POST` | `/v1/guardian/register` | — | 200 | 보호자·가족 생성 |

#### 2. 자녀 관리 — 4개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/children` | `ChildBody` | 201 | 자녀 등록 + 동의 저장 |
| 2 | `DELETE` | `/v1/children/{child_id}` | — | 204 | 자녀 삭제(연결 해제) |
| 3 | `POST` | `/v1/children/{child_id}/block-policy` | `PolicyBody` | 200 | 잠금 정책 저장 |
| 4 | `POST` | `/v1/children/{child_id}/block-reselect` | — | 200 | 차단 앱 재선택 열기 |

#### 3. 기기 연결·상태 — 5개

| # | 메서드 | 경로 | 요청 | 성공 응답 | 동작 |
| --- | --- | --- | --- | --- | --- |
| 1 | `POST` | `/v1/devices/event-key` | — | 200 | 기기 이벤트 키 발급 |
| 2 | `POST` | `/v1/devices/status` | `DeviceStatusBody` | 204 | 기기 권한·상태 보고 |
| 3 | `POST` | `/v1/pairing/claim` | `ClaimBody` | 200 | 연결 코드 사용(기기 연결) |
| 4 | `POST` | `/v1/pairing/codes` | `PairingCodeBody` | 201 | 연결 코드 발급 |
| 5 | `POST` | `/v1/pairing/release` | — | 204 | 기기 연결 해제 |

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
- 답변에는 **근거(세트·`page`·`quote`) 또는 "근거 부족" 고지**가 붙는다. 자녀 실명·연락처·좌표는 프롬프트로 보내지 않는다(마스킹 후 전송).
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
| 서버 테스트 | `uv run pytest` (96개 테스트 파일) | 2026-11-16 기준 **649건 통과** |
| 앱 정적 검사 | `flutter analyze` · `flutter test` | CI 통과 |
| **재확인(11/18 pull 후)** | `2e25cb0` → `53cdf85` 13커밋 반영 후 재추출 | **58 path · 59 operation · 31 schema 동일**. `server/app/api/*`·`errors.py` 변경 없음 |
| **미검증** | 4xx·429 실제 부하 측정, 동시 30건 경합 | **미실시** — 12/31 통합 테스트에서 보강 |

> **한계를 먼저 적는다.** `limit_exceeded`·`assistant_busy`는 코드와 단위 테스트로만 확인했고, 실제 동시 부하로 재현하지 않았다. 제출 문서에 "검증 완료"로 쓰지 않는다.

---

## 8. 5개 기획 문서와 다른 지점 (결정 필요)

| # | 항목 | 5개 문서 | 구현(이 문서) | 권고 |
| --- | --- | --- | --- | --- |
| 1 | 대화 API 경로 | `POST /v1/chat/sessions` …, `/v1/chat/*` | **`/v1/assistant/*`** (thread·messages·proposals) | **문서를 구현에 맞춘다.** FR-WEB-002 매핑표를 `assistant`로 고치고, 필요하면 `/v1/chat/*` alias를 2주만 둔다 |
| 2 | 오류 코드 | 대문자 6개(`INVALID_INPUT`…) | 소문자 **24개**(`validation_failed`…) | 기술 명세서 6.2 표를 2.4로 교체 |
| 3 | 생성 작업 상태 | 예시가 "검수 대기"를 암시 | `queued/running/succeeded/failed` + `step` 5단계 | API 예시·화면 문구를 실제 상태값으로 수정 |
| 4 | 엔드포인트 수 | 17~18개 | **59개** | 이 문서를 정본으로 삼고, 5개 문서에는 "상세는 API 설계서" 링크 |
| 5 | 멱등성 | "모든 쓰기 API가 `Idempotency-Key`" | 본문 `id`·상태 검사·부분 유니크 4방식 | 문장을 "쓰기 API는 4가지 방식 중 하나로 중복을 막는다"로 수정 |
| 6 | 페이지네이션 | 규정 없음 | 관리자만 `limit`/`offset` | 2.8 문장을 5개 문서에 반영 |
| 7 | 자녀 인증 | "익명 세션"만 서술 | 익명 토큰 + `child_devices` 행 검사 | 아키텍처 문서에 판정 규칙 추가 |
| 8 | 문항 근거 | FR-AI-001 "모든 문항에 `page`·`quote`" | 기본팩 시드 문항은 `items.source`가 **NULL** | 인수조건을 **"AI 생성 문항에 한함"**으로 정정 |

> **11/18 변경 반영** — PM 결정으로 자녀 앱의 보호자 확인 단계(C-13, 두 자리 수 곱셈)가 제거됐다(`docs/features/family-agreement.md`·`block.md`·`location.md` 갱신). 화면 흐름만 바뀌고 **`/v1/agreements*` 계약은 변경 없다.**

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
