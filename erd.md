```mermaid
erDiagram
    USERS ||--o{ GAMES : "winner_id"
    USERS ||--o{ USER_GAMES : "user_id"
    GAMES ||--o{ USER_GAMES : "game_id"

    USERS {
        INT id PK
        VARCHAR kakao_id "UNIQUE"
        VARCHAR nickname
        VARCHAR hashed_password
        TEXT profile_image_url
        TIMESTAMP created_at
        TIMESTAMP updated_at
    }

    GAMES {
        INT id PK
        INT winner_id
        SMALLINT round_count
        TIMESTAMP created_at
    }

    USER_GAMES {
        INT id PK
        INT user_id
        INT game_id
        BIGINT tolls_paid
        SMALLINT tiles_purchased
        SMALLINT buildings_built
        SMALLINT placement
    }
```
