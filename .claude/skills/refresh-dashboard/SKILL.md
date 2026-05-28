---
name: refresh-dashboard
description: 지정한 Linear 프로젝트의 오남의 티켓을 노션 MCP가 아닌 Linear MCP로 가져와 public/passkeys/data.json을 갱신하고 그 파일만 git commit·push 한다(이 push가 GitHub Actions 배포를 트리거). 토큰/.env 불필요 — Claude의 Linear MCP 연결 사용. 트리거 - "대시보드 갱신", "일정 새로고침", "/refresh-dashboard", 또는 Linear 프로젝트를 지정하며 갱신을 요청할 때.
---

# refresh-dashboard

work-visualization 대시보드의 데이터를 **Linear**에서 갱신하는 스킬이다.
Claude의 Linear MCP 연결을 쓰므로 토큰/.env 가 필요 없다.

역할: 지정 Linear 프로젝트의 **오남의(인턴)** 티켓 조회 → `public/passkeys/data.json` 갱신 → **data.json만** commit → push.
배포는 `.github/workflows/deploy.yml`이 push를 받아 자동 처리한다.

## 입력 (프로젝트 지정)

- 인자로 **Linear 프로젝트**(이름 또는 ID)가 주어지면 그 프로젝트로 한정한다. 없으면 기본값을 쓴다.
- 기본 프로젝트: `패스키 구현` (id `10dafbd1-ee37-4f7b-bdd3-c9553fcf7048`, 팀 `BE/Internal Internship`).
- **그 프로젝트 안의 오남의 티켓만 산정한다** (assignee=`오남의`). 본 대시보드는 인턴 오남의의 일정을 시각화하는 용도이므로 기본 assignee를 고정한다. 다른 사람 티켓을 보고 싶으면 호출 시 assignee를 명시적으로 override.

## 절차 (프로젝트 루트에서)

1. **조회**: Linear MCP `list_issues`로 가져온다.
   - `assignee: "오남의"`, `project: "<프로젝트 id 또는 이름>"`, `limit: 100`, `includeArchived: false`.
   - ⚠️ 출력이 토큰 한도를 넘으면 결과가 파일로 저장된다(예: `.../tool-results/...list_issues-*.txt`). **그 파일 내용을 컨텍스트로 읽지 말고**, 아래 2번을 그 파일 경로에 대해 python으로 실행한다.
   - `hasNextPage`가 true면 `cursor`로 이어서 가져온다(프로젝트가 100건 초과일 때만).

2. **변환 + 파일 생성**: 저장된 결과(JSON)를 python으로 파싱해 `public/passkeys/data.json`을 쓴다. 큰 출력을 컨텍스트에 올리지 않기 위해 파일→파일 변환으로 처리한다.

   매핑 규칙 (날짜는 **분리된 SoT**):
   - `title` ← 이슈 제목
   - `status` ← `statusType`: `completed`→`Done`, `started`→`In progress`, 그 외→`Not started`. **`canceled`/`triage`는 제외**.
   - `track` ← description의 `트랙: …` (없으면 null)
   - `type` ← description의 `종류: …` (없으면 null)
   - **`end` ← Linear `dueDate` (단일 SoT)**. dueDate가 없으면 description의 시작일과 같게.
   - **`start` ← description의 `기간: A ~ ...`의 A** (또는 `일자: A`). description의 종료일 부분(B)은 **무시** — 종료는 dueDate가 결정한다. description에 날짜 줄이 없으면 dueDate로 폴백(1일짜리).
   - `point` ← `estimate.value` (없으면 1)
   - `start` 오름차순 정렬(없는 건 뒤로)

   참고 python (경로만 바꿔 실행):
   ```python
   import json, re, sys, datetime
   data = json.load(open(sys.argv[1], encoding="utf-8"))
   issues = data.get("issues", [])
   smap = lambda st: {"completed":"Done","started":"In progress"}.get(st,"Not started")
   def field(d,k):
       m = re.search(k+r"[:\s]*([^/\n]+?)\s*(?:/|$)", d or ""); return m.group(1).strip() if m else None
   def start_from_desc(d):
       if not d: return None
       m = re.search(r"기간[:\s]*([0-9]{4}-[0-9]{2}-[0-9]{2})", d)
       if m: return m.group(1)
       m = re.search(r"일자[:\s]*([0-9]{4}-[0-9]{2}-[0-9]{2})", d)
       if m: return m.group(1)
       return None
   tasks=[]
   for it in issues:
       st=it.get("statusType")
       if st in ("canceled","triage"): continue
       d=it.get("description") or ""
       due=it.get("dueDate")
       s_desc=start_from_desc(d)
       end=due
       start=s_desc if s_desc else due
       est=it.get("estimate") or {}; pt=est.get("value") if isinstance(est,dict) else None
       tasks.append({"title":it.get("title",""),"track":field(d,"트랙"),"type":field(d,"종류"),
                     "status":smap(st),"start":start,"end":end,"point":pt if pt is not None else 1})
   tasks.sort(key=lambda t:(t["start"] is None, t["start"] or ""))
   out={"generated_at":datetime.datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%S.000Z"),
        "source":"linear","project":"<프로젝트 이름>","tasks":tasks}
   open("public/passkeys/data.json","w",encoding="utf-8").write(json.dumps(out,ensure_ascii=False,indent=2)+"\n")
   print("tasks:",len(tasks))
   ```

3. **변경 확인**: `git status --porcelain public/passkeys/data.json`. 비어 있으면 "데이터 변경 없음" 보고 후 종료.

4. **스테이징**: `git add public/passkeys/data.json` — **오직 이 파일만**.

5. **커밋**: `git commit -m "chore: refresh linear data $(date -u +%Y-%m-%dT%H:%M:%SZ)"`

6. **푸시**: `git push`. 거부되면 `git pull --rebase` 후 다시 push (충돌 시 `git checkout --theirs public/passkeys/data.json` 채택).

7. **보고**: 커밋·푸시 완료 + "GitHub Actions가 곧 배포합니다. 노션 임베드 새로고침하면 반영됩니다." 안내.

## 주의

- Linear MCP 연결을 전제로 한다. 없으면 동작하지 않으므로 사용자에게 안내한다.
- **날짜 SoT 분리**: 종료일은 Linear 정식 필드 `dueDate`가 SoT. 시작일은 description의 `기간: A ~ ...`의 A (또는 `일자: A`)에서 추출. description의 종료일 부분(B)은 파싱하지 않으니 dueDate와 어긋나도 무방. dueDate가 description의 시작일보다 빠르면 start>end로 깨질 수 있으니 사용자가 Linear UI에서 dueDate를 조정해야 한다.
- 트랙/종류는 description의 `트랙: / 종류:` 형식에서 파싱한다. 이 형식이 없는 티켓은 트랙/종류 null로 들어간다(진척 산정엔 point로 포함).
- `git add .`/`-A` 금지 — 항상 `public/passkeys/data.json`만.
- 다른 프로젝트/타인 티켓을 포함하지 않는다(공개 노출 방지).
