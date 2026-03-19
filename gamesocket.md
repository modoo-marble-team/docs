# Game Socket Specification

이 문서는 `backend/app/game/*`, `backend/app/main.py` 현재 구현 기준의 게임 소켓 명세다. 예전 문서와 달리 실제 동작하는 payload, sync 규칙, disconnect 처리까지 반영했다.

### Socket Authentication

연결 시 `socket.auth.token`에 access token을 넣어야 한다.

```ts
const socket = io(SOCKET_URL, {
  auth: {
    token: accessToken,
  },
})
```

인증 실패 시 connect 자체가 거절된다.

실패 조건

- token 누락
- JWT 파싱 실패
- `sub` 누락 또는 정수 변환 실패
- 삭제된 사용자 / 없는 사용자

---

## 1. Core Rules

### Server Authoritative

- 클라이언트는 결과가 아니라 의도만 보낸다.
- 주사위, 이동, 통행료, 기회/이벤트 카드, 파산, turn advance는 서버가 확정한다.
- 클라이언트가 상태를 로컬에서 확정하면 안 된다.

### Revision

- 서버 상태가 바뀔 때마다 `revision`이 1씩 증가한다.
- `game:sync`는 `knownRevision` 기반으로 diff replay 또는 snapshot을 돌려준다.
- `game:action`, `game:prompt_response`에 `knownRevision`을 같이 보낼 수 있다.
- action/prompt_response에서 `knownRevision`이 현재 서버 revision과 다르면 서버는 요청을 처리하지 않고 `game:error(code=DESYNC)`만 보낸다.

### Money Unit

- 현재 게임 금액은 `만원 단위 정수`
- 예: `500` = 500만원

### AFK Timeout Rule

- 각 턴에는 현재 `30초` 타임아웃이 있다.
- 타임아웃 시 서버가 턴을 자동 진행한다.
- `WAIT_ROLL` 상태에서 응답이 없으면 서버가 주사위를 자동으로 굴린다.
- 구매/건설/인수처럼 스킵 가능한 prompt는 자동으로 `SKIP` 한다.
- 통행료처럼 필수 처리 prompt는 서버가 강제로 처리한다.
- 여행처럼 추가 payload가 필요한 선택 prompt는 서버가 유효한 선택지 중 하나를 랜덤으로 골라 처리한다.
- 자동 처리 후 추가 prompt가 생기면 같은 규칙으로 연쇄 처리하고, 더 이상 prompt가 없으면 턴을 종료한다.

### Identifier Type

실구현은 ID 타입이 완전히 단일하지 않다.

- snapshot / prompt: `playerId`, `ownerId`, `currentPlayerId`는 주로 문자열
- event payload: `playerId`, `toPlayerId` 등은 주로 숫자

프론트는 문자열/숫자 둘 다 수용해야 한다.

---

## 2. Client -> Server

## 2.1 `game:sync`

목적

- 최초 진입
- 재연결
- desync 복구

요청

```json
{
  "gameId": "17",
  "knownRevision": 10
}
```

필드

| Field             | Type    | Required | Note                                |
| ----------------- | ------- | -------- | ----------------------------------- |
| `gameId`        | string  | yes      | 게임 ID                             |
| `knownRevision` | integer | no       | 생략 또는 `< 0`이면 full snapshot |

동작

- `knownRevision < 0` 또는 생략: full snapshot 1회 송신
- `knownRevision == currentRevision`: 빈 patch + `SYNCED` 이벤트 송신
- `knownRevision > currentRevision`: `game:error(code=DESYNC)` + full snapshot 송신
- `knownRevision < currentRevision`이고 patch log가 연속적이면 누락 patch들을 순서대로 replay
- patch log가 비어 있거나 중간이 비면 full snapshot 강제 송신

patch log 정책

- 최근 200개 revision 유지
- action 기반 patch와 timer 기반 patch 모두 저장

성공 응답

