# 노션 임베드 대시보드 디자인 리서치

> 조사 일자: 2026-05-27  
> 대상: 정적 HTML + Chart.js, 노션 /embed 블록, 700~1100px 폭, 라이트/다크 모드 지원, ~18개 작업 일정 데이터

**Research value: high** -- 정보 위계, KPI 카드 해부, 색상 토큰 설계, Chart.js 다크모드 연동까지 권위 있는 출처에서 구체적 수치와 코드 패턴이 확인되었음.

---

## 1. 정보 위계 — F-패턴/Z-패턴과 시선 흐름 설계

### 권장 사항

- 서양권 사용자는 좌상단 → 우상단 → 좌측 하단 방향으로 F자형으로 화면을 훑는다.
- 좌상단 영역이 전체 주의의 약 80%를 가져간다. 우하단은 10% 미만이다.
- 화면을 4분면으로 나누면: 1사분면(좌상) = 핵심 지표, 2사분면(우상) = 보조 지표, 하단 절반 = 상세·트렌드 맥락.

### 근거

Nielsen Norman Group의 F-패턴 아이트래킹 연구 및 Improvado 대시보드 설계 가이드(2026)에서 동일하게 확인. "Information in the top-left gets 80% of attention; bottom-right gets <10%."

### 이 프로젝트 적용

| 영역 | 배치 콘텐츠 | 이유 |
|------|------------|------|
| 최상단 한 줄 | 전체 완료율 KPI 카드 1~3개 | 가장 먼저 눈이 닿는 위치 |
| 두 번째 행 | 버업 차트 또는 트랙별 진행률 막대 | 핵심 트렌드를 빠르게 전달 |
| 세 번째 행 | 칸반 파이프라인 요약 | 현황 파악 |
| 최하단 | 캘린더 히트맵 | 보조 맥락 — 스크롤해서 보는 것이 자연스러움 |

---

## 2. 위젯 개수 — 인지 부하 기준

### 권장 사항

- 한 화면에 5~9개 KPI가 적정 상한이다.
- 12개를 초과하면 참여율이 40% 낮아진다는 데이터가 있다.
- Arkatechture 프레임워크: "핵심 지표는 1~3개로 제한하라."
- Notion 임베드 맥락에서는 위젯 4~6개가 실용적 권고치다(각 위젯이 별도 HTTP 요청).

### 근거

Miller's Law(작업 기억 용량 7±2)와 Nielsen Norman Group의 인지 부하 연구. Progressive disclosure(점진적 공개)는 인지 부하를 최대 55% 낮춘다.

### 이 프로젝트 적용

~18개 작업 데이터는 KPI 카드 3개 + 차트/뷰 3~4개로 집약 가능하다. 개별 작업 목록은 대시보드에서 제거하고, 집계 지표만 표시한다.

---

## 3. KPI 카드 디자인 패턴

### 카드의 5개 필수 구성 요소

| 요소 | 설명 | 이 프로젝트 예시 |
|------|------|----------------|
| 날짜 범위 | 어떤 기간의 수치인지 | "이번 스프린트" / "전체 기간" |
| 지표명 | 간결한 레이블, 툴팁으로 보완 | "완료 작업" / "지연 중" |
| 핵심 값 | 가장 큰 폰트로, 단위 포함 | "7 / 18" |
| 맥락 정보 | 목표 대비, 전 주 대비 변화 | "목표 대비 -2" / "+3 vs 지난 주" |
| 스파크라인 | 작은 선 차트로 트렌드 방향 표시 | 최근 4주 완료 추이 |

### 시각 설계 원칙

- 지표값 > 지표명 > 맥락 정보 순으로 폰트 크기를 줄여 위계를 만든다.
- 색상 의미 고정: 녹색 = 목표 달성, 황색 = 경고, 적색 = 이탈.
- 색각 이상(남성 약 8%)을 위해 색상만으로 상태를 전달하지 말고, ▲▼ 기호나 텍스트를 병행한다.
- 모든 카드는 레이아웃 구조를 동일하게 유지한다(레이블 위치, 값 위치, 스파크라인 위치).

