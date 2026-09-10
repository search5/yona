# Tailwind CSS + htmx 전환 백로그

[[docs/yona-wiki/plans/p3-45-tailwind-htmx-template-refactor]](대상 계획서)의 파일 단위 진행 상태
추적표. yona 템플릿(`src/main/resources/templates/**/*.html`, 총 **182개**, 2026-09-10 기준)을
Bootstrap 2.3.1 + jQuery 3.3.1 → Tailwind CSS + htmx로 점진 전환하기 위한 그룹별 작업 순서표.
형식·원칙은 `docs/TEMPLATE_BACKLOG.md`(legacy→Thymeleaf 이식 백로그, 242/242 완료)를 그대로
차용하되, 대조 대상이 "legacy 원본"이 아니라 "현재 yona 화면 자신"이라는 점만 다르다.

## 작업 원칙 (반드시 준수)

- **재설계 금지.** 이 백로그의 모든 항목은 "화면을 새로 디자인"하는 게 아니라 **현재 화면의
  정보구조·기능·필드를 그대로 유지한 채 마크업/CSS/상호작용 구현만 교체**하는 작업이다. 레이아웃
  구조, 조건 분기, 표시되는 필드와 그 순서, 텍스트/메시지 키는 손대지 않는다. 예외적으로 시각적
  재디자인이 필요하다고 판단되면 코드를 먼저 바꾸지 말고 사용자 확인부터 받는다.
- **전환 단위는 파일(화면) 전체다.** 한 파일 안에서 레이아웃 클래스(그리드/여백/타이포그래피)를
  절반만 Tailwind로 바꾸지 않는다 — Bootstrap 그리드(`.row`/`.span*`)와 Tailwind 그리드(`flex`/
  `grid`)가 한 파일 안에 섞이면 레이아웃이 쉽게 깨진다. 다만 그 파일이 include하는 **공용 파샬이
  아직 미전환**이면 파샬 내부는 과도기적으로 Bootstrap 클래스가 남을 수 있다(`[~]`로 표시).
- **Preflight는 전체 전환 완료 전까지 끈 상태를 유지한다**(`tailwind.config.js`의
  `corePlugins.preflight: false`) — 이유는 계획서의 "Bootstrap 2.3.1과의 공존 전략" 참고. 개별
  파일 작업 중에 이 설정을 임의로 켜지 않는다.
- **htmx 전환은 "서버가 HTML 조각을 다시 그려주는" 상호작용에만 적용한다.** 마크다운 에디터, code/
  diff 뷰어, Select2, Pikaday, Jcrop, ViewerJS, `atjs` 멘션 자동완성 등 클라이언트 전용 위젯
  라이브러리는 이번 라운드에서 교체하지 않는다(계획서 "비범위" 참고).
- 작업 순서는 **의존성 우선**(레이아웃/공용 파샬 → 규모 작은 화면군 → 나머지 도메인)이며, 그룹
  내부는 파일시스템 순서를 따른다. 번호는 전체 작업 순서(1~182)를 나타내지만, 레이아웃/공용 파샬
  이후의 그룹 간에는 의존이 얕으므로 반드시 순서대로 할 필요는 없다.
- **화면 그룹마다 다음 순서를 지킨다**:
  1. 대상 파일을 읽고 현재 사용 중인 Bootstrap 클래스/jQuery 훅(`id`/`data-*`/이벤트 바인딩)을
     목록화한다.
  2. 대응하는 `*TemplateEquivalenceSpec.kt`가 있으면, Bootstrap 클래스명이나 jQuery가 만드는
     DOM 구조를 셀렉터로 쓰는 단언이 있는지 확인한다.
  3. Tailwind 유틸리티 클래스로 마크업을 재작성. 서버 왕복형 상호작용이 있으면 htmx 속성으로 전환
     (계획서의 htmx 패턴 카탈로그 참고). 컨트롤러가 JSON만 반환한다면 HTML 조각을 반환하는 분기를
     추가한다.
  4. 기존 스펙의 셀렉터를 새 마크업 기준으로 갱신(가능하면 CSS 클래스 대신 `id`/`data-*` 기준으로
     옮겨 이후 스타일 변경에 강건하게 만든다). 필요하면 htmx 응답(`HX-Request` 헤더)을 검증하는
     테스트를 추가.
  5. `./gradlew test --tests "..."`로 회귀 확인 후 전체 스위트로 마무리 확인.
  6. 그룹 완료 시 브라우저로 골든 패스 스모크 체크 1회(자동화 테스트가 못 잡는 시각 회귀 확인).
  7. 이 문서의 상태(`[ ]`/`[~]` → `[x]`)와 완료 로그, 필요시 계획서의 리스크 표를 갱신하고 커밋.

