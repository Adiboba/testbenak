# 2주차 활동지 — 코딩 에이전트와 컨텍스트

| | |
| :-- | :-- |
| 팀명 | gra|
| 작성일 |10.9 |
| 참여자 |adib |

---

## 0. 준비

 `memo-seed` 저장소를 엽니다. 다음 파일이 있는지 확인하세요.

- [X] `schema.sql`
- [X] `service.js`
- [X] `routes.js`
- [X] `CONVENTIONS.md`

---

## 1. 조 나누기

팀을 두 조로 나눕니다. (4인 → 2:2 / 3인 → 1:2)

| 조 | 참여자 |
| :-- | :-- |
| A조 | adib |
| B조 |adib |

**두 조는 같은 과제를 동시에 수행합니다.** 서로의 화면을 보지 마세요.

### 오늘의 과제 (두 조 공통)

> 메모 검색 기능을 추가하라. 제목과 본문에서 키워드로 찾을 수 있어야 한다.

---

## 2. 에이전트에게 준 것

### A조 — 이것만 붙여넣습니다

```
메모 검색 기능 만들어줘. 제목이랑 본문에서 키워드로 찾을 수 있게.
```

파일은 **하나도 주지 않습니다.**

### B조 — 네 칸을 모두 채웁니다

```
[지시]
메모 검색 기능을 추가해줘. 제목과 본문에서 키워드로 검색된다.

[규약]
(CONVENTIONS.md 내용 전체를 붙여넣기)

[근거]
(schema.sql, service.js, routes.js 내용 전체를 붙여넣기)

[종료조건]
- GET /memos/search?q=키워드 로 호출된다
- 제목 또는 본문에 키워드가 포함된 메모만 반환한다
- 본인 메모만 반환한다
- q가 비어 있으면 400과 { ok: false, error } 를 반환한다
```

### 실제로 붙여넣은 것 (원문 그대로, 요약 금지)

```
--- A조에 보낸 것 ---
메모 검색 기능 만들어줘. 제목이랑 본문에서 키워드로 찾을 수 있게.

--- B조에 보낸 것 ---
[지시]
메모 검색 기능을 추가해줘. 제목과 본문에서 키워드로 검색된다.

[규약]
# 프로젝트 규약
이 문서는 코드를 작성할 때 지켜야 할 규칙입니다.
## 계층 분리

routes.js는 HTTP 요청과 응답만 다룹니다. SQL을 직접 쓰지 않습니다.
데이터베이스 접근은 service.js에만 둡니다.
## 응답 형식
모든 응답은 다음 두 형태 중 하나입니다.
{ "ok": true,  "data": ... }
{ "ok": false, "error": "ERROR_CODE" }
에러 코드는 대문자와 밑줄로 씁니다. (예: MEMO_NOT_FOUND)
## 명명 규칙

함수명은 동사로 시작합니다. list, get, create, update, remove
데이터베이스 컬럼은 스네이크 케이스를 씁니다. user_id, created_at
자바스크립트 변수는 카멜 케이스를 씁니다. userId, createdAt
## 입력 검증

사용자 입력은 반드시 검증합니다.
검증에 실패하면 400과 함께 { ok: false, error } 를 반환합니다.
## 권한

모든 조회와 수정은 본인 소유 데이터로 한정합니다.
모든 쿼리에 user_id 조건을 포함합니다.
[근거]
--- schema.sql ---
-- memo-seed 데이터베이스 스키마
CREATE TABLE users (
  id         INTEGER PRIMARY KEY,
  email      TEXT NOT NULL UNIQUE,
  name       TEXT NOT NULL,
  created_at TEXT NOT NULL
);
CREATE TABLE memos (
  id         INTEGER PRIMARY KEY,
  user_id    INTEGER NOT NULL,
  title      TEXT NOT NULL,
  body       TEXT NOT NULL,
  created_at TEXT NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
CREATE INDEX idx_memos_user ON memos(user_id);
--- service.js ---
const db = require('./db');
/**
 * 사용자의 메모 목록을 최신순으로 조회한다.
 */
function listMemos(userId) {
  return db.all(
    SELECT id, title, created_at        FROM memos       WHERE user_id = ?       ORDER BY created_at DESC,
    [userId]
  );
}
/**
 * 메모 한 건을 조회한다. 본인 메모가 아니면 null을 반환한다.
 */
function getMemo(userId, memoId) {
  return db.get(
    SELECT id, title, body, created_at        FROM memos       WHERE id = ? AND user_id = ?,
    [memoId, userId]
  );
}
/**
 * 메모를 생성한다.
 */
function createMemo(userId, title, body) {
  return db.run(
    INSERT INTO memos (user_id, title, body, created_at)      VALUES (?, ?, ?, datetime('now')),
    [userId, title, body]
  );
}
module.exports = { listMemos, getMemo, createMemo };
--- routes.js ---
const express = require('express');
const service = require('./service');
const router = express.Router();
// 메모 목록 조회
router.get('/memos', async (req, res) => {
  const memos = await service.listMemos(req.user.id);
  res.json({ ok: true, data: memos });
});
// 메모 단건 조회
router.get('/memos/:id', async (req, res) => {
  const memo = await service.getMemo(req.user.id, req.params.id);
  if (!memo) {
    return res.status(404).json({ ok: false, error: 'MEMO_NOT_FOUND' });
  }
  res.json({ ok: true, data: memo });
});
// 메모 생성
router.post('/memos', async (req, res) => {
  const { title, body } = req.body;
  if (!title || !body) {
    return res.status(400).json({ ok: false, error: 'TITLE_AND_BODY_REQUIRED' });
  }
  const result = await service.createMemo(req.user.id, title, body);
  res.status(201).json({ ok: true, data: { id: result.lastID } });
});
module.exports = router;
[종료조건]
- GET /memos/search?q=키워드 로 호출된다
- 제목 또는 본문에 키워드가 포함된 메모만 반환한다
- 본인 메모만 반환한다
- q가 비어 있으면 400과 { ok: false, error } 를 반환한다

this>
```

