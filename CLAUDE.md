# 독서노트 — 작업 규칙 (CLAUDE.md)

이 저장소에서 작업할 때 항상 지켜야 하는 규칙이에요. 무엇을 언제 왜 바꿨는지는 `HISTORY.md`에 있어요.

- **무엇:** 레이아웃을 자유롭게 꾸미는 독서노트 웹앱
- **파일:** `index.html` 하나 (GitHub Pages에 그대로 올려서 씀)
- **현재 버전:** 1.9.0 (2026-11-07)
- **사용자:** 한국어 사용자, 아이폰 · 안드로이드 · PC를 모두 씀
- **소통:** 사용자에게는 항상 **한국어**로, 개발 용어보다 쉬운 말로 설명해 주세요.

---

## 1. 절대 규칙 (처음부터 정한 필수 항목)

어떤 수정을 하더라도 아래가 깨지면 안 돼요.

1. **데이터 가져오기 · 내보내기** (JSON 백업 / 가져오기)
2. **실시간 자동 저장** + 저장된 **날짜와 시간**을 화면에 표시 (`#saveStat`, 저장 기록). 자동 저장은 그대로 두고, **'저장' 버튼**(`#btnSave`)으로 직접 저장도 할 수 있어야 해요 (v1.8.0 요청)
3. **업데이트가 있으면 버전으로 체크**하고 팝업으로 알림 → 업데이트 뒤에는 "업데이트 완료" 팝업
4. **모든 자료를 PDF로 내보내기** 가능
5. 폰트는 **Pretendard**
6. 배경색은 **흰색**
7. **HTML 파일 하나** (GitHub 업로드용). 빌드 도구 · 프레임워크 · 별도 CSS/JS 파일을 쓰지 않음
8. 모든 것은 **수정 · 삭제 · 정렬** 가능
9. 삭제할 때는 **한 번 더 확인** (`confirmDialog`)

처음 요청한 화면 구성:
- 섹션: Info. · Motivation · Liked · Unliked · Learned · Action · Comment (이후 Words 추가)
- 레이아웃은 위치를 자유롭게 바꿀 수 있고, 한 행은 **최대 3단**
- 섹션 제목 수정 · 삭제 가능
- **Info**는 그 안에서 책 제목 · 작가 · 출판사 · 별점 같은 **필드를 옵션으로 설정**
- 소설은 **등장인물 마인드맵** (v1.6.0부터 부모 · 자식 가지)
- 책마다 **표지 이미지**를 직접 넣을 수 있음

---

## 2. 데이터 보존 규칙 (가장 중요)

사용자가 "데이터 보존이 중요하다"고 했어요. 데이터에 손대는 수정은 아래를 모두 지켜야 해요.

### 2-1. 읽기와 저장
- **'읽기 실패'와 '기록 없음'은 다른 상황이에요.** `loadState()`가 `error`를 돌려주면 아무것도 새로 만들거나 저장하지 않아요. 이때는 `bootReadError()`로 다시 시도 화면만 보여 줘요. 예시 책으로 덮어쓰던 버그가 v1.7.0에서 고친 가장 심각한 문제였어요.
- **모든 데이터 변경은 `commit(what, book)`을 거쳐요.** `state`를 바꾸고 `commit`을 부르지 않으면 저장되지 않아요.
  - `commit`은 `changeSeq++` 한 뒤 450ms 후에 `saveNow()`를 실행해요.
  - 저장 기록 문구 `what`은 한국어로 짧게 써요 (예: `'메모 삭제'`).