- 서버는 `game:patch`를 1회 이상 보낸다.
- active prompt가 있으면 추가로 `game:prompt`를 prompt owner에게 보낸다.

오류

| Event          | code                | 의미                           |
| -------------- | ------------------- | ------------------------------ |
| `game:error` | `AUTH_REQUIRED`   | 소켓 인증 없음                 |
| `game:error` | `INVALID_REQUEST` | `gameId` 누락                |
| `game:error` | `DESYNC`          | revision 불일치 또는 복구 필요 |

## 2.1.1 `game:sync_timer`

목적

- 브라우저 새로고침 / 탭 복구 후 남은 턴 시간을 서버 기준으로 다시 맞춤
- prompt가 열려 있다면 prompt 남은 시간도 함께 복구

요청

```json
{
  "gameId": "17"
}
```

동작

- 게임 멤버만 호출 가능하다.
- 서버는 현재 runtime timer 기준으로 `game:timer_sync`를 요청자에게만 보낸다.
- `game:sync`와 달리 state snapshot / patch replay는 하지 않는다.
- `prompt` 타이머 정보는 현재 prompt owner에게만 포함된다.

오류

| Event          | code                | 의미                    |
| -------------- | ------------------- | ----------------------- |
| `game:error` | `AUTH_REQUIRED`   | 소켓 인증 없음          |
| `game:error` | `INVALID_REQUEST` | `gameId` 누락         |
| `game:error` | `GAME_NOT_FOUND`  | 게임 상태 없음          |
| `game:error` | `NOT_GAME_MEMBER` | 해당 게임 참가자 아님   |

## 2.2 `game:action`

목적

- 일반 게임 액션 요청

공통 envelope

```json
{
  "gameId": "17",
  "actionId": "game-action-1742373000",
  "type": "ROLL_DICE",
  "knownRevision": 12,
  "payload": {}
}
```

필드

| Field             | Type    | Required | Note                     |
| ----------------- | ------- | -------- | ------------------------ |
| `gameId`        | string  | yes      | 게임 ID                  |
| `actionId`      | string  | yes      | 클라이언트 액션 식별자   |
| `type`          | string  | yes      | 아래 action type 참조    |
| `knownRevision` | integer | no       | 현재 클라이언트 revision |
| `payload`       | object  | no       | 액션별 payload           |

정식 action type

- `ROLL_DICE`
- `BUY_PROPERTY`
- `CITY_BUILD`
- `SELL_PROPERTY`
- `END_TURN`

호환 action type

- `TRAVEL`
  - 현재 백엔드는 `TRAVEL_SELECT` prompt가 떠 있는 상태에서만 허용한다.
  - 새 클라이언트는 가능하면 `game:prompt_response`를 우선 사용하고, `TRAVEL`은 레거시 호환으로만 본다.

### `ROLL_DICE`

```json
{
  "gameId": "17",
  "actionId": "a-1",
  "type": "ROLL_DICE",
  "payload": {
    "diceId": 0
  }
}
```

메모

- `diceId`는 현재 서버에서 의미 있게 사용하지 않는다.

### `BUY_PROPERTY`

```json
{
  "gameId": "17",
  "actionId": "a-2",
  "type": "BUY_PROPERTY",
  "payload": {
    "tileId": 4
  }
}
```

동작

- 해당 타일이 무주지면 매입
- 본인 소유 타일 건설은 `CITY_BUILD`를 사용
- 타인 소유 타일이면 거절

### `CITY_BUILD`

```json
{
  "gameId": "17",
  "actionId": "a-3",
  "type": "CITY_BUILD",
  "payload": {
    "tileId": 4
  }
}
```

동작

