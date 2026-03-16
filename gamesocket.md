## 0. 구현 가이드(필수 규칙)

1) 서버는 게임 단위로 액션을 **순차 처리**한다(큐/락 권장).
2) 클라이언트는 `revision` 기준으로 **중복/역순 패킷을 무시**한다.
3) 입력 UX
   - `game:ack`에서 `ok=false`면 즉시 UI 에러 표시
   - 내 턴이 아니거나 phase가 맞지 않으면 서버가 거절(`NOT_YOUR_TURN`, `INVALID_PHASE`)
4) 재접속 시 클라이언트는 `game:sync`를 호출하고 `snapshot`/누락 patch로 복구한다.

## 1. 전송 방식 / 룸(Room) 규칙

### 1.1 Socket.io 이벤트 이름

#### 클라이언트 -> 서버

- `game:action` : 게임 행위(의도) 요청
- `game:sync` : 재접속/동기화 요청
- `game:prompt_response` : 서버가 보낸 개인 선택(prompt)에 대한 응답

#### 서버 -> 클라이언트

- `game:ack` : 요청 처리 결과(요청자에게만)
- `game:patch` : 상태 변경(게임 룸 전체 브로드캐스트)
- `game:prompt` : 특정 플레이어에게만 필요한 선택 요청(개인 룸)
- `game:error` : 치명 오류/강제 동기화 필요 등(선택)

### 1.2 룸(Room)

- 게임 룸: `game:{gameId}`
  - 해당 게임 참가자 전원에게 브로드캐스트
- 개인 룸: `user:{userId}`
  - 특정 플레이어에게만 보내는 메시지(개인 선택, 비공개 정보 등)

### 1.3 서버 권위(Server authoritative)

- 클라이언트는 “의도(intent)”만 전송한다.
- 주사위 결과, 이동, 돈 계산, 소유권/건물 단계 등 '결과(outcome)'는 서버가 확정한다.
- 클라이언트가 결과를 보내는 형태는 금지한다.

---

## 2. 공통 규칙 / 데이터 타입

### 2.1 숫자/문자 타입 규칙

- 아래 값들은 **문자열이 아니라 숫자(number)** 로 보낸다:
  - `turn`, `revision`, `tileId`, `playerId`, `tier`, `balance`, `amount`, `stateDuration`, `buildingLevel`
- 돈 단위
  - 전 구간 **만원 단위 정수**로 고정한다.
  - 예: `5000 = 5000만원`
  - 프론트/백엔드/스토리지/로그 전 구간에 동일하게 적용한다.

### 2.2 Enum 정의

#### PlayerState

- `NORMAL` : 일반 상태
- `LOCKED` : 섬에 갇힘 (정확한 정의: 더블이 나와야 타일을 움직이는 등의 행동이 가능한 상태)

#### TileType

- `PROPERTY` : 땅/건물 칸
- `EVENT` : 이벤트 칸
- `CHANCE` : 찬스 칸
- `MOVE_TO_ISLAND` : 섬으로 이동 칸
- `ISLAND` : 섬 칸
- `START` : 시작 칸

#### BuildingLevel (서버 권위, 자동 구매/판매 단계)

- `0` : 땅만(토지)
- `1..3` : 주택 1,2,3
- `4..6` : 호텔 1,2,3
- `7` : 랜드마크

#### ActionType (클라 -> 서버)

- `ROLL_DICE`
- `BUY_PROPERTY`
- `SELL_PROPERTY`
- `END_TURN`

#### ServerEventType (서버 -> 클라, 연출/로그용)

- `DICE_ROLLED`
- `PLAYER_MOVED`
- `LANDED`
- `PAID_TOLL`
- `BOUGHT_PROPERTY`
- `SOLD_PROPERTY`
- `TURN_ENDED`
- `PLAYER_STATE_CHANGED`
- `CHANCE_RESOLVED`
- `SYNCED`

