# 끼워 넣은 span이 주변 글꼴을 따르지 않는 이유

> 분류: 기타 · 최근 갱신: 2026-10-21

## 한 줄 요약
본문 일부를 `<span class="mark">`로 감싸 강조했는데 그 부분만 작은 기본 글꼴로 보이면, 공통 CSS에 `span { font-size: …; font-family: … }` 같은 태그 전역 규칙이 있는 것이다. 표시 클래스에 `font-size: inherit; font-family: inherit;`을 주면 바깥 글자를 따른다.

## 내용
- 상속(inherit)은 그 요소에 직접 걸린 규칙이 **없을 때만** 일어난다. 태그 선택자(`span { … }`)라도 직접 걸리면 상속값보다 우선한다.
- 그래서 원문이 22pt·특정 글꼴이어도, 프로그램이 낱말 단위로 끼운 `span`은 전역 규칙의 13px·기본 글꼴로 바뀐다. 끼운 위치가 맞아도 그 부분만 작아 보인다.
- 전역 규칙을 지우면 그 CSS를 쓰는 다른 화면이 모두 영향을 받는다. 문제 화면의 컨테이너 안 표시 클래스에만 `inherit`을 준다.
- 색·배경은 그대로 두고 글꼴 관련 속성만 상속시키면 강조 표시는 유지된다.
- 확인은 개발자 도구의 Computed 값으로 바깥 문단과 표시 요소의 `font-size`(px로 계산된 값), `font-family`가 같은지 본다.

## 예시
```css
/* 공통 CSS (건드리지 않음) */
span { font-size: 13px; font-family: 'Noto Sans KR', sans-serif; }

/* 문제 화면에만 */
.result-box .mark-error,
.result-box .mark-suggest,
.result-box .mark-done {
  font-size: inherit;
  font-family: inherit;
}
```

## 주의할 점
- `font: inherit` 단축 속성은 굵기·줄 높이 등도 함께 상속시켜 표시 모양이 바뀔 수 있다. 필요한 속성만 지정한다.
- 같은 결과 창을 여러 화면이 공유하면 한 번 고친 것이 모두에 적용된다. 영향 화면을 함께 확인한다.

## 참고
- MDN: inherit, CSS 상속과 명시도(specificity)
