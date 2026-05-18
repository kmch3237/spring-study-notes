# SP16 - Spring MVC 게시판(BBS) 실전 정리

> 프로젝트명: `sp16_board`  
> 핵심 주제: 3계층 구조 + MyBatis + DBCP2 커넥션풀 + 페이징 + 검색 + 이전/다음글

---

## 1. 전체 흐름 요약

```
[브라우저 요청]
    ↓
[web.xml]
  ├── ContextLoaderListener → root-context.xml 로딩 (DB, MyBatis, 트랜잭션 등)
  └── DispatcherServlet   → servlet-context.xml 로딩 (컨트롤러 스캔, 뷰리졸버)
    ↓
[DispatcherServlet]
    ↓ URL 매핑
[@Controller]
    ↓ 비즈니스 로직 호출
[BoardService (인터페이스)]
    ↓ DB 작업 위임
[BoardMapper (인터페이스, @Mapper)]
    ↓ SQL 실행
[bbsMapper.xml]
    ↓
[Oracle DB - BBS 테이블]
```

---

## 2. 설정 파일 구조

### 2-1. web.xml - 웹 애플리케이션 진입점

```xml
<!-- ① ContextLoaderListener : 서버 시작 시 root-context.xml 읽어서 공용 Bean 등록 -->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

<!-- ② DispatcherServlet : 모든 요청(/)을 받아서 컨트롤러에 전달 -->
<servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>  <!-- 서버 시작 시 즉시 로드 (첫 요청 지연 제거) -->
</servlet>

<!-- ③ 한글 인코딩 필터 : 모든 요청/응답을 UTF-8로 강제 변환 -->
<filter>
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
    <init-param><param-name>encoding</param-name><param-value>UTF-8</param-value></init-param>
    <init-param><param-name>forceEncoding</param-name><param-value>true</param-value></init-param>
</filter>
```

**포인트**
- `ContextLoaderListener` : Service, DAO, DB 같은 공용 Bean 담당 (= Boot Context)
- `DispatcherServlet` : Controller, ViewResolver 같은 웹 Bean 담당 (= Servlet Context)
- `load-on-startup` 숫자가 작을수록 먼저 초기화 (1 = 최우선)

---

### 2-2. root-context.xml

```xml
<!-- mybatis-context.xml을 임포트해서 DB 관련 설정을 위임 -->
<import resource="classpath://mybatis/mybatis-context.xml"/>
```

---

### 2-3. mybatis-context.xml - DB 핵심 설정

**① jdbc.properties에서 DB 접속 정보 읽기**
```xml
<context:property-placeholder location="classpath:/mybatis/jdbc.properties"/>
```

**② DBCP2 커넥션 풀 (BasicDataSource)**
```xml
<bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">
    <property name="driverClassName" value="${jdbc.driverClass}"/>
    <property name="url"            value="${jdbc.uri}"/>
    <property name="username"       value="${jdbc.username}"/>
    <property name="password"       value="${jdbc.password}"/>
    <property name="initialSize"    value="20"/>  <!-- 서버 시작 시 미리 연결 20개 생성 -->
    <property name="maxTotal"       value="20"/>  <!-- 동시에 사용 가능한 최대 커넥션 수 -->
    <property name="maxIdle"        value="20"/>  <!-- 반납 시 풀에 유지할 최대 커넥션 수 -->
    <property name="minIdle"        value="20"/>  <!-- 항상 유지할 최소 커넥션 수 -->
    <property name="maxWaitMillis"  value="10000"/> <!-- 풀이 꽉 찼을 때 대기 시간 (10초) -->
</bean>
```

> **커넥션 풀이란?**  
> 매번 DB 연결을 새로 만들면 느리고 자원 낭비. 미리 연결 N개를 만들어 재사용하는 방식.  
> 요청 올 때 빌리고(getConnection), 끝나면 반납(close → 실제로는 반납).