- `isDirty()`는 `changeSeq !== savedSeq`예요. "저장 안 된 변경이 있는지"는 꼭 이걸로 판단해요. `saveTimer`나 `pendingMsg`로 판단하면 안 돼요.
- 저장이 실패하면 자동으로 다시 시도해요 (`retryTimer`, 2초부터 최대 60초 간격).
- 화면을 벗어날 때(`visibilitychange` · `pagehide` · `freeze` · `beforeunload`)는 `emergencySave()`가 실행돼요.
  - 비상 사본은 `localStorage 'rn:emergency'`에 동기로 남겨요.
  - 이미 저장된 표지는 용량 때문에 `'rn:keep-cover'` 표시로 바꿔 넣어요.
  - 비상 사본은 그 내용이 실제로 저장된 뒤에만 지워요 (`emSeq` 비교).
- 시각을 찍을 때는 `stamp()`를 써요. 항상 앞선 값보다 커서 비상 사본과 저장본의 순서를 비교할 수 있어요.

### 2-2. 지우기
- 무언가를 지우는 기능은 아래 순서를 **반드시** 지켜요.
  ```js
  if (!await confirmDialog({ title: '…삭제할까요?', message: '…' })) return;
  const hold = holdBook(b);          // 지우기 직전 책 사본
  /* …삭제… */
  commit('○○ 삭제', b); /* 화면 갱신 */
  offerUndo(hold, '○○을 삭제했어요');   // '되돌리기' 알림
  ```
- 큰 변경 직전에는 **복원 지점**을 만들어요: `makeSnap(reason)`.
  - 대상: 책 삭제 · 가져오기 · 모든 데이터 삭제 · 복원.
  - 변경 전 상태가 필요하면 먼저 `captureState()`로 굳혀 둔 뒤 `makeSnap(reason, cap)`.
  - 새 이유를 추가하면 `SNAP_REASON`에도 한국어 이름을 넣어요.
- 사용자가 입력한 값이 사라질 수 있는 변경은 미리 물어요 (예: 필드 종류 변경 → `fieldText()` 비교).

### 2-3. 데이터 구조를 바꿀 때
- 옛 데이터가 **그대로 열려야** 해요. 불러올 때 `migrate()` → `normalizeBook()` · `normalizeSettings()`가 정리해요.
- **새 설정 키는 `normalizeSettings()`에 꼭 추가해요.** 빠뜨리면 불러올 때 그 값이 지워져요.
- 새 최상위 키는 `migrate()` 기본값에 추가해요.
- 한 번만 하는 이전 작업은 `state.flags.xxx`로 표시해요 (예: `flags.wordsTpl` = 템플릿에 Words 한 번 추가).
- 가져온 JSON은 믿지 않아요. id는 `safeId`, 표지는 `COVER_RX`와 길이 제한, 값은 문자열로 변환, 카테고리 중첩은 1단계로 펼쳐요.
- 메모 항목 · 챕터 객체의 **모르는 필드는 지우지 않고 그대로 둬요** (`normItem`은 `Object.assign` 후 아는 필드만 정리). id를 바꿀 때는 그 id를 가리키는 값도 함께 바꿔요 (예: 챕터 id → 메모의 `ch`).
- 백업 형식은 `exportShape()` 하나로 통일해요: `{ app:'reading-note', version, exportedAt, books, template, settings }`.

### 2-4. 여러 탭
- BroadcastChannel `'reading-note'`로 저장을 알리고, 없는 브라우저는 `localStorage 'rn:ping'`로 알려요.
- 다른 탭이 더 최근에 저장했으면(`otherTabNewer`) 덮어쓰기 전에 충돌 창(`showConflict`)으로 물어요.

### 2-5. 저장소 이름표 — 절대 바꾸지 마세요

| 위치 | 이름 | 용도 |
|---|---|---|
| IndexedDB | DB `'reading-note'` v1, store `'kv'` | 기본 저장소 |
| IndexedDB kv | `'state'` | 전체 데이터 |
| IndexedDB kv | `'snaps'`, `'snap:<id>'` | 복원 지점 목록 / 본문(JSON 글자) — 최대 10개, 약 30MB |
| localStorage | `rn:state` | IndexedDB가 안 될 때만 쓰는 대신 저장본 |
| localStorage | `rn:emergency` | 화면을 벗어날 때 남기는 비상 사본 |
| localStorage | `rn:where` | 마지막으로 저장한 곳 (`idb` / `ls`) — "저장한 적 있음" 표시도 겸함 |
| localStorage | `rn:seenVersion` | "업데이트 완료" 팝업용 |
| localStorage | `rn:ping` | BroadcastChannel이 없을 때 탭끼리 알림 |

