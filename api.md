# API / WebSocket 명세서

## 범위
- 게임 본플레이 계약은 `docs/gamesocket.md`를 단일 기준으로 사용한다.
- 이 문서는 로비, 대기방, 프레즌스, DM의 REST/Socket 계약을 정의한다.
- 문서 간 충돌 시 우선순위:
  1. `docs/gamesocket.md`
  2. `docs/api.md`
  3. 레거시 구현 동작

## 공통
| 항목 | 값 |
|---|---|
| REST Base URL | `/api` |
| Socket URL | `VITE_SOCKET_URL` (기본 `http://localhost:3000`) |
| REST 인증 | `Authorization: Bearer {accessToken}` |
| Socket 인증 | 연결 전 `socket.auth = { token: accessToken }` |
| Content-Type | `application/json` |
| 시간 포맷 | ISO-8601 UTC |

### 공통 에러 포맷(전체 API)
```json
{
  "code": "STRING_ERROR_CODE",
  "message": "사용자 노출 메시지",
  "detail": "선택 상세 정보"
}
```

## REST API

## 로비 / 대기방 / 프레즌스
| ID | Method | Path | Auth | 설명 |
|---|---|---|---|---|
| SYS-001 | GET | `/health` | N | 헬스체크 |
| LOBBY-001 | GET | `/rooms` | N | 로비 방 목록 스냅샷 조회 |
| ROOM-001 | POST | `/rooms` | Y | 방 생성 |
| ROOM-002 | POST | `/rooms/{roomId}/join` | Y | 방 입장 |
| ROOM-003 | POST | `/rooms/{roomId}/leave` | Y | 방 퇴장 |
| ROOM-004 | PATCH | `/rooms/{roomId}/ready` | Y | 준비 상태 변경 |
| ROOM-005 | POST | `/rooms/{roomId}/start` | Y(host) | 게임 시작 |
| PRESENCE-001 | GET | `/users/online` | Y | 온라인 유저 초기 스냅샷 |

### GET `/rooms` query
| 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `status` | `waiting \| playing` | N | 상태 필터 |
| `exclude_private` | boolean | N | 비공개 방 제외 |
| `keyword` | string | N | 방 제목 검색 |

### ROOM-002: POST `/rooms/{roomId}/join` 응답 계약
#### 필수 필드
| 필드 | 타입 | 비고 |
|---|---|---|
| `room_id` | string | 입장한 방 ID |
| `title` | string | 방 제목 |
| `status` | `waiting \| playing` | 방 상태 |
| `max_players` | number | 최대 인원 |
| `is_private` | boolean | 비공개 여부 |
| `players` | array | 현재 방 참가자 목록 |
| `chat_messages` | array | 채팅 히스토리(빈 배열 허용, 필드 자체는 필수) |

#### `players[]` 필수 필드
| 필드 | 타입 |
|---|---|
| `id` | string |
| `nickname` | string |
| `is_ready` | boolean |
| `is_host` | boolean |

#### `chat_messages[]` 필수 필드
| 필드 | 타입 |
|---|---|
| `id` | string |
| `sender_id` | string |
| `sender_nickname` | string |
| `message` | string |
| `sent_at` | string (ISO-8601 UTC) |
| `type` | `talk` |

#### 예시
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

### ROOM-003 퇴장 시퀀스 계약
- 클라이언트는 `POST /rooms/{roomId}/leave`를 먼저 호출해야 한다.
- REST 퇴장 성공 시, 클라이언트는 실시간 분리를 위해 `leave_room`을 즉시 emit하는 것을 권장한다.
- REST-only도 fallback으로 허용한다(서버는 REST 결과만으로 퇴장 상태를 확정 처리해야 함).
- 권장 순서:
  1. `POST /rooms/{roomId}/leave`
  2. 성공 시 `{ room_id }` payload로 `leave_room` emit
  3. 클라이언트 라우팅/상태 정리