- 자기 턴의 `WAIT_ROLL`, `RESOLVING` 상태에서 허용
- 본인 소유 `PROPERTY` 타일만 다음 건설 단계로 1단계 증설
- 최대 단계(7)에 도달했거나 잔액이 부족하면 거절
- 성공해도 턴은 즉시 종료되지 않음
- 착지 후 뜨는 `BUILD_OR_SKIP` prompt 흐름과 별개로, 자기 턴에는 다른 소유 타일도 자유롭게 증설 가능
- 주사위를 이미 굴린 뒤에도 자기 턴 시간이 남아 있고 prompt가 없으면 계속 호출 가능

### `SELL_PROPERTY`

```json
{
  "gameId": "17",
  "actionId": "a-4",
  "type": "SELL_PROPERTY",
  "payload": {
    "tileId": 4,
    "buildingLevel": 0
  }
}
```

필드

- `buildingLevel` 생략 가능
- 생략 시 현재 단계 기준으로 1단계 매각 처리
- 자기 턴의 `WAIT_ROLL`, `RESOLVING` 상태에서 허용
- 성공해도 턴은 즉시 종료되지 않음
- 주사위를 이미 굴린 뒤에도 자기 턴 시간이 남아 있고 prompt가 없으면 계속 호출 가능

### `END_TURN`

```json
{
  "gameId": "17",
  "actionId": "a-5",
  "type": "END_TURN",
  "payload": {}
}
```

동작

- 자기 턴의 `RESOLVING` 상태에서만 허용
- 아직 주사위를 굴리지 않은 `WAIT_ROLL` 상태에서는 거절
- 현재 처리해야 할 prompt가 남아 있으면 거절
- 성공 시 다음 플레이어로 턴이 넘어가며 turn timer가 재시작

### `TRAVEL` (legacy compatibility)

```json
{
  "gameId": "17",
  "actionId": "a-6",
  "type": "TRAVEL",
  "payload": {
    "toIndex": 4
  }
}
```

허용 payload 키

- `targetTileId`
- `toTileId`
- `toIndex`

## 2.3 `game:prompt_response`

목적

- 서버 prompt 응답

요청

```json
{
  "gameId": "17",
  "promptId": "prompt-1234567890",
  "choice": "BUY",
  "knownRevision": 12,
  "payload": {}
}
```

필드

| Field             | Type    | Required | Note                       |
| ----------------- | ------- | -------- | -------------------------- |
| `gameId`        | string  | yes      | 게임 ID                    |
| `promptId`      | string  | yes      | prompt ID                  |
| `choice`        | string  | yes      | 대문자 canonical choice    |
| `knownRevision` | integer | no       | 현재 클라이언트 revision   |
| `payload`       | object  | no       | prompt type별 부가 payload |

주요 prompt별 choice

| Prompt Type       | Allowed choices       |
| ----------------- | --------------------- |
| `BUY_OR_SKIP`   | `BUY`, `SKIP`     |
| `BUILD_OR_SKIP` | `BUILD`, `SKIP`   |
| `PAY_TOLL`      | `PAY_TOLL`          |
| `ACQUISITION_OR_SKIP` | `ACQUIRE`, `SKIP` |
| `TRAVEL_SELECT` | `CONFIRM`, `SKIP` |

주요 prompt payload

### `BUY_OR_SKIP.payload`

```json
{
  "tileId": 4,
  "tileName": "군산",
  "price": 300,
  "buildingLevel": 0
}
```

### `BUILD_OR_SKIP.payload`

```json
{
  "tileId": 4,
  "tileName": "군산",
  "price": 180,
  "buildCost": 180,
  "buildingLevel": 0,
  "nextToll": 70
}
```

메모

- `price` 와 `buildCost` 는 현재 구현에서 같은 값을 가진다.
- 프론트는 건설 팝업에서 `buildCost`, `nextToll` 값을 그대로 표시해야 한다.
- `buildingLevel` 은 현재 단계이고, 실제 건설 후 단계는 `buildingLevel + 1` 이다.

### `PAY_TOLL.payload`

```json
{
  "tileId": 4,
  "tileName": "군산",
  "ownerId": 1,
  "ownerName": "host",
  "acquisitionCost": 700,
  "toll": 120,
  "amount": 120,
  "buildingLevel": 2
}
```

