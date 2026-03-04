# API / WebSocket 명세서 (절충안, 2026-03-04)

## 1) 목적
- 프론트 기존 구현(`lobby`, `waiting-room`, `presence`)은 최대한 유지한다.
- 게임 로직(주사위, 매수/건설/매각, 턴 진행, 패널티)은 전부 WebSocket으로 전환한다.
- REST는 초기 스냅샷/방 관리/폴백 용도로 최소 유지한다.

## 2) 공통 규칙
| 항목 | 값 |
|---|---|
| REST Base URL | `/api` |
| Socket URL | `VITE_SOCKET_URL` (기본 `http://localhost:3000`) |
| 인증(REST) | `Authorization: Bearer {accessToken}` |
| 인증(Socket) | 연결 전 `socket.auth = { token }` 설정 |
| Content-Type | `application/json` |
| 날짜 포맷 | ISO-8601 UTC (`2026-03-04T08:00:00Z`) |
| 금액 단위 | 정수(원 단위) |

### 에러 포맷
REST / Socket Ack 공통으로 아래 형식을 사용한다.

```json
{
  "code": "STRING_ERROR_CODE",
  "message": "사용자 표시용 메시지",
  "detail": "옵션 상세"
}
```

## 3) REST API

| ID | Method | Path | Auth | 설명 |
|---|---|---|---|---|
| SYS-001 | GET | `/health` | N | 헬스체크 |
| LOBBY-001 | GET | `/rooms` | N | 로비 방 목록 스냅샷 조회 |
| ROOM-001 | POST | `/rooms` | Y | 방 생성 |
| ROOM-002 | POST | `/rooms/{roomId}/join` | Y | 방 입장 |
| ROOM-003 | POST | `/rooms/{roomId}/leave` | Y | 방 퇴장 |
| ROOM-004 | PATCH | `/rooms/{roomId}/ready` | Y | 준비 상태 토글 |
| ROOM-005 | POST | `/rooms/{roomId}/start` | Y (host) | 게임 시작 |
| PRESENCE-001 | GET | `/users/online` | Y | 온라인 유저 초기 스냅샷(폴백) |
| GAME-LEGACY-001 | GET | `/game/{roomId}/state` | Y | 초기 동기화용 폴백(권장: 제거 예정) |

### `GET /rooms` Query
| 이름 | 타입 | 필수 | 설명 |
|---|---|---|---|
| `status` | `waiting \| playing` | N | 상태 필터 |
| `exclude_private` | `true \| false` | N | 비공개 방 제외 |
| `keyword` | string | N | 방 제목 검색 |

## 4) WebSocket 이벤트 명세

## 4-1) 로비/대기방/채팅
| Event | 방향 | Payload | 설명 |
|---|---|---|---|
| `lobby_updated` | Server -> Client | `{ action, room }` | 로비 목록 변경 알림(생성/수정/삭제/상태변경) |
| `enter_room` | Client -> Server | `{ room_id }` | 대기방 소켓 입장 |
| `leave_room` | Client -> Server | `{ room_id }` | 대기방 소켓 퇴장 |
| `player_ready` | Server -> Client | `{ player_id, is_ready, all_ready }` | 준비 상태 변경 |
| `host_changed` | Server -> Client | `{ new_host_id, new_host_nickname }` | 방장 변경 |
| `send_chat` | Client -> Server | `{ room_id, message }` | 채팅 전송 |
| `chat` | Server -> Client | `{ room_id, sender_id, sender_nickname, message, sent_at }` | 채팅 수신 |
| `game_start` | Server -> Client | `{ game_id, game_state }` | 게임 시작 + 초기 상태 전달 |

## 4-2) 프레즌스/DM
| Event | 방향 | Payload | 설명 |
|---|---|---|---|
| `online_users` | Server -> Client | `{ users: [{ id, nickname, status }] }` | 온라인 유저 목록 실시간 브로드캐스트 |
| `dm_send` | Client -> Server | `{ receiver_id, message }` | DM 전송 |
| `dm_receive` | Server -> Client | `{ sender_id, sender_nickname, message, sent_at }` | DM 수신 |

