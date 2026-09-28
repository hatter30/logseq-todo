# 포켓 할 일 (Logseq × GitHub 모바일 웹앱)

GitHub 레포에 저장된 Logseq 그래프를 iPhone에서 할 일 앱처럼 쓰기 위한 단일 HTML 웹앱.
GitHub Pages에 올려 Safari "홈 화면에 추가"로 사용한다.

## 원칙
- 파일은 `index.html` 하나. 빌드 도구, 프레임워크, 서버 없음. 외부 리소스는 Google Fonts(IBM Plex Sans KR)뿐.
- 토큰은 코드에 절대 넣지 않는다. 사용자가 설정 화면에서 입력하면 localStorage에만 저장.
- 탭 내용은 앱이 정하지 않는다. 탭 하나 = 사용자가 쓴 쿼리 하나(이름, 쿼리, 정렬, 묶기).
- UI 문구는 한국어, 짧은 평서문. 모바일(iOS Safari) 우선, safe-area 대응, 라이트/다크 토큰.

## 구조 (index.html 안의 script)
- `// CORE-START` ~ `// CORE-END`: DOM 없는 순수 함수. Node로 테스트 가능.
  - `parseFile(path, text, journalRegex)` → `{page, blocks}`. 블록: marker, pri, title, extra, scheduled, deadline, repeat, refs, pathRefs(조상 참조 + 페이지 이름/tags/alias 상속), line, raw
  - 블록의 자기 줄 범위는 `b.line` ~ `b.end`(제외). `applyBlame(blocks, ranges)`가 그 줄들의 git blame 시각 중 최신을 `b.edited`(로컬 YYYY-MM-DD)로 넣는다.
  - `parseQuery(q)` → OR 그룹 배열, `matchQuery(block, groups, today)`
  - 파일 수정: `appendBlock`, `setMarker`, `setScheduled`, `advanceRepeats`(`.+`, `++`, `+` 반복), `setTitle`(첫 줄 글자만. 들여쓰기·대시·상태 유지, 줄바꿈은 공백; 새 줄은 `titleLine(raw, title)`), `addChildren`(b의 마지막 하위(손자 포함) 다음에 직계 하위 추가. 들여쓰기는 기존 하위를 따르고 없으면 탭), `removeBlocks`(블록을 하위째 삭제, 아래쪽부터). 둘 다 `subtreeEnd(ls, i)`로 하위 범위를 잡는다
  - 수정 함수는 `b.line`의 줄이 `b.raw`와 같은지 확인하고, 다르면 `indexOf(raw)`로 다시 찾는다. 못 찾으면 에러.
- 그 아래: 저장소(localStorage `lqtodo.v1`, IndexedDB `lqtodo`/`files` {path, sha, text, blame}), GitHub API, 렌더링, 시트(빠른 추가 / 블록 동작 / 탭 편집), 이벤트.
- 블록 동작 시트: 누른 블록의 첫 줄과 모든 하위 블록 첫 줄을 입력창으로 보여 준다(하위 블록 상태는 오른쪽에 표시만). `+ 하위 블록`으로 직계 하위를 여러 줄 추가(새 줄에서 Enter면 한 줄 더). 하위 블록 ✕는 하위째 '삭제 예정' 표시(↺로 되살림), 새 줄의 ✕는 바로 제거. 맨 아래 `이 블록 삭제`는 두 번 눌러야 실행(첫 번째는 하위 개수 안내)되고 `removeBlocks`로 하위째 삭제. 저장 전 변경은 `actPending()` → `{edits, news, dels}`(dels는 부모가 같이 지워지는 블록 제외). `저장`은 한 커밋. 상태/예정일 버튼을 누르면 이 변경도 같은 커밋에 넣는다(`applyPending`: 글자 편집 → 하위 삭제 → 하위 추가 → 첫 줄 raw가 바뀐 `bb`로 상태/날짜 적용).

