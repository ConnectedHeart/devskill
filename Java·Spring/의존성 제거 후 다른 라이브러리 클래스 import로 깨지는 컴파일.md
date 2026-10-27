# 의존성 제거 후 다른 라이브러리 클래스 import로 깨지는 컴파일

> 분류: Java·Spring · 최근 갱신: 2026-10-27

## 한 줄 요약
보안 취약점 대응으로 jar를 빼면, 그 jar 안의 유틸 클래스(예: Tomcat의 Base64)를 우연히 import하던 코드가 컴파일 오류를 낸다. 같은 기능의 표준 라이브러리로 import를 바꾸면 된다.

## 내용
- IDE 자동 import는 클래스패스에 있는 같은 이름 클래스 중 아무거나 고를 수 있어서, 서버 내장 라이브러리(`org.apache.tomcat.util.codec.binary.Base64`)처럼 의도하지 않은 패키지가 들어가곤 한다.
- CVE 대응으로 `tomcat-util`, `tomcat-coyote` 같은 의존성을 pom에서 빼면 이런 import가 "package does not exist"로 깨진다.
- 해결: 이미 의존성에 있는 Apache Commons Codec(`org.apache.commons.codec.binary.Base64`)이나 JDK 8+ `java.util.Base64`로 바꾼다. Commons Codec의 `Base64.decodeBase64(String)` 등은 이름·동작이 같아 import만 바꾸면 된다.

## 핵심 코드
### 1) import 교체
```java
// 전
import org.apache.tomcat.util.codec.binary.Base64;
// 후 (같은 API)
import org.apache.commons.codec.binary.Base64;
// 또는 JDK 표준
byte[] bytes = java.util.Base64.getDecoder().decode(str);
```
의존성을 빼기 전에 그 jar의 패키지를 쓰는 곳 찾기: `grep -rn "org.apache.tomcat.util" src/`

### 2) 빼기 전에 사용처와 유입 경로 확인
```bash
# 소스에서 그 jar 패키지를 쓰는 곳
grep -rn "import org.apache.tomcat" src/main/java
# 어떤 의존성을 타고 들어오는지(직접인지 전이인지)
mvn dependency:tree -Dincludes=org.apache.tomcat
```

### 3) pom: 전이 의존성 제외와 대체 의존성 명시
```xml
<dependency>
    <groupId>com.example</groupId>
    <artifactId>some-lib</artifactId>              <!-- tomcat-util을 끌고 오는 의존성 -->
    <version>1.0.0</version>
    <exclusions>
        <exclusion>
            <groupId>org.apache.tomcat</groupId>
            <artifactId>tomcat-util</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- 우연히 다른 jar로 들어오던 클래스라면 이제 직접 선언한다 -->
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
    <version>1.15</version>
</dependency>
```

### 4) `java.util.Base64`로 바꿀 때 동작 맞추기
```java
byte[] a = java.util.Base64.getMimeDecoder().decode(textWithNewlines); // 줄바꿈·공백 섞인 입력
byte[] b = java.util.Base64.getUrlDecoder().decode(urlSafeText);       // '-', '_' 를 쓰는 입력
String s = java.util.Base64.getEncoder().encodeToString(bytes);        // 줄바꿈 없는 출력
// 기본 getDecoder()는 줄바꿈이 섞이면 IllegalArgumentException
```

## 주의할 점
- `java.util.Base64`로 바꿀 때는 줄바꿈·URL-safe 문자 처리 등 동작 차이를 확인한다(Commons Codec은 공백·줄바꿈을 관대하게 처리한다).
- 의존성 제거 커밋에서 전체 빌드를 한 번 돌려 확인한다.

## 참고
- Apache Commons Codec 문서, Java SE `java.util.Base64`