---

## 3. 클라이언트 -> 서버

## 3.1 `game:action`

목적: 게임 행위(의도)를 서버에 요청한다.

### 3.1.1 공통 Envelope(필수)

```json
{
  "gameId": "g1",
  "actionId": "uuid-123",
  "type": "ROLL_DICE",
  "payload": {}
}
```

- `gameId` : 게임 식별자
- `actionId` : 요청 식별자(UUID 권장). 요청자 ACK 매칭용
- `type` : ActionType
- `payload` : 액션별 입력 데이터

서버 응답 흐름(권장):

1) 요청자에게 `game:ack` (개인 룸/해당 소켓)
2) 게임 룸에 `game:patch` 브로드캐스트

---

### 3.1.2 액션별 payload

#### A) ROLL_DICE

```json
{
  "gameId": "g1",
  "actionId": "uuid-1",
  "type": "ROLL_DICE",
  "payload": {
    "diceId": 0
  }
}
```

- `diceId`는 구현목록에는 없으나(현시점 0으로 고정) 113466 같은 주사위 세트 추가 가능

#### B) BUY_PROPERTY

```json
{
  "gameId": "g1",
  "actionId": "uuid-2",
  "type": "BUY_PROPERTY",
  "payload": {
    "tileId": 15
  }
}
```

- 서버가 현재 `buildingLevel`을 기준으로 다음 단계를 자동 구매한다(서버 권위).

#### C) SELL_PROPERTY

```json
{
  "gameId": "g1",
  "actionId": "uuid-3",
  "type": "SELL_PROPERTY",
  "payload": {
    "tileId": 15,
    "buildingLevel": 0
  }
}
```

- `buildingLevel` : 가장 높은 단계의 건축물을 자동으로 판매한다(서버 권위).

#### D) END_TURN

```json
{
  "gameId": "g1",
  "actionId": "uuid-4",
  "type": "END_TURN",
  "payload": {}
}
```

---

## 3.2 `game:sync`

목적: 재접속 / 동기화 / 상태 복구.

```json
{
  "gameId": "g1",
  "knownRevision": 40
}
```

- 클라이언트가 알고 있는 마지막 `revision`을 보낸다.
- 서버는 다음 중 하나로 응답한다:
  - 누락된 변경분만 `game:patch`로 보내기
  - 또는 `snapshot` 포함한 `game:patch`로 전체 상태 보내기

---

## 3.3 `game:prompt_response`

목적: 서버가 보낸 개인 선택(prompt)에 응답한다.

```json
{
  "gameId": "g1",
  "promptId": "pr-9",
  "choice": "BUY",
  "payload": {}
}
```

- `promptId` : 서버가 준 선택 요청 ID
- `choice` : 프롬프트 타입별 canonical 선택 값(섹션 `6.1 Prompt Choice Canonical`)을 사용
- 서버는 처리 후 `game:ack` + `game:patch`(브로드캐스트)로 결과를 확정한다.

---

## 4. 서버 -> 클라이언트

## 4.1 `game:ack` (요청자에게만)

목적: 요청의 성공/실패를 빠르게 알려준다(입력 UI 제어/에러 표시용).

성공 예시:

```json
{
  "gameId": "g1",
  "actionId": "uuid-1",
  "ok": true,
  "error": null,
  "revision": 42
}
```

실패 예시:

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

권장 에러 코드 (확정 X):

- `NOT_IN_GAME` : 게임 참가자가 아님
- `NOT_YOUR_TURN` : 플레이어 턴이 아님
- `INVALID_PHASE` : 현재 단계에서 불가능한 행동
- `INVALID_TILE` : 잘못된 타일 ID
- `INSUFFICIENT_FUNDS` : 잔액 부족
- `NOT_OWNER` : 소유자가 아님
- `UNKNOWN_ACTION` : 알 수 없는 액션

---

## 4.2 `game:patch` (게임 룸 브로드캐스트)

