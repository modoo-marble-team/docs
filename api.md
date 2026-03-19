# REST API Specification
## 0. Common Rules

- 기본경로: `/api`
- 헬스체크: `/health`, `/api/health`

### Authentication

- Access token 인증이 필요한 엔드포인트는 `Authorization: Bearer <accessToken>` 헤더를 사용한다.
- Refresh는 HttpOnly 쿠키 `modoo_refresh_token`를 사용한다.
- Socket 인증은 별도 문서 [`gamesocket.md`](./gamesocket.md)를 따른다.

### Gameplay Contract Note

- 실제 게임 진행 중 prompt / event payload 계약은 REST 문서가 아니라 [`gamesocket.md`](./gamesocket.md)가 기준이다.
- 특히 카드 설명 문구는 `CHANCE_RESOLVED.chance.description`, 착지 이벤트 타일 정보는 `LANDED.tile` nested payload를 기준으로 처리해야 한다.

### Content Type

- JSON body가 있는 요청은 `Content-Type: application/json`

### Error Response Shapes

구현상 에러 응답 형태가 두 가지다.

1. Custom `ApiError`

```json
{
  "code": "ROOM_NOT_FOUND",
  "message": "방을 찾을 수 없습니다.",
  "detail": null
}
```

2. FastAPI `HTTPException` / validation error

```json
{
  "detail": "Invalid token"
}
```

```json
{
  "detail": [
    {
      "loc": ["body", "nickname"],
      "msg": "String should have at most 10 characters",
      "type": "string_too_long"
    }
  ]
}
```

### Common Auth Failures

아래 엔드포인트들은 구현상 별도 `code` 없이 `detail`만 내려간다.

| HTTP | detail |
|---|---|
| `401` | `Authorization header missing` |
| `401` | `Invalid Authorization header` |
| `401` | `Token expired` |
| `401` | `Invalid token` |
| `401` | `Invalid token payload` |

---

## 1. System

### `GET /health`

- 목적: 서버 liveness 확인
- 인증: 없음
- 요청 바디: 없음

성공 응답

| HTTP | Body |
|---|---|
| `200` | `{"status":"ok","title":"모두의마블 API","version":"0.1.0"}` |

### `GET /api/health`

- 목적: API prefix 하위 health check
- 인증: 없음
- 요청 바디: 없음

성공 응답

| HTTP | Body |
|---|---|
| `200` | `{"status":"ok","title":"모두의마블 API","version":"0.1.0"}` |

---

## 2. Auth

### `GET /api/auth/kakao/login`

- 목적: 카카오 OAuth 인가 페이지로 리다이렉트
- 인증: 없음
- 요청 바디: 없음

요청 헤더

| Header | Required | Value |
|---|---|---|
| 없음 | - | - |

성공 응답

| HTTP | Header | Body |
|---|---|---|
| `302` | `Location: https://kauth.kakao.com/oauth/authorize?...` | 없음 |

에러

| HTTP | code | Body |
|---|---|---|
| `500` | 없음 | `{"detail":"Kakao OAuth env missing"}` |

### `GET /api/auth/kakao/callback`

- 목적: 카카오 code를 access token + refresh cookie로 교환하고 프론트 콜백 URL로 리다이렉트
- 인증: 없음

쿼리

| Name | Type | Required |
|---|---|---|
| `code` | string | yes |

성공 응답

| HTTP | Header | Body |
|---|---|---|
| `302` | `Set-Cookie: modoo_refresh_token=...; HttpOnly; Path=/api/auth` | 없음 |
| `302` | `Location: {FRONTEND_LOGIN_REDIRECT}?access_token=...&is_new_user=true|false` | 없음 |

에러

| HTTP | code | Body |
|---|---|---|
| `400` | 없음 | `{"detail":"Kakao token exchange failed"}` 등 |
| `500` | 없음 | `{"detail":"Kakao OAuth env missing"}` |
| `500` | 없음 | `{"detail":"FRONTEND_LOGIN_REDIRECT missing"}` |
| `422` | 없음 | FastAPI validation error |