---

## 3. 버전 · 배포 규칙

### 3-1. 버전 번호 (SemVer)
- **새 기능** → 둘째 자리 (1.7.0 → **1.8.0**)
- **버그 수정만** → 셋째 자리 (1.7.0 → **1.7.1**)
- 첫째 자리는 데이터 구조를 크게 바꿀 때만 올려요. 그때도 옛 데이터 이전은 필수예요.

### 3-2. 새 버전 내는 순서 (매번)
1. 시작 전에 지금 상태를 git에 커밋해 두어 되돌릴 수 있게 해요.
2. 수정 → 아래 4장의 버그 점검.
3. `<meta name="app-version" content="X.Y.Z">` 값을 올려요.
4. `<script type="application/json" id="changelog">` 배열 **맨 위**에 항목을 추가해요.
   ```json
   {
     "version": "1.8.0",
     "date": "2026-11-07",
     "notes": [
       "사용자가 알아들을 수 있는 말로, 해요체로, 한 줄씩",
       "3~6줄 정도"
     ]
   }
   ```
5. `HISTORY.md`에 같은 버전 항목을 추가해요 (사용자 요청 · 바뀐 점 · 정한 규칙 · 고친 버그의 원인).
6. 커밋 메시지는 `vX.Y.Z: 한 줄 요약`, 태그는 `vX.Y.Z`. 예전에는 버전마다 `index_vXYZ.html` 사본을 따로 저장했는데, 이제는 git 태그가 그 역할을 해요.

### 3-3. 업데이트 알림이 동작하는 방식 — 이름을 바꾸면 안 돼요
- 열려 있는 앱이 `location.pathname?vc=시각`으로 자기 파일을 다시 받아요. 그 안의 `meta[name="app-version"]`과 `#changelog`를 읽어서 더 높은 버전이면 "새 버전이 나왔어요" 팝업을 띄워요.
  - 실행 시점: 시작 5초 뒤, 30분마다, 화면으로 돌아왔을 때(마지막 확인 뒤 5분이 지났으면).
  - 글 쓰는 중이면 15초 미뤄요.
- 새로고침 뒤 `rn:seenVersion`보다 높으면 "vX.Y.Z로 업데이트되었어요" 팝업에 그 사이 변경 내역을 보여 줘요.
- 그래서 **`app-version` 메타 이름, `changelog` id, JSON 형식(`version`/`date`/`notes`)은 바꾸면 안 돼요.** 이미 사용자 폰에 열려 있는 옛 버전이 이걸 읽어요.
- 버전은 반드시 이전보다 커야 해요 (`semver()`로 비교).
- `file://`로 열면 업데이트 확인은 꺼져요 (정상).

---

## 4. 매 작업마다 하는 점검

사용자가 매번 "버그 없이", "업로드 전에 버그 점검"을 요청했어요. 수정 뒤에는 실제 브라우저로 확인해요.

### 4-1. 기본
- 로컬 서버: `python3 -m http.server 8765` → `http://localhost:8765/`
- 화면 폭 3가지로 확인: PC(1280px) · 휴대폰(390px) · 아이폰 UA(사파리 전용 처리 확인).
- 콘솔 오류 0개.
- **한글 입력**(IME 조합 중 Enter · Tab), 붙여넣기, 새로고침 뒤 내용이 남는지.
- 새 기능이 **JSON 내보내기 → 가져오기**, **PDF**, **인쇄(Ctrl+P)**에도 반영되는지.
- 수정 · 삭제(확인창 + 되돌리기) · 정렬(끌기)이 되는지.

