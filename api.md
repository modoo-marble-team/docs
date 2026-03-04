# API / WebSocket 명세서 (gamesocket 기준 정렬본)

## 1. 문서 목적
- 본 문서는 `docs/gamesocket.md`를 게임 영역의 단일 기준으로 사용한다.
- 기존 프론트 구현을 최대한 유지하기 위해 REST는 로비/대기방/프레즌스 중심으로 유지한다.
- 게임 로직은 모두 `game:*` 소켓 이벤트로 처리한다.

## 2. 공통 규칙
| 항목 | 값 |
|---|---|
| REST Base URL | `/api` |
| Socket URL | `VITE_SOCKET_URL` (기본 `http://localhost:3000`) |
| REST 인증 | `Authorization: Bearer {accessToken}` |
| Socket 인증 | 연결 전 `socket.auth = { token: accessToken }` |
| Content-Type | `application/json` |
| 시간 포맷 | ISO-8601 UTC |
| 금액 단위 | 정수(프로젝트 합의 단위 고정, 프론트/백엔드 동일 적용) |

### 공통 에러 포맷
```json
{
  "code": "STRING_ERROR_CODE",
  "message": "사용자 표시용 메시지",
  "detail": "옵션"
}
```

## 3. REST API (유지 범위)
| ID | Method | Path | Auth | 설명 |
|---|---|---|---|---|
| SYS-001 | GET | `/health` | N | 헬스체크 |
| LOBBY-001 | GET | `/rooms` | N | 로비 방 목록 스냅샷 |
| ROOM-001 | POST | `/rooms` | Y | 방 생성 |
| ROOM-002 | POST | `/rooms/{roomId}/join` | Y | 방 입장 |
| ROOM-003 | POST | `/rooms/{roomId}/leave` | Y | 방 퇴장 |
| ROOM-004 | PATCH | `/rooms/{roomId}/ready` | Y | 준비 상태 변경 |
| ROOM-005 | POST | `/rooms/{roomId}/start` | Y (host) | 게임 시작 |
| PRESENCE-001 | GET | `/users/online` | Y | 온라인 유저 초기 스냅샷 |

### `GET /rooms` query
| 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `status` | `waiting \| playing` | N | 상태 필터 |
| `exclude_private` | boolean | N | 비공개 방 제외 |
| `keyword` | string | N | 방 제목 검색 |

## 4. WebSocket 명세

## 4.1 로비/대기방/채팅
| Event | 방향 | Payload | 설명 |
|---|---|---|---|
| `lobby_updated` | Server -> Client | `{ action, room }` | 로비 목록 변경 알림 |
| `enter_room` | Client -> Server | `{ room_id }` | 대기방 소켓 입장 |
| `leave_room` | Client -> Server | `{ room_id }` | 대기방 소켓 퇴장 |
| `player_ready` | Server -> Client | `{ player_id, is_ready, all_ready }` | 준비 상태 변경 |
| `host_changed` | Server -> Client | `{ new_host_id, new_host_nickname }` | 방장 변경 |
| `send_chat` | Client -> Server | `{ room_id, message }` | 채팅 전송 |
| `chat` | Server -> Client | `{ room_id, sender_id, sender_nickname, message, sent_at }` | 채팅 수신 |
| `game_start` | Server -> Client | `{ game_id }` | 게임 시작 알림(진행 상태는 `game:sync`/`game:patch`로 수신) |

## 4.2 프레즌스/DM
| Event | 방향 | Payload | 설명 |
|---|---|---|---|
| `online_users` | Server -> Client | `{ users: [{ id, nickname, status }] }` | 온라인 유저 갱신 |
| `dm_send` | Client -> Server | `{ receiver_id, message }` | DM 전송 |
| `dm_receive` | Server -> Client | `{ sender_id, sender_nickname, message, sent_at }` | DM 수신 |

## 4.3 게임 (단일 기준: `docs/gamesocket.md`)

### 클라이언트 -> 서버
| Event | 설명 |
|---|---|
| `game:action` | 게임 행위(의도) 요청 |
| `game:sync` | 재접속/초기 진입 동기화 요청 |
| `game:prompt_response` | 개인 선택 프롬프트 응답 |

### 서버 -> 클라이언트
| Event | 설명 |
|---|---|
| `game:ack` | 요청 처리 결과(요청자에게만) |
| `game:patch` | 상태 변경 전파(게임 룸 브로드캐스트) |
| `game:prompt` | 개인 선택 요청(개인 룸) |
| `game:error` | 치명 오류/강제 동기화 요청(선택) |

### 룸 규칙
- 게임 룸: `game:{gameId}`
- 개인 룸: `user:{userId}`

### `game:action` envelope
```json
{
  "gameId": "g1",
  "actionId": "uuid-123",
  "type": "ROLL_DICE",
  "payload": {}
}
```

### ActionType
- `ROLL_DICE`
- `BUY_PROPERTY`
- `SELL_PROPERTY`
- `END_TURN`

### `game:sync`
```json
{
  "gameId": "g1",
  "knownRevision": 40
}
```

### `game:prompt_response`
```json
{
  "gameId": "g1",
  "promptId": "pr-9",
  "choice": "BUY",
  "payload": {}
}
```

### `game:ack` 예시
```json
{
  "gameId": "g1",
  "actionId": "uuid-1",
  "ok": true,
  "error": null,
  "revision": 42
}
```

```json
{
  "gameId": "g1",
  "actionId": "uuid-2",
  "ok": false,
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "잔액이 부족합니다."
  },
  "revision": 42
}
```

### `game:patch` envelope
```json
{
  "gameId": "g1",
  "revision": 42,
  "turn": 10,
  "events": [],
  "patch": [],
  "snapshot": null
}
```

### patch operation
- `set`: 값 설정
- `inc`: 숫자 증감
- `push`: 배열 추가
- `remove`: 배열/키 삭제

## 5. 상태 권위 및 적용 규칙
- 서버 authoritative: 주사위/이동/통행료/소유권/건설 레벨 확정은 서버만 수행한다.
- 클라이언트는 의도(intent)만 보낸다.
- 클라이언트는 `revision` 기준으로 중복/역순 patch를 무시한다.
- `game:patch.snapshot` 존재 시 로컬 상태를 snapshot으로 교체한다.
- 액션 실패는 `game:ack.ok=false`를 기준으로 즉시 UI 에러를 표시한다.

## 6. 데이터 타입 기준 (gamesocket 합의)
- 식별자/턴/revision/타일/레벨/금액 관련 값은 number 기준 사용
- 빌딩 레벨: `0..7`
- 주요 enum: `ActionType`, `ServerEventType`, `TileType`, `PlayerState`는 `docs/gamesocket.md` 정의를 따른다.

## 7. 호환성 정책
- 아래 레거시 게임 이벤트/REST는 신규 구현 기준에서 사용하지 않는다.
  - 레거시 이벤트: `roll_dice`, `game_state`, `turn_start`, `player_moved`, `tile_purchased`, `toll_paid` ...
  - 레거시 REST: `/game/{roomId}/buy`, `/game/{roomId}/build`, `/game/{roomId}/sell`, `/game/{roomId}/state`
- 다만 마이그레이션 기간에는 서버에서 임시 어댑터를 둘 수 있다.
- 기준 문서 충돌 시 우선순위:
  1. `docs/gamesocket.md`
  2. 본 문서(`docs/api.md`)
  3. 레거시 구현

