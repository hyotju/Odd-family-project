# 괴짜가족 챗봇 AI — 사용자 데이터 SQL 설계 (제안)

> **주의**: 이 문서는 "만약 이 프로젝트에 SQL을 적용했다면 어떻게 설계했을지"에 대한 **가정 기반 제안**입니다.
> 실제 리포지토리(`4-base/`)에는 SQL/RDBMS가 사용되지 않으며, 게임 상태는 `chatbot_service.py`의
> `self.sessions = {}` 인메모리 딕셔너리로만 관리되어 서버 재시작 시 초기화됩니다.
> 이력서 등에 활용 시 "구현함"이 아니라 "설계 관점에서 제안함"으로 표현하는 것을 권장합니다.

## 배경: 실제 코드에서 관리되는 상태 값

- `session_id`(=username), `chapter`, `cleared[]`, `game_over`, `data` — `chatbot_service.py:71-80`
- 챕터3 힌트 모드 `hint_mode` — `chatbot_service.py:668-680`
- 실패 시 `fail_id` / `combo_key` — `chatbot_service.py:787`, `app.py:137-165`
- 챕터2 설득 점수 `score` — `chatbot_service.py:558-560`

이 값들을 영속화한다고 가정하고 아래와 같이 테이블을 설계할 수 있습니다.

---

## 1. 테이블 설계 (CREATE TABLE)

```sql
-- 사용자
CREATE TABLE users (
    user_id     INTEGER PRIMARY KEY AUTOINCREMENT,
    username    VARCHAR(50) NOT NULL UNIQUE,
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 현재 진행 상태 (in-memory self.sessions 딕셔너리를 대체)
CREATE TABLE game_sessions (
    session_id      INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         INTEGER NOT NULL UNIQUE REFERENCES users(user_id),
    current_chapter INTEGER NOT NULL DEFAULT 1,
    current_step    INTEGER NOT NULL DEFAULT 0,
    game_over       BOOLEAN NOT NULL DEFAULT 0,
    state_data      JSON,          -- ch1/ch2/ch3 핸들러별 임시 상태(meat, heat, score 등)
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 클리어한 챕터 이력 (state["cleared"] 리스트를 정규화)
CREATE TABLE chapter_clears (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER NOT NULL REFERENCES users(user_id),
    chapter     INTEGER NOT NULL,
    cleared_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, chapter)
);

-- 챕터별 성공/실패 기록 (분석용 로그)
CREATE TABLE play_logs (
    log_id      INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER NOT NULL REFERENCES users(user_id),
    chapter     INTEGER NOT NULL,
    result      VARCHAR(10) NOT NULL CHECK (result IN ('success','fail')),
    fail_id     VARCHAR(30),       -- 'CH1_FAIL', 'CH3_20대_동안_통통_귀여운' 등
    score       INTEGER,           -- 챕터2 설득 점수
    used_ml     INTEGER,           -- 챕터3 사용 용량
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 챕터3 힌트 사용 이력
CREATE TABLE hint_usage (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER NOT NULL REFERENCES users(user_id),
    chapter     INTEGER NOT NULL,
    category    VARCHAR(10),       -- 나이/얼굴/몸매/스타일
    used_at     DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 2. 실제 코드 흐름에 대응하는 쿼리

### `"init"` 입력 시 — 신규/기존 유저 처리
(`chatbot_service.py:798-801` 대체)

```sql
INSERT INTO users (username) VALUES (?)
ON CONFLICT(username) DO NOTHING;

INSERT INTO game_sessions (user_id, current_chapter, current_step, game_over, state_data)
SELECT user_id, 1, 0, 0, '{}' FROM users WHERE username = ?
ON CONFLICT(user_id) DO UPDATE SET
    current_chapter = 1, current_step = 0, game_over = 0, state_data = '{}';
```

### 매 턴마다 상태 갱신
(`_get_state` 및 각 챕터 핸들러가 `state`를 mutate하는 부분 대체)

```sql
UPDATE game_sessions
SET current_chapter = ?, current_step = ?, state_data = ?, updated_at = CURRENT_TIMESTAMP
WHERE user_id = (SELECT user_id FROM users WHERE username = ?);
```

### 챕터 클리어 시
(`chatbot_service.py:273-274` 등 `state["cleared"].append(...)` 대체)

```sql
INSERT INTO chapter_clears (user_id, chapter)
VALUES ((SELECT user_id FROM users WHERE username = ?), ?)
ON CONFLICT DO NOTHING;

INSERT INTO play_logs (user_id, chapter, result, score, used_ml)
VALUES ((SELECT user_id FROM users WHERE username = ?), ?, 'success', ?, ?);
```

### 실패 시
(`/fail?id=CH3_...` 라우트, `app.py:137-154` 대체)

```sql
INSERT INTO play_logs (user_id, chapter, result, fail_id)
VALUES ((SELECT user_id FROM users WHERE username = ?), ?, 'fail', ?);
```

---

## 3. 포트폴리오용 분석 쿼리

단순 CRUD보다 집계 쿼리가 SQL 역량 어필에 더 설득력이 있습니다.

### 챕터별 이탈률 (퍼널 분석)

```sql
SELECT
    chapter,
    COUNT(*) AS attempts,
    SUM(CASE WHEN result = 'fail' THEN 1 ELSE 0 END) AS fails,
    ROUND(100.0 * SUM(CASE WHEN result = 'fail' THEN 1 ELSE 0 END) / COUNT(*), 1) AS fail_rate_pct
FROM play_logs
GROUP BY chapter
ORDER BY chapter;
```

### 챕터3에서 가장 많이 틀리는 실패 조합(엔딩) Top 5

```sql
SELECT fail_id, COUNT(*) AS cnt
FROM play_logs
WHERE chapter = 3 AND result = 'fail'
GROUP BY fail_id
ORDER BY cnt DESC
LIMIT 5;
```

### 힌트를 쓴 유저와 안 쓴 유저의 클리어율 비교

```sql
SELECT
    CASE WHEN h.user_id IS NULL THEN '힌트 미사용' ELSE '힌트 사용' END AS hint_group,
    COUNT(DISTINCT c.user_id) AS cleared_users
FROM chapter_clears c
LEFT JOIN hint_usage h ON h.user_id = c.user_id AND h.chapter = 3
WHERE c.chapter = 3
GROUP BY hint_group;
```

---

## 참고: 현재 프로젝트의 실제 데이터 저장 방식

| 데이터 종류 | 실제 저장 방식 | 파일 |
|---|---|---|
| 게임 진행 상태 (챕터, 스텝, 점수 등) | 인메모리 딕셔너리 (`self.sessions`) | `4-base/services/chatbot_service.py` |
| 챕터 데이터/대사/선택지 | JSON 파일 | `4-base/static/data/chatbot/ch1_data.json`, `ch3_data.json` |
| 실패 엔딩 정보 | JSON 파일 | `4-base/static/data/chatbot/bad_ending.json`, `ch3_results.json` |
| RAG 검색용 벡터 | ChromaDB (벡터 DB) | `4-base/static/data/chatbot/chardb_embedding/` |

관계형 SQL 데이터베이스는 이 프로젝트에 실제로 존재하지 않으며, 위 설계는 순수 제안입니다.
