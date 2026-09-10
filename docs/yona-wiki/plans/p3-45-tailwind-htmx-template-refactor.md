---
type: plan
id: P3-45
title: "템플릿을 Tailwind CSS + htmx 기반으로 리팩터링"
status: planned
priority: 0          # 전략 우선순위표(index.md 상단)에는 미등록 — P3-17 이후 항목들과 동일하게
                       # 사용자가 직접 지시한 독립 항목이라 즉시 착수 가능(선행 조건 없음)
depends_on: []
blocks: []
source: docs/parity/tickets/p3-45.md
created: 2026-09-10
updated: 2026-09-10
tags: [plan, p3, frontend, tailwindcss, htmx]
---

# 템플릿을 Tailwind CSS + htmx 기반으로 리팩터링

## 배경

`docs/TEMPLATE_BACKLOG.md`로 legacy → Thymeleaf 이식이 242/242 완료되어, 현재 yona의 182개
템플릿(`src/main/resources/templates/**/*.html`)은 legacy와 기능적으로 동등하다. 하지만 마크업/CSS는
여전히 **Bootstrap 2.3.1**(`static/bootstrap/`, 2012년 릴리스 — 지금의 Bootstrap 5와도 완전히 다른
구세대 API)이고, 상호작용은 **jQuery 3.3.1** 기반 수동 DOM 조작(`static/javascripts/` 전역에 흩어진
`$.ajax`/`$.post`/이벤트 위임)이다. 빌드 파이프라인 자체가 없다 — `package.json`이 저장소 어디에도
없고, 모든 정적 자산(`static/**`)은 Spring Boot가 가공 없이 그대로 서빙한다(`site/layout.html`이
`<link>`/`<script>` 태그로 직접 참조, 캐시 무효화도 `yobi.css?v=20260614-2`처럼 수동 쿼리스트링 1건뿐).

2026-09-10, 사용자가 `search5/yona`(자신의 fork, `git@github.com:search5/yona.git`)를 가리키는
`tailwindcss` 브랜치를 만들고 이 리팩터링을 직접 지시했다. `origin`은 계속 `yona-projects/yona`를
가리키므로, 이 작업은 `tailwindcss` 브랜치에서 진행되고 `search5` remote로 푸시된다(`main`/`next`
브랜치와는 무관하게 독립적으로 진행 가능 — 병합 여부는 이후 사용자 결정 사항).

## 범위

### 포함
- Tailwind CSS 빌드 파이프라인 신설(Node 빌드 체인 없이 standalone CLI로, 아래 설계 개요 참고)
- htmx 도입 — 서버 왕복형 부분 갱신(댓글 CRUD, 목록 필터/페이지네이션, 알림 배지 등)을 jQuery
  수동 AJAX 대신 htmx 속성(`hx-get`/`hx-post`/`hx-put`/`hx-delete`/`hx-target`/`hx-swap`)으로 전환
- 182개 템플릿 전수를 그룹 단위(레이아웃 → 공용 파샬 → 도메인별)로 점진 전환, 진행 상태는
  [`docs/TAILWIND_HTMX_BACKLOG.md`](../../TAILWIND_HTMX_BACKLOG.md)에서 파일 단위로 추적
- 전환 완료 시점의 정리: Bootstrap CSS/JS, 대체된 jQuery 플러그인·커스텀 스크립트 제거,
  Tailwind Preflight(기본 리셋) 활성화

### 제외 (비범위)
- **화면 재설계(정보구조/필드/기능 변경)는 하지 않는다** — 이번 작업은 순수 기술 스택 교체다.
  "더 낫다고 생각되는" 레이아웃 변경, 필드 생략/추가는 금지(TEMPLATE_BACKLOG의 이식 원칙과 동일 정신).
  실제 리디자인이 필요하면 이 계획과 별개로 사용자 지시를 받는다.