## 상태 기호

| 기호 | 의미 |
|---|---|
| `[ ]` | 미착수 |
| `[~]` | 부분 전환(예: 파일 자체는 Tailwind로 바뀌었으나 include하는 공용 파샬이 아직 미전환) |
| `[x]` | 전환 완료 — 대응 스펙 GREEN + 그룹 스모크 체크 완료 |

---

## 그룹 1 — 레이아웃 & 전역 뼈대 (2개, #1~2)

다른 모든 템플릿이 extends/include 하므로 최우선. Phase 0(빌드 파이프라인)이 끝난 직후 착수.

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 1 | [ ] | `site/layout.html` | GNB/사이드바/푸터/스크립트 슬롯 전부 포함하는 최상위 셸. 댓글 에디터/파일 업로더 등 `th:fragment` 다수 정의 |
| 2 | [ ] | `site/layout_framed.html` | 사이드바+iframe 프레임 셸(`/user/sidebar` 등에서 사용) |

## 그룹 2 — `common/*` 공용 파샬 (16개, #3~18)

htmx 패턴 카탈로그(댓글 CRUD, 파일 업로드)를 여기서 먼저 실증한다.

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 3 | [ ] | `common/attachmentFile.html` | |
| 4 | [ ] | `common/branchItem.html` | |
| 5 | [ ] | `common/calendar.html` | Pikaday 연동 — 위젯 자체는 비범위, 감싸는 마크업만 대상 |
| 6 | [ ] | `common/child_commentForm.html` | htmx 전환 대상(댓글 작성) |
| 7 | [ ] | `common/childComments.html` | |
| 8 | [ ] | `common/childCommentsAnchorDiv.html` | |
| 9 | [ ] | `common/commentDeleteModal.html` | htmx 전환 대상(댓글 삭제 — `hx-delete`) |
| 10 | [ ] | `common/commentFormOnThread.html` | htmx 전환 대상(댓글 작성) |
| 11 | [ ] | `common/commentUpdateForm.html` | htmx 전환 대상(댓글 수정 — `hx-put`) |
| 12 | [ ] | `common/commitMsg.html` | |
| 13 | [ ] | `common/mySeriesMenuTab.html` | |
| 14 | [ ] | `common/partial_history.html` | |
| 15 | [ ] | `common/reviewForm.html` | htmx 전환 검토 대상 |
| 16 | [ ] | `common/select2.html` | Select2 위젯 자체는 비범위, 감싸는 마크업만 대상 |
| 17 | [ ] | `common/uploadForm.html` | 파일 업로드 — 드래그앤드롭은 클라이언트 로직 유지, 폼 마크업만 대상 |
| 18 | [ ] | `common/usermenu_tab_content_list.html` | |

## 그룹 3 — `site/*` 관리자 화면 (레이아웃 제외, 14개, #19~32)

규모가 작고 서로 의존이 얕아 그룹 1·2 다음으로 착수하기 좋다.

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 19 | [ ] | `site/data.html` | |
| 20 | [ ] | `site/diagnostic.html` | |
| 21 | [ ] | `site/issueList.html` | |
| 22 | [ ] | `site/mail.html` | |
| 23 | [ ] | `site/massMail.html` | |
| 24 | [ ] | `site/oauth_apps.html` | |
| 25 | [ ] | `site/partial_pagination.html` | htmx 전환 대상(페이지네이션) |
| 26 | [ ] | `site/partial_paginationForUserList.html` | htmx 전환 대상(페이지네이션) |
| 27 | [ ] | `site/postList.html` | |
| 28 | [ ] | `site/projectList.html` | |
| 29 | [ ] | `site/setting.html` | |
| 30 | [ ] | `site/sso.html` | |
| 31 | [ ] | `site/update.html` | |
| 32 | [ ] | `site/userList.html` | |