### 안티패턴

- 맥락 없는 숫자: "완료 7"만 표시하고 전체 대비 비율이나 목표가 없는 경우.
- 허영 지표(Vanity Metric): 데이터가 변해도 어떤 행동을 취해야 할지 알 수 없는 지표.
- 색상 불일치: 하락 추세에 녹색을 표시하는 실수.
- 과한 정밀도: 소수점 세 자리를 표시할 필요가 없다.

---

## 4. 임베드/좁은 폭 환경 대응

### Notion iframe 특성 이해

- Notion /embed 블록은 iframe이며, 콘텐츠의 폭은 Notion 페이지 폭 설정에 따라 700~1100px 사이에서 결정된다.
- iframe 내부는 독립적인 문서이므로, 내부에서 viewport 단위(`vw`, `vh`)는 iframe 크기 기준으로 동작한다.
- 세로 스크롤은 iframe 높이를 충분히 지정하거나, Notion이 자동으로 높이를 조정하도록 허용해야 한다.

### 레이아웃 권장 방식

```css
/* 반응형 그리드: 700px 이상에서 2열, 미만에서 1열 */
.dashboard-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
  gap: 1rem;
}

/* KPI 카드 행: 항상 동일 간격 */
.kpi-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 0.75rem;
}
```

- `auto-fit` + `minmax`를 조합하면 폭이 줄어들 때 자동으로 1열로 전환된다.
- `maintainAspectRatio: false`를 Chart.js에 설정하고, 차트 컨테이너에 명시적 `height`를 지정하면 폭 변화에도 차트가 깨지지 않는다.
- 세로 스크롤 친화적 배치: 모든 위젯을 세로로 쌓는 단열 구조를 기본으로 하고, 공간이 충분할 때만 가로 배치로 확장한다.

### 좁은 폭(700px 미만) 예외 대응

- KPI 카드 3개 → 폭 350px 이하에서는 1열 나열로 전환.
- 칸반 파이프라인 → 좌우 스크롤보다 단계별 accordion 또는 세로 리스트가 낫다.

---

## 5. 라이트/다크 모드 색상 토큰 설계

### 3계층 토큰 구조

```
Primitive (원시값)  →  Semantic (역할)  →  Component (컴포넌트)
--blue-600: #0052CC   --color-accent:          --card-border:
                       var(--blue-600)           var(--color-accent)
```

- **Primitive**: 팔레트의 실제 색상값. 레이아웃 코드에서 직접 참조하지 않는다.
- **Semantic**: 역할(surface, text, accent, status)을 이름으로 갖는 토큰. 테마가 바뀌어도 이름은 변하지 않는다.
- **Component**: 각 위젯/카드가 semantic 토큰을 참조한다.

### 이 프로젝트용 최소 토큰 세트

```css
:root {
  /* Surface (배경 계층) */
  --color-bg-page: #F5F5F5;
  --color-bg-card: #FFFFFF;
  --color-bg-elevated: #FFFFFF;

  /* Text */
  --color-text-primary: #1A1A1A;
  --color-text-secondary: #6B7280;
  --color-text-muted: #9CA3AF;

  /* Status */
  --color-status-success: #16A34A;
  --color-status-warning: #D97706;
  --color-status-danger: #DC2626;
  --color-status-neutral: #6B7280;

  /* Border */
  --color-border: #E5E7EB;

  /* Chart 팔레트 (JavaScript에서 읽어 쓸 것) */
  --chart-color-1: #3B82F6;
  --chart-color-2: #10B981;
  --chart-color-3: #F59E0B;
  --chart-color-4: #6366F1;
  --chart-grid: #E5E7EB;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-bg-page: #111827;
    --color-bg-card: #1F2937;
    --color-bg-elevated: #374151;
    --color-text-primary: #F9FAFB;
    --color-text-secondary: #9CA3AF;
    --color-text-muted: #6B7280;
    --color-border: #374151;
    --chart-grid: #374151;
    /* status 색상은 채도를 약간 낮춰 다크 배경에서 눈부심 방지 */
    --color-status-success: #22C55E;
    --color-status-warning: #FBBF24;
    --color-status-danger: #F87171;
  }
}
```

