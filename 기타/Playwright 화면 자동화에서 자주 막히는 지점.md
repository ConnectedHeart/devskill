# Playwright 화면 자동화에서 자주 막히는 지점

> 분류: 기타 · 최근 갱신: 2026-10-28

## 한 줄 요약
오래된 웹 화면을 Playwright로 자동화할 때는 숫자로 시작하는 id 선택자, `window.event`에 기대는 핸들러, 다시 그려지는 목록, 예상 못 한 확인 창, 느린 응답과 네트워크 끊김 때문에 자주 멈춘다. 실제 클릭을 쓰고, 건마다 예외를 가두고, 실패한 데이터는 그 자리에서 정리한다.

## 내용
### 선택자
- CSS에서 id가 숫자로 시작하면(`2step`) `#2step`은 **문법 오류인 선택자**다. 클릭이 오류로 끝나거나 무시되어, 탭이 안 열린 채 다음 단계(숨은 탭 안의 트리·버튼)에서 시간 초과가 난다. `[id="2step"]`으로 쓴다.
- 탭을 눌렀으면 탭 내용이 실제로 보이는지 확인한 뒤 진행한다. 값을 직접 넣는 필드는 탭이 숨어 있어도 동작해서 문제가 늦게 드러난다.

### 함수 직접 호출 대신 실제 클릭
- 오래된 화면(IE 시절 코드)의 핸들러는 전역 `window.event`로 "어느 요소를 눌렀는지"를 확인하는 경우가 있다. 스크립트에서 함수를 직접 부르면 이벤트가 없어 아무것도 하지 않고 끝난다(오류도 안 난다).
- 트리 노드 클릭 함수를 직접 부르는 것도 같은 이유로 실패하거나 화면 상태가 준비되지 않아 예외가 난다.
- 사람이 하는 순서대로 행을 클릭하고 버튼을 클릭한다. 결과(목록 개수 등)를 다시 읽어 바뀌었는지 확인한다.
- `catch (e) { alert(e.description) }` 같은 IE 전용 속성은 다른 브라우저에서 "undefined"만 띄운다. 알림 문구로 원인을 알 수 없으면 `page.on('pageerror')`, 콘솔 로그로 실제 예외를 잡는다.

### 다시 그려지는 목록과 이전 상태
- 검색 버튼을 누른 직후 결과 행을 누르면, 응답이 목록을 다시 그리기 전의 **이전 행**을 누를 수 있다. 검색 응답(`waitForResponse`)을 기다린 뒤 행을 다시 찾아 누른다.
- 입력 창이 지난번 값(지난 결재선, 최근 수신자 등)을 자동으로 불러오는 화면이 있다. 시작할 때 기본 항목만 남기고 비운 뒤 넣어야 "이미 등록됨" 알림이나 다른 사람이 섞이는 일이 없다.

### 확인 창(레이어)은 골라서 답하기
- 상태에 따라 "부재중인 사용자입니다. 추가하시겠습니까?" 같은 확인 창이 뜬다. 스크립트가 모르는 창이 뜨면 그 자리에서 멈춘 것처럼 보인다.
- 예상한 확인 창 하나에만 자동으로 답하고, 그 사실을 로그에 남긴다. 모든 창에 무조건 [예]를 누르면 잘못된 데이터가 만들어져도 모른다.
- 사전 조건을 바꾸는 작업(예: 부재 설정)은 그 조건이 필요한 단계 직전에 하고, 끝나면 되돌린다. 다른 곳에 같은 계정으로 열린 화면이 설정을 덮어쓸 수 있으므로 필요한 단계 직전에 다시 설정하고 조회로 확인한다.

### 대기 시간
- 대기 한도를 모두 짧게 줄이면 편집기 로딩, 팝업 준비, 느린 API가 그 한도를 넘겨 대부분 실패한다. 무거운 단계(편집기·팝업 준비, API 응답, 저장 완료)만 길게(예: 30초), 버튼·검색 대기는 짧게(예: 10초) 둔다.
- 한 건의 시간 초과 예외가 **실행 전체**를 멈추지 않게 건별 `try/catch`로 가둔다.

### 실패한 데이터 정리
- 건마다 "어디까지 진행됐는지"를 기록해 두고, 저장 전에 멈추면 바로 지우고, 저장 후에 멈추면 취소 후 삭제를 시도한다. 상태를 조회해서 판단하는 방식은 판단이 틀리면 정리를 건너뛴다.
- 지우지 못한 것은 ID와 정리 방법(쿼리 등)을 파일에 남기고, 다음 실행 시작 때 먼저 정리하게 한다.
- 진행 상태를 파일에 저장하면 끊겨도 같은 명령으로 이어서 실행할 수 있다.

### 네트워크가 끊기는 환경
- 긴 실행 중 무선 연결이 끊기고 다른 네트워크로 자동 연결되는 일이 있다. 오류 코드(`ENOTFOUND`, `ETIMEDOUT`, `ECONNRESET`)로 네트워크 오류를 구분해 재연결 작업을 실행하고, 대상 서버가 응답할 때까지 기다린 뒤 그 건만 다시 시도한다.
- 실행 중 사람이 보고 싶어 하면 화면을 띄우는 대신(`headless: false`, 사람이 누르면 포커스가 바뀌어 실패) 진행 로그를 따라가는 별도 창을 띄우는 편이 안전하다.