## 그룹 4 — 최상위 인증/랜딩 (6개, #33~38)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 33 | [ ] | `login.html` | |
| 34 | [ ] | `login_2fa.html` | |
| 35 | [ ] | `signup.html` | |
| 36 | [ ] | `index.html` | 최초 진입 화면(프로젝트/공지 목록) |
| 37 | [ ] | `bootstrap-setup.html` | 최초 설정 마법사 |
| 38 | [ ] | `bootstrap-restart.html` | |

## 그룹 5 — `index/*` (2개, #39~40)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 39 | [ ] | `index/notifications.html` | |
| 40 | [ ] | `index/partial_notifications.html` | htmx 전환 대상(알림 배지 폴링) |

## 그룹 6 — `user/*` (24개, #41~64)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 41 | [ ] | `user/edit.html` | |
| 42 | [ ] | `user/edit_emails.html` | |
| 43 | [ ] | `user/edit_gpg_keys.html` | |
| 44 | [ ] | `user/edit_gpg_keys_new.html` | |
| 45 | [ ] | `user/edit_notifications.html` | |
| 46 | [ ] | `user/edit_oauth_apps.html` | |
| 47 | [ ] | `user/edit_oauth_apps_owned.html` | |
| 48 | [ ] | `user/edit_oauth_apps_owned_new.html` | |
| 49 | [ ] | `user/edit_password.html` | |
| 50 | [ ] | `user/edit_security.html` | |
| 51 | [ ] | `user/edit_security_backup_codes.html` | |
| 52 | [ ] | `user/edit_security_totp_new.html` | |
| 53 | [ ] | `user/edit_security_webauthn_new.html` | |
| 54 | [ ] | `user/edit_ssh_keys.html` | |
| 55 | [ ] | `user/edit_ssh_keys_new.html` | |
| 56 | [ ] | `user/edit_token.html` | |
| 57 | [ ] | `user/edit_tokens.html` | |
| 58 | [ ] | `user/edit_tokens_new.html` | |
| 59 | [ ] | `user/lostPassword.html` | |
| 60 | [ ] | `user/partial_edit_tabmenu.html` | |
| 61 | [ ] | `user/resetPassword.html` | |
| 62 | [ ] | `user/userFiles.html` | |
| 63 | [ ] | `user/verified.html` | |
| 64 | [ ] | `user/view.html` | |

## 그룹 7 — `organization/*` (17개, #65~81)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 65 | [ ] | `organization/view.html` | |
| 66 | [ ] | `organization/create.html` | |
| 67 | [ ] | `organization/delete.html` | |
| 68 | [ ] | `organization/header.html` | |
| 69 | [ ] | `organization/list.html` | |
| 70 | [ ] | `organization/members.html` | |
| 71 | [ ] | `organization/menu.html` | |
| 72 | [ ] | `organization/setting.html` | |
| 73 | [ ] | `organization/partial_settingmenu.html` | |
| 74 | [ ] | `organization/boardList.html` | |
| 75 | [ ] | `organization/boardList_partial.html` | htmx 전환 대상 |
| 76 | [ ] | `organization/issueList.html` | |
| 77 | [ ] | `organization/issueList_partial.html` | htmx 전환 대상 |
| 78 | [ ] | `organization/issueList_quicksearch.html` | htmx 전환 대상 |
| 79 | [ ] | `organization/issueSearch_partial.html` | htmx 전환 대상 |
| 80 | [ ] | `organization/pullRequestList.html` | |
| 81 | [ ] | `organization/pullRequestList_partial.html` | htmx 전환 대상 |

