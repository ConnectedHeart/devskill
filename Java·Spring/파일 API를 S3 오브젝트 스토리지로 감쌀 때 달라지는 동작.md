# 파일 API를 S3 오브젝트 스토리지로 감쌀 때 달라지는 동작

> 분류: Java·Spring · 최근 갱신: 2026-10-27

## 한 줄 요약
`java.io.File`과 같은 모양의 래퍼로 로컬 디스크와 S3 호환 스토리지를 함께 지원하면, 폴더 개념이 없는 오브젝트 스토리지 특성 때문에 `exists`·`listFiles`·`renameTo`·`mkdir` 등이 디스크와 다르게 동작한다. 래퍼를 거치지 않고 `new File`을 쓰거나 경로만 다른 서버에 넘기는 코드는 S3 모드에서 조용히 깨진다.

## 내용
### 오브젝트 스토리지(S3)란
- 원래 Amazon S3의 이름이지만 API가 사실상 표준이 되어 MinIO, Ceph 등 "S3 호환 스토리지"가 많다. 엔드포인트 주소를 직접 지정하고 path-style로 접속하면 사내에 설치한 호환 스토리지에도 붙는다.
- 버킷 안에 `키(경로 문자열) → 내용`이 평평하게 저장된다. `root/board/2026/a.pdf`의 `/`는 폴더가 아니라 키의 일부다.
- 읽기·쓰기·삭제·목록을 모두 HTTP로 요청한다. 여러 WAS가 같은 저장소를 보므로 이중화 환경에서 공유 마운트 없이 파일을 함께 쓸 수 있다.

### 래퍼 설계의 흔한 형태
- 기동 시 설정값 하나로 저장 모드(파일/오브젝트)를 정해 static으로 둔다 → 바꾸려면 재기동.
- 파일 모드에서는 내부 `File`에 그대로 위임해 `new File`과 똑같다.
- 오브젝트 모드에서는 공통 루트 이후 경로를 키로 바꿔 S3 API를 부른다. 임시·연계 폴더처럼 로컬이어야 하는 경로 목록은 예외로 디스크를 쓴다.

### 디스크와 달라지는 메소드
| 메소드 | 오브젝트 모드 동작 | 함정 |
|---|---|---|
| `exists()` | 키 존재 조회. 없으면 확장자 유무로 파일/폴더를 추정 | 확장자 없는 파일은 폴더로 오인 |
| `isDirectory()` | 오브젝트가 있거나 확장자가 있으면 false | 추정 기반 |
| `mkdir()`/`mkdirs()` | 폴더 개념이 없어 로컬에 만듦 | 다른 서버엔 없음(무해하지만 의미 없음) |
| `length()`/`lastModified()` | 메타데이터 HEAD 요청 | 호출마다 네트워크 요청 1회 |
| `listFiles()` | prefix 조회 | 하위 전체가 재귀로 나오고, `abc` prefix가 `abcd...`도 잡음 → 끝에 `/`를 붙이거나 필터로 바로 아래만 추려야 함 |
| `renameTo()` | 복사 후 원본 삭제 | 원자적이지 않음. 중간 실패 시 둘 다 있거나 둘 다 없을 수 있음 |
| `delete()` | 오브젝트 삭제 | 로컬에 다른 프로세스가 만든 사본은 남음 |

### S3 모드에서 깨지는 전형적인 코드
- `new File`, `FileInputStream`, `RandomAccessFile`, `Files.*`로 업무 파일을 직접 읽고 쓰기. 기본값이 파일 모드인 개발 환경에선 잘 돌아서 발견이 늦다. 예: Range(206) 스트리밍을 `RandomAccessFile`로 구현하면, 업로드 후 로컬 사본을 지우는 오브젝트 모드에서 404가 난다.
- 래퍼에서 `getFile()`로 내부 `File`을 꺼내 이어 쓰기.
- 파일 내용 대신 **경로만** 외부 서버에 넘기는 연동: 외부 서버는 공유 디스크를 전제로 하므로 파일을 못 읽고 "200 + 빈 결과"를 돌려준다.
- 외부 서버가 디스크에 직접 파일을 쓰고 우리 서버가 래퍼로 확인하는 구조: 래퍼는 S3를 조회하므로 디스크의 파일을 찾지 못한다.

## 핵심 코드
### 1) 모드 결정과 키 변환
```java
public final class StorageFile {
    static volatile boolean objectMode;              // 기동 시 한 번 설정
    static AmazonS3 s3; static String bucket; static String root = "/data/files/";

    private final File file; private final String key; private final boolean local;

    public StorageFile(String path) {
        this.file = new File(path);
        this.local = !objectMode || isLocalOnly(path);           // 임시·연계 폴더는 로컬
        this.key = path.startsWith(root) ? path.substring(root.length()) : path;
    }
    public boolean exists() { return local ? file.exists() : s3.doesObjectExist(bucket, key); }
    public long length()    { return local ? file.length()
                                          : s3.getObjectMetadata(bucket, key).getContentLength(); }
}
```

### 2) 바로 아래 항목만 나열 (prefix 함정 피하기)
```java
ListObjectsV2Request req = new ListObjectsV2Request()
        .withBucketName(bucket)
        .withPrefix(dirKey.endsWith("/") ? dirKey : dirKey + "/")  // abc/ 로 끝내야 abcd 제외
        .withDelimiter("/");                                       // 하위 폴더는 CommonPrefixes로
ListObjectsV2Result res = s3.listObjectsV2(req);
for (S3ObjectSummary o : res.getObjectSummaries()) System.out.println(o.getKey());
```

### 3) Range 스트리밍을 스토리지 독립적으로
```java
// RandomAccessFile 대신 범위 GET — 로컬이면 FileChannel, 오브젝트면 withRange
GetObjectRequest get = new GetObjectRequest(bucket, key).withRange(start, end);
try (S3ObjectInputStream in = s3.getObject(get).getObjectContent();
     OutputStream out = response.getOutputStream()) {
    response.setStatus(206);
    response.setHeader("Content-Range", "bytes " + start + "-" + end + "/" + total);
    response.setContentLengthLong(end - start + 1);
    in.transferTo(out);                                       // Java 8이면 버퍼 루프
}
```

### 4) 코드 리뷰용 grep
```bash
# 바꾼 Java에서 래퍼를 거치지 않는 파일 I/O 찾기
git diff --name-only -- '*.java' | xargs grep -nE \
  'new (java\.io\.)?File\(|FileInputStream\(|FileOutputStream\(|RandomAccessFile\(|Files\.(read|write|copy|move)|\.getFile\(\)'
```

## 주의할 점
- 기본값이 파일 모드인 환경에서만 테스트하면 이 차이를 못 본다. 오브젝트 모드 검증은 MinIO 같은 호환 스토리지를 로컬에 띄워 확인하거나, 못 하면 "코드 대조만 수행"이라고 명시한다.
- `length()`·`exists()`를 루프 안에서 부르면 파일 수만큼 HTTP 요청이 나간다.
- 출력 스트림은 try-with-resources로 닫아야 업로드가 끝난다(닫을 때 전송하는 구현이 많다).
- 접속 키는 설정 파일에 평문으로 두지 말고 비밀 관리 방식을 따른다.

## 참고
- AWS SDK for Java: ListObjectsV2 (prefix, delimiter), GetObjectRequest.withRange
- [파일 경로만 넘기는 서버 간 연동과 공유 저장소](../기타/파일 경로만 넘기는 서버 간 연동과 공유 저장소.md)