목적: 서버 권위 상태 변경을 모든 클라이언트에 전달한다.

### 4.2.1 공통 Envelope

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

- `revision` : 서버가 증가시키는 순번(단조 증가). 클라는 `revision`이 작거나 같은 패킷은 무시한다.
- `turn` : 현재 턴 수(게임 규칙에 따라 정의)
- `events` : 연출/로그/애니메이션용 이벤트(순서 중요)
- `patch` : 상태 변경 최소 단위(커스텀 패치 포맷)
- `snapshot` : 전체 상태(동기화/첫 진입 시 사용). 있으면 로컬 상태를 통째로 교체한다.

---

### 4.2.2 patch 오퍼레이션(커스텀 포맷)

```json
{
  "op": "set",
  "path": "players.1.balance",
  "value": 950000
}
```

op:

- `set` : 값 설정
- `inc` : 숫자 증감
- `push` : 배열에 추가
- `remove` : 배열/키 삭제

---

### 4.2.3 예시: 주사위 -> 이동 -> 부동산 칸 도달 -> 통행료 지불

```json
{
  "gameId": "g1",
  "revision": 10,
  "turn": 10,
  "events": [
    { "type": "DICE_ROLLED", "playerId": 1, "dice": [5, 5] },
    { "type": "PLAYER_MOVED", "playerId": 1, "fromTileId": 10, "toTileId": 20 },
    {
      "type": "LANDED",
      "playerId": 1,
      "tile": {
        "id": 20,
        "name": "강남",
        "tier": 6,
        "tileType": "PROPERTY",
        "ownerId": 3,
        "buildingLevel": 0,
        "toll": 5000
      }
    },
    { "type": "PAID_TOLL", "fromPlayerId": 1, "toPlayerId": 3, "amount": 5000 }
  ],
  "patch": [
    { "op": "set", "path": "players.1.currentTileId", "value": 20 },
    { "op": "inc", "path": "players.1.balance", "value": -5000 },
    { "op": "inc", "path": "players.3.balance", "value": 5000 }
  ],
  "snapshot": null
}
```

---

### 4.2.4 예시: 주사위 -> 이동 -> 찬스칸 도달 -> 찬스 적용

```json
{
  "gameId": "g1",
  "revision": 11,
  "turn": 10,
  "events": [
    { "type": "DICE_ROLLED", "playerId": 1, "dice": [1, 2] },
    { "type": "PLAYER_MOVED", "playerId": 1, "fromTileId": 18, "toTileId": 21 },
    {
      "type": "CHANCE_RESOLVED",
      "playerId": 1,
      "tileId": 21,
      "chance": {
        "type": "GAIN_MONEY",
        "power": 5000
      }
    }
  ],
  "patch": [
    { "op": "set", "path": "players.1.currentTileId", "value": 21 },
    { "op": "inc", "path": "players.1.balance", "value": 5000 }
  ],
  "snapshot": null
}
```

찬스 타입 예시:

- `GAIN_MONEY`, `LOSE_MONEY`, `MOVE_FORWARD`, `MOVE_BACKWARD`, `STEAL_PROPERTY`, `GIVE_PROPERTY`

---

## 4.3 `game:prompt` (개인 룸으로만)

목적: 특정 플레이어의 선택이 필요한 상황에서 서버가 선택지를 요청한다.

예: 구매 선택 요청

```json
{
  "gameId": "g1",
  "revision": 12,
  "turn": 10,
  "promptId": "pr-9",
  "type": "BUY_OR_SKIP",
  "payload": {
    "tileId": 20,
    "price": 3000,
    "timeoutMs": 15000
  }
}
```

- 프롬프트를 받은 플레이어는 `game:prompt_response`로 응답한다.

---

## 4.4 `game:error` (선택)

목적: 액션 단위가 아닌 치명 오류/동기화 필요 상황 알림.