## 데이터 흐름
- 시작: IndexedDB 캐시로 즉시 렌더 → 백그라운드 `sync(true)`.
- `sync`: `git/trees/{branch}?recursive=1`로 `journals/`, `pages/`의 `.md` 목록과 sha를 받고, 캐시와 sha가 다른 파일만 `git/blobs/{sha}`로 받음(동시 6개). 사라진 파일은 캐시에서 삭제. `range`(일) 설정 시 오래된 저널 제외.
- 이어서 blame이 없거나 sha가 바뀐 파일은 GraphQL `blame`으로 줄별 마지막 커밋 시각(authoredDate)을 받음(파일 10개씩 한 요청, 동시 3개). 같은 요청의 `file(path){oid}`가 캐시 sha와 같을 때만 `{sha, ranges:[[시작줄,끝줄,ms]]}`로 저장. 실패해도 동기화는 계속.
- 보기/탭 전환/쿼리: 네트워크 없음. 메모리 인덱스만 사용.
- 쓰기(`writeFile`): contents API로 해당 파일 최신본 GET → transform → PUT(sha 포함). 409/422면 1회 재시도. 성공 시 캐시에 새 sha와 텍스트 반영 후 `rebuild()`, 이어서 그 파일 blame만 다시 받음.
- 앱 복귀 시 마지막 동기화 후 2분이 지났을 때만 자동 동기화.

## Logseq 파일 규칙
- 저널: `journals/yyyy_MM_dd.md`(형식은 설정값, config.edn의 `:journal/file-name-format`과 맞춤). 페이지: `pages/이름.md`, `___`는 네임스페이스 `/`.
- 블록은 `- `로 시작, 들여쓰기는 탭(스페이스 2칸도 허용). 속성 `key:: value`, `:LOGBOOK:` ~ `:END:` 드로어는 무시.
- 예정일 줄: `  SCHEDULED: <2026-09-30 Wed>` (블록 들여쓰기 + 2칸). 반복 표기와 시간은 보존.
- 빠른 추가는 항상 오늘 저널 끝에 붙인다. 현재 탭 쿼리의 첫 OR 그룹에 있는 긍정 참조(#태그, [[페이지]])를 자동으로 붙이는 옵션이 있고, `due:today/tomorrow`면 날짜 기본값도 따라간다.
- 상태는 NOW/LATER 흐름만 쓴다. 블록 시트의 상태 버튼은 NOW, LATER, DONE, 일반 블록. 체크 해제와 반복 완료 후 상태는 LATER. 빠른 추가는 상태 버튼(NOW, LATER, 일반 블록)으로 고르며 열 때마다 NOW가 기본. 빠른 추가에도 블록 시트와 같은 `+ 하위 블록` 줄(`addKidRow(#addKids)`, ✕, Enter로 한 줄 더)이 있고, 하위 블록은 이번에 붙인 마지막 블록 아래에 `addChildren`으로 들어간다. 읽기는 모든 마커(TODO, DOING 등)를 그대로 인식한다.

## 쿼리 문법
단어, `[[페이지]]`, `#태그`, `is:now|later|open|done|task|repeat`, `marker:now,later`,
`due:today|tomorrow|overdue|week|+Nd|any|none|YYYY-MM-DD`, `in:journal|page`, `page:이름`,
`from:7d`, `to:today`(저널 날짜), `edited:today|yesterday|7d|YYYY-MM-DD|any|none`(줄의 마지막 커밋 날짜), `pri:a`, 앞에 `-`면 제외, 대문자 `OR`로 그룹 구분. 따옴표로 공백 포함 값.

## 테스트
브라우저(Node 없이): `python -m http.server 8765 --bind 127.0.0.1` 후 `http://127.0.0.1:8765/test.html`.
test.html이 index.html의 CORE 부분을 뽑아 샘플 마크다운으로 검증한다. file://로 열면 동작하지 않는다.

Node가 있으면:
```bash
python3 -c "s=open('index.html').read();c=s.split('// CORE-START')[1].split('// CORE-END')[0];open('core.js','w').write(c+'\nmodule.exports={parseFile,parseQuery,matchQuery,appendBlock,setMarker,setScheduled,advanceRepeats,fmtToRegex,applyBlame,titleLine,setTitle,addChildren,removeBlocks};')"
node t.js   # core.js를 require해서 샘플 마크다운으로 검증
```
전체 스크립트 문법 확인: script 부분을 app.js로 뽑아 `node --check app.js`.

## 다음 작업 후보
- [ ] 동기화 최적화: 브랜치 최신 커밋 sha(`GET /repos/{o}/{r}/commits/{branch}` 또는 `git/ref/heads/{branch}`)를 저장해 두고, 같으면 trees 요청을 건너뛰기.
- [ ] 데스크톱 Logseq와의 충돌 안내 개선(쓰기 전 원문이 캐시와 다르면 알림).
- [ ] 필요 시: 오프라인 쓰기 대기열, 여러 줄 본문 편집, 마감일(DEADLINE) 설정.