메모

- `toll` 과 `amount` 는 현재 구현에서 같은 값을 가진다.
- 프론트는 통행료 표시에서 `amount`, `toll` 둘 다 허용하는 편이 안전하다.

`TRAVEL_SELECT` confirm 예시

```json
{
  "gameId": "17",
  "promptId": "prompt-1234567890",
  "choice": "CONFIRM",
  "payload": {
    "targetTileId": 4
  }
}
```

타인 소유 땅 도착 flow

- 서버는 먼저 `PAY_TOLL` prompt를 보낸다.
- `PAY_TOLL` 성공 후, 플레이어가 파산하지 않았으면 같은 턴에 `ACQUISITION_OR_SKIP` prompt를 추가로 보낸다.
- `ACQUISITION_OR_SKIP.payload`에는 `tileId`, `tileName`, `ownerId`, `ownerName`, `buildingLevel`, `acquisitionCost`, `toll`이 들어간다.
- `ACQUIRE`는 "땅 가격 + 현재 건물 단계까지 들어간 전체 건설비"를 원주인에게 추가 지급하고 소유권과 건물 단계를 그대로 넘긴다.
- `SKIP`은 추가 정산 없이 prompt만 종료한다.

오류

| Event        | code                      | 의미                                   |
| ------------ | ------------------------- | -------------------------------------- |
| `game:ack` | `PROMPT_NOT_FOUND`      | active prompt 없음 / promptId mismatch |
| `game:ack` | `INVALID_PHASE`         | 현재 phase에서 prompt 처리 불가        |
| `game:ack` | `NOT_PROMPT_OWNER`      | 다른 플레이어 prompt                   |
| `game:ack` | `INVALID_PROMPT_CHOICE` | choice 불일치                          |
| `game:ack` | `INVALID_TILE`          | 여행 목적지 누락/범위 오류             |

---

## 3. Server -> Client

## 3.1 `game:ack`

목적

- action/prompt_response의 즉시 처리 결과
- 요청자에게만 전송

성공 예시

```json
{
  "gameId": "17",
  "actionId": "a-1",
  "type": "ROLL_DICE",
  "ok": true,
  "error": null,
  "revision": 13
}
```

prompt response 성공 예시

```json
{
  "gameId": "17",
  "actionId": "prompt:prompt-1234567890",
  "type": "PROMPT_RESPONSE",
  "ok": true,
  "error": null,
  "revision": 14,
  "promptId": "prompt-1234567890"
}
```

실패 예시

```json
{
  "gameId": "17",
  "actionId": "a-2",
  "type": "BUY_PROPERTY",
  "ok": false,
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Not enough funds."
  },
  "revision": 13,
  "promptId": null
}
```

주요 ack error codes

- `INVALID_REQUEST`
- `GAME_NOT_FOUND`
- `NOT_YOUR_TURN`
- `INVALID_PHASE`
- `INVALID_TILE`
- `INSUFFICIENT_FUNDS`
- `NOT_OWNER`
- `PROMPT_NOT_FOUND`
- `NOT_PROMPT_OWNER`
- `INVALID_PROMPT_CHOICE`
- `UNKNOWN_ACTION`
- `RETRY_LATER`

## 3.2 `game:patch`

목적

- authoritative state delta 또는 full snapshot

공통 envelope

```json
{
  "gameId": "17",
  "revision": 13,
  "turn": 4,
  "events": [],
  "patch": [],
  "snapshot": null
}
```

필드

| Field        | Type           | Required    | Note                                                   |
| ------------ | -------------- | ----------- | ------------------------------------------------------ |
| `gameId`   | string         | yes         |                                                        |
| `revision` | integer\| null | usually yes | disconnect/reconnect 브로드캐스트도 동일 envelope 사용 |
| `turn`     | integer\| null | usually yes |                                                        |
| `events`   | array          | yes         |                                                        |
| `patch`    | array          | yes         |                                                        |
| `snapshot` | object\| null  | yes         |                                                        |