```json
{
  "gameId": "g1",
  "code": "DESYNC",
  "message": "클라이언트 상태가 오래되었습니다. sync가 필요합니다."
}
```

---

## 5. 스냅샷(Snapshot) 규격 (동기화/첫 접속)

`game:patch`에서 `snapshot`이 존재하면, 클라이언트는 로컬 상태를 snapshot으로 “교체”한다.

```json
{
  "gameId": "g1",
  "revision": 40,
  "turn": 10,
  "events": [{ "type": "SYNCED" }],
  "patch": [],
  "snapshot": {
    "turn": 10,
    "currentPlayerId": 1,
    "phase": "WAIT_ROLL",
    "players": [
      {
        "id": 1,
        "name": "홍길동",
        "balance": 1050000,
        "currentTileId": 5,
        "playerState": "NORMAL",
        "stateDuration": 0,
        "ownedTiles": [
          { "tileId": 5, "buildingLevel": 0 },
          { "tileId": 10, "buildingLevel": 0 }
        ]
      },
      {
        "id": 2,
        "name": "김철수",
        "balance": 800000,
        "currentTileId": 12,
        "playerState": "LOCKED",
        "stateDuration": 3,
        "ownedTiles": []
      }
    ],
    "tiles": [
      {
        "id": 20,
        "name": "강남",
        "tier": 6,
        "tileType": "PROPERTY",
        "ownerId": 3,
        "buildingLevel": 0,
        "toll": 5000
      }
    ]
  }
}
```

---

## 6. FE/BE 계약 고정안 (2026-03-10)

| 항목                                                             | 확정값                                                                                                                                                                                                                                                   | 임시호환 필요여부   | 종료 시점             |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- | --------------------- |
| 식별자(`game:action`, `game:sync`, `game:prompt_response`) | `gameId`만 허용. `roomId`는 게임 소켓 payload에서 미사용                                                                                                                                                                                             | 불필요              | 즉시(2026-03-10)      |
| ActionType(건설 포함)                                            | `ROLL_DICE`, `BUY_PROPERTY`, `SELL_PROPERTY`, `END_TURN` 고정. 건설은 `BUY_PROPERTY` 재사용(자동 단계 상승). `BUILD_PROPERTY` 미지원                                                                                                         | 불필요              | 즉시(2026-03-10)      |
| `game:prompt` 식별자                                           | `promptId`가 정식 필드. `id`는 레거시 별칭으로만 허용                                                                                                                                                                                                | 필요                | 2026-03-20 (W12 종료) |
| `game:prompt` 타임아웃 필드                                    | 정식은 `timeoutSec`. `payload.timeoutMs`는 레거시 별칭                                                                                                                                                                                               | 필요                | 2026-03-20 (W12 종료) |
| `game:prompt`의 `choices`                                    | 선택형 프롬프트(`BUY_OR_SKIP` 등)에서는 필수, 알림형 프롬프트에서는 선택                                                                                                                                                                               | 불필요              | 즉시(2026-03-10)      |
| `game:prompt_response` 이후 `game:ack`                       | 프롬프트 응답에 대한 `ack`에는 `promptId`를 포함해야 함                                                                                                                                                                                              | 불필요              | 즉시(2026-03-10)      |
| 프롬프트 실패 코드 규칙                                          | `PROMPT_NOT_FOUND`, `PROMPT_EXPIRED`, `INVALID_PROMPT_CHOICE`, `NOT_PROMPT_OWNER`, `INVALID_PHASE` 사용                                                                                                                                        | 불필요              | 즉시(2026-03-10)      |
| snapshot 최종 스키마                                             | `snapshot.turn`, `snapshot.currentPlayerId`, `snapshot.phase`, `snapshot.players[]`, `snapshot.tiles[]` 고정. 플레이어 위치=`currentTileId`, 소유권=`ownedTiles[]` + `tiles[].ownerId`, 상태=`playerState`                             | 불필요              | 즉시(2026-03-10)      |
| PlayerState 최종값                                               | `NORMAL`, `LOCKED`, `BANKRUPT`                                                                                                                                                                                                                     | 불필요              | 즉시(2026-03-10)      |
| phase 최종값                                                     | `WAIT_ROLL`, `MOVING`, `RESOLVING`, `WAIT_PROMPT`, `TURN_END`, `GAME_OVER`                                                                                                                                                                   | 불필요              | 즉시(2026-03-10)      |
| ServerEventType 최종값                                           | `DICE_ROLLED`, `PLAYER_MOVED`, `LANDED`, `PAID_TOLL`, `BOUGHT_PROPERTY`, `SOLD_PROPERTY`, `TURN_ENDED`, `PLAYER_STATE_CHANGED`, `CHANCE_RESOLVED`, `SYNCED` (`TURN_ENDED` 포함)                                                    | 불필요              | 즉시(2026-03-10)      |
| revision / sync 정책                                             | `knownRevision`과 서버 `revision` 동일 시 `SYNCED` 이벤트만 포함된 빈 patch 가능. 누락분 기준 단위는 **revision count**이며 `serverRevision - knownRevision > 200`이면 snapshot 강제 반환(또는 서버 보관분 없음/클라 revision 미래값 포함) | 불필요              | 즉시(2026-03-10)      |
| out-of-order 패킷 처리                                           | 클라는 `revision <= localRevision` 패킷을 무시                                                                                                                                                                                                         | 불필요              | 즉시(2026-03-10)      |
| money 단위                                                       | canonical unit은**만원 단위 정수**. BE 전 구간 동일 적용                                                                                                                                                                                           | 불필요              | 즉시(2026-03-10)      |
| `PAID_TOLL` 이후 인수 플로우                                   | v1 미지원(인수 prompt 미발행). 랜드마크 제외/건물 승계 규칙은 차기 버전에서 별도 추가                                                                                                                                                                    | 불필요              | 즉시(2026-03-10)      |
| 임시 호환 종료 계획(요약)                                        | `roomId fallback`은 즉시 종료. 프롬프트 레거시 별칭(`id`, `timeoutMs`)만 한시 지원                                                                                                                                                                 | 필요(별칭 2개 한정) | 2026-03-20 (W12 종료) |