### 핵심 원칙

- 컴포넌트 코드는 `--color-bg-card`처럼 semantic 토큰만 참조한다. `#FFFFFF`를 직접 쓰지 않는다.
- 다크 모드에서 배경을 단순히 반전(흰→검)하면 명도 대비가 무너진다. Surface를 `page < card < elevated` 3단계 계층으로 설계해야 한다.
- Notion 자체 라이트/다크 테마와 독립적으로, `prefers-color-scheme` 미디어 쿼리로 시스템 설정을 따르는 것이 가장 안전하다.

---

## 6. Chart.js 미니멀 설정 패턴

### CSS 변수 → Chart.js 색상 연동

Chart.js는 CSS 변수를 직접 읽지 못한다. `getComputedStyle`로 값을 가져와서 주입해야 한다.

```javascript
function cssVar(name) {
  return getComputedStyle(document.documentElement)
    .getPropertyValue(name)
    .trim();
}

// 차트 생성 시점에 현재 테마의 색상값을 읽어 주입
function buildChartColors() {
  return {
    primary: cssVar('--chart-color-1'),
    secondary: cssVar('--chart-color-2'),
    grid: cssVar('--chart-grid'),
    text: cssVar('--color-text-secondary'),
  };
}

// 테마 변경 감지 → 차트 전체 재생성 또는 update()
window.matchMedia('(prefers-color-scheme: dark)')
  .addEventListener('change', () => {
    // 각 차트에 대해 dataset color 갱신 후 chart.update() 호출
    rebuildAllCharts();
  });
```

### 미니멀 기본 옵션 템플릿

```javascript
const minimalDefaults = {
  responsive: true,
  maintainAspectRatio: false,   // 컨테이너 height에 차트를 맞춤
  animation: { duration: 600, easing: 'easeInOutQuart' },
  plugins: {
    legend: { display: false },  // 범례 제거 (카드 제목으로 대체)
    tooltip: {
      backgroundColor: 'var(--color-bg-elevated)', // 주의: 실제로는 cssVar()로 읽어야 함
      titleColor: cssVar('--color-text-primary'),
      bodyColor: cssVar('--color-text-secondary'),
    },
  },
  scales: {
    x: {
      grid: { display: false },          // 세로 그리드 제거
      border: { display: false },
      ticks: { color: cssVar('--color-text-muted') },
    },
    y: {
      grid: {
        display: true,
        color: cssVar('--chart-grid'),   // 옅은 가로선만
        drawBorder: false,
      },
      border: { display: false, dash: [4, 4] },
      ticks: {
        color: cssVar('--color-text-muted'),
        maxTicksLimit: 5,               // 눈금 개수 제한
      },
    },
  },
};
```

### 차트 유형별 권고

| 콘텐츠 | Chart.js 타입 | 핵심 설정 |
|--------|--------------|----------|
| 버업 차트 | `line` (stepped 또는 `tension: 0`) | `fill: true`로 면적 강조, 이상적 라인 별도 overlay |
| 트랙별 진행률 | `bar` (horizontal, `indexAxis: 'y'`) | 세로 그리드 off, 100% 너비 기준선 표시 |
| 칸반 파이프라인 | 차트 불필요 — HTML/CSS로 구현 | `flex`나 `grid` 기반이 더 적합 |
| 캘린더 히트맵 | 차트 불필요 — CSS grid 52×7 | Chart.js Matrix 플러그인 가능하나 순수 CSS가 가볍다 |

---

## 7. 질문 주도 설계 (Question-Driven Design)

### 방법론

Stephen Few의 정의: "대시보드는 하나 이상의 목표를 달성하는 데 필요한 가장 중요한 정보를 한 화면에 응집시켜 한눈에 모니터링할 수 있도록 만든 시각적 표시."

