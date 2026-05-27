# work-visualization

노션 "패스키 일정 캘린더" DB의 작업 현황을 **GitHub Pages 정적 대시보드**로 시각화한다. 노션 페이지에 `/embed`로 붙여 노션 안에서 본다.

- **시각화**: KPI 카드(지연 작업 카드는 클릭하면 지연 목록 표시) + Gantt 타임라인(메인) + 트랙별 진행률 막대 + Kanban 파이프라인
- **경로**: 대시보드는 `/passkeys/` 경로에서 열린다 (루트는 안내 페이지)
- **진척률**: 작업 기간을 무게로 쓰는 기간 가중(duration-weighted) 방식
- **갱신**: Claude(노션 MCP 연결)에게 갱신을 요청하면 데이터를 가져와 `public/passkeys/data.json`을 갱신·커밋·push 한다. **별도 토큰/Integration 설정이 필요 없다.**
- **기술**: 외부 차트 라이브러리 없이 순수 HTML/CSS/JS
- **배포 URL**: `https://flex-nameui.github.io/work-visualization/passkeys/`

<!-- screenshot -->

색 의미(전 위젯 공통): 완료=초록, 진행 중=파랑, 대기=회색.

---

## 1. 로컬에서 확인

```bash
npm install     # 로컬 미리보기용 http-server 설치
npm run dev     # 로컬 정적 서버 (캐시 끔)
# 브라우저에서 http://localhost:8080/passkeys/ 접속  (파일을 직접 더블클릭하지 말 것)
```

저장소에 포함된 `public/passkeys/data.json`(실제 노션 데이터)으로 바로 화면이 보인다.

---

## 2. 데이터 갱신 (표준 흐름)

노션을 수정한 뒤, **Claude에게 "대시보드 갱신해줘"** 라고 하거나 `/refresh-dashboard` 스킬을 호출한다.

Claude가 하는 일:
1. **노션 MCP**로 최신 데이터를 가져온다 (토큰 불필요 — Claude의 노션 연결 사용)
2. `public/passkeys/data.json`을 갱신한다
3. 그 파일만 git commit → push (push가 GitHub Actions 배포를 트리거)

배포가 끝나면 노션 임베드를 새로고침하면 최신 데이터가 보인다.

- 다른 DB를 보려면: `/refresh-dashboard <노션 DB 링크>`
- 갱신은 Claude를 통해서만 일어난다(노션을 자동 감시하지 않음).

---

## 3. 깃허브 셋업 (최초 1회)

1. 코드를 GitHub 리포(`flex-nameui/work-visualization`)에 push
2. **Settings → Pages → Source: "GitHub Actions"** 로 설정
3. 끝. **GitHub Secret이 필요 없다** — 데이터 갱신은 로컬 Claude(MCP)가, 배포는 CI가 한다.

`public/**`가 main에 push될 때마다, 또는 Actions 탭에서 "Deploy Dashboard"를 수동 실행할 때 배포된다.

---

## 4. 노션에 임베드

1. 배포 URL: `https://flex-nameui.github.io/work-visualization/passkeys/`
2. 노션 페이지에서 `/embed` 입력 → URL 붙여넣기
3. 데이터 갱신(섹션 2) 후 노션 임베드를 새로고침하면 반영된다

---

## 5. 트러블슈팅

| 증상 | 확인할 것 |
|------|-----------|
| 갱신이 안 됨 | Claude에 노션 MCP가 연결돼 있는지 |
| 노션 데이터를 못 읽음 | 해당 DB가 MCP 연결 계정에서 접근 가능한지 |
| Pages 404 | Settings → Pages Source가 "GitHub Actions"인지 |
| "데이터를 불러오지 못했어요" | `public/passkeys/data.json`이 커밋·배포됐는지, 그리고 `http://.../passkeys/`로 여는지(파일 직접 열기 X) |
| `git push` 거부됨 | `git pull --rebase` 후 다시 push (스킬은 자동 처리) |

---

## 디렉토리 구조

```
work-visualization/
├── .claude/skills/refresh-dashboard/SKILL.md   # 갱신 스킬 (노션 MCP 사용)
├── .github/workflows/deploy.yml                # 배포 전담 워크플로
├── public/
│   ├── index.html                              # 루트 안내 (→ /passkeys)
│   └── passkeys/
│       ├── index.html                          # 대시보드 본체
│       └── data.json                           # 데이터(커밋됨)
├── docs/plans/                                 # 구현 계획
├── .gitignore
└── package.json
```
