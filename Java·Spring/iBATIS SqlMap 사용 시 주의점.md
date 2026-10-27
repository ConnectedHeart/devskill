# iBATIS SqlMap 사용 시 주의점

> 분류: Java·Spring · 최근 갱신: 2026-10-27

## 한 줄 요약
iBATIS 2(SqlMap)에서 `resultClass`가 없는 `<select>`는 항상 null을 돌려주고, Map 파라미터의 키는 대소문자를 구분하며, CDATA 안의 주석은 SQL에 그대로 들어간다.

## 내용
### `resultClass`/`resultMap` 없는 `<select>`는 null
- `<select id="..." parameterClass="Map">`처럼 결과 매핑이 없으면, `SqlExecutor.handleResults`가 시작 부분에서 결과 매핑이 없는 경우 행 읽기를 통째로 건너뛴다. DB에 값이 있어도 `queryForObject`는 null이다.
- 오류가 나지 않아서, 받은 값을 화면에서 쓰지 않는 동안은 아무도 모른다. 단일 값 조회에는 `resultClass="String"`(또는 `int`, `java.lang.Integer`)을 꼭 적는다.

### Map 파라미터 키는 대소문자를 구분
- SQL이 `#ITEMID#`를 쓰는데 Java가 `map.put("itemID", ...)`로 넣으면 값이 null로 바인딩된다. UPDATE라면 0건 갱신, SELECT라면 0건 조회이고 역시 오류는 없다.
- 키 이름은 SQL 쪽을 기준으로 맞추거나 상수로 공유한다. 한쪽만 고치면(예: 분기 조건만 고쳐서 실제로 실행되게 만들면) 이번에는 0건 갱신이 조용히 실행된다. 두 문제가 겹쳐 있을 수 있다.

### CDATA 안에 쓴 주석은 SQL 문자열이 된다
- `<![CDATA[ ... ]]>` 안에 XML 주석(`<!-- -->`)을 넣으면 그대로 SQL 문자열에 섞여 DB 문법 오류가 난다. 설명은 `<insert>`·`<select>` 태그 바깥(위)에 XML 주석으로 쓰거나, SQL 주석(`--`, `/* */`)을 쓴다.

### 파라미터 유무에 따른 기본값
- 고정값으로 넣던 컬럼을 파라미터로 바꿀 때 기존 호출처를 모두 고치기 어려우면 `<isNotEmpty property="flag">#flag#</isNotEmpty><isEmpty property="flag">'Y'</isEmpty>`처럼 분기하면 기존 호출은 예전과 같이 동작한다.
- `<isEqual property="flag" compareValue="1">`로 SET 절을 조립하는 쿼리는, 어떤 값에도 맞지 않는 파라미터가 오면 `SET` 뒤가 비어 문법 오류가 난다. 호출 전에 값 범위를 좁힌다.

### IN 절에 긴 목록 넘기기
- `#list[]#` 로 IN 절을 만들 때 Oracle은 한 IN 절에 1000개까지만 허용한다(`ORA-01795`). 목록을 500개 정도로 잘라 여러 번 실행한다.

## 핵심 코드
### 1) 단일 값 조회의 `resultClass`, 파라미터 유무에 따른 기본값
```xml
<!-- 설명 주석은 태그 바깥에 -->
<select id="getFlag" parameterClass="Map" resultClass="String">
    SELECT flag FROM config_item WHERE item_key = #itemKey#
</select>

<insert id="insertShare" parameterClass="Map">
    INSERT INTO file_share (item_id, seq, shared_yn)
    VALUES (#itemId#, #seq#,
        <isNotEmpty property="sharedYn">#sharedYn#</isNotEmpty>
        <isEmpty property="sharedYn">'Y'</isEmpty>)
</insert>
```

### 2) CDATA 안에서는 SQL 주석만
```xml
<!-- 설명은 여기(태그 바깥)에 XML 주석으로 -->
<select id="getStatus" parameterClass="java.util.Map" resultClass="java.lang.String">
<![CDATA[
    /* CDATA 안에는 SQL 주석만 쓴다 */
    SELECT status FROM orders WHERE order_id = #orderId# AND amount < #limit#
]]>
</select>
```

### 3) Map 키를 상수로 공유하고 0건을 로그로 남기기
```java
private static final String KEY_ORDER_ID = "orderId";   // SQL의 #orderId#와 글자 단위로 같게

Map<String, Object> param = new HashMap<>();
param.put(KEY_ORDER_ID, orderId);
String status = (String) sqlMapClient.queryForObject("getStatus", param);
int updated = sqlMapClient.update("updateStatus", param);
if (updated == 0) {
    log.warn("0 rows updated: {}", param);   // 키 이름 불일치·조건 불일치 의심
}
```

### 4) SET 절 조립과 IN 절 목록
```xml
<update id="updateStatus" parameterClass="java.util.Map">
    UPDATE orders
    <dynamic prepend="SET">
        <isEqual property="mode" compareValue="1" prepend=",">status = 'A'</isEqual>
        <isEqual property="mode" compareValue="2" prepend=",">status = 'B'</isEqual>
    </dynamic>
    WHERE order_id = #orderId#
</update>

<update id="markSent" parameterClass="java.util.Map">
    UPDATE orders SET send_flag = 'Y'
     WHERE order_id IN
    <iterate property="idList" open="(" close=")" conjunction=",">
        #idList[]#
    </iterate>
</update>
```
`mode`가 1·2가 아니면 `SET`이 비어 문법 오류이므로 호출 전에 값 범위를 확인한다.

### 5) IN 절 목록을 잘라 여러 번 실행
```java
// IN 절 분할
for (int i = 0; i < ids.size(); i += 500) {
    map.put("idList", ids.subList(i, Math.min(i + 500, ids.size())));
    dao.update("markSent", map);
}
```

## 주의할 점
- 위 문제들은 전부 "예외 없이 null 또는 0건"이다. 점검할 때는 쿼리 정의에 `resultClass`가 있는지, SQL의 `#키#`와 Java의 `put` 키가 글자 단위로 같은지 기계적으로 대조한다.
- XML 파싱이 통과했다는 것과 SQL이 DB에서 실행된다는 것은 다르다. 실제 DB에서 돌려보기 전에는 "확인됨"으로 적지 않는다.

## 참고
- iBATIS 2 Data Mapper Developer Guide: Result Maps, Parameter Maps, Dynamic SQL
- Oracle: ORA-01795 maximum number of expressions in a list