**③ SqlSessionFactory**
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
    <property name="dataSource"       ref="dataSource"/>
    <property name="configLocation"   value="classpath:/mybatis/mybatis-config.xml"/>
    <property name="mapperLocations"  value="classpath:/mybatis/mapper/**/*.xml"/>
</bean>
```

**④ SqlSessionTemplate**
```xml
<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
    <constructor-arg index="0" ref="sqlSessionFactory"/>
</bean>
```
- MyBatis SQL 실행, 예외 처리, 세션 관리 담당

**⑤ 트랜잭션**
```xml
<bean id="transactionManager"
      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>

<bean id="transactionTemplate"
      class="org.springframework.transaction.support.TransactionTemplate">
    <property name="transactionManager" ref="transactionManager"/>
</bean>
```

**⑥ Mapper 자동 스캔**
```xml
<!-- com.doit.app.mapper 패키지 아래 모든 인터페이스를 Mapper로 자동 등록 -->
<mybatis-spring:scan base-package="com.doit.app.mapper"/>
```

---

### 2-4. jdbc.properties - DB 접속 정보

```properties
jdbc.driverClass=oracle.jdbc.driver.OracleDriver
jdbc.uri=jdbc:oracle:thin:@//127.0.0.1:1521/xe
jdbc.username=scott
jdbc.password=tiger
```

---

### 2-5. mybatis-config.xml - MyBatis 동작 설정

```xml
<settings>
    <!-- 2차 캐시 OFF : 데이터 정합성 문제 방지 -->
    <setting name="cacheEnabled"         value="false"/>
    
    <!-- DB 자동생성 키(시퀀스 등)를 Java 객체에 채워줌 -->
    <setting name="useGeneratedKeys"     value="true"/>
    
    <!-- PreparedStatement 재사용 → 성능 향상 -->
    <setting name="defaultExecutorType"  value="REUSE"/>
    
    <!-- 콘솔에 SQL 출력 (학습/디버그용) -->
    <setting name="logImpl"              value="STDOUT_LOGGING"/>
</settings>

<!-- 긴 클래스명을 짧게 쓰기 위한 별칭 등록 -->
<typeAliases>
    <typeAlias alias="hashMap" type="java.util.HashMap"/>
    <typeAlias alias="map"     type="java.util.map"/>
</typeAliases>
```

---

### 2-6. servlet-context.xml - 웹 MVC 설정

```xml
<!-- @Controller 어노테이션 기반 요청 처리 활성화 -->
<annotation-driven/>
<!-- HandlerMapping : URL → 컨트롤러 연결 -->
<!-- HandlerAdapter : 컨트롤러 실행 담당 -->

<!-- 정적 자원 (CSS, JS, 이미지 등) 처리 -->
<resources mapping="/resources/**" location="/resources/"/>

<!-- ViewResolver : 컨트롤러가 반환한 뷰 이름 → JSP 경로 변환 -->
<beans:bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <beans:property name="prefix" value="/WEB-INF/views/"/>
    <beans:property name="suffix" value=".jsp"/>
</beans:bean>
<!-- 예) "home" → /WEB-INF/views/home.jsp -->

<!-- 컨트롤러 자동 스캔 -->
<context:component-scan base-package="com.doit.app"/>
```

---

## 3. 물리적 패키지 구조

```
com.doit.app
├── domain/           → DTO (데이터 객체)
│   └── Board.java
├── service/          → Service 인터페이스 (비즈니스 로직 명세)
│   └── BoardService.java
├── mapper/           → @Mapper 인터페이스 (SQL 호출 명세)
│   └── BoardMapper.java
└── HomeController.java

resources/mybatis/
├── jdbc.properties   → DB 접속 정보
├── mybatis-config.xml
├── mybatis-context.xml
└── mapper/
    └── bbsMapper.xml → 실제 SQL 쿼리
