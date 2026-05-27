---
name: refresh-dashboard
description: 노션 패스키 일정 DB에서 최신 데이터를 노션 MCP로 가져와 public/passkeys/data.json을 갱신하고 그 파일만 git commit·push 한다(이 push가 GitHub Actions 배포를 트리거). 토큰/.env 불필요 — Claude의 노션 MCP 연결을 사용. 트리거 - "대시보드 갱신", "노션 데이터 새로고침", "/refresh-dashboard", 또는 노션 DB 링크를 주며 갱신을 요청할 때.
---

# refresh-dashboard

work-visualization 대시보드의 데이터를 갱신하는 스킬이다.
**Claude의 노션 MCP 연결로 데이터를 가져오므로 NOTION_TOKEN/.env 가 필요 없다.**

역할: 노션 MCP로 조회 → `public/passkeys/data.json` 갱신 → **data.json만** commit → push.
배포(GitHub Pages)는 `.github/workflows/deploy.yml`이 push를 받아 자동 처리한다.

## 입력 (선택)

- 인자로 노션 DB 링크/ID가 주어지면 그 DB를, 없으면 기본 패스키 DB를 쓴다.
- 기본 데이터 소스: `collection://1a623255-da14-4636-a230-5eb1d7a8b22e`
  (DB 컨테이너 URL의 ID는 `734bc4ff43134fe78afd32a4b3e5b5be`이고, 그 안의 데이터 소스가 위 collection 이다.)
- 링크가 주어지면 먼저 노션 MCP `notion-fetch`로 그 DB를 조회해 `<data-source url="collection://...">` 값을 찾아 사용한다.

## 절차 (프로젝트 루트에서)

1. **데이터 조회**: 노션 MCP 쿼리 도구로 데이터 소스의 모든 행을 가져온다.
   - 기본 (SQL 모드, `notion-query-data-sources`):
     - `data_source_urls`: `["collection://1a623255-da14-4636-a230-5eb1d7a8b22e"]`
     - `query`: `SELECT "제목","트랙","종류","status","date:일자:start","date:일자:end" FROM "collection://1a623255-da14-4636-a230-5eb1d7a8b22e" ORDER BY "date:일자:start" ASC`
   - 429(rate limit)가 나면 몇 초 쉬고 재시도하거나, **view 모드**로 대체한다
     (`mode: "view"`, `view_url`: 해당 DB의 뷰 URL — 모든 행/컬럼을 반환).
   - `has_more`가 true면 `next_cursor`로 이어서 모두 가져온다.

2. **정규화**: 각 행을 아래 형태로 변환한다. (노션 page ID 등 내부 식별자는 넣지 않는다 — 공개 노출 최소화)
   - `title`: `제목`, `track`: `트랙`, `type`: `종류`(없으면 `null`), `status`: `status`
   - `start`: `date:일자:start`, `end`: `date:일자:end`(없으면 `start`와 동일)
   - `start` 오름차순 정렬

3. **파일 작성**: `public/passkeys/data.json` 에 아래 구조로 저장(2-space pretty JSON).
   ```json
   { "generated_at": "<현재 ISO8601 UTC>", "database_id": "1a623255-da14-4636-a230-5eb1d7a8b22e", "tasks": [ /* 정규화된 작업들 */ ] }
   ```

4. **변경 확인**: `git status --porcelain public/passkeys/data.json`.
   - 비어 있으면 "데이터에 변경이 없어 커밋하지 않았습니다." 보고 후 종료.

5. **스테이징 (data.json만)**: `git add public/passkeys/data.json` — **오직 이 파일만**.

6. **커밋**: `git commit -m "chore: refresh notion data $(date -u +%Y-%m-%dT%H:%M:%SZ)"`

7. **푸시**: `git push`. 거부되면(remote 앞섬) `git pull --rebase` 후 다시 push.
   - rebase 중 `public/passkeys/data.json` 충돌 시 방금 만든 최신을 채택:
     `git checkout --theirs public/passkeys/data.json && git add public/passkeys/data.json && git rebase --continue`.

8. **보고**: 커밋·푸시 완료를 알리고 "GitHub Actions가 곧 배포합니다. 배포 후 노션 임베드를 새로고침하면 반영됩니다."라고 안내한다. (선택: `gh run watch`로 배포 확인 제안.)

## 주의

- 이 스킬은 **노션 MCP 연결**을 전제로 한다. 연결이 없으면 동작하지 않으므로 사용자에게 MCP 연결을 안내한다.
- `git add .` / `git add -A` 금지 — 항상 `public/passkeys/data.json`만 스테이징한다.
- 배포를 직접 트리거하지 않는다(push가 `deploy.yml`을 작동시킨다).
