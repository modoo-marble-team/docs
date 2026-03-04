## API 공통 규격

| 항목 | 값 |
|---|---|
| Base URL | `https://api.modoo-marble.com/v1` (개발: `http://localhost:8000/v1`) |
| 인증 방식 | `Authorization: Bearer {JWT}` — 모든 JWT 필요 API에 헤더 포함 |
| Content-Type | `application/json` (모든 요청/응답) |
| 토큰 만료 | 24시간. 만료 시 `401 Unauthorized` → 재로그인 |
| 에러 응답 구조 | 아래 참고 |
| 날짜 형식 | ISO 8601 UTC — 예: `2026-02-23T08:00:00Z` |
| 금액 단위 | 서버·API 모두 원(₩) 정수. FE에서 억/만원 단위 변환 표시 |

### 에러 응답(JSON)
```json
{
  "detail": "에러 메시지",
  "code": "ERROR_CODE"
}
```
## API 목록

### Auth
| ID | Method | Path | 설명 |
|---|---|---|---|
| [AUTH-001](jsons/AUTH-001.md) | POST | /auth/kakao/callback | 카카오 인가코드를 받아 액세스 토큰과 교환 후 유저 정보를 조회한다. |
| AUTH-002 | POST | /auth/guest | 로그인 없이 게스트 세션 토큰을 발급한다. |

### User
| ID | Method | Path | 설명 |
|---|---|---|---|
| USER-001 | GET | /users/me | 로그인한 유저의 닉네임과 프로필을 반환한다. |
| USER-002 | GET | /users/me/detail | 로그인한 유저의 프로필과 전적(승/패/총 게임 수)을 반환한다. |
| USER-003 | PATCH | /users/me/nickname | 신규 가입 후 닉네임을 설정한다. |

### Lobby
| ID | Method | Path | 설명 |
|---|---|---|---|
| LOBBY-001 | GET | /rooms | 로비 방 목록을 반환한다. |
| LOBBY-002 | POST | /rooms | 새 방을 생성한다. 비밀방 시 `password` 필수(숫자 4자리). |
| LOBBY-003 | POST | /rooms/{id}/join | 방에 참가한다. |