### `POST /api/auth/kakao/callback`

- 목적: 카카오 code를 access token + refresh cookie로 교환
- 인증: 없음

요청 바디

```json
{
  "code": "kakao-auth-code"
}
```

성공 응답

| HTTP | Header | Body |
|---|---|---|
| `200` | `Set-Cookie: modoo_refresh_token=...; HttpOnly; Path=/api/auth` | 아래 예시 참조 |

```json
{
  "access_token": "jwt-access-token",
  "user": {
    "id": 1,
    "nickname": "player1",
    "profile_image_url": "https://...",
    "is_guest": false
  },
  "is_new_user": true
}
```

에러

| HTTP | code | Body |
|---|---|---|
| `400` | 없음 | `{"detail":"Kakao token exchange failed"}` 등 |
| `500` | 없음 | `{"detail":"Kakao OAuth env missing"}` |
| `422` | 없음 | FastAPI validation error |

### `POST /api/auth/guest`

- 목적: 게스트 계정 생성 및 로그인
- 인증: 없음
- 요청 바디: 없음

성공 응답

| HTTP | Header | Body |
|---|---|---|
| `200` | `Set-Cookie: modoo_refresh_token=...; HttpOnly; Path=/api/auth` | `AuthResponse` |

```json
{
  "access_token": "jwt-access-token",
  "user": {
    "id": 12,
    "nickname": "게스트1234",
    "profile_image_url": null,
    "is_guest": true
  },
  "is_new_user": false
}
```

에러

| HTTP | code | Body |
|---|---|---|
| `409` | 없음 | `{"detail":"Guest creation conflict"}` |

### `GET /api/auth/session`

- 목적: 현재 access token 기준 세션 조회
- 인증: Access token 필요
- 요청 바디: 없음

성공 응답

| HTTP | Body |
|---|---|
| `200` | `{"id":1,"nickname":"player1","profile_image_url":"https://...","is_guest":false}` |

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | 없음 | `{"detail":"User not found"}` |

### `POST /api/auth/refresh`

- 목적: refresh cookie를 검증하고 새 access token + 새 refresh cookie 발급
- 인증: refresh cookie 필요
- 요청 바디: 없음

요청 헤더

| Header | Required | Value |
|---|---|---|
| `Cookie` | yes | `modoo_refresh_token=<refresh-token>` |

성공 응답

| HTTP | Header | Body |
|---|---|---|
| `200` | `Set-Cookie: modoo_refresh_token=...; HttpOnly; Path=/api/auth` | 아래 예시 참조 |

