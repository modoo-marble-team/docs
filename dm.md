# DM 명세 (무영속 / 실시간 전용)

## 정책 요약
- 영속 저장하지 않는다.
- 서버사이드 캐싱을 두지 않는다.
- 새로고침 시 대화 내역은 휘발된다.
- 수신자가 온라인이 아닐 때는 전송할 수 없다.

## REST API
- 제공하지 않는다.
- 아래 API는 사용하지 않는다.
  - `GET /dm/rooms`
  - `GET /dm/rooms/{target_user_id}/messages`
  - `PATCH /dm/rooms/{target_user_id}/read`

## Socket 이벤트
| 이벤트 | 방향 | Payload | 설명 |
|---|---|---|---|
| `dm_send` | Client -> Server | `{ receiver_id, message, client_message_id? }` | DM 전송 요청 |
| `dm_receive` | Server -> Client | `{ message_id, sender_id, sender_nickname, message, sent_at }` | DM 수신 이벤트 |

## 동작 규칙
- `dm_send`는 수신자 온라인 상태에서만 성공한다.
- 수신자 오프라인이면 서버는 전송을 거절하고 에러를 반환한다.
- 서버는 메시지를 저장하지 않는다.
- 클라이언트는 수신한 메시지를 메모리 상태로만 유지한다.
- 페이지 새로고침/재접속 후 이전 메시지는 복구되지 않는다.

## 멱등성(idempotency)
- `dm_receive.message_id`는 필수다.
- 클라이언트는 `message_id` 기준으로 중복 삽입을 방지한다.

## 에러 코드
- `DM_TARGET_NOT_FOUND`
- `DM_TARGET_OFFLINE`
- `DM_NOT_ALLOWED_IN_PLAYING`
- `DM_MESSAGE_TOO_LONG`
- `DM_RATE_LIMITED`