patch operation

```json
{
  "op": "set",
  "path": "players.1.currentTileId",
  "value": 4
}
```

허용 op

- `set`
- `inc`
- `push`
- `remove`

## 3.3 `game:prompt`

목적

- 특정 플레이어에게만 선택 요청

예시

```json
{
  "id": "prompt-1234567890",
  "promptId": "prompt-1234567890",
  "type": "BUY_OR_SKIP",
  "playerId": "1",
  "title": "구산 구매",
  "message": "구산을 500만원에 구매하시겠습니까?",
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
}
```

실제 prompt type

- `BUY_OR_SKIP`
- `BUILD_OR_SKIP`
- `PAY_TOLL`
- `ACQUISITION_OR_SKIP`
- `TRAVEL_SELECT`

prompt type 메모

- `PAY_TOLL`
  - 타인 소유 땅 도착 시 먼저 뜨는 확인 prompt
  - choice는 `PAY_TOLL` 하나만 허용
  - 성공 후 플레이어가 파산하지 않았으면 `ACQUISITION_OR_SKIP`가 이어질 수 있다.
- `ACQUISITION_OR_SKIP`
  - 통행료 지불 직후 인수 여부를 고르는 prompt
  - choice는 `ACQUIRE`, `SKIP`
  - `payload.acquisitionCost`는 원주인에게 추가 지급할 인수 금액이다.

## 3.4 `game:timer_sync`

목적

- 서버 기준 현재 남은 턴 시간 / prompt 시간을 복구
- `game:sync_timer` 요청자에게만 전송

예시

```json
{
  "gameId": "17",
  "revision": 13,
  "serverTimeMs": 1742373000000,
  "turnDeadlineAtMs": 1742373030000,
  "turnRemainingMs": 30000,
  "turnRemainingSec": 30,
  "prompt": {
    "promptId": "prompt-1234567890",
    "type": "BUY_OR_SKIP",
    "timeoutSec": 30,
    "deadlineAtMs": 1742373021000,
    "remainingMs": 21000,
    "remainingSec": 21
  }
}
```

필드 메모

- `serverTimeMs`: 응답 생성 시점의 서버 epoch milliseconds
- `turnDeadlineAtMs`: 현재 턴 자동 처리 예정 시각. 타이머가 없으면 `null`
- `turnRemainingMs`, `turnRemainingSec`: 현재 턴 남은 시간. 타이머가 없으면 `null`
- `prompt`: 현재 사용자에게 열린 prompt가 있을 때만 포함, 아니면 `null`

프론트 처리 규칙

- 창 복구 시 먼저 `game:sync`로 상태를 맞추고, 이어서 `game:sync_timer`로 남은 시간을 덮어쓴다.
- 로컬 카운트다운을 신뢰하지 말고 `turnRemaining*` / `promptRemaining*` 값을 서버 기준으로 재설정해야 한다.

## 3.5 `game:error`

목적

- ack와 별개인 공통 오류 알림
- 주로 auth / desync / 잘못된 sync 요청

예시

```json
{
  "gameId": "17",
  "code": "DESYNC",
  "message": "클라이언트 상태가 오래되었습니다."
}
```

주요 error codes

- `AUTH_REQUIRED`
- `INVALID_REQUEST`
- `DESYNC`
- `NOT_GAME_MEMBER`

---

## 4. Snapshot Schema

`game:patch.snapshot`과 REST `GET /api/games/{game_id}`는 같은 shape를 사용한다.

```json
{
  "roomId": "room-1234abcd",
  "gameId": "17",
  "revision": 13,
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
  "prompt": null,
  "isGameOver": false,
  "winnerId": null
}
```

게임 종료 후 snapshot / full sync는 아래 규칙을 따른다.