> 요약하지 마세요. 나중에 이 기록이 무엇이 결과를 만들었는지 확인하는 근거가 됩니다.

---

## 3. 결과 확인
|group A|
| | 확인 항목 | 결과 |
| :-: | :-- | :-- |
| ① | 실행 성공까지 걸린 시간 | 2분 |
| ② | 없는 함수·컬럼을 지어낸 개수 | - |
| | → 지어낸 이름 |SQL을 쓰지 않아 컬럼을 지어낼 기회 자체가 없었다. 대신 프로젝트에 없는 구조(React 컴포넌트 상태, 브라우저 필터링)를 만들었다.|
| ③ | `CONVENTIONS.md` 위반 개수 | 4개 |
| | → 무엇을 어겼는가 |계층 분리 없음(service.js·routes.js 자체가 없음) / 응답 형식 { ok, data } 아님 / 에러 코드 없음 / 빈 입력 400 검증 없음 |
| ④ | 사람이 직접 고친 지점 |0곳 |
| | → 어디를 어떻게 |고칠 것이 없어서가 아니라 부분 수정이 성립하지 않기 때문이다. 백엔드 엔드포인트가 아니라 브라우저에서 동작하는 화면이므로 service.js·routes.js에 붙일 지점 자체가 없고 전면 재작성이 필요하다.|
| ⑤ | **본인 메모만 반환되는가** |아니오 |

|group B|
| | 확인 항목 | 결과 |
| :-: | :-- | :-- |
| ① | 실행 성공까지 걸린 시간 | 1분 |
| ② | 없는 함수·컬럼을 지어낸 개수 | 0개 |
| | → 지어낸 이름 |없음. id, title, body, created_at, user_id 모두 schema.sql에 존재|
| ③ | `CONVENTIONS.md` 위반 개수 | 0개 |
| | → 무엇을 어겼는가 |없음. 계층 분리·응답 형식·명명 규칙·user_id 조건 모두 준수 |
| ④ | 사람이 직접 고친 지점 | 0곳 |
| | → 어디를 어떻게 |코드를 읽고 판단했으나 고칠 지점을 찾지 못했다. 라우트 순서(/memos/search를 /memos/:id 위에 두는 것)는 에이전트가 먼저 경고했으므로 사람의 수정으로 세지 않았다.|
| ⑤ | **본인 메모만 반환되는가** | 예 |

### ⑤번을 반드시 확인하세요

생성된 SQL에 `user_id` 조건이 들어 있는지 보세요.

없다면 **코드는 정상 동작하지만 남의 메모까지 검색됩니다.** 에러도 나지 않습니다.

---

## 4. 두 조의 결과 비교

작업이 끝나면 두 조가 만든 코드를 나란히 놓고 함께 답하세요.

**4-1. 두 결과의 가장 큰 차이는 무엇입니까?**

```
코드의 품질 차이가 아니라 만들어진 결과물의 종류가 달랐다.

B조는 기존 service.js에 searchMemos 함수를, routes.js에 GET /memos/search 라우트를 추가했다. 프로젝트에 그대로 붙일 수 있다.

A조는 브라우저에서 동작하는 React 검색 화면을 만들었다. 메모를 컴포넌트 상태에 두고 자바스크립트로 필터링하므로 SQL도, 라우트도, 데이터베이스 접근도 없다. 코드 자체는 동작하지만 이 프로젝트에 붙일 수 있는 지점이 없다.

같은 과제, 같은 모델인데 A조는 서버 기능을 만들어야 할 자리에서 화면을 만들었다. 두 결과는 비교 가능한 두 개의 답이 아니라 서로 다른 층위의 산출물이다.
```