### 6.1 Prompt Choice Canonical

| prompt type      | choice canonical 값 | 비고               |
| ---------------- | ------------------- | ------------------ |
| `BUY_OR_SKIP`  | `BUY`, `SKIP`   | 기본 매수 프롬프트 |
| `CONFIRM_ONLY` | `CONFIRM`         | 확인 전용 프롬프트 |

- 위 표에 없는 조합은 `INVALID_PROMPT_CHOICE`로 거절한다.
- `choice` 값은 대문자 스네이크/상수형 문자열로 고정한다.

### prompt/snapshot 필드 요약

```json
{
  "game:prompt": {
    "promptId": "pr-1",
    "type": "BUY_OR_SKIP",
    "timeoutSec": 15,
    "choices": [{"id":"BUY","label":"구매"}]
  },
  "game:ack": {
    "gameId": "g1",
    "actionId": "uuid-1",
    "promptId": "pr-1",
    "ok": true,
    "error": null
  },
  "snapshot": {
    "turn": 10,
    "currentPlayerId": 1,
    "phase": "WAIT_ROLL",
    "players": [
      {
        "id": 1,
        "balance": 1050000,
        "currentTileId": 5,
        "playerState": "NORMAL",
        "stateDuration": 0,
        "ownedTiles": [{"tileId": 5, "buildingLevel": 0}]
      }
    ],
    "tiles": [
      {
        "id": 5,
        "ownerId": 1,
        "buildingLevel": 0,
        "tileType": "PROPERTY",
        "toll": 50000
      }
    ]
  }
}
```