- `phase`는 항상 `GAME_OVER`
- `isGameOver`는 항상 `true`
- `winnerId`는 승자 `playerId` 문자열이며 승자 없음이면 `null`
- `prompt`는 `null`

### Phase

현재 구현에서 실제로 쓰는 phase

- `WAIT_ROLL`
- `RESOLVING`
- `WAIT_PROMPT`
- `GAME_OVER`

참고

- 과거 문서에 있던 `MOVING`, `TURN_END`는 현재 백엔드 canonical phase가 아니다.

### Player State

snapshot의 `playerState`는 내부 enum과 달리 소문자 문자열로 직렬화된다.

- `normal`
- `locked`
- `bankrupt`

### Tile Type

snapshot의 `tiles[].type`은 board enum value 그대로 내려간다.

- `START`
- `PROPERTY`
- `EVENT`
- `CHANCE`
- `MOVE_TO_ISLAND`
- `ISLAND`
- `TRAVEL`
- `AI` 칸은 폐기됨

---

## 5. Event Types

현재 서버가 쓰는 이벤트 타입

- `DICE_ROLLED`
- `PLAYER_MOVED`
- `LANDED`
- `PAID_TOLL`
- `BOUGHT_PROPERTY`
- `ACQUIRED_PROPERTY`
- `SOLD_PROPERTY`
- `TURN_ENDED`
- `PLAYER_STATE_CHANGED`
- `CHANCE_RESOLVED`
- `SYNCED`
- `GAME_OVER`
- `PLAYER_DISCONNECTED`
- `PLAYER_RECONNECTED`

대표 payload 예시

### `DICE_ROLLED`

```json
{
  "type": "DICE_ROLLED",
  "playerId": 1,
  "dice": [2, 2],
  "isDouble": true
}
```

### `PLAYER_MOVED`

```json
{
  "type": "PLAYER_MOVED",
  "playerId": 1,
  "fromTileId": 0,
  "toTileId": 4,
  "trigger": "normal"
}
```

### `LANDED`

```json
{
  "type": "LANDED",
  "playerId": 1,
  "tile": {
    "tileId": 4,
    "name": "구산",
    "tileType": "PROPERTY",
    "tier": 1,
    "price": 500
  }
}
```

메모

- 현재 구현의 `LANDED` 는 타일 정보를 top-level `tileId` 로 내리지 않고 nested `tile` 객체 안에 담는다.
- 프론트는 `event.tile.tileId`, `event.tile.name`, `event.tile.tileType` 을 읽을 수 있어야 한다.
- `LANDED` 처리에서 top-level `tileId` 만 가정하면 `알 수 없는 칸` 같은 표시 오류가 날 수 있다.

### `PAID_TOLL`

```json
{
  "type": "PAID_TOLL",
  "fromPlayerId": 2,
  "toPlayerId": 1,
  "amount": 500,
  "tileId": 4
}
```

메모

- 타인 소유 땅 도착 시 `PAID_TOLL` 이후 같은 턴에 `ACQUISITION_OR_SKIP` prompt가 이어질 수 있다.
- 플레이어가 통행료로 파산하면 추가 인수 prompt는 보내지지 않는다.

### `CHANCE_RESOLVED`

```json
{
  "type": "CHANCE_RESOLVED",
  "playerId": 1,
  "tileId": 3,
  "chance": {
    "type": "GAIN_MONEY",
    "power": 300,
    "description": "보너스 300만원을 획득합니다."
  }
}
```

메모

- `CHANCE` 칸과 `EVENT` 칸 모두 현재 구현에서는 결과 이벤트 타입으로 `CHANCE_RESOLVED` 를 사용한다.
- 사용자에게 보여줄 카드 문구는 하드코딩된 기본 문자열이 아니라 `chance.description` 을 우선 사용해야 한다.
- `chance.type` 은 효과 분기용이고, 실제 카드 문구는 `chance.description` 이 기준이다.
- 일부 카드(`STEAL_PROPERTY`, `GIVE_PROPERTY`)는 `chance` 내부에 `fromPlayerId`, `toPlayerId`, `tileId` 같은 추가 필드가 들어갈 수 있다.