## DM API (신규)
| ID | Method | Path | Auth | 설명 |
|---|---|---|---|---|
| DM-001 | GET | `/dm/rooms` | Y | DM 방 목록 + 마지막 메시지 + unread_count 조회 |
| DM-002 | GET | `/dm/rooms/{target_user_id}/messages` | Y | 메시지 히스토리 조회(커서 페이지네이션) |
| DM-003 | PATCH | `/dm/rooms/{target_user_id}/read` | Y | 읽음 처리(`unread_count=0`) |

### DM-002 query
| 이름 | 타입 | 필수 | 기본값 | 비고 |
|---|---|---|---|---|
| `cursor` | string | N | null | 이전 페이지 앵커 |
| `limit` | number | N | 30 | `1..100` |

## WebSocket API

## 로비 / 대기방
| 이벤트 | 방향 | Payload | 설명 |
|---|---|---|---|
| `lobby_updated` | Server -> Client | `{ action, room }` | 로비 목록 변경 알림 |
| `enter_room` | Client -> Server | `{ room_id }` | 대기방 소켓 입장 |
| `leave_room` | Client -> Server | `{ room_id }` | 대기방 소켓 퇴장 |
| `player_ready` | Server -> Client | `{ player_id, is_ready, all_ready }` | 준비 상태 변경 |
| `host_changed` | Server -> Client | `{ new_host_id, new_host_nickname }` | 방장 변경 |
| `send_chat` | Client -> Server | `{ room_id, message }` | 대기방 채팅 전송 |
| `chat` | Server -> Client | `{ room_id, sender_id, sender_nickname, message, sent_at }` | 대기방 채팅 수신 |
| `game_start` | Server -> Client | `{ game_id, room_id, game_state? }` | 게임 시작 신호 |

### `game_start` payload 계약(확정)
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
| 이벤트 | 방향 | Payload | 설명 |
|---|---|---|---|
| `online_users` | Server -> Client | `{ users: [{ id, nickname, status }] }` | 온라인 유저 스냅샷/변경 이벤트 |

### `online_users` 초기 정책(확정)
- 소켓 연결 직후 초기 스냅샷 emit은 보장하지 않는다.
- 클라이언트는 반드시 `GET /users/online`으로 bootstrap 해야 한다.
- bootstrap 이후에는 `online_users` 소켓 이벤트로 실시간 반영한다.
- 서버 초기 emit은 최적화로 제공 가능하나, 클라이언트가 의존하면 안 된다.

## DM Socket
| 이벤트 | 방향 | Payload | 설명 |
|---|---|---|---|
| `dm_send` | Client -> Server | `{ receiver_id, message, client_message_id? }` | DM 전송 |
| `dm_receive` | Server -> Client | `{ message_id, sender_id, sender_nickname, message, sent_at }` | DM 수신 |

### DM 멱등성(idempotency)
- `dm_receive.message_id`는 필수다.
- 클라이언트는 `message_id` 기준으로 중복 삽입을 방지해야 한다.
- 서버는 저장 성공 후에만 `dm_receive`를 발행한다.

## 게임 Socket
- 게임 본플레이는 `docs/gamesocket.md`를 따른다.
  - Client -> Server: `game:action`, `game:sync`, `game:prompt_response`
  - Server -> Client: `game:ack`, `game:patch`, `game:prompt`, `game:error`

## DM 정책
- 저장: 영속 저장
- TTL: 365일
- 삭제 모델: 사용자 뷰 기준 소프트 삭제(hide)
- 하드 삭제: TTL 만료 배치 또는 운영 정책에 따라 수행
- unread_count 단일 진실원천: 서버

## DM 에러 코드
- `DM_TARGET_NOT_FOUND`
- `DM_NOT_ALLOWED_IN_PLAYING`
- `DM_ROOM_NOT_FOUND`
- `DM_MESSAGE_TOO_LONG`
- `DM_RATE_LIMITED`