```

---

## 4. 계층별 클래스 상세

### 4-1. Board.java (Domain / DTO)

```java
// Oracle BBS 테이블의 컬럼과 1:1 매핑되는 Java 객체
public class Board {
    private long   num;      // NUMBER → long 타입 사용 (큰 숫자 대비)
    private String name;     // 작성자명
    private String pwd;      // 비밀번호
    private String subject;  // 제목
    private String content;  // 내용
    private String ipAddr;   // 접속 IP (ipAddr ↔ DB컬럼 IPADDR : 캐멀케이스 주의)
    private String regDate;  // 등록일 (reg_date → regDate : 캐멀케이스 매핑)
    private int    hitCount; // 조회수

    // 모든 필드에 getter/setter 구성
}
```

> **주의:** DB 컬럼 `REG_DATE`는 Java에서 `regDate`로 쓴다 (MyBatis 자동 매핑).  
> MyBatis 설정에서 `mapUnderscoreToCamelCase`를 켜거나 ResultMap으로 명시적 매핑.

---

### 4-2. BoardService.java (Service 인터페이스)

```java
public interface BoardService {
    void        insertBoard(Board dto)           throws Exception; // 게시물 작성
    void        updateBoard(Board dto)           throws Exception; // 게시물 수정
    void        deleteBoard(long num)            throws Exception; // 게시물 삭제
    void        updateHit(long num)              throws Exception; // 조회수 증가
    int         dataCount(Map<String, Object> map);               // 게시물 수 (검색 포함)
    List<Board> listBoard(Map<String, Object> map);               // 목록 조회 (페이징+검색)
    Board       findById(long num);                               // 상세 조회
    Board       findByPrev(Map<String, Object> map);              // 이전글
    Board       findByNext(Map<String, Object> map);              // 다음글
}
```

> `Map<String, Object>` 사용 이유:  
> 페이징 파라미터(offset, size), 검색 파라미터(searchType, keyword) 등  
> 여러 값을 한 번에 묶어서 전달하기 위해.

---

### 4-3. BoardMapper.java (@Mapper 인터페이스)

```java
@Mapper  // MyBatis가 구현 클래스를 자동 생성해서 Spring Bean으로 등록
public interface BoardMapper {
    void        insertBoard(Board dto)           throws Exception;
    void        updateBoard(Board dto)           throws Exception;
    void        deleteBoard(long num)            throws Exception;
    void        updateHit(long num)              throws Exception;
    int         dataCount(Map<String, Object> map);
    List<Board> listBoard(Map<String, Object> map);
    Board       findById(long num);
    Board       findByPrev(Map<String, Object> map);
    Board       findByNext(Map<String, Object> map);
}
```

**@Mapper 핵심 규칙**

| 규칙 | 내용 |
|------|------|
| 메서드 이름 | bbsMapper.xml의 SQL `id`와 반드시 일치해야 함 |
| namespace | 인터페이스의 풀 패키지 경로와 일치 (`com.doit.app.mapper.BoardMapper`) |
| 구현 클래스 | MyBatis가 자동 생성 → 개발자가 implement 불필요 |
| 대체 어노테이션 | `@Repository`로 대체 가능 (DataAccessException 변환 제공) |

**흐름 옵션**
```
Controller → Service → Mapper              (가장 일반적)
Controller → Service → Repository → Mapper (DAO 클래스 추가 시)
```

---

### 4-4. bbsMapper.xml (SQL Mapper)

```xml
<mapper namespace="com.doit.app.mapper.BoardMapper">
    <!-- namespace = BoardMapper 인터페이스의 풀 패키지 경로 -->
    
    <!-- 메서드 id = BoardMapper 인터페이스의 메서드 이름과 일치 -->
