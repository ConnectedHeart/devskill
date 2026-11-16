# HWP·PDF 문서에서 텍스트 뽑기

> 분류: 기타 · 최근 갱신: 2026-11-11

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

### 서식의 "본문"만 뽑기: 표 칸 필드와 누름틀
- 결재·보고 서식처럼 머리말·수신·결재란이 있는 문서에서 본문만 뽑으려면, 서식이 본문 자리에 붙인 **필드 이름**(예: `body`)을 찾는다.
- 본문 자리는 두 가지로 만들어질 수 있다. ① 표 셀에 필드 이름을 준 "셀 필드" ② 문단 안 "누름틀"(클릭하여 입력, hwpx에서는 `<hp:fieldBegin type="CLICK_HERE" name="body">` … `<hp:fieldEnd>`). 표 셀 이름만 찾는 추출기는 누름틀 서식을 "본문 칸 없음"으로 보고 문서 전체를 뽑는다.
- 찾는 순서: 셀 필드 → 누름틀 → 둘 다 없으면 문서 전체. 누름틀은 여러 문단·표 셀 안에 걸칠 수 있고, 안에 하이퍼링크 같은 다른 필드가 중첩될 수 있으므로 **같은 id의 끝**까지 짝을 맞춰 읽는다.
- 본문을 안 쓴 문서는 누름틀에 "내용을 입력하세요" 같은 안내 문구만 남는다. 안내 문구만 있으면 빈 본문(null)으로 돌려야 AI 요약 등에 서식 껍데기가 넘어가지 않는다.
- 확장자가 `.hwpx`인데 내용이 hwp 바이너리인 파일도 있다. 확장자만 믿고 고르면 열지 못하므로 zip 시그니처(`PK`)를 먼저 확인한다.

### 뽑은 텍스트의 한계
- 슬라이드·표의 순서가 뒤섞이고 그림 안 글자는 빠진다. "문서에 없다"고 결론 내리기 전에 영문 키워드로 한 번 더 검색하고, 중요한 내용은 원본을 직접 연다.
- 추출 프로그램과 결과 파일은 프로젝트 밖(임시 폴더)에 두어 저장소에 섞이지 않게 한다.

## 핵심 코드

### 1) 문단 텍스트 추출 개념과 명령
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

### 2) hwplib 추출 프로그램 (표·글상자 안 글자 포함)
```xml
<!-- Maven: 버전은 Maven Central 의 최신 값을 쓴다 -->
<dependency>
  <groupId>kr.dogfoot</groupId>
  <artifactId>hwplib</artifactId>
  <version><!-- 최신 버전 --></version>
</dependency>
```
```java
import kr.dogfoot.hwplib.object.HWPFile;
import kr.dogfoot.hwplib.reader.HWPReader;
import kr.dogfoot.hwplib.tool.textextractor.TextExtractMethod;
import kr.dogfoot.hwplib.tool.textextractor.TextExtractor;

public class HwpToText {
    public static void main(String[] args) throws Exception {
        HWPFile hwp = HWPReader.fromFile(args[0]);
        // 표·글상자 같은 컨트롤 안 글자를 문단 사이에 끼워 넣어 뽑는다
        String text = TextExtractor.extract(hwp, TextExtractMethod.InsertControlTextBetweenParagraphText);
        System.out.println(text);
    }
}
```
```bash
# hwplib 과 그 의존 jar 를 lib/ 에 두고 (Windows 는 ; 로 구분)
javac -cp "lib/*" HwpToText.java
java -cp "lib/*:." HwpToText guide.hwp > guide.txt
```

