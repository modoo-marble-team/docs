# API / WebSocket 명세 인덱스

## 문서 목적
- 이 문서는 공통 규칙과 상세 명세 문서의 진입점이다.
- 상세 계약은 도메인 문서에서 관리한다.

## 우선순위
문서 간 충돌 시 아래 순서를 따른다.
1. `docs/gamesocket.md`
2. `docs/lobby.md`
3. `docs/dm.md`
4. `docs/api.md`
5. 레거시 구현 동작

## 공통 규칙
| 항목 | 값 |
|---|---|
| REST Base URL | `/api` |
| Socket URL | `VITE_SOCKET_URL` (기본 `http://localhost:3000`) |
| REST 인증 | `Authorization: Bearer {accessToken}` |
| Socket 인증 | 연결 전 `socket.auth = { token: accessToken }` |
| Content-Type | `application/json` |
| 시간 포맷 | ISO-8601 UTC |
| 금액 단위 | 전 구간 **만원 단위 정수** |

### 공통 에러 포맷
```json
{
  "code": "STRING_ERROR_CODE",
  "message": "사용자 노출 메시지",
  "detail": "선택 상세 정보"
}
```

## 상세 문서
- 로비/대기방/프레즌스: `docs/lobby.md`
- DM 실시간 전용(무영속): `docs/dm.md`
- 게임 본플레이 소켓: `docs/gamesocket.md` (섹션 `6. FE/BE 계약 고정안 (2026-03-10)` 참고)