- 클라이언트 전용 위젯 라이브러리 교체는 범위 밖 — 마크다운 에디터, code/diff 뷰어, Select2,
  Pikaday(달력), Jcrop, ViewerJS, `atjs`(멘션 자동완성) 등은 jQuery 의존이 남더라도 이번 라운드에서
  건드리지 않는다(htmx는 "서버가 HTML 조각을 다시 그려주는" 상호작용에만 적용, 순수 클라이언트
  위젯 교체는 각자 리스크가 커 별도 계획으로 분리하는 게 안전).
- [[p3-13|P3-13]](프런트엔드 SPA 분리)과 경합하지 않는다 — 서버 렌더링(Thymeleaf)을 유지한 채
  진행하는 대안 경로이며, 두 계획은 상호 배타적이다(SPA로 가면 이 작업의 htmx 관례는 무의미해짐).
  현재 [[p3-13|P3-13]]은 "계획서만 존재, 미착수, 프레임워크 미결정"이라 이 작업과 충돌하지 않는다.
- Grid/Flex 레이아웃을 넘어서는 반응형 모바일 최적화 심화는 범위 밖(기존 수준 유지, Tailwind
  반응형 유틸리티로 자연히 개선되는 수준까지만).

## 의존성

- **선행 조건**: 없음(TEMPLATE_BACKLOG 242/242 완료가 사실상의 토대이지만 블로킹 의존은 아님 —
  이미 완료돼 있음). 즉시 착수 가능.
- **후속 파급**: 없음. 이 작업이 끝나야 착수 가능한 다른 계획은 현재 없음.
- **병렬 진행 위험**: `main`/`next` 브랜치에서 진행 중인 다른 작업(P1/P2 버그 수정, P3 기능 추가)이
  같은 템플릿 파일을 건드리면 `tailwindcss` 브랜치와 diverge가 커진다. 그룹 단위로 자주 rebase하거나,
  전환이 끝난 그룹부터 `next`에 병합하는 전략을 권장(아래 "병합 전략" 참고).

## 설계 개요

### 1. Tailwind CSS 빌드 파이프라인

- **Node.js 프로젝트 빌드 체인을 새로 들이지 않는다.** 현재 저장소에 `package.json`이 전혀 없고
  Gradle이 유일한 빌드 도구다 — Tailwind의 **standalone CLI 바이너리**(Node 런타임 불요, Tailwind
  공식 배포)를 사용해 이 관례를 깨지 않는다.
- 버전을 명시적으로 고정한다(예: `tailwindcss-{os}-{arch}` 특정 릴리스 태그 — 실제 도입 시점에
  최신 안정판으로 고정하고 이 문서에 버전을 기록).