```json
{
  "access_token": "new-jwt-access-token",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | `{"detail":"Refresh token missing"}` |
| `401` | 없음 | `{"detail":"Refresh token expired"}` |
| `401` | 없음 | `{"detail":"Invalid refresh token"}` |

### `POST /api/auth/logout`

- 목적: refresh session revoke 후 refresh cookie 제거
- 인증: 없음
- 요청 바디: 없음

요청 헤더

| Header | Required | Value |
|---|---|---|
| `Cookie` | no | `modoo_refresh_token=<refresh-token>` |

성공 응답

| HTTP | Header | Body |
|---|---|---|
| `200` | `Set-Cookie: modoo_refresh_token=; Max-Age=0; Path=/api/auth` | `{"success":true}` |

에러

- 구현상 logout은 refresh token 파싱 실패를 무시하고 성공 응답을 반환한다.

---

## 3. Users

### `GET /api/users/me`

- 목적: 마이페이지용 사용자 정보 조회
- 인증: Access token 필요
- 권한: 게스트 불가
- 요청 바디: 없음

성공 응답

```json
{
  "id": 1,
  "nickname": "player1",
  "profile_image_url": "https://...",
  "is_guest": false,
  "stats": {
    "total_games": 0,
    "wins": 0,
    "losses": 0
  }
}
```

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `403` | 없음 | `{"detail":"Guest not allowed"}` |
| `404` | 없음 | `{"detail":"User not found"}` |

### `GET /api/users/me/context`

- 목적: 현재 사용자가 어느 방/게임에 참가 중인지와 앱 재진입 시 어디로 복귀해야 하는지 조회
- 인증: Access token 필요
- 권한: 게스트 포함 사용 가능
- 요청 바디: 없음

성공 응답

```json
{
  "room_id": "room-1234abcd",
  "room_title": "친구방",
  "room_status": "playing",
  "game_id": "17",
  "presence_status": "playing",
  "resume_target": "game"
}
```

필드 설명

- `room_id`: 현재 참가 중인 방 ID. 없으면 `null`
- `room_title`: 현재 방 제목. 방을 찾을 수 없으면 `null`
- `room_status`: 현재 방 상태. `waiting` 또는 `playing`, 없으면 `null`
- `game_id`: 현재 진행 중인 게임 ID. 없으면 `null`
- `presence_status`: presence 저장값. 보통 `lobby`, `in_room`, `playing` 중 하나
- `resume_target`: 프론트가 기본 복귀 대상으로 써야 하는 위치

`resume_target` 값

- `lobby`: 참가 중인 방/게임 없음
- `room`: 방에는 참가 중이지만 진행 중 게임은 없음
- `game`: 진행 중 게임이 있음

구현 메모

- `room_id`는 현재 방 매핑 또는 진행 중 게임 상태에서 복구될 수 있다.
- `game_id`는 활성 게임 매핑, 레거시 게임 매핑, 방의 `game_id` 중 현재 값으로 계산된다.
- 게임 종료 후에는 방이 유지되므로 보통 `room_status=waiting`, `game_id=null`, `resume_target=room`으로 내려간다.

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |

### `PATCH /api/users/me/nickname`

- 목적: 닉네임 변경
- 인증: Access token 필요
- 권한: 게스트 불가

요청 바디

```json
{
  "nickname": "새닉네임"
}
```

유효성

- Pydantic: 길이 `2..10`
- Service regex: 영문/숫자/한글만 허용

성공 응답

| HTTP | Body |
|---|---|
| `200` | `{"id":1,"nickname":"새닉네임"}` |

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `403` | 없음 | `{"detail":"Guest not allowed"}` |
| `400` | 없음 | `{"detail":"Invalid nickname"}` |
| `404` | 없음 | `{"detail":"User not found"}` |
| `409` | 없음 | `{"detail":"Nickname already exists"}` |
| `422` | 없음 | FastAPI validation error |

### `GET /api/users/online`

- 목적: 현재 온라인 사용자 목록 조회
- 인증: Access token 필요
- 요청 바디: 없음

성공 응답

```json
{
  "users": [
    {
      "id": "1",
      "nickname": "player1",
      "status": "lobby"
    }
  ]
}
```

`status` 값

- `lobby`
- `in_room`
- `playing`

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |

---

## 4. Lobby / Room

### `GET /api/rooms`

- 목적: 대기방 목록 조회
- 인증: 없음
- 요청 바디: 없음

쿼리

| Name | Type | Required | Description |
|---|---|---|---|
| `status` | string | no | 예: `waiting`, `playing` |
| `exclude_private` | boolean | no | 비공개 방 제외 여부 |
| `keyword` | string | no | 제목 substring 검색 |

성공 응답

```json
{
  "rooms": [
    {
      "id": "room-1234abcd",
      "title": "친구방",
      "status": "waiting",
      "current_players": 2,
      "max_players": 4,
      "is_private": false,
      "host_id": "1",
      "host_nickname": "host"
    }
  ],
  "total": 1
}
```

에러

- 구현상 custom error를 직접 던지지 않는다.

### `POST /api/rooms`

- 목적: 방 생성
- 인증: Access token 필요

요청 바디

```json
{
  "title": "친구방",
  "is_private": true,
  "password": "1234",
  "max_players": 4
}
```

유효성

- `title`: `1..30`
- `password`: 비공개 방이면 필요, 형식은 숫자 4자리
- `max_players`: `2..4`

권한

- 로그인 사용자 누구나 생성 가능

성공 응답

```json
{
  "id": "room-1234abcd",
  "title": "친구방",
  "status": "waiting",
  "current_players": 1,
  "max_players": 4,
  "is_private": true,
  "host_id": "1",
  "host_nickname": "host"
}
```

부가 동작

- Socket `lobby_updated(action=created)` 브로드캐스트

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | `USER_NOT_FOUND` | `ApiError` |
| `409` | `ALREADY_JOINED_ROOM` | `ApiError` |
| `400` | `ROOM_PASSWORD_REQUIRED` | `ApiError` |
| `422` | 없음 | FastAPI validation error |

### `POST /api/rooms/{room_id}/join`

- 목적: 방 입장
- 인증: Access token 필요
- 권한: 방 멤버 또는 신규 입장자

경로 파라미터

| Name | Type | Required |
|---|---|---|
| `room_id` | string | yes |

요청 바디

```json
{
  "password": "1234"
}
```

비공개 방이 아니면 body 없이 호출 가능하다.

성공 응답

```json
{
  "room_id": "room-1234abcd",
  "title": "친구방",
  "status": "waiting",
  "max_players": 4,
  "is_private": true,
  "players": [
    {
      "id": "1",
      "nickname": "host",
      "is_ready": false,
      "is_host": true
    },
    {
      "id": "2",
      "nickname": "guest",
      "is_ready": false,
      "is_host": false
    }
  ],
  "chat_messages": []
}
```

부가 동작

- Socket `lobby_updated(action=updated)`
- Socket `room_updated`

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | `ROOM_NOT_FOUND` | `ApiError` |
| `404` | `USER_NOT_FOUND` | `ApiError` |
| `409` | `ALREADY_JOINED_ROOM` | `ApiError` |
| `409` | `ROOM_NOT_READY_TO_START` | `ApiError` |
| `409` | `ROOM_FULL` | `ApiError` |
| `400` | `ROOM_PASSWORD_REQUIRED` | `ApiError` |
| `403` | `ROOM_PASSWORD_MISMATCH` | `ApiError` |
| `422` | 없음 | FastAPI validation error |

### `POST /api/rooms/{room_id}/leave`

- 목적: 방 퇴장
- 인증: Access token 필요
- 권한: 해당 방 멤버만 가능

성공 응답

```json
{
  "success": true,
  "new_host_id": "2"
}
```

`new_host_id`

- 방이 비면 `null`
- 호스트 변경이 없으면 `null`
- 호스트가 나가서 위임되면 새 호스트 ID 문자열

부가 동작

- 방이 비면 `lobby_updated(action=removed)`
- 아니면 `lobby_updated(action=updated)`, `room_updated`, 필요 시 `host_changed`

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | `ROOM_NOT_FOUND` | `ApiError` |
| `403` | `NOT_ROOM_MEMBER` | `ApiError` |

### `PATCH /api/rooms/{room_id}/ready`

- 목적: 준비 상태 토글
- 인증: Access token 필요
- 권한: 방 멤버

성공 응답

```json
{
  "is_ready": true
}
```

구현 메모

- 호스트는 준비 토글 대상이 아니며 현재 구현은 에러 대신 `{"is_ready": false}`를 반환한다.

부가 동작

- Socket `player_ready`
- Socket `room_updated`

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | `ROOM_NOT_FOUND` | `ApiError` |
| `403` | `NOT_ROOM_MEMBER` | `ApiError` |

### `POST /api/rooms/{room_id}/start`

- 목적: 대기방 게임 시작
- 인증: Access token 필요
- 권한: 호스트만 가능

성공 응답

```json
{
  "success": true,
  "game_id": "17"
}
```

부가 동작

- DB `games`, `user_games` 생성
- Redis game state 초기화
- 참여자 presence를 `playing`으로 변경
- Socket `lobby_updated(action=status_changed)`
- Socket `game_start`를 `room:{room_id}`와 각 `user:{userId}`로 송신
- 서버 turn timer 시작

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | `ROOM_NOT_FOUND` | `ApiError` |
| `403` | `ONLY_HOST_CAN_START` | `ApiError` |
| `409` | `ROOM_NOT_READY_TO_START` | `ApiError` |

---

## 5. Game

### `GET /api/games/{game_id}`

- 목적: 게임 스냅샷 조회
- 인증: Access token 필요
- 권한: 현재 구현상 인증만 검사하며 게임 참가자 여부는 REST에서 별도 막지 않는다

경로 파라미터

| Name | Type | Required |
|---|---|---|
| `game_id` | string | yes |

성공 응답

아래 스냅샷은 Socket `game:patch.snapshot`과 동일한 형태다.

```json
{
  "roomId": "room-1234abcd",
  "gameId": "17",
  "revision": 12,
  "phase": "WAIT_PROMPT",
  "players": [
    {
      "playerId": "1",
      "nickname": "host",
      "currentTileId": 4,
      "balance": 4500,
      "ownedTiles": [4],
      "isInJail": false,
      "stateDuration": 0,
      "isBankrupt": false,
      "playerState": "normal",
      "color": "#EF5350"
    }
  ],
  "tiles": [
    {
      "id": 4,
      "name": "구산",
      "type": "PROPERTY",
      "ownerId": "1",
      "buildingLevel": 0,
      "price": 500
    }
  ],
  "currentPlayerId": "1",
  "round": 1,
  "turnTimeoutSec": 30,
  "prompt": {
    "id": "prompt-1234567890",
    "promptId": "prompt-1234567890",
    "type": "BUY_OR_SKIP",
    "playerId": "1",
    "title": "구산 purchase",
    "message": "Buy 구산 for 500만원?",
    "timeoutSec": 30,
    "choices": [
      {
        "id": "buy",
        "label": "구매",
        "value": "BUY"
      },
      {
        "id": "skip",
        "label": "건너뛰기",
        "value": "SKIP"
      }
    ],
    "payload": {
      "tileId": 4,
      "tileName": "구산",
      "price": 500,
      "buildingLevel": 0
    }
  },
  "isGameOver": false,
  "winnerId": null
}
```

에러

| HTTP | code | Body |
|---|---|---|
| `401` | 없음 | 공통 auth failure |
| `404` | `GAME_NOT_FOUND` | `{"code":"GAME_NOT_FOUND","message":"게임을 찾을 수 없습니다.","detail":null}` |

---

종료된 게임 스냅샷에서는 아래 필드가 함께 보장된다.

- `phase = "GAME_OVER"`
- `isGameOver = true`
- `winnerId = "<승자 playerId>"` 또는 승자 없음이면 `null`
- `prompt = null`

예시

```json
{
  "roomId": "room-1234abcd",
  "gameId": "17",
  "revision": 21,
  "phase": "GAME_OVER",
  "players": [],
  "tiles": [],
  "currentPlayerId": "1",
  "round": 8,
  "turnTimeoutSec": 30,
  "prompt": null,
  "isGameOver": true,
  "winnerId": "1"
}
```

## 6. Implementation Notes

- 금액 단위는 현재 백엔드 전역에서 `만원 단위 정수`를 사용한다.
- Room/Game ID는 문자열로 내려가지만, 일부 이벤트 payload의 `playerId`는 숫자로 내려간다. 프론트는 문자열/숫자 모두 허용해야 한다.
- 타인 소유 땅 도착 시 prompt flow는 `PAY_TOLL` 이후 `ACQUISITION_OR_SKIP`가 이어지는 2단계 구조다.
- 인수 prompt payload에는 `tileId`, `tileName`, `ownerId`, `ownerName`, `buildingLevel`, `acquisitionCost`, `toll`이 포함될 수 있다.
- 인수 금액은 "땅 가격 + 현재 건물 단계까지 들어간 전체 건설비"다.
- 게임 실시간 이벤트, prompt, patch, sync 규격은 [`gamesocket.md`](./gamesocket.md)를 기준으로 본다.