**4-2. A조의 실패는 모델 탓입니까, 우리가 주지 않은 탓입니까? 근거를 들어 적으세요.**

```
우리가 주지 않은 탓이다. 근거는 세 가지다.

첫째, 같은 모델이었다. A조와 B조는 같은 에이전트에게 몇 분 간격으로 같은 과제를 주었다. 달라진 변수는 함께 준 자료뿐이다. 모델의 능력 문제라면 B조에서도 같은 실패가 나와야 했지만 그렇지 않았다.

둘째, A조의 출력은 혼란스럽지 않았다. 실패한 코드가 아니라 잘 정리된 코드였다. 토큰 단위 AND 검색, 매칭 위치 주변 본문 자르기, 하이라이트 처리까지 요구하지 않은 기능이 들어 있었다. 능력이 부족해서 실패한 것이 아니라, 어디에 그 능력을 써야 하는지를 몰랐던 것이다.

셋째, 자료가 없을 때 에이전트는 멈추지 않고 대신할 것을 찾는다. A조의 에이전트는 프로젝트 구조를 알 수 없자 다른 맥락에서 프로젝트를 추측했고, 그 추측 위에 일관된 코드를 만들었다. 빈칸은 빈칸으로 남지 않는다.

다만 한 가지 단서를 남긴다. A조 대화는 완전한 무맥락 상태가 아니었다. 에이전트가 이전 대화에서 얻은 사용자 개인의 기술 스택(MySQL·서블릿)을 언급했기 때문이다. 따라서 이 조건은 "맥락 없음"이 아니라 "프로젝트 맥락 대신 다른 맥락이 있었음"에 가깝다. 이 점은 오히려 위 셋째 근거를 뒷받침한다.
```

**4-3. B조가 준 자료 중 결과를 가장 크게 바꾼 것 하나를 꼽는다면 무엇입니까? 왜 그렇게 생각합니까?**

```
schema.sql을 꼽는다.

나머지 자료는 모두 스키마를 전제로 해야 작동하기 때문이다. CONVENTIONS.md는 "모든 쿼리에 user_id 조건을 포함하라"고 말하지만, memos 테이블에 user_id 컬럼이 있다는 사실을 모르면 이 규칙은 실행할 수 없다. [종료조건]의 "본인 메모만 반환한다"도 마찬가지로 목표일 뿐 수단이 아니다. schema.sql만이 무엇이 실제로 존재하는지를 알려준다.

출력에서 흔적도 확인된다. ORDER BY created_at DESC는 시각 컬럼의 존재와 이름을 알았다는 뜻이고, 검색 대상으로 content나 text가 아니라 title·body를 고른 것도 스키마를 봤기 때문이다. 마지막에 덧붙인 인덱스 관련 언급도 idx_memos_user를 읽은 결과로 보인다.

A조와의 대비도 같은 방향이다. A조는 스키마가 없자 컬럼을 틀리게 쓴 것이 아니라 데이터 모델 자체를 새로 지어냈다. schema.sql이 막아주는 것은 바로 이 지어내기다.

다만 service.js를 꼽는 반론도 성립한다. service.js의 쿼리에는 memos, user_id, created_at, title, body가 모두 등장하므로 스키마 없이도 필요한 정보를 얻을 수 있고, 여기에 더해 db.all의 호출 방식과 getMemo의 AND user_id = ? 패턴이라는 따라 쓸 본보기까지 제공한다. 우리는 service.js가 컬럼을 사용 중인 모습만 보여줄 뿐 전체 컬럼 구성과 제약(NOT NULL, 외래키)을 확정해 주지는 못한다고 보아 schema.sql을 택했다.

이 둘을 실제로 가르려면 조건을 하나 더 두면 된다. [지시]와 [종료조건]에 schema.sql만 주고 CONVENTIONS.md와 service.js는 빼는 것이다. 그래도 WHERE에 user_id가 들어온다면 스키마의 기여가 확인되고, 들어오지 않는다면 본보기 역할을 한 service.js 쪽이 컸다는 뜻이 된다.
```

---

## 5. PROMPTS.md 기록

위 2번의 프롬프트 원문을 저장소의 `PROMPTS.md`에 추가하고 커밋하세요.

```markdown
## 2026-__-__ · 메모 검색 기능 (2주차 활동)

**지시**
(붙여넣은 프롬프트 원문)

**채택 여부**
(전체 채택 / 일부 채택 — 무엇을 어떻게 수정했는지 / 미채택)

**참고**
(있으면)
```

- [X ] `PROMPTS.md`에 추가하고 커밋했습니다

---

## 6. 제출 확인

- [X] 이 활동지를 저장소에 커밋했습니다
- [X] `PROMPTS.md`를 커밋했습니다