- 소스: `src/main/tailwind/input.css`(신설, `@tailwind base;`는 **비활성화** — 아래 "Bootstrap과의
  공존" 참고. `@tailwind components; @tailwind utilities;`만 사용).
- 설정: `tailwind.config.js`(신설) — `content`에 `src/main/resources/templates/**/*.html`을 스캔
  대상으로 지정, `corePlugins.preflight: false`(Bootstrap 리셋과 충돌 방지, 아래 참고).
- 출력: `src/main/resources/static/stylesheets/tailwind.css`(컴파일 산출물 — **git에 커밋**한다.
  이유: standalone CLI를 CI/배포 환경에 매번 설치하는 것보다, 컴파일된 CSS를 커밋해두는 편이
  기존 "정적 자산은 가공 없이 그대로 서빙" 관례와 일치하고 배포 단순성이 높다. 대신 PR/커밋 시
  소스(`input.css`+템플릿 변경)와 산출물(`tailwind.css`)이 항상 함께 갱신됐는지 리뷰 시 확인).
- Gradle 연동: `tasks.register<Exec>("tailwindBuild")`를 신설해 `processResources`보다 먼저 실행되게
  의존 배선(`tasks.named("processResources") { dependsOn("tailwindBuild") }`), 로컬 개발 중에는
  `--watch` 모드를 별도 커맨드로 안내(Gradle 태스크로 강제하지 않음 — watch는 데몬 프로세스라
  빌드 태스크와 성격이 다름).
- CI는 하나만 결정하면 된다: **바이너리를 매 빌드 다운로드할지, 로컬에서 빌드해 산출물만 커밋할지.**
  위 결정(산출물 커밋)에 따라 CI는 `tailwindBuild`를 건너뛰어도 무방 — 이 프로젝트 CI 파이프라인
  설정 위치를 착수 시 먼저 확인하고 반영한다(gradlew 기반 확인 필요, 이 문서 작성 시점엔 미확인).

### 2. Bootstrap 2.3.1과의 공존 전략 (점진적 전환의 핵심)

182개 템플릿을 한 번에 바꾸는 "빅뱅"은 리스크가 너무 크다. 그런데 `site/layout.html`/
`site/layout_framed.html`은 사실상 모든 페이지가 공유하므로, 이 두 파일에 Tailwind를 얹는 순간
전역 영향이 생긴다. 이를 안전하게 점진화하는 핵심 결정:

- **Tailwind Preflight(전역 리셋)를 전환 완료 전까지 끈다**(`corePlugins.preflight: false`).
  Preflight는 `margin`/`box-sizing`/헤딩 폰트크기 등 브라우저 기본 스타일을 초기화하는데, 이걸
  켠 채로 레이아웃에 Tailwind CSS를 얹으면 아직 전환 안 된 Bootstrap 기반 하위 템플릿들의 여백/
  타이포그래피가 전부 깨진다. 꺼두면 Tailwind는 "필요한 곳에만 유틸리티 클래스를 추가하는" 방식으로
  기존 Bootstrap 스타일 위에 안전하게 공존한다.
- 즉 **레이아웃(그룹 1)도 Tailwind CSS `<link>` 자체는 처음부터(Phase 0에서) 로드**하지만, 실제로
  Bootstrap 클래스를 Tailwind 유틸리티로 **치환**하는 작업은 그룹 단위로 점진 진행한다. 두 클래스
  체계가 한 페이지 안에 공존해도(예: `class="btn btn-primary px-4"`) 문제 없다 — 특이도 충돌이
  나는 지점만 그때그때 확인.
- **전환 단위는 "화면(템플릿) 그룹" 전체다, 파일 절반만 바꾸지 않는다** — 한 화면 안에서 Bootstrap
  그리드(`.row`/`.span*`)와 Tailwind 그리드(`flex`/`grid`)를 동시에 쓰면 레이아웃이 깨지기 쉬우므로,
  파일 하나를 열면 그 파일의 레이아웃 클래스는 전부 Tailwind로 바꾸고 끝낸다(버튼/뱃지 같은 낱개
  컴포넌트 클래스는 그 파일이 include하는 공용 파샬이 아직 미전환이면 과도기적으로 남을 수 있음 —
  이 경우 백로그에 `[~]`로 표시하고 사유를 남긴다).
- **최종 정리 단계(전 그룹 완료 후)**: `corePlugins.preflight: true`로 전환, `static/bootstrap/`
  디렉터리 삭제, `bootstrap-better-typeahead.js`/`bootstrap-switch.js` 등 Bootstrap 전용 JS 제거,
  `site/layout.html`의 관련 `<link>`/`<script>` 정리. 이 단계에서 전체 화면 수동 스모크 테스트
  1회 필요(Preflight ON이 전환 완료 화면들에도 의도치 않은 영향을 줄 수 있음).

### 3. htmx 도입 범위와 패턴 카탈로그

htmx는 CDN이 아니라 기존 관례(`static/javascripts/lib/`)대로 **자체 호스팅**한다
(`static/javascripts/lib/htmx/htmx.min.js`, 버전 고정). 빌드 스텝 불요 — `<script>` 한 줄 추가.

기존 코드에서 이미 확인된, htmx로 대체 가능한 jQuery 수동 AJAX 패턴(TEMPLATE_BACKLOG 완료 로그에
기록된 것들 위주로 우선순위):

| 현재 구현(jQuery) | 위치 | htmx 대체 패턴 |
|---|---|---|
| 댓글 인라인 수정 — `fetch PUT` + 수동 DOM 치환 | `common/commentUpdateForm.html`(TASK-0243) | `hx-put="/.../comments/{id}"` + `hx-target="closest .comment"` + `hx-swap="outerHTML"` |
| 댓글 삭제 — 확인창 + `DELETE` + DOM 제거 | `common/commentDeleteModal.html`(TASK-0243, `yobi.Comment.js` 없어 인라인 스크립트로 구현) | `hx-delete` + `hx-confirm="..."` + `hx-target="closest .comment"` + `hx-swap="outerHTML swap:200ms"` |
| 알림 파셜 폴링/새로고침 | `index/partial_notifications.html` | `hx-get` + `hx-trigger="every 30s"`(또는 `load`) + `hx-swap="innerHTML"` |
| 목록 페이지네이션 파셜 | `site/partial_pagination.html`, `issue/partial_list.html`, `organization/*_partial.html`, `pullrequest/partial_list.html` | `hx-get` + `hx-target="#list-container"` + `hx-push-url="true"`(뒤로가기 대응) |
| 이슈 라벨 선택/조직 이슈 검색 등 파셜 갱신 | `issue/partial_select_label.html`, `organization/issueSearch_partial.html` 등 | `hx-get`/`hx-post` + `hx-trigger="change"`(select) 또는 `hx-trigger="keyup changed delay:300ms"`(검색창) |
| 마일스톤 상태 갱신 | `milestone/partial_status.html` | `hx-patch` + `hx-swap="outerHTML"` |

컨트롤러 쪽 변경 필요 여부: htmx는 일반 HTML 폼/링크와 동일하게 동작하므로 **기존 REST 엔드포인트를
그대로 재사용**하는 게 원칙이다(`common/commentUpdateForm.html`이 이미 `PUT .../comments/{id}`를
쓰고 있듯). 다만 응답 바디가 "JSON"인 엔드포인트는 htmx가 그대로 못 쓰므로(htmx는 응답 HTML을
그 자리에 스왑) **HTML 조각을 반환하는 부분 뷰(partial fragment) 렌더링**이 필요한 경우 컨트롤러에
분기를 추가해야 한다 — 이건 화면 그룹 작업 중 개별적으로 발견/처리한다(TEMPLATE_BACKLOG가 "뷰만
옮기고 끝나지 않음, 컨트롤러도 이식 대상"이라고 명시한 것과 같은 종류의 판단).

### 4. 검증 전략(테스트)

기존 `TemplateEquivalenceSpec.kt` 계열 하네스(`AbstractIntegrationTest` + 실제 MockMvc 렌더링 +
Jsoup CSS 셀렉터 단언)를 그대로 재사용한다. 단, 이번엔 **"legacy와 동일한가"가 아니라 "새 마크업이
같은 데이터/기능을 여전히 노출하는가"**를 검증하는 성격으로 바뀐다:

- 화면 그룹 전환 전, 그 그룹에 대응하는 기존 스펙 파일들을 찾아 **Bootstrap 클래스명/jQuery
  DOM id를 셀렉터로 쓰는 단언이 있는지 확인**한다(예: `.btn.btn-primary`, `#comment-form`).
  있으면 전환과 함께 새 Tailwind/htmx 마크업의 셀렉터로 갱신 — 셀렉터가 CSS 클래스가 아니라
  **의미 있는 `data-*` 속성이나 `id`**를 기준으로 하도록 옮겨가면 이후 스타일 변경에 더 강건해진다
  (권장: 상호작용 훅에는 `data-testid` 또는 기존 `id` 유지, 스타일 클래스와 테스트 셀렉터를 분리).
- htmx 상호작용(부분 갱신)은 MockMvc로 `hx-*` 요청 헤더(`HX-Request: true`)를 흉내내 해당
  엔드포인트가 여전히 올바른 HTML 조각을 반환하는지 검증하는 테스트를 추가한다.
- **자동화 테스트로 못 잡는 것**: 순수 시각적 회귀(레이아웃 깨짐, 여백/정렬)는 CSS 셀렉터 단언으로
  검증되지 않는다. 그룹 하나를 끝낼 때마다 `claude-in-chrome`(또는 `run` 스킬)으로 실제 화면을
  띄워 골든 패스 스모크 체크를 1회 수행하고, 결과(정상/이슈)를 백로그에 기록한다.
- 전체 회귀 스위트(`./gradlew test`, 필요시 `-Dyona.it.db=h2`)는 그룹마다 그린 유지가 원칙 —
  백엔드 로직은 건드리지 않으므로 대부분의 기존 스펙은 셀렉터 갱신 없이 그대로 통과해야 하고,
  실패하는 스펙이 바로 "이 화면이 Bootstrap 클래스에 의존한 단언을 갖고 있었다"는 신호다.

### 5. 병합 전략

- `tailwindcss` 브랜치는 `search5` remote(개인 fork)로만 푸시 — `origin`(`yona-projects/yona`)에는
  이 브랜치를 직접 올리지 않는다(사용자가 별도 remote를 분리 지정한 의도로 해석).
- 그룹 단위 작업 완료 시마다 `tailwindcss` 브랜치에 커밋, 전체 182개가 끝나기 전에 `next`로
  병합할지 여부는 사용자 판단(부분 병합 시 Bootstrap/Tailwind 공존 기간이 `next`에도 노출됨 —
  위 "공존 전략"이 안전한 이유가 바로 이 부분 병합 가능성 때문이기도 하다).
- `next`가 이 기간 동안 계속 전진하므로, 그룹 5개 정도 완료할 때마다 `git fetch origin && git rebase
  origin/next`로 드리프트를 주기적으로 해소할 것을 권장(각 그룹 완료 시점을 자연스러운 rebase
  체크포인트로 삼는다).

## 단계별 작업 계획

### Phase 0 — 빌드 파이프라인 및 디자인 토큰 (선행, 1회성)

1. Tailwind CLI(standalone) 버전 확정 및 `tailwind.config.js`/`src/main/tailwind/input.css` 신설,
   `corePlugins.preflight: false`로 설정
2. Gradle `tailwindBuild` Exec 태스크 추가, `processResources` 의존 배선
3. `htmx.min.js` 자체 호스팅 파일 추가(`static/javascripts/lib/htmx/`)
4. 디자인 토큰 정의 — 현재 `yobi.css`/`usermenu.css` 등에서 실제 사용 중인 브랜드 색상(GNB 배경색,
   프라이머리 버튼색 등)을 추출해 `tailwind.config.js`의 `theme.extend.colors`에 등록(완전히 새
   팔레트를 만들지 않고 기존 색을 토큰화 — "재설계 금지" 원칙과 일치)
5. `site/layout.html`/`site/layout_framed.html`에 `tailwind.css`/`htmx.min.js` `<link>`/`<script>`
   추가(이 시점엔 아직 아무 템플릿도 Tailwind 클래스를 안 씀 — 로드만 해두는 단계)
6. 검증: `./gradlew build`로 `tailwindBuild` 정상 실행 확인, 임의 템플릿에 테스트용 Tailwind
   클래스 하나 추가해 실제로 스타일이 반영되는지 브라우저로 1회 확인 후 제거

### Phase 1 — 그룹 1: 레이아웃 & 전역 뼈대 (2개 파일)

`site/layout.html`, `site/layout_framed.html`. 모든 페이지가 의존하므로 최우선. GNB/사이드바/푸터
등 반복 요소를 Tailwind로 전환하면 이후 그룹 작업 시 "이미 전환된 셸 안에 콘텐츠만 채우는" 형태가
되어 효율이 높아진다.

### Phase 2 — 그룹 2: `common/*` 공용 파샬 (16개 파일)

거의 모든 화면이 include. 이 중 htmx 전환 대상(댓글 폼/수정/삭제, 파일 업로드)이 몰려 있어 Phase 3
설계 개요의 htmx 패턴 카탈로그를 여기서 먼저 실증한다.

### Phase 3 이후 — 도메인별 그룹 (나머지 164개 파일)

`docs/TAILWIND_HTMX_BACKLOG.md`의 그룹 순서(레이아웃/공용 파샬 다음은 규모가 작고 의존이 적은
`site/*` 관리자 화면 → 최상위 인증/랜딩 → 나머지 도메인)를 따른다. 그룹 내부 순서는 자유(의존
방향이 얕음).

## 완료 기준 (Definition of Done)

- [ ] Phase 0(빌드 파이프라인) 완료 — `./gradlew build`가 `tailwind.css`를 재생성하고, htmx가
      로드되어 있음을 아무 페이지에서나 확인 가능
- [ ] `docs/TAILWIND_HTMX_BACKLOG.md`의 182개 항목이 전부 `[x]`(그룹 완료 시마다 갱신)
- [ ] 각 그룹 전환 시 대응 `*TemplateEquivalenceSpec.kt`가 새 마크업 기준으로 갱신되어 GREEN
- [ ] htmx 패턴 카탈로그(댓글/페이지네이션/알림 등)가 최소 1개 화면군에서 실제로 동작 확인
      (jQuery `$.ajax` 대신 `hx-*` 속성으로 서버 왕복 확인)
- [ ] 전체 회귀 스위트(`./gradlew test`) 그린 유지 — 백엔드 로직 변경이 없으므로 회귀 0건이 원칙
- [ ] 최종 정리 단계: `corePlugins.preflight: true` 전환 + `static/bootstrap/` 삭제 +
      전 화면 수동 스모크 체크 완료
- [ ] 코드/템플릿 리뷰 기준 커버리지 회귀 없음(백엔드 코드는 변경하지 않으므로 JaCoCo 수치 자체는
      불변이 기본 — 컨트롤러에 partial-fragment 분기를 추가한 경우에만 해당 분기 커버리지 확인)

## 리스크 / 미결정 사항

| 항목 | 내용 | 해소 방법 |
|---|---|---|
| Tailwind CLI 버전 고정 | 아직 실제 버전을 선택/기록하지 않음 | Phase 0 착수 시 최신 안정판으로 확정, 이 문서에 버전 기록 |
| CI 파이프라인 확인 | 이 저장소의 CI 설정(빌드 시 `tailwindBuild` 실행 여부)을 아직 확인하지 않음 | Phase 0에서 CI 설정 파일 위치 확인 후 반영 |
| 부분 병합 시 공존 기간 노출 | 그룹 일부만 끝난 상태로 `next`에 병합하면 Bootstrap+Tailwind 공존이 `next`에도 보임(안전하게 설계했지만 완전 무위험은 아님) | 병합 시점은 사용자 판단, 이 문서의 "공존 전략"이 안전망 역할 |
| htmx 대상 파셜의 컨트롤러 응답 형식(JSON vs HTML) 전수 미확인 | 182개 파일 전부를 이 단계에서 미리 조사하지 않음 | 그룹 작업 착수 시 화면별로 개별 확인·처리(TEMPLATE_BACKLOG와 동일 방식) |
| 시각적 회귀는 자동화로 못 잡음 | CSS 셀렉터 단언은 구조 회귀만 잡고 여백/정렬 등 시각 회귀는 못 잡음 | 그룹 완료마다 수동 브라우저 스모크 체크를 DoD에 명시 |

## 관련

- 백로그 원본(신규 등록): [`docs/parity/tickets/p3-45.md`](../../parity/tickets/p3-45.md)
- 파일 단위 진행 백로그: [`docs/TAILWIND_HTMX_BACKLOG.md`](../../TAILWIND_HTMX_BACKLOG.md)
- 선행 완료 작업: `docs/TEMPLATE_BACKLOG.md`(legacy→Thymeleaf 이식, 242/242 완료 — 이 작업의 전제)
- 경쟁/대안 계획: [[p3-13-decoupled-spa-frontend]](프런트엔드 SPA 분리 — 상호 배타적 대안 경로)
- 관련 소스: `src/main/resources/templates/**`, `src/main/resources/static/bootstrap/`,
  `src/main/resources/static/javascripts/**`, `src/test/kotlin/.../web/*TemplateEquivalenceSpec.kt`
- 관련 브랜치: `tailwindcss`(로컬), `search5/tailwindcss`(원격, `git@github.com:search5/yona.git`)