## 4-3) 게임 (핵심: 전부 WebSocket)
### 클라이언트 -> 서버 액션 이벤트
| Event | Payload | Ack | 설명 |
|---|---|---|---|
| `request_game_state` | `{ room_id }` | `{ ok, state? , error? }` | 게임 상태 강제 동기화 |
| `roll_dice` | `{ room_id }` | `{ ok, error? }` | 주사위 굴리기 |
| `buy_tile` | `{ room_id, tile_index }` | `{ ok, error? }` | 타일 매수 |
| `build_tile` | `{ room_id, tile_index }` | `{ ok, error? }` | 건설 |
| `sell_tile` | `{ room_id, tile_index, level? }` | `{ ok, error? }` | 매각 |
| `confirm_penalty` | `{ room_id, player_id }` | `{ ok, error? }` | 패널티 확인 |

### 서버 -> 클라이언트 상태 이벤트
| Event | Payload | 설명 |
|---|---|---|
| `game_state` | `{ players, tiles, current_turn, round, timeout_sec }` | 정합성 기준 상태(서버 권위) |
| `turn_start` | `{ player_id, round, timeout_sec }` | 턴 시작 |
| `dice_rolled` | `{ player_id, dice, is_double, double_count }` | 주사위 결과 |
| `player_moved` | `{ player_id, from_index, to_index, trigger, pass_go, pass_go_salary }` | 이동 결과 |
| `tile_purchased` | `{ player_id, tile_index, tile_name, price }` | 타일 매수 반영 |
| `toll_paid` | `{ payer_id, owner_id, tile_index, amount }` | 통행료 정산 |
| `event_card_drawn` | `{ player_id, card }` | 이벤트 카드 |
| `chance_card_drawn` | `{ player_id, card }` | 찬스 카드 |
| `player_sent_to_jail` | `{ player_id }` | 무인도/감금 처리 |
| `player_bankrupt` | `{ player_id }` | 파산 처리 |
| `ai_penalty_loading` | `{}` | AI 패널티 로딩 |
| `ai_penalty` | `{ ... }` | AI 패널티 결과 |
| `game_over` | `{ reason, rankings }` | 게임 종료 |

### Socket Ack 규격
모든 게임 액션 이벤트는 아래 Ack를 반환한다.

```json
{
  "ok": true
}
```

```json
{
  "ok": false,
  "error": {
    "code": "NOT_YOUR_TURN",
    "message": "현재 턴이 아닙니다."
  }
}
```

## 5) 상태 권위 규칙
- 게임 상태의 최종 권위는 서버이다.
- 클라이언트는 연출(주사위 애니메이션, 모달 노출)만 담당한다.
- 돈/소유권/턴 변경은 서버 이벤트(`game_state` 포함)로만 확정한다.
- 클라이언트의 낙관적 업데이트는 허용하되, `game_state` 수신 시 서버 상태로 덮어쓴다.

## 6) 프론트 적용 기준 (최소 변경)
| 영역 | 현재 | 절충안 |
|---|---|---|
| 로비 목록 | REST + `lobby_updated` 무효화 | 유지 (변경 최소) |
| 대기방 채팅/이벤트 | Socket | 유지 |
| 온라인 유저 | REST 초기 + Socket 갱신 | 유지 |
| 게임 액션(`buy/build/sell/roll`) | REST + Socket 혼합 | Socket 100% 전환 |
| 게임 초기 동기화 | REST `getState` + Socket | `request_game_state` 우선, REST는 임시 폴백 |

## 7) Deprecated 대상
- `POST /game/{roomId}/buy`
- `POST /game/{roomId}/build`
- `POST /game/{roomId}/sell`

위 3개는 서버/프론트 전환 완료 시 제거한다.

## 8) 예시 플로우
1. 클라이언트 `buy_tile` emit (`{ room_id, tile_index }`) + Ack 대기
2. 서버 Ack `ok: true`
3. 서버 `tile_purchased` 브로드캐스트
4. 서버 `game_state` 브로드캐스트
5. 클라이언트는 `game_state` 기준으로 store를 최종 갱신