### 4-2. 데이터 보존 회귀 점검
v1.7.0에서 자동 테스트로 확인한 시나리오예요. 저장 코드를 건드리면 다시 확인해요.
1. 처음 실행 → 예시 책 저장. 아무것도 안 바꾸고 새로고침하면 다시 저장하지 않음.
2. 여러 종류 입력 뒤 새로고침 → `state` 전체가 같음.
3. 입력하고 0.45초 안에 탭 닫기 → 다음에 열면 남아 있음.
4. 저장소 읽기 1회 실패 → 재시도로 정상. 계속 실패 → 다시 시도 화면, **아무것도 쓰지 않음**.
5. IndexedDB 열기가 멈춤 → 5초 뒤 재시도 / 계속 멈추면 다시 시도 화면.
6. 저장 중에 앱이 꺼짐 → 비상 사본으로 복구, 표지는 저장본에서.
7. 늦게 끝난 예전 저장이 더 최신 비상 사본을 지우지 않음.
8. 저장 실패 → "저장 실패 · 다시 시도 중" 표시 → 자동 재시도로 성공. IndexedDB만 실패하면 localStorage에 대신 저장했다가 다음에 다시 옮김.
9. IndexedDB가 없는 브라우저 → localStorage만으로 동작.
10. 6MB 표지가 있어도 비상 사본이 들어감.
11. 탭 2개 동시 수정 → 충돌 창 / BroadcastChannel이 없어도 알림.
12. 되돌리기(메모 · 단어 · 섹션 · 인물+가지 · 관계 · 책), 그 사이 다른 변경이 있으면 되돌리기 거절.
13. 복원 지점: 하루 1회 자동, 책 삭제 · 전체 삭제 직전 생성, 복원 · 내려받기 · 삭제, 10개 제한.
14. 내보내기 → 모든 데이터 삭제 → 가져오기(덮어쓰기) = 책 · 설정이 완전히 같음.
15. 이전 버전에서 쓰던 데이터를 새 버전으로 열어도 그대로.

### 4-3. 콘솔에서 쓸 수 있는 전역 (테스트용)
- `state`, `commit()`, `saveNow()`, `isDirty()`, `saving`, `loadState()`, `idbRun()`
- `listSnaps()`, `makeSnap()`, `curBook()`, `roleField()`, `makeBook()`, `render()`
- Playwright에서 저장 완료를 기다릴 때: `state && !isDirty() && !saving`
- 확인창을 띄우는 함수(`deleteBook` 등)를 `page.evaluate`에서 그대로 await하면 멈춰요. `(()=>{ deleteBook(b); })()`처럼 감싸서 부르고, 확인창 버튼을 눌러요.

---

## 5. 코드 구조 (`index.html` 안의 순서)

| 구역 | 내용 |
|---|---|
| `<head>` | 사용법 주석, `app-version`, Pretendard, SortableJS, `#changelog` JSON, 전체 CSS |
| `0. 기본값 · 유틸` | `APP_VERSION`, `TAB_ID`, `IS_IOS`, `$`/`$$`, `esc`, `icon()`, 마크다운 `md()` |
| `1. 데이터 정의` | `SEC`(섹션 종류), `FIELD_TYPES`, `PRESETS`, 색 · 팔레트, `normalizeBook`, `migrate`, `makeBook`, `sampleBook` |
| `2. 저장소` | IndexedDB/localStorage, `commit`/`saveNow`, 비상 사본, 다중 탭, 복원 지점, 되돌리기 |
| `3. 공통 UI` | `modal`, `confirmDialog`, `openMenu`, `toast`, `busy` |
| `4. 라우팅 · 렌더링` | `#/`(서재), `#/book/<id>`, `renderLibrary`, `renderBook`, Sortable 연결 |
| `4-b. 챕터 · 쓰인 문장 · 직접 저장` | 목차 편집 `openToc`, 챕터별 모아 보기 `openChOverview`, 문장 팝업 `openSentPop`, `manualSave` |
| `5. 등장인물 맵` | SVG 맵, 노드 · 관계 · 가지, `drawMap`, `autoLayout` |
| `6. 이벤트 (위임)` | `ACT` 객체 + `data-act` 클릭 위임, document 단위 `input`/`keydown` 처리 |
| `7. 가져오기 · 내보내기` | JSON, PDF(html2pdf), 인쇄용 `#printRoot` |
| `8. 설정 · 저장 기록 · 버전` | 설정 창, 복원 지점 UI, 업데이트 확인 |
| `9. 시작` | `boot()` — 읽기 → 렌더 → 이벤트 연결 |