</mapper>
```

---

## 5. Oracle DB 설계 (BBS 테이블)

```sql
-- 테이블 구조
DESC BBS;
/*
NUM      NOT NULL NUMBER         -- 게시물 번호 (시퀀스로 자동 증가)
NAME     NOT NULL VARCHAR2(30)   -- 작성자명
PWD      NOT NULL VARCHAR2(50)   -- 비밀번호
SUBJECT  NOT NULL VARCHAR2(300)  -- 제목
CONTENT  NOT NULL VARCHAR2(4000) -- 내용
IPADDR   NOT NULL VARCHAR2(50)   -- 작성자 IP
HITCOUNT          NUMBER         -- 조회수
REG_DATE          DATE           -- 작성일
*/

-- 시퀀스 생성 (자동 번호 부여)
CREATE SEQUENCE BBS_SEQ
    INCREMENT BY 1
    START WITH 1
    NOMAXVALUE
    NOCYCLE
    NOCACHE;

-- 게시물 삽입 예시
INSERT INTO BBS(NUM, NAME, PWD, SUBJECT, CONTENT, IPADDR, HITCOUNT, REG_DATE)
VALUES(BBS_SEQ.NEXTVAL, '홍길동', 'doit5601', '제목', '내용', '192.168.0.1', 0, SYSDATE);
```

---

## 6. 페이징 처리 (OFFSET / FETCH)

Oracle 12c 이상에서 지원하는 페이징 문법:

```sql
SELECT NUM, NAME, SUBJECT, HITCOUNT, TO_CHAR(REG_DATE, 'YYYY-MM-DD') AS REG_DATE
FROM BBS
ORDER BY NUM DESC
OFFSET 20 ROWS      -- 상위 20개 건너뜀 (3페이지 시작 = (3-1) * 10 = 20)
FETCH FIRST 10 ROWS ONLY;  -- 그 다음 10개만 가져옴
```

**파라미터 계산 공식**
```
size   = 한 페이지에 보여줄 게시물 수 (보통 10 or 20)
offset = (현재페이지 - 1) * size

예) 3페이지, 한 페이지 10개 → offset = (3-1)*10 = 20
```

> MyBatis에서 `#{offset}`, `#{size}` 파라미터로 전달.  
> Service/DAO 계층에서 offset 값을 계산해서 Map에 담아 Mapper로 전달.

---

## 7. 검색 처리 (INSTR 함수 활용)

```sql
-- 제목+내용 검색
WHERE (INSTR(SUBJECT, '키워드') > 0 OR INSTR(CONTENT, '키워드') > 0)

-- 작성일 검색
WHERE (TO_CHAR(REG_DATE, 'YYYY-MM-DD') = '키워드'
    OR TO_CHAR(REG_DATE, 'YYYYMMDD') = '키워드')

-- 작성자 검색
WHERE (INSTR(NAME, '키워드') > 0)
```

> `INSTR(컬럼, '키워드')` : 컬럼에서 키워드의 위치 반환. 0이면 없는 것, 1 이상이면 있는 것.  
> LIKE 대신 INSTR을 쓰는 이유: 와일드카드(%) 없이도 부분 일치 검색 가능하고 직관적.

---

## 8. 이전글 / 다음글 조회

```sql
-- 다음글 (현재 NUM보다 큰 것 중 가장 작은 것)
SELECT NUM, SUBJECT FROM BBS
WHERE NUM > #{currentNum}
ORDER BY NUM ASC
FETCH FIRST 1 ROWS ONLY;

-- 이전글 (현재 NUM보다 작은 것 중 가장 큰 것)
SELECT NUM, SUBJECT FROM BBS
WHERE NUM < #{currentNum}
ORDER BY NUM DESC
FETCH FIRST 1 ROWS ONLY;
```

---

## 9. 테스트 코드 이해

