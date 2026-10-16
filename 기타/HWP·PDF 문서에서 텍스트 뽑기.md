# HWP·PDF 문서에서 텍스트 뽑기

> 분류: 기타 · 최근 갱신: 2026-10-15

## 한 줄 요약
HWP는 전용 바이너리라 일반 도구로 못 읽으므로 hwplib(Java)이나 pyhwp(Python) 같은 파서가 필요하고, PDF는 `pdftotext`로 뽑되 한글 CID 글꼴 데이터(poppler-data)가 없으면 한글만 빠진 채 추출된다.

## 내용
### HWP
- `.hwp`는 한글 전용 바이너리 형식(OLE 복합 문서)이다. 텍스트 에디터나 일반 문서 도구로는 내용을 볼 수 없다.
- Java 프로젝트라면 `hwplib` jar가 이미 의존성에 있는 경우가 있다. 작은 추출 프로그램을 만들어 문단 텍스트를 뽑을 수 있다. Python이 있으면 `pyhwp`의 `hwp5txt`가 간단하다.
- `.hwpx`는 zip 안의 XML이라 압축을 풀고 XML 텍스트를 읽으면 된다.

### PDF
- `pdftotext 파일.pdf 결과.txt`(poppler)로 텍스트를 뽑는다. `-layout`을 주면 표 모양이 어느 정도 유지된다.
- 한글 PDF에서 `Unknown character collection 'Adobe-Korea1'` 경고와 함께 영문·숫자만 남는다면 poppler의 CID 글꼴 데이터(poppler-data)가 설치되지 않은 것이다. 데이터를 설치하거나 다른 도구(한글/Word로 열어 텍스트 저장, Python `pdfplumber`·`PyMuPDF`)를 쓴다.
- 텍스트 추출이 안 되는 스캔 PDF는 `pdftoppm`으로 페이지를 이미지로 만들어 보거나 OCR이 필요하다.

### 뽑은 텍스트의 한계
- 슬라이드·표의 순서가 뒤섞이고 그림 안 글자는 빠진다. "문서에 없다"고 결론 내리기 전에 영문 키워드로 한 번 더 검색하고, 중요한 내용은 원본을 직접 연다.
- 추출 프로그램과 결과 파일은 프로젝트 밖(임시 폴더)에 두어 저장소에 섞이지 않게 한다.

## 예시
```java
// hwplib 으로 문단 텍스트 추출 (개념)
HWPFile hwp = HWPReader.fromFile(path);
for (Section s : hwp.getBodyText().getSectionList()) {
    for (Paragraph p : s.getParagraphs()) {
        System.out.println(p.getNormalString());
    }
}
```
```bash
pdftotext -layout guide.pdf guide.txt
hwp5txt guide.hwp > guide.txt          # pyhwp
```

## 주의할 점
- 문서에 비밀번호·인증서 경로·내부 주소가 섞여 있으면 추출 결과를 공유 폴더나 대화 기록에 그대로 남기지 않는다.

## 참고
- hwplib (GitHub: neolord0/hwplib), pyhwp
- poppler-utils: pdftotext, pdftoppm, poppler-data