### 스크립트 수정 시 함정
- 셸 명령(heredoc, `sed`, `echo`)으로 코드를 파일에 넣으면 정규식의 `\d`가 `d`로 바뀌는 등 백슬래시가 사라질 수 있다. 넣은 뒤에는 **실제 파일의 함수**를 불러 결과를 확인한다. 따로 시험한 코드가 맞아도 파일에 들어간 줄이 다를 수 있다.

## 핵심 코드
### 1) 숫자로 시작하는 id와 탭 열림 확인
```javascript
// 바꿔야 할 곳: 탭·내용 id
await page.locator('[id="2step"]').click();                 // '#2step' 은 잘못된 선택자
await page.locator('[id="2stepBody"]').waitFor({ state: 'visible', timeout: 10000 });
```

### 2) 검색 응답을 기다린 뒤 결과 행 클릭, 안 되면 버튼으로
```javascript
const [resp] = await Promise.all([
  page.waitForResponse(r => r.url().includes('/api/users/search') && r.ok(), { timeout: 30000 }),
  page.locator('#btnSearch').click(),
]);
const row = page.locator('#resultList tr', { hasText: '<사용자 이름>' }).first();  // 다시 찾는다
await row.click();
await page.locator('#btnAdd').click();                       // 함수 직접 호출 대신 실제 버튼
if (await page.locator('#selectedList tr').count() === 0) throw new Error('추가 안 됨');
```

### 3) 예상한 확인 창 하나에만 답하기
```javascript
const confirmLayer = page.locator('.confirm-layer', { hasText: '부재중' });
if (await confirmLayer.isVisible({ timeout: 2000 }).catch(() => false)) {
  await confirmLayer.locator('button', { hasText: '예' }).click();
  log('자동응답:[예] 부재중 사용자 추가');
}
// 다른 문구의 창은 답하지 않고 실패로 기록
const other = page.locator('.confirm-layer:visible');
if (await other.count() > 0) throw new Error('예상 밖 확인 창: ' + await other.first().innerText());
```

### 4) 건별로 예외를 가두고 단계에 맞게 정리
```javascript
for (const item of items) {
  let stage = 'start', docId = null;
  try {
    docId = await openDraft(item); stage = 'drafted';
    await fillAndSave(item);       stage = 'saved';
    await approve(docId);          stage = 'done';
    state.done[item.id] = docId;
  } catch (e) {
    log(`FAIL ${item.id} stage=${stage} ${e.message}`);
    if (stage === 'drafted') await deleteDraft(docId);              // 저장 전: 바로 삭제
    else if (stage === 'saved') await cancelThenDelete(docId);      // 저장 후: 취소 후 삭제
    if (docId && !(await isDeleted(docId))) state.pendingCleanup.push(docId);  // 다음 실행이 정리
  }
  saveState(state);                                                  // 끊겨도 이어서 실행
}
```

### 5) 네트워크 오류면 재연결 후 그 건만 재시도
```javascript
const NET_ERR = /ENOTFOUND|ETIMEDOUT|ECONNRESET|ECONNREFUSED/;
async function withNetRetry(fn) {
  try { return await fn(); }
  catch (e) {
    if (!NET_ERR.test(String(e && e.message))) throw e;
    execSync('schtasks /Run /TN "<재연결 작업 이름>"');               // 바꿔야 할 곳: 재연결 방법
    const until = Date.now() + 60000;
    while (Date.now() < until) {
      try { await request.get(BASE + '/health', { timeout: 5000 }); break; } catch (_) {}
      await new Promise(r => setTimeout(r, 3000));
    }
    return await fn();                                                // 한 번만 다시 시도
  }
}
```

### 6) 실제 파일에 들어간 함수로 확인
```bash
# 코드를 파일에 넣은 뒤, 시험 코드가 아니라 파일의 함수를 직접 불러 본다
node -e "const m=require('./script.js'); console.log(m.makeTitle('A1','r2'))"
grep -n 'r\\d+' script.js     # 정규식이 깨지지 않았는지 눈으로도 확인
```

## 주의할 점
- 화면 자동화로 만든 데이터는 화면 기본값(공개 범위 등)을 그대로 따른다. 조건이 중요한 시험 데이터라면 기본값을 명시적으로 지정하고 다시 읽어 확인한다.
- 같은 계정으로 다른 사람이 동시에 접속해 있으면 설정이 덮어써질 수 있다. 시험 계정은 실행 동안 다른 곳에서 쓰지 않게 알린다.
- 자동화가 만든 "완료" 결과와 실제 권한 동작(누가 열 수 있는가)은 별개다. 후자는 계정별로 열어 보는 확인을 따로 해야 한다.

## 참고
- Playwright 문서: Locators, `waitForResponse`, `page.on('dialog')`, `page.on('pageerror')`
- [목록 화면에서 선택 행과 전역 변수가 어긋나는 버그](목록 화면에서 선택 행과 전역 변수가 어긋나는 버그.md)