### 9-1. DataSourceTest.java - DB 연결 테스트

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/*.xml"})
public class DataSourceTest {

    @Inject
    private DataSource ds;  // 스프링이 DataSource Bean을 주입

    @Test
    public void testConnection() throws Exception {
        // try-with-resources : Java 7+ 문법
        // AutoCloseable 구현 객체는 블록 종료 시 자동으로 close() 호출
        try (Connection conn = ds.getConnection()) {
            System.out.println("DB 연결 성공: " + conn);
        }
    }
}
```

**경로 패턴 설명: `file:src/main/webapp/WEB-INF/spring/**/*.xml`**

| 패턴 | 의미 |
|------|------|
| `*` (싱글) | 현재 폴더 안의 파일/폴더 (1단계만) |
| `**` (더블) | 현재 폴더 포함 모든 하위 폴더 (깊이 제한 없음) |
| `**/*.xml` | 몇 단계 아래에 있든 `.xml` 파일 전부 |

> 이 방식을 **Ant-style Path Pattern** 이라고 부른다.  
> 스프링뿐 아니라 다양한 Java 라이브러리에서 공통적으로 사용.

---

### 9-2. MyBatisTest.java - SqlSession 연동 테스트

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"file:src/main/webapp/WEB-INF/spring/**/*.xml"})
public class MyBatisTest {

    @Inject
    private SqlSessionFactory sqlFactory;

    @Test
    public void testFactory() {
        System.out.println(sqlFactory);  // null이면 설정 오류
    }

    @Test
    public void testSession() throws Exception {
        try (SqlSession session = sqlFactory.openSession()) {
            System.out.println("MyBatis 세션 연동 성공: " + session);
        }
    }
}
```

---

## 10. pom.xml 주요 개념

> pom = **Project Object Model**  
> Maven 프로젝트의 설계도. 프로젝트 구조, 의존 라이브러리, 빌드 설정을 담당.

**라이브러리 추가 방법**
1. [mvnrepository.com](http://mvnrepository.com) 에서 필요한 라이브러리 검색
2. 원하는 버전 선택 → XML 코드 복사
3. `pom.xml`의 `<dependencies>` 태그 안에 붙여넣기
4. Maven이 자동으로 `.jar` 다운로드 & 관리

---

## 11. property name 작성 규칙

```xml
<!-- Bean 설정 시 property name은 변수명이 아니라 setter 메서드 기반 -->
<property name="dataSource" ref="dataSource"/>
<!-- setDataSource(...) 메서드 → set 제거 + 첫 글자 소문자 = dataSource -->

<property name="sqlSessionFactory" ref="sqlSessionFactory"/>
<!-- setSqlSessionFactory(...) → sqlSessionFactory -->
```

---

## 12. 핵심 개념 요약표

| 개념 | 설명 |
|------|------|
| `pom.xml` | Maven 프로젝트 설계도. 의존성/빌드 관리 |
| `ContextLoaderListener` | 서버 시작 시 공용 Bean(DB, Service 등) 로딩 |
| `DispatcherServlet` | 모든 HTTP 요청의 진입점. 컨트롤러에 위임 |
| `CharacterEncodingFilter` | 한글 깨짐 방지. 모든 요청/응답 UTF-8 처리 |
| `DBCP2 BasicDataSource` | DB 커넥션 풀. 미리 연결 생성 후 재사용 |
| `SqlSessionFactory` | MyBatis 세션 생성 공장 |
| `SqlSessionTemplate` | SQL 실행 + 예외처리 담당 |
| `@Mapper` | 인터페이스를 Mapper로 등록. 구현 클래스 자동 생성 |
| `BoardService` | 비즈니스 로직 인터페이스 |
| `Board (DTO)` | DB 테이블과 매핑되는 데이터 객체 |
| `OFFSET/FETCH` | Oracle 페이징 문법 |
| `INSTR` | Oracle 부분 문자열 검색 함수 |
| `BBS_SEQ` | 게시물 번호 자동 생성 시퀀스 |
| `Ant-style Path` | `**` 와일드카드로 하위 폴더 일괄 지정 |
| `try-with-resources` | AutoCloseable 객체를 자동 close (Java 7+) |
