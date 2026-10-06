# JavaScript var 스코프와 중첩 루프 버그

> 분류: 기타 · 최근 갱신: 2026-10-06

## 한 줄 요약
`var`는 블록이 아니라 함수 스코프라서, 한 함수 안의 바깥 루프와 안쪽 루프가 같은 이름(`i`)을 쓰면 하나의 변수를 공유해 바깥 루프의 위치가 망가진다.

## 내용
- `for (var i = 0; ...)`의 `i`는 그 `for` 블록이 아니라 감싸고 있는 함수 전체에 속한다. 같은 함수 안에서 `var i`를 다시 선언해도 새 변수가 생기지 않고 같은 변수에 값을 덮어쓴다.
- 안쪽 루프가 끝나면 `i`는 안쪽 루프의 종료 값이 된다. 그 값이 바깥 루프의 현재 위치보다 작으면 바깥 루프가 앞쪽으로 되돌아가 같은 항목을 다시 처리하고, 크면 항목을 건너뛴다.
- 증상이 조건에 따라 달라져 찾기 어렵다.
  - 다시 처리되는 항목이 "이미 처리됨" 검사로 조용히 건너뛰어지면 결과는 정상으로 보인다.
  - 다시 지나갈 때 경고창을 띄우는 항목이 있으면 그 경고만 여러 번 뜬다. "특정 항목이 목록 중간에 있을 때만, 뒤에 있는 항목 수만큼 반복"처럼 위치와 개수에 비례하는 증상이면 이 버그를 의심한다.
  - 안쪽 루프가 조건부로만 실행되면(예: 항목이 실제로 추가됐을 때만) 재현 조건이 더 좁아진다.
- 고치는 방법은 안쪽 변수 이름을 바꾸거나 `let`을 쓰는 것이다. `let`은 블록 스코프라 루프마다 별도 변수다.

## 예시
```js
function addAll(items) {
  for (var i = 0; i < items.length; i++) {
    if (items[i].blocked) { alert('선택할 수 없습니다'); continue; }
    if (exists(items[i])) continue;       // 중복은 조용히 건너뜀
    var row = append(items[i]);
    var inputs = row.getElementsByTagName('input');
    for (var i = 0; i < inputs.length; i++) {   // 바깥 i를 덮어씀
      inputs[i].checked = true;
    }
    // 여기서 i == inputs.length → 바깥 루프가 앞쪽부터 다시 훑음
  }
}

// 수정: 다른 이름을 쓰거나 let 사용
for (var k = 0; k < inputs.length; k++) { inputs[k].checked = true; }
```

## 주의할 점
- 같은 코드가 복사돼 다른 화면에도 있을 가능성이 높다. 한 곳을 고치면 같은 패턴을 검색해 함께 확인한다.
- 오래된 브라우저 지원 때문에 `var`만 쓰는 코드베이스에서는 중첩 루프 변수를 `i`, `j`, `k`로 구분하는 규칙을 지킨다.
- ESLint의 `no-redeclare`, `block-scoped-var`, `no-var` 규칙이 이런 재선언을 잡아 준다.

## 참고
- MDN — var, let, Hoisting