## 그룹 8 — `project/*` (22개, #82~103)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 82 | [ ] | `project/home.html` | |
| 83 | [ ] | `project/header.html` | |
| 84 | [ ] | `project/menu.html` | |
| 85 | [ ] | `project/create.html` | |
| 86 | [ ] | `project/list.html` | |
| 87 | [ ] | `project/members.html` | |
| 88 | [ ] | `project/setting.html` | |
| 89 | [ ] | `project/setting_menu.html` | |
| 90 | [ ] | `project/setting_branch_protection.html` | |
| 91 | [ ] | `project/setting_deploykeys.html` | |
| 92 | [ ] | `project/setting_webhook.html` | |
| 93 | [ ] | `project/change_vcs.html` | |
| 94 | [ ] | `project/delete.html` | |
| 95 | [ ] | `project/fork.html` | |
| 96 | [ ] | `project/importing.html` | |
| 97 | [ ] | `project/issuelabels.html` | |
| 98 | [ ] | `project/partial_issuelabels_list.html` | htmx 전환 대상 |
| 99 | [ ] | `project/partial_issuelabels_editcategory.html` | htmx 전환 대상 |
| 100 | [ ] | `project/partial_issuelabels_editlabel.html` | htmx 전환 대상 |
| 101 | [ ] | `project/statistics.html` | |
| 102 | [ ] | `project/transfer.html` | |
| 103 | [ ] | `project/watchers.html` | |

## 그룹 9 — `issue/*` (14개, #104~117)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 104 | [ ] | `issue/list.html` | |
| 105 | [ ] | `issue/view.html` | |
| 106 | [ ] | `issue/create.html` | |
| 107 | [ ] | `issue/edit.html` | |
| 108 | [ ] | `issue/my_list.html` | |
| 109 | [ ] | `issue/my_partial_list.html` | htmx 전환 대상 |
| 110 | [ ] | `issue/my_partial_list_quicksearch.html` | htmx 전환 대상 |
| 111 | [ ] | `issue/my_partial_search.html` | htmx 전환 대상 |
| 112 | [ ] | `issue/partial_list.html` | htmx 전환 대상(페이지네이션) |
| 113 | [ ] | `issue/partial_massupdate.html` | |
| 114 | [ ] | `issue/partial_select_label.html` | htmx 전환 대상 |
| 115 | [ ] | `issue/partial_show_selected_label.html` | htmx 전환 대상 |
| 116 | [ ] | `issue/partial_view_child.html` | |
| 117 | [ ] | `issue/partial_view_childIssueList.html` | |

## 그룹 10 — `board/*` (4개, #118~121)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 118 | [ ] | `board/list.html` | |
| 119 | [ ] | `board/view.html` | |
| 120 | [ ] | `board/create.html` | |
| 121 | [ ] | `board/edit.html` | |

## 그룹 11 — `milestone/*` (5개, #122~126)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 122 | [ ] | `milestone/list.html` | |
| 123 | [ ] | `milestone/view.html` | |
| 124 | [ ] | `milestone/create.html` | |
| 125 | [ ] | `milestone/edit.html` | |
| 126 | [ ] | `milestone/partial_status.html` | htmx 전환 대상 |

## 그룹 12 — `pullrequest/*` (21개, #127~147)

코드/diff 렌더링이 몰려 있어 클라이언트 위젯(비범위) 비중이 큼 — 마크업만 대상으로 신중히 판단.

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 127 | [ ] | `pullrequest/list.html` | |
| 128 | [ ] | `pullrequest/view.html` | |
| 129 | [ ] | `pullrequest/create.html` | |
| 130 | [ ] | `pullrequest/edit.html` | |
| 131 | [ ] | `pullrequest/clone.html` | |
| 132 | [ ] | `pullrequest/partial_list.html` | htmx 전환 대상(페이지네이션) |
| 133 | [ ] | `pullrequest/partial_search.html` | htmx 전환 대상 |
| 134 | [ ] | `pullrequest/partial_state.html` | htmx 전환 대상 |
| 135 | [ ] | `pullrequest/partial_info.html` | |
| 136 | [ ] | `pullrequest/partial_branch.html` | |
| 137 | [ ] | `pullrequest/partial_forklist.html` | |
| 138 | [ ] | `pullrequest/partial_recently_pushed_branches.html` | |
| 139 | [ ] | `pullrequest/partial_merge_result.html` | htmx 전환 대상 |
| 140 | [ ] | `pullrequest/partial_pull_request_event.html` | |
| 141 | [ ] | `pullrequest/partial_reviewlist.html` | |
| 142 | [ ] | `pullrequest/partial_diff.html` | diff 뷰어(클라이언트 위젯 비중 큼) — 마크업만 대상 |
| 143 | [ ] | `pullrequest/partial_filediff.html` | 상동 |
| 144 | [ ] | `pullrequest/partial_diff_line.html` | 상동 |
| 145 | [ ] | `pullrequest/partial_diff_comment_on_line.html` | htmx 전환 대상(라인 댓글) |
| 146 | [ ] | `pullrequest/partial_comment_thread.html` | htmx 전환 대상 |
| 147 | [ ] | `pullrequest/partial_comment_form_on_thread.html` | htmx 전환 대상 |