Arkatechture 10-Question Framework에서 핵심 질문:

1. 이 대시보드를 한 문장으로 설명하면? → 범위를 고정한다.
2. 이 대시보드가 구동해야 할 구체적 결정(Decision)은 무엇인가?
3. 핵심 지표는 1~3개로 제한할 수 있는가?
4. 역사적 트렌드 비교가 필요한가, 현재 상태만 필요한가?
5. 대상 독자의 기술 수준은?

"Decision Test": 이 숫자가 바뀌면 무엇을 다르게 할 것인가? 대답이 모호하면 그 지표는 제거한다.

### 이 프로젝트에 적용할 핵심 질문 목록

이 대시보드가 답해야 할 질문을 먼저 정의한 뒤 위젯을 선택한다.

| # | 질문 | 답하는 위젯 |
|---|------|-----------|
| Q1 | 전체 작업 중 지금 몇 %가 완료됐는가? | KPI 카드 (완료율) |
| Q2 | 일정대로 가고 있는가, 지연되고 있는가? | 버업 차트 (실제선 vs 이상선) |
| Q3 | 어떤 트랙/영역이 막혀 있는가? | 트랙별 진행률 막대 |
| Q4 | 현재 각 작업이 어느 단계에 있는가? | 칸반 파이프라인 |
| Q5 | 언제 작업이 몰리고 언제 공백인가? | 캘린더 히트맵 |

Q1~Q3이 핵심. Q4~Q5는 보조. 답할 수 없는 위젯은 추가하지 않는다.

---

## 권장 레이아웃 스케치 (세로 스크롤 기준)

```
┌─────────────────────────────────────────┐
│ [KPI: 완료율]  [KPI: 지연 수]  [KPI: D-day] │  ← 최상단, 한눈에
├─────────────────────────────────────────┤
│         버업 차트 (번다운/번업)              │  ← 일정 준수 여부
├─────────────────────────────────────────┤
│  트랙 A ██████████░░░ 68%               │
│  트랙 B ████████░░░░░ 55%               │  ← 트랙별 진행률
│  트랙 C ████░░░░░░░░░ 30%               │
├─────────────────────────────────────────┤
│ [대기] [진행 중] [검토] [완료]             │  ← 칸반 파이프라인
├─────────────────────────────────────────┤
│         캘린더 히트맵 (옵션)               │  ← 스크롤 후
└─────────────────────────────────────────┘
```

700px 폭에서: KPI 카드 3개가 1열로 쌓이고, 차트는 전폭 사용.  
1100px 폭에서: KPI 카드 3개가 수평 배열, 일부 섹션 2열 가능.

---

## 출처

- [Improvado — Dashboard Design Guide 2026](https://improvado.io/blog/dashboard-design-guide)
- [Nastengraph — Anatomy of the KPI Card](https://nastengraph.substack.com/p/anatomy-of-the-kpi-card)
- [Muzli — Dark Mode Design Systems](https://muz.li/blog/dark-mode-design-systems-a-complete-guide-to-patterns-tokens-and-hierarchy/)
- [Splendide Mendax — Color in Chart.js (2024)](https://splendide-mendax.com/posts/2024-08-27_color_in_chart_js)
- [Arkatechture — 10 Questions to Ask When Designing a Dashboard](https://www.arkatechture.com/blog/10-questions-to-ask-when-designing-a-dashboard)
- [Perceptual Edge (Stephen Few) — Assessing Dashboard Design](https://www.perceptualedge.com/blog/?p=672)
- [Chart.js GitHub Discussion — Dark Mode / prefers-color-scheme](https://github.com/chartjs/Chart.js/discussions/9214)
- [Nielsen Norman Group — F-Pattern Eye Tracking (via Improvado/UXPin citations)]
- [UXPin — Effective Dashboard Design Principles](https://www.uxpin.com/studio/blog/dashboard-design-principles/)