### 3) hwpx: zip 안 XML에서 글자 읽기 (Java 8 표준 API만)
```java
// import java.util.*; java.util.zip.*; javax.xml.parsers.*; org.w3c.dom.*;
static String hwpxToText(String path) throws Exception {
    StringBuilder out = new StringBuilder();
    DocumentBuilderFactory f = DocumentBuilderFactory.newInstance();
    f.setNamespaceAware(true);
    try (ZipFile zip = new ZipFile(path)) {
        List<String> sections = new ArrayList<String>();
        for (Enumeration<? extends ZipEntry> e = zip.entries(); e.hasMoreElements(); ) {
            String n = e.nextElement().getName();
            if (n.startsWith("Contents/section") && n.endsWith(".xml")) sections.add(n);
        }
        Collections.sort(sections);                 // section0, section1 … (10개 이상이면 숫자로 정렬)
        for (String n : sections) {
            Document doc = f.newDocumentBuilder().parse(zip.getInputStream(zip.getEntry(n)));
            NodeList ts = doc.getElementsByTagNameNS("*", "t");     // 글자는 <hp:t>
            Node lastP = null;
            for (int i = 0; i < ts.getLength(); i++) {
                Node p = ts.item(i).getParentNode();
                while (p != null && !"p".equals(p.getLocalName())) p = p.getParentNode();   // 소속 문단 <hp:p>
                if (lastP != null && p != lastP) out.append('\n');
                out.append(ts.item(i).getTextContent());
                lastP = p;
            }
            out.append('\n');
        }
    }
    return out.toString();
}
```

### 4) hwpx 누름틀(CLICK_HERE) 본문만 읽기
```java
// sectionN.xml 문서 순서대로 훑으며 name="body" 누름틀의 시작~짝이 맞는 끝 사이 글자만 모은다
static String clickHereBody(Document doc, String fieldName) {
    StringBuilder sb = new StringBuilder();
    String bodyId = null;              // 본문 누름틀의 id (fieldEnd의 beginIDRef와 짝)
    NodeList all = doc.getElementsByTagNameNS("*", "*");
    for (int i = 0; i < all.getLength(); i++) {
        Element el = (Element) all.item(i);
        String tag = el.getLocalName();
        if ("fieldBegin".equals(tag) && bodyId == null
                && "CLICK_HERE".equals(el.getAttribute("type")) && fieldName.equals(el.getAttribute("name"))) {
            bodyId = el.getAttribute("id");
        } else if ("fieldEnd".equals(tag) && bodyId != null && bodyId.equals(el.getAttribute("beginIDRef"))) {
            break;                     // 중첩된 다른 필드의 끝에서는 멈추지 않는다
        } else if ("t".equals(tag) && bodyId != null) {
            sb.append(el.getTextContent());
        } else if ("p".equals(tag) && bodyId != null && sb.length() > 0) {
            sb.append('\n');
        }
    }
    if (bodyId == null) return null;   // 누름틀 없음 → 호출한 쪽이 문서 전체로
    String text = sb.toString().trim();
    return text.isEmpty() || text.equals(GUIDE_TEXT) ? "" : text;   // 안내 문구만 있으면 빈 본문
}
```

### 5) PDF 한글 누락·스캔 PDF 처리
```bash
pdffonts guide.pdf                                   # 글꼴·CID(Adobe-Korea1) 사용 여부 확인
sudo apt-get install poppler-utils poppler-data      # Debian/Ubuntu
sudo dnf install poppler-utils poppler-data          # RHEL 계열
pdftotext -layout -enc UTF-8 -f 3 -l 5 guide.pdf -   # 3~5쪽만 표준출력으로

# 스캔 PDF: 이미지로 바꾼 뒤 OCR (tesseract + 한국어 데이터 필요)
pdftoppm -r 300 -png -f 1 -l 1 scan.pdf page         # page-1.png (쪽수에 따라 page-01.png)
tesseract page-1.png page-1 -l kor                   # page-1.txt 생성
```

## 주의할 점
- 문서에 비밀번호·인증서 경로·내부 주소가 섞여 있으면 추출 결과를 공유 폴더나 대화 기록에 그대로 남기지 않는다.

## 참고
- hwplib (GitHub: neolord0/hwplib), pyhwp
- poppler-utils: pdftotext, pdftoppm, poppler-data