## 그룹 13 — `reviewthread/*` (2개, #148~149)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 148 | [ ] | `reviewthread/list.html` | |
| 149 | [ ] | `reviewthread/partial_list.html` | htmx 전환 대상 |

## 그룹 14 — `code/*` (11개, #150~160)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 150 | [ ] | `code/view.html` | |
| 151 | [ ] | `code/history.html` | |
| 152 | [ ] | `code/branches.html` | |
| 153 | [ ] | `code/tags.html` | |
| 154 | [ ] | `code/diff.html` | diff 뷰어(클라이언트 위젯 비중 큼) — 마크업만 대상 |
| 155 | [ ] | `code/svnDiff.html` | 상동 |
| 156 | [ ] | `code/compare.html` | |
| 157 | [ ] | `code/compare_svn.html` | |
| 158 | [ ] | `code/nohead.html` | |
| 159 | [ ] | `code/nohead_svn.html` | |
| 160 | [ ] | `code/partial_nonrange_codecomment_thread.html` | htmx 전환 대상 |

## 그룹 15 — `wiki/*` (4개, #161~164)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 161 | [ ] | `wiki/view.html` | |
| 162 | [ ] | `wiki/edit.html` | |
| 163 | [ ] | `wiki/history.html` | |
| 164 | [ ] | `wiki/search.html` | |

## 그룹 16 — 기타 단일 화면 (8개, #165~172)

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 165 | [ ] | `search/list.html` | |
| 166 | [ ] | `migration/home.html` | |
| 167 | [ ] | `oauth2/consent.html` | |
| 168 | [ ] | `help/toc.html` | |
| 169 | [ ] | `help/keymap.html` | |
| 170 | [ ] | `help/markdown.html` | |
| 171 | [ ] | `help/experimental.html` | |
| 172 | [ ] | `help/UIKit.html` | |

## 그룹 17 — `error/*` (10개, #173~182)

레이아웃 의존 여부가 파일마다 다를 수 있음(Spring Boot `DefaultErrorViewResolver`가 상태코드로
암묵 매칭하는 `error/401.html` 사례가 TEMPLATE_BACKLOG에 기록돼 있음 — 삭제/구조 변경 시 특히 주의).

| # | 상태 | 경로 | 비고 |
|---|---|---|---|
| 173 | [ ] | `error/400.html` | |
| 174 | [ ] | `error/401.html` | 암묵적 상태코드 매칭으로 실제 사용 중(TEMPLATE_BACKLOG #145 참고) — 삭제 금지 |
| 175 | [ ] | `error/403.html` | |
| 176 | [ ] | `error/404.html` | |
| 177 | [ ] | `error/413.html` | |
| 178 | [ ] | `error/500.html` | |
| 179 | [ ] | `error/badrequest.html` | |
| 180 | [ ] | `error/forbidden.html` | |
| 181 | [ ] | `error/forbidden_organization.html` | |
| 182 | [ ] | `error/notfound.html` | |

---

## 진행 로그

(그룹 완료 시마다 아래에 `### 그룹 N 완료 (YYYY-MM-DD)` 형식으로 TEMPLATE_BACKLOG와 동일하게
작성 — 발견한 이슈, 갱신한 스펙, 스모크 체크 결과 포함)
