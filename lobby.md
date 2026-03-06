# 로비 / 대기방 / 프레즌스 명세

## REST API

| ID           | Method | Path                      | Auth    | 설명                     |
| ------------ | ------ | ------------------------- | ------- | ------------------------ |
| SYS-001      | GET    | `/health`               | N       | 헬스체크                 |
| LOBBY-001    | GET    | `/rooms`                | N       | 로비 방 목록 스냅샷 조회 |
| ROOM-001     | POST   | `/rooms`                | Y       | 방 생성                  |
| ROOM-002     | POST   | `/rooms/{roomId}/join`  | Y       | 방 입장                  |
| ROOM-003     | POST   | `/rooms/{roomId}/leave` | Y       | 방 퇴장                  |
| ROOM-004     | PATCH  | `/rooms/{roomId}/ready` | Y       | 준비 상태 변경           |
| ROOM-005     | POST   | `/rooms/{roomId}/start` | Y(host) | 게임 시작                |
| PRESENCE-001 | GET    | `/users/online`         | Y       | 온라인 유저 초기 스냅샷  |

### GET `/rooms` query

| 이름                | 타입                  | 필수 | 설명           |
| ------------------- | --------------------- | ---- | -------------- |
| `status`          | `waiting \| playing` | N    | 상태 필터      |
| `exclude_private` | boolean               | N    | 비공개 방 제외 |
| `keyword`         | string                | N    | 방 제목 검색   |

## ROOM-002: POST `/rooms/{roomId}/join` 응답 계약

### 필수 필드

| 필드              | 타입                  | 비고                                          |
| ----------------- | --------------------- | --------------------------------------------- |
| `room_id`       | string                | 입장한 방 ID                                  |
| `title`         | string                | 방 제목                                       |
| `status`        | `waiting \| playing` | 방 상태                                       |
| `max_players`   | number                | 최대 인원                                     |
| `is_private`    | boolean               | 비공개 여부                                   |
| `players`       | array                 | 현재 방 참가자 목록                           |
| `chat_messages` | array                 | 채팅 히스토리(빈 배열 허용, 필드 자체는 필수) |

### `players[]` 필수 필드

| 필드         | 타입    |
| ------------ | ------- |
| `id`       | string  |
| `nickname` | string  |
| `is_ready` | boolean |
| `is_host`  | boolean |

### `chat_messages[]` 필수 필드

| 필드                | 타입                  |
| ------------------- | --------------------- |
| `id`              | string                |
| `sender_id`       | string                |
| `sender_nickname` | string                |
| `message`         | string                |
| `sent_at`         | string (ISO-8601 UTC) |
| `type`            | `talk`              |

### 응답 예시

```json
{
  "room_id": "room-7",
  "title": "즐거운 게임 한판!",
  "status": "waiting",
  "max_players": 4,
  "is_private": false,
  "players": [
    { "id": "u-1", "nickname": "홍길동", "is_ready": false, "is_host": true }
  ],
  "chat_messages": []
}
```

## ROOM-003 퇴장 시퀀스 계약

- 클라이언트는 `POST /rooms/{roomId}/leave`를 먼저 호출한다.
- REST 성공 후 `leave_room` emit을 권장한다(실시간 분리 목적).
- REST-only도 fallback으로 허용한다.
- 권장 순서:
  1. `POST /rooms/{roomId}/leave`
  2. 성공 시 `{ room_id }` payload로 `leave_room` emit
  3. 클라이언트 라우팅/상태 정리

## WebSocket 이벤트

## 로비 / 대기방

| 이벤트            | 방향             | Payload                                                       | 설명                |
| ----------------- | ---------------- | ------------------------------------------------------------- | ------------------- |
| `lobby_updated` | Server -> Client | `{ action, room }`                                          | 로비 목록 변경 알림 |
| `enter_room`    | Client -> Server | `{ room_id }`                                               | 대기방 소켓 입장    |
| `leave_room`    | Client -> Server | `{ room_id }`                                               | 대기방 소켓 퇴장    |
| `player_ready`  | Server -> Client | `{ player_id, is_ready, all_ready }`                        | 준비 상태 변경      |
| `host_changed`  | Server -> Client | `{ new_host_id, new_host_nickname }`                        | 방장 변경           |
| `send_chat`     | Client -> Server | `{ room_id, message }`                                      | 대기방 채팅 전송    |
| `chat`          | Server -> Client | `{ room_id, sender_id, sender_nickname, message, sent_at }` | 대기방 채팅 수신    |
| `game_start`    | Server -> Client | `{ game_id, room_id, game_state? }`                         | 게임 시작 신호      |

### `game_start` payload 계약

- 이벤트명: `game_start` 유지
- 필수: `game_id`(string), `room_id`(string)
- 선택: `game_state`(object)

```json
{
  "game_id": "g-123",
  "room_id": "room-7"
}
```

## 프레즌스

| 이벤트           | 방향             | Payload                                   | 설명                           |
| ---------------- | ---------------- | ----------------------------------------- | ------------------------------ |
| `online_users` | Server -> Client | `{ users: [{ id, nickname, status }] }` | 온라인 유저 스냅샷/변경 이벤트 |

### `online_users` 초기 정책(확정)

- 소켓 연결 직후 1회 emit은 보장하지 않는다.
- 클라이언트는 반드시 `GET /users/online`으로 bootstrap 한다.
- bootstrap 이후 `online_users` 소켓 이벤트로 실시간 반영한다.
- 서버 초기 emit은 최적화로 제공 가능하나, 클라이언트는 의존하면 안 된다.