데이터 모양 (요약):
- `state = { schema, books[], template, settings, saveLog[], savedAt, lastBackupAt, firstUseAt, backupSnoozeUntil, flags }`
- `book = { id, kind:'general'|'novel', createdAt, updatedAt, cover(dataURL|null), sections:{id→section}, rows:[{id, ratio[], cols:[[sectionId…]]}], chapters:[{id, kind:'part'|'ch'|'extra', title, page}] }`
  - 목차(`chapters`)는 책마다 따로예요. 기본 템플릿에는 넣지 않아요.
  - 순서는 **파트 → 챕터**. 챕터는 목차에서 바로 앞에 있는 파트에 속해요. `extra`(화면 이름 '기타': 프롤로그 · 에필로그 · 작가의 말 등)는 어디에도 속하지 않아요.
  - `page` = 시작 쪽. 메모의 챕터는 `effCh()`로 정해요: **직접 고른 `ch` → 쪽 번호(`autoChOf`) → 임시로 이어받은 `ch`(`chSoft:true`)**. 챕터를 보여 주거나 묶는 곳은 `e.ch` 대신 꼭 `effCh(b, e)`를 써요.
- 섹션 `type`: `info` · `motivation` · `liked` · `unliked` · `learned` · `action` · `comment` · `words` · `characters` · `custom`
  - Info: `fields:[{id, role, label, type, value, hidden, options?}]`
  - 메모형: `entries:[{id, text, page, done, word?, sentence?, ch?, chSoft?} | {id, kind:'group', title, collapsed, items:[…]}]` (카테고리는 1단계만)
    - `word` · `sentence`(단어가 쓰인 문장)는 Words에만, `ch`는 고른 챕터 id, `chSoft`는 앞 메모를 따라 임시로 정해진 챕터 표시
    - 섹션 보기 설정: `byCh`(챕터별로 묶어 보기), `sentHide`(Words: 문장은 눌러서 보기)
- `settings.pdf = { fs, lh, margin, cols, cover, map, sent, page, chap, byCh, head }` — PDF 상세 설정 (`normPdf`가 정리)
  - 등장인물: `map:{ nodes:[{id, name, role, desc, color, main, parent, collapsed, x, y}], edges:[{id, from, to, label, dir, style}] }`

---

## 6. 화면 · 디자인 규칙
- 흰 배경. 색은 `:root` CSS 변수(`--ink`, `--muted`, `--line`, `--accent`…)를 써요.
- 포인트 색은 사용자가 바꿀 수 있어요 (`--accent*`는 `applyTheme()`가 설정). 섹션 색은 `--c` / `--c-ink`.
- 색 값은 `'#RRGGBB'`(직접 고른 색) 또는 `'@팔레트ID'`(팔레트 연결) 둘 중 하나예요. 팔레트 색을 바꾸면 연결된 섹션 · 인물이 함께 바뀌어요. 글자 색은 `readableOn()`으로 흰 배경 대비 4.5 이상을 맞춰요.
- 레이아웃: 행마다 1~3단 + 비율(`RATIOS`).
  - **휴대폰에서도 정한 단 수를 유지**해요 ("1단 2단 3단의 의미가 없다"는 피드백으로 정함).
  - 설정 `stackMobile`을 켰을 때만 1단으로 쌓아요.
  - 섹션 안쪽 반응형은 `@container`로 처리해요.