### `ACQUIRED_PROPERTY`

```json
{
  "type": "ACQUIRED_PROPERTY",
  "playerId": 2,
  "fromPlayerId": 1,
  "toPlayerId": 2,
  "tileId": 4,
  "amount": 800,
  "buildingLevel": 2
}
```

### `TURN_ENDED`

```json
{
  "type": "TURN_ENDED",
  "playerId": 1,
  "nextPlayerId": 2,
  "turn": 5,
  "round": 2
}
```

### `GAME_OVER`

last player standing 종료

```json
{
  "type": "GAME_OVER",
  "reason": "last_player_standing",
  "winner": {
    "playerId": 1,
    "nickname": "host",
    "balance": 6100
  }
}
```

일반 종료 / 라운드 종료

```json
{
  "type": "GAME_OVER",
  "reason": "max_rounds",
  "winner": {
    "playerId": 1,
    "nickname": "host",
    "balance": 6100
  }
}
```

disconnect timeout 종료

```json
{
  "type": "GAME_OVER",
  "reason": "disconnect_timeout",
  "winner": {
    "playerId": 2,
    "nickname": "guest",
    "balance": 5300
  }
}
```

### `PLAYER_DISCONNECTED`

```json
{
  "type": "PLAYER_DISCONNECTED",
  "playerId": 2,
  "timeoutSeconds": 60
}
```

### `PLAYER_RECONNECTED`

```json
{
  "type": "PLAYER_RECONNECTED",
  "playerId": 2
}
```

---

## 6. Disconnect / Reconnect Flow

### Disconnect

- 사용자의 마지막 game socket이 끊기면 서버는 해당 사용자가 active game에 있는지 확인한다.
- active game 참가자이고 파산 상태가 아니면:
  - disconnect 시각을 Redis에 기록
  - `PLAYER_DISCONNECTED` 이벤트를 `game:{gameId}` room에 브로드캐스트
  - 60초 유예 후에도 복귀하지 않으면 강제 파산 처리

### Reconnect

- 클라이언트가 다시 접속한 뒤 `game:sync`를 호출하면:
  - reconnect 예약이 제거된다
  - presence는 `playing`으로 복구된다
  - 필요한 patch/snapshot이 먼저 전송된다
  - 이후 `PLAYER_RECONNECTED` 이벤트가 `game:{gameId}` room에 브로드캐스트된다

### Disconnect Timeout Bankruptcy

유예 시간 만료 시 서버는 아래를 수행한다.

- 플레이어 잔액 `0`
- `playerState = BANKRUPT`
- `stateDuration = 0`
- `consecutiveDoubles = 0`
- 모든 소유 타일 해제
- `ownedTiles = []`
- `buildingLevels = {}`
- 현재 턴 플레이어였다면 다음 플레이어로 강제 턴 진행
- 생존 플레이어가 1명 이하가 되면 즉시 `GAME_OVER(reason=disconnect_timeout)`
- 게임이 종료되면 room 자체는 유지된다.
- 종료 직후 room은 `waiting` 상태로 복귀하고 `game_id` 는 비워진다.
- 연결 중인 참가자의 presence는 `in_room` 으로 복귀한다.
- game state / patch log / active game 매핑은 정리된다.

---

## 7. Current Implementation Notes

- `game:sync`는 소켓 핸들러와 runtime이 중복으로 patch를 보내지 않도록 정리되어 있다.
- turn timer가 발생시킨 자동 prompt 처리 / 자동 턴 종료 patch도 sync replay 대상에 포함된다.
- 게임 중 socket disconnect가 발생해도 room membership을 자동으로 제거하지 않는다. 대기방(`status=waiting`)일 때만 disconnect 기반 room 정리가 일어난다.
