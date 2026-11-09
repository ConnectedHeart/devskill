# 정적 HTML에서 로컬 폴더 읽기(File System Access API)

> 분류: 기타 · 최근 갱신: 2026-11-06

## 한 줄 요약
더블클릭으로 연 `index.html`은 보안상 자기 폴더의 파일 목록을 읽을 수 없다. 사용자가 폴더를 한 번 고르게 하면(`showDirectoryPicker`) Chrome·Edge에서 하위 파일을 읽어 여러 결과물을 한 화면에 모아 보여 줄 수 있고, 폴더 핸들을 IndexedDB에 두면 다음부터는 "허용"만 누르면 된다.

## 내용
- `file://` 페이지는 `fetch`로 옆 파일을 읽거나 디렉터리를 나열할 수 없다. 서버 없이 "폴더에 넣은 결과물을 자동으로 모아 보기"를 하려면 사용자 동작으로 폴더 접근 권한을 받아야 한다.
- `window.showDirectoryPicker()`(Chrome·Edge): 고른 폴더의 `FileSystemDirectoryHandle`을 준다. `entries()`로 하위를 재귀 탐색하고 `getFile().text()`로 읽는다.
- 핸들은 구조화 복제가 되므로 IndexedDB에 저장할 수 있다. 다음에 열 때 `queryPermission`/`requestPermission`으로 다시 허용만 받으면 그사이 넣은 폴더까지 읽힌다.
- 지원하지 않는 브라우저는 `<input type="file" webkitdirectory>`로 폴더를 매번 고르게 한다(파일 목록과 `webkitRelativePath`가 온다).
- 모아 보기용 데이터는 항목마다 작은 메타 파일(JSON)과 화면 조각(HTML)으로 저장해 두면, 모음 페이지는 그것만 읽어 목록·검색·필터를 만든다. 같은 항목이 여러 번 들어 있으면 메타의 추출 시각으로 최신 것만 남긴다.
- "새로 들어옴" 표시는 지난번에 본 키 목록을 `localStorage`에 두고 비교한다. 저장소 키는 페이지 위치별이라 폴더마다 따로 관리된다.
- 압축 파일(zip)은 읽지 않으므로 풀어서 넣게 안내한다. 탐색 깊이 상한을 둔다.

## 핵심 코드
### 1) 폴더 고르기와 재귀 읽기
```javascript
async function pickFolder() {
  const dir = await window.showDirectoryPicker();      // 사용자 클릭 안에서만 호출 가능
  await saveHandle(dir);                               // 다음에 쓰려고 저장
  return dir;
}
async function collect(dir, depth = 0, out = []) {
  if (depth > 5) return out;                           // 깊이 상한
  for await (const [name, h] of dir.entries()) {
    if (h.kind === 'directory') {
      try {
        const meta = await (await (await h.getFileHandle('meta.json')).getFile()).text();
        const html = await (await (await h.getFileHandle('section.html')).getFile()).text();
        out.push({ ...JSON.parse(meta), html });
      } catch { await collect(h, depth + 1, out); }    // 메타가 없으면 더 들어간다
    }
  }
  return out;
}
```

### 2) 핸들 저장과 다시 허용 받기
```javascript
const DB = 'viewer', STORE = 'h';
function idb() { return new Promise((ok, ng) => { const r = indexedDB.open(DB, 1);
  r.onupgradeneeded = () => r.result.createObjectStore(STORE); r.onsuccess = () => ok(r.result); r.onerror = ng; }); }
async function saveHandle(h) { const db = await idb(); db.transaction(STORE, 'readwrite').objectStore(STORE).put(h, 'root'); }
async function loadHandle() { const db = await idb();
  return new Promise(ok => { const q = db.transaction(STORE).objectStore(STORE).get('root'); q.onsuccess = () => ok(q.result); }); }
async function reopen() {                               // [다시 읽기] 버튼 클릭에서 호출
  const h = await loadHandle(); if (!h) return pickFolder();
  if (await h.queryPermission({ mode: 'read' }) !== 'granted' &&
      await h.requestPermission({ mode: 'read' }) !== 'granted') return null;
  return h;
}
```

### 3) 미지원 브라우저 대체
```html
<input type="file" webkitdirectory multiple onchange="readFiles(this.files)">
<!-- files[i].webkitRelativePath 로 하위 경로를 알 수 있다 -->
```

### 4) 최신 것만 남기기와 새 항목 표시
```javascript
const byKey = new Map();
for (const it of items) {                              // 다른 출처가 섞이면 키를 "출처+번호"로
  const k = it.source + '#' + it.id, old = byKey.get(k);
  if (!old || old.exportedAt < it.exportedAt) byKey.set(k, it);
}
const seen = new Set(JSON.parse(localStorage.getItem('seen') || '[]'));
const fresh = [...byKey.keys()].filter(k => !seen.has(k));
localStorage.setItem('seen', JSON.stringify([...byKey.keys()]));
```

## 주의할 점
- `showDirectoryPicker`와 `requestPermission`은 사용자 클릭 같은 동작 안에서만 부를 수 있다. 페이지를 열자마자 자동으로 읽게 할 수는 없다.
- Firefox·Safari는 디렉터리 선택 API를 지원하지 않는다(대체 방식 필요).
- 폴더를 넣은 뒤에는 다시 읽어야 반영된다. 페이지가 폴더 변화를 감지하지는 않는다.
- 다른 출처(서버)에서 온 같은 번호를 번호만으로 합치면 하나가 숨겨진다. 키에 출처를 넣는다.

## 참고
- MDN: Window.showDirectoryPicker(), FileSystemHandle.requestPermission(), HTMLInputElement.webkitdirectory