- UI 문구는 **한국어 해요체**로 짧고 친절하게 써요 (`'저장했어요'`, `'삭제할까요?'`, `'되돌렸어요'`).
- 아이콘은 내장 `icon(name, size)` SVG를 써요. 외부 아이콘 폰트는 쓰지 않아요.
- 사용자 글을 HTML에 넣을 때는 항상 `esc()` 또는 `md()`를 거쳐요.

## 7. 모바일 · 입력 규칙 (버그로 배운 것)
- **iOS**
  - 입력칸 글자 크기는 16px 이상이어야 해요 (확대 방지). iOS에서는 viewport에 `maximum-scale=1`을 덧붙여요.
  - `.md-view`에는 `onclick="void 0"` + `cursor:pointer`가 있어야 탭이 인식돼요. 지우면 "글자가 안 적히는" 버그가 다시 생겨요.
- **포커스할 때 레이아웃이 밀리면 안 돼요.** 안드로이드에서 엉뚱한 곳이 눌려요. 버튼은 항상 자리에 두고 `visibility`로 숨겨요.
  - 누르는 도중(`mdPtrDown`)에 화면을 바꿔야 하면 미뤘다가 손을 뗀 뒤에 해요 (`mdPending` · `sentPending` · `tapQueue` → `mdFlush`).
  - 입력 중인 섹션은 다시 그리지 않아요. 챕터별 보기의 재묶음은 섹션 밖으로 나간 뒤에 해요 (`regroupSoon`).
- **한글 입력:** Enter · Tab · Escape를 처리하기 전에 `e.isComposing || e.keyCode === 229`를 확인해요.
- 글 쓰는 중에는 업데이트 팝업을 미뤄요 (`userIsTyping()`).
- 중요한 선택 창은 `modal({ focusDialog:true })`로 열어요. 쓰던 Enter · Space가 버튼을 누르지 않게 해요.
- 메뉴는 스크롤이 40px 넘을 때만 닫아요 (관성 스크롤 대응).
- Tab 이동 판단은 `lastTabAt > lastPtrAt` 기준이에요.
- 끌기(SortableJS)는 `forceFallback: true`로 써요. 중첩 그룹의 `put`은 **같은 그룹 이름일 때만** 허용해요. 다른 종류 섹션으로 메모가 넘어가지 않게 하려는 거예요.

## 8. 마크다운
- 직접 만든 파서 `md()`를 써요.
  - 지원 문법: 굵게 · 기울임 · 취소선 · 형광펜(`==`) · 제목 · 목록 · 할 일 · 인용 · 코드 · 링크 · 표 · 구분선.
  - 편집 단축키: Ctrl/⌘ + B · I · K.
  - Enter로 목록 자동 이어쓰기, Tab으로 들여쓰기.
- **XSS 안전**이 필수예요. 원본 HTML은 통과시키지 않고, 링크는 `http(s)://`와 `mailto:`만 허용해요.
- 내부 토큰은 사용자 영역 문자 ``~``를 써요. 소스에는 **반드시 `\uXXXX` 이스케이프로** 적어요. 실제 문자를 넣으면 깨져요. 이 문자는 입력에서 제거돼요.

## 9. 외부 라이브러리 (버전 고정)
- Pretendard `v1.3.9` (jsdelivr, dynamic subset)
- SortableJS `1.15.6`
- html2pdf.js `0.14.0` — PDF 만들 때만 늦게 불러와요 (`H2P_URL`).
- 새 라이브러리는 꼭 필요할 때만 추가해요. jsdelivr에서 **버전을 고정**해서 불러와요.
