# 스프링 프레임워크 학습 (토비의 스프링 3.1)


## 1장 오브젝트와 의존관계
https://github.com/RedRabbit-88/SpringFramework/wiki/1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84#1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84


## 2장 테스트
https://github.com/RedRabbit-88/SpringFramework/wiki/2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8#2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8


## 3장 템플릿
https://github.com/RedRabbit-88/SpringFramework/wiki/3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF#3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF


## 4장 예외
https://github.com/RedRabbit-88/SpringFramework/wiki/4%EC%9E%A5-%EC%98%88%EC%99%B8#4%EC%9E%A5-%EC%98%88%EC%99%B8


## 5장 서비스 추상화

https://github.com/RedRabbit-88/SpringFramework/wiki/5%EC%9E%A5-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%83%81%ED%99%94#5%EC%9E%A5-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%83%81%ED%99%94


## 6장 AOP
https://github.com/RedRabbit-88/SpringFramework/wiki/6%EC%9E%A5-AOP#6%EC%9E%A5-aop


## 7장 스프링 핵심 기술의 응용

* 스프링의 3대 핵심 기술: **Ioc/DI, 서비스 추상화, AOP**


### 7.1 SQL과 DAO의 분리

* SQL 변경이 필요한 상황이 발생하면 SQL을 담고 있는 DAO 코드가 수정될 수 밖에 없음
<br>-> SQL을 적절히 분리해 DAO 코드와 다른 파일이나 위치에 두고 관리가 필요


### 7.1.1 XML 설정을 이용한 분리

* 가장 쉬운 방법은 SQL을 스프링 XML 설정파일로 분리하는 것
<br>-> 매번 새로운 SQL이 추가될 때마다 프로퍼티를 추가하고 DI를 위한 변수와 수정자 메서드도 만들어야 함.
```java
// 리스트 7-1 add() 메서드를 위한 SQL 필드
public class UserDaoJdbc implements UserDao {
	private String sqlAdd;

	public void setSqlAdd(String sqlAdd) {
		this.sqlAdd = sqlAdd;
	}
	...
}

// 리스트 7-3 설정파일에 넣은 SQL 문장 -> 새로운 SQL이 추가될 때마다 매번 설정파일을 변경해야 함!
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
	<property name="sqlAdd" value="insert into users(id, name, password, email
		,level, login recommend) values(?, ?, ?, ?, ?, ?, ?)" />
	...
</bean>
```

* SQL 맵 프로퍼티 방식
  * SQL을 별도의 빈이 아닌 하나의 컬렉션으로 담아두는 방식 사용
  * sqlMap 프로퍼티의 타입은 Map이기 때문에 `<map>` 태그를 사용.
  * SQL이 추가될 때 entry만 추가하면 되므로 조금 더 편리
  <br>-> SQL을 가져올 때 문자열 키값을 사용하기 때문에 오타 같은 실수도 런타임 전에 확인 불가
```java
// 리스트 7-4 맵 타입의 SQL 정보 프로퍼티
public class UserDaoJdbc implements UserDao {
	private Map<String, String> sqlMap;

	public void setSqlMap(Map<String, String> sqlMap) {
		this.sqlMap = sqlMap;
	}
	...
}

// 리스트 7-6 맵을 이용한 SQL 설정
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
	<property name="sqlMap">
		<map>
			<entry key="add" value="insert into users(id, name, password, email
				,level, login recommend) values(?, ?, ?, ?, ?, ?, ?)" />
			...
		</map>
	</property>
</bean>
```


*** 7.1.2 SQL 제공 서비스

* SQL과 DI 설정정보가 섞여있으면 보기에도 지저분하고 관리하기에도 좋지 않다.

* 스프링 설정파일로부터 생성된 오브젝트와 정보는 애플리케이션 재시작 전에는 변경이 매우 어려움.
<br>-> **DAO가 사용할 SQL을 제공하는 기능을 독립시켜야 함!**

* SQL 서비스 인터페이스
  * 가장 먼저 할 일은 SQL 서비스의 인터페이스를 설계하는 것
  * SQL에 대한 키 값을 전달하면 그에 해당하는 SQL을 돌려주게 설계
```java
// 리스트 7-7 SqlService 인터페이스
package springbook.user.sqlservice;

public interface SqlService {
	// SqlRetrievalFailureException는 런타임 예외이므로 특별히 복구해야할 일이 없으면 무시
	String getSql(String key) throws SqlRetrievalFailureException;
}

// 리스트 7-8 조회 실패 시 예외
public class SqlRetrievalFailureException extends RuntimeException {
	public SqlRetrievalFailureException(String message) {
		super(message);
	}
	
	public SqlRetrievalFailureException(String message, Throwable cause) {
		super(message, cause);
	}
}

// 리스트 7-9 SqlService 프로퍼티 추가
public class UserDaoJdbc implements UserDao {
	...
	private SqlService sqlService;
	
	public void setSqlService(SqlService sqlService) {
		this.sqlService = sqlService;
	}
}

// 리스트 7-10 sqlService를 사용하도록 수정한 메서드
public void add(User user) {
	this.jdbcTemplate.update(this.sqlService.getSql("userAdd"),
		user.getId(), user.getName(), user.getPassword(), user.getEmail(), 
		user.getLevel().intValue(), user.getLogin(), user.getRecommend());
}
```

* 스프링 설정을 사용하는 단순 SQL 서비스
  * 설정파일에 `<map>`으로 정의된 SQL을 가져와서 Map으로 저장.
  * `getSql()` 메서드를 이용해서 Map에 저장된 SQL을 리턴하도록 설계
  * UserDao를 포함한 모든 DAO는 SQL을 어디서 저장하고 가져오는지에 대해 신경 안 써도 됨.
```java
// 리스트 7-11 맵을 이용한 SqlService의 구현
package springbook.user.sqlservice;

public class SimpleSqlService implements SqlService {
	private Map<String, String> sqlMap;
	
	public void setSqlMap(Map<String, String> sqlMap) {
		this.sqlMap = sqlMap;
	}
	
	public String getSql(String key) throws SqlRetrievalFailureException {
		String sql = sqlMap.get(key); // 내부 sqlMap에서 SQL을 가져옴
		if (sql == null)
			throw new SqlRetrievalFailureException(key + "에 대한 SQL을 찾을 수 없습니다.");
		else
			return sql;
	}
}

// SimpleSqlService를 빈으로 등록하고 UserDao에 DI
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
	<property name="sqlService" ref="sqlService" />
</bean>

<bean id="sqlService" class="springbook.user.sqlservice.SimpleSqlService">
	<property name="sqlMap">
		<map>
			<entry key="userAdd" value="insert into users(id, name, password, email
				,level, login recommend) values(?, ?, ?, ?, ?, ?, ?)" />
			...
		</map>
	</property>
</bean>
```


### 7.2 인터페이스의 분리와 자기참조 빈


### 7.2.1 XML 파일 매핑

* 스프링의 XML 설정파일에서 `<bean>` 태그 안에 SQL 정보를 넣어놓고 활용하는 건 좋은 방법이 아님.
<br>-> **SQL을 저장해두는 전용 포맷을 가진 독립적인 파일을 이용하는 편이 나음!**

* JAXB (Java Architecture for XML Binding)
  * XML 문서정보를 거의 동일한 구조의 오브젝트로 직접 매핑해줌.
  * DOM은 XML 정보를 리플렉션 API를 사용해서 오브젝트에 조작하는 것처럼 간접적으로 접근해서 불편.
  * JAXB는 XML 정보를 그대로 담고 있는 오브젝트 트리 구조로 만들어주기 때문에 XML 정보를 오브젝트처럼 다룰 수 있어 편리.

* JAXB는 XML 문서의 구조를 정의한 스키마를 이용해서, 매핑할 오브젝트의 클래스까지 자동으로 만들어주는 **컴파일러**도 제공
  * 스키마 컴파일러를 통해 자동생성된 오브젝트에는 매핑정보가 애노테이션으로 담겨 있음.
  * JAXB API는 애노테이션에 담긴 정보를 이용해서 XML과 매핑된 오브젝트 트리 사이의 자동변환 작업을 수행
  <br>`XML 스키마 <-> 스키마 컴파일러 <-> XML정보를 담을 수 있는 매핑용 클래스`

* SQL 맵을 위한 스키마 작성과 컴파일
  * 스키마 파일을 생성 후 JAXB 컴파일러로 컴파일 시 자동으로 XML 문서 바인딩용 클래스가 생성됨.
  * `xjc -p springbook.user.sqlservice.jaxb sqlmap.xsd -d src`
  <br>          생성할 클래스의 패키지    변환할 스키마 파일  파일저장 위치
```java
// 리스트 7-12 SQL 맵 XML 문서
<sqlmap>
	<sql key="userAdd">insert into users(id, name, password, email, level, login recommend) values(?, ?, ?, ?, ?, ?, ?)</sql>
	...
</sqlmap>

// 리스트 7-13 SQL 맵 문서에 대한 스키마
<?xml version="1.0" encoding="UTF-8"?>
<schema xmlns="http://www.w3.org/2001/XMLSchema"
	targetNamespace="http://www.epril.com/sqlmap"
	xmlns:tns="http://www.epril.com/sqlmap" elementFormDefault="qualified">

	<element name="sqlmap">
		<complexType>
			<sequence>
				<element name"sql" maxOccurs="unbounded" type="tns:sqlType" />
			</sequence>
		</complexType>
	</element>

	<element name="sqlType"> // <sql>에 대한 정의를 시작
		<simpleContent>
			<extension base="string"> // SQL 문장을 넣을 String 타입을 정의
				<attribute name"key" use="required" type="string" /> // 검색을 위한 키값
			</sequence>
		</simpleContent>
	</element>
</schema>

// 리스트 7-14 SqlmapType 클래스
// 변환 작업에서 참고할 정보를 애노테이션으로 갖고 있음
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "", propOrder = { "sql" })
@XmlRootElement(name = "sqlmap")
public class Sqlmap {
    @XmlElement(required = true)
    protected List<SqlType> sql; // <sql> 태그의 정보를 담은 SqlType 오브젝트를 리스트로 보유

    public List<SqlType> getSql() {
        if (sql == null) {
            sql = new ArrayList<SqlType>();
        }
        return this.sql;
    }
}

// 리스트 7-15 SqlType 클래스
@XmlAccessorType(XmlAccessType.FIELD)
@XmlType(name = "sqlType", propOrder = { "value" })
public class SqlType { // <sql> 태그 한 개당 SqlType 오브젝트가 하나씩 생성됨.
    @XmlValue
    protected String value;
    @XmlAttribute(required = true)
    protected String key;

    public String getValue() {
        return value;
    }

    public void setValue(String value) {
        this.value = value;
    }

    public String getKey() {
        return key;
    }

    public void setKey(String value) {
        this.key = value;
    }
}
```

* 언마샬링
  * 언마샬링(Unmarshalling): XML 문서를 읽어서 자바의 오브젝트로 변환하는 것
  * 마샬링(Marshalling): 바인딩 오브젝트를 XML 문서로 변환하는 것
```java
// 리스트 7-16 테스트용 SQL 맵 XML 문서
<?xml version="1.0" encoding="UTF-8"?>
<sqlmap xmlns="http://www.epril.com/sqlmap" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.epril.com/sqlmap ../../../../../sqlmap.xsd ">
	<sql key="add">insert</sql>
	<sql key="get">select</sql>
	<sql key="delete">delete</sql>
</sqlmap>

// 리스트 7-17 JAXB 학습 테스트
public class JaxbTest {
	@Test
	public void readSqlmap() throws JAXBException, IOException {
		
		String contextPath = Sqlmap.class.getPackage().getName(); 
		JAXBContext context = JAXBContext.newInstance(contextPath); // 바인딩용 클래스들 위치를 가지고 JAXB 컨텍스트를 생성
		Unmarshaller unmarshaller = context.createUnmarshaller(); // 언마샬러 생성
		// 언마샬링을 하면 매핑된 오브젝트 트리의 루트인 Sqlmap을 돌려준다.
		Sqlmap sqlmap = (Sqlmap) unmarshaller.unmarshal(
				getClass().getResourceAsStream("sqlmap.xml")); // 테스트 클래스와 같은 폴더에 있는 XML 파일을 사용
		
		List<SqlType> sqlList = sqlmap.getSql();

		// 리스트에 담긴 Sql 오브젝트를 가져와 XML 문서와 같은 정보를 갖고 있는지 확인
		assertThat(sqlList.size(), is(3));
		assertThat(sqlList.get(0).getKey(), is("add"));
		assertThat(sqlList.get(0).getValue(), is("insert"));
		assertThat(sqlList.get(1).getKey(), is("get"));
		assertThat(sqlList.get(1).getValue(), is("select"));
		assertThat(sqlList.get(2).getKey(), is("delete"));
		assertThat(sqlList.get(2).getValue(), is("delete"));
	}
}
```


### 7.2.2 XML 파일을 이용하는 SQL 서비스

* SQL 맵 XML 파일
  * SQL을 DAO 로직의 일부라고 볼 수 있으므로 DAO와 같은 패키지에 두는 게 좋음
```java
<?xml version="1.0" encoding="UTF-8"?>
<sqlmap xmlns="http://www.epril.com/sqlmap" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.epril.com/sqlmap http://www.epril.com/sqlmap/sqlmap.xsd ">
	<sql key="userAdd">insert into users(id, name, password, email, level, login, recommend) values(?,?,?,?,?,?,?)</sql>
	<sql key="userGet">select * from users where id = ?</sql>
	<sql key="userGetAll">select * from users order by id</sql>
	<sql key="userDeleteAll">delete from users</sql>
	<sql key="userGetCount">select count(*) from users</sql>
	<sql key="userUpdate">update users set name = ?, password = ?, email = ?, level = ?, login = ?, recommend = ? where id = ?</sql>
</sqlmap>
```

* XML SQL 서비스
  * XML 문서에서 SQL을 가져올 때는 JAXB API를 사용
  * 언제 JAXB를 사용해 XML 문서를 가져와야 할까?
    * DAO가 SQL을 요청할 때마다 매번 XML 파일을 읽는 건 비효율적
    * XML 파일로부터 읽은 내용은 어딘가에 저장해두고 DAO에서 요청이 올 때만 사용하도록 구성
  * SQL 문장을 스프링의 빈 설정에서 완벽하게 분리하는데 성공
  * 향후 SQL 수정이 필요시 별도의 XML 문서만 제공하면 됨.
```java
// 리스트 7-19 생성자 초기화 방법을 사용하는 XmlSqlService 클래스
package springbook.user.sqlservice;
...
public class XmlSqlService implements SqlService {
	// 읽어온 SQL을 저장해둘 맵
	private Map<String, String> sqlMap = new HashMap<String, String>();
	
	// 스프링이 오브젝트를 만드는 시점에 SQL을 읽어오도록 생성자를 이용
	public XmlSqlService() {
		// JAXB API를 이용해 XML 문서를 오브젝트 트리로 읽어옴.
		String contextPath = Sqlmap.class.getPackage().getName(); 
		try {
			JAXBContext context = JAXBContext.newInstance(contextPath);
			Unmarshaller unmarshaller = context.createUnmarshaller();
			InputStream is = UserDao.class.getResourceAsStream("sqlmap.xml");
			Sqlmap sqlmap = (Sqlmap)unmarshaller.unmarshal(is);

			// 읽어온 SQL을 맵으로 저장
			for(SqlType sql : sqlmap.getSql()) {
				sqlMap.put(sql.getKey(), sql.getValue());
			}
		} catch (JAXBException e) { // 복구 불가능한 예외를 불필요한 throws를 피하도록 런타임 예외로 포장
			throw new RuntimeException(e);
		} 
	}

	public String getSql(String key) throws SqlRetrievalFailureException {
		String sql = sqlMap.get(key);
		if (sql == null)  
			throw new SqlRetrievalFailureException(key + "를 이용해서 SQL을 찾을 수 없습니다.");
		else
			return sql;
	}
}

// 리스트 7-20 sqlService 설정 변경
<bean id="sqlService" class="springbook.user.sqlservice.XmlSqlService">
</bean>
```


### 7.2.3 빈의 초기화 작업

* 몇 가지 개선 필요사항
  * 생성자에서 예외가 발생할 수도 있는 복잡한 초기화 작업을 다루는 것은 좋지 않음.
  <br>생성자 예외는 다루기 힘들고, 상속에도 불편하며 보안 문제도 있음.
  <br>-> **초기 상태를 가진 오브젝트를 만들어놓고 별도의 초기화 메서드를 사용하는 방법이 나음.**
  * 읽어들일 파일의 위치와 이름이 코드에 고정되어 있음.
  <br>-> **유연한 변경을 위해 외부에서 DI로 설정하게 변경해야 함.**
  * XmlSqlService 오브젝트에 대한 제어권이 작성한 코드에 있다면 오브젝트 생성 시점에 초기화 메서드를 호출하면 됨.
  <br>-> XmlSqlService 오브젝트는 빈이므로 제어권이 스프링에 있음!
  * 애노테이션을 이용한 빈 설정을 지원해주는 몇 가지 빈 후처리기가 존재
    * context 네임스페이스를 사용해서 `<context:annotation-config/>` 태그를 설정파일에 넣을 경우
    <br>빈 설정 기능에 사용할 수 있는 특별한 애노테이션 기능을 부여해주는 빈 후처리기들이 등록됨.
```java
// 리스트 7-21 SQL 맵 파일 이름 프로퍼티
private String sqlmapFile;

public void setSqlMapFile(String sqlmapFile) {
	this.sqlmapFile = sqlmapFile;
}

// 리스트 7-22 생성자 대신 사용할 초기화 메서드
public void loadSql() {
	String contextPath = Sqlmap.class.getPackage().getName();
	try {
		...
		// 프로퍼티로 설정을 통해 제공받은 파일 이름을 사용
		InputStream is = UserDao.class.getResourceAsStream(this.sqlmapFile);
		...
	}
}

// 리스트 7-23 XmlSqlService 오브젝트의 초기화 방법
// XmlSqlService 오브젝트의 제어권이 스프링에 있어서 사용 불가!
XmlSqlService sqlProvider = new XmlSqlService();
sqlProvider.setSqlmapFile("sqlmap.xml");
sqlProvider.loadSql(); // 초기화

// 리스트 7-24 context 네임스페이스 선언과 annotation-config 태그 설정
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xmlns:context="http://www.springframework.org/schema/context"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
						http://www.springframework.org/schema/aop 
						http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
						http://www.springframework.org/schema/context 
						http://www.springframework.org/schema/context/spring-context-3.0.xsd
						http://www.springframework.org/schema/tx 
						http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">
	// @Transactional이 붙은 타입와 메서드에 트랜잭션 부가기능을 담은 프록시를 추가하도록 만들어주는 후처리기 등록
	<tx:annotation-driven />
	
	// 코드의 애노테이션을 이용해서 부가적인 빈 설정 또는 초기화 작업을 해주는 후처리기 등록
	<context:annotation-config />
	...
</beans>
```

* `@PostConstruct`
  * 빈 오브젝트의 초기화 메서드를 지정하는데 사용
  * `@PostConstruct` 애노테이션이 붙은 메서드가 수행되는 방식
    1. 클래스로 등록된 빈의 오브젝트를 생성
    2. DI 작업 수행
    3. `@PostConstruct`가 붙은 메서드를 자동으로 실행
  * `@PostConstruct` 애노테이션은 빈 오브젝트가 생성되고 의존 오브젝트와 설정 값을 넣어주는 DI 작업까지 마친 후에 호출됨.
  <br>-> **`@PostConstruct`를 단 메서드의 코드는 모든 프로퍼티의 값이 준비됐다고 가정하고 작성하면 됨!**
  * 스프링 컨테이너의 초기 작업 순서
    1. XML 빈 설정을 읽는다.
    <br>`applicationContext.xml`
    2. 빈의 오브젝트를 생성한다.
    <br>`<bean id=".." class="ClassName" >`
    3. 프로퍼티에 의존 오브젝트 또는 값을 주입한다.
    ```java
    <property name=".." value="xyz" />
    <property name=".." ref="beanId" />
    ```
    4. 빈이나 태그로 등록된 후처리기를 동작시킨다. -> 코드에 달린 애노테이션에 대한 부가작업 진행
    ```java
    @PostConstruct
    public void init() { ... }
    ```
    
```java
// 리스트 7-25 @PostConstruct 초기화 메서드
public class XmlSqlService implements SqlService {
	...
	@PostConstruct // loadSql() 메서드를 빈의 초기화 메서드로 지정
	public void loadSql() { ... }
}

// 리스트 7-26 sqlmapFile 프로퍼티 추가
<bean id="sqlService" class="springbook.user.sqlservice.XmlSqlService">
	<property name="sqlmapFile" value="sqlmap.xml" />
</bean>
```


### 7.2.4 변화를 위한 준비: 인터페이스 분리

* XmlSqlService는 특정 포맷의 XML에서 SQL 데이터를 가져오고 이를 HashMap 타입의 맵 오브젝트에 저장
  * **SQL을 가져오는 방법에 있어서 특정 기술에 고정되어 있음!**
  * SQL을 가져오는 것과 보관해두고 사용하는 것은 독자적인 이유로 변경 가능한 독립적인 전략

* 책임에 따른 인터페이스 정의
  * XmlSqlService에서 독립적으로 변경 가능한 책임
    * SQL 정보를 외부의 리소스로부터 읽어오는 기능
    * 읽어온 SQL을 보관해두고 있다가 필요할 때 제공해주는 기능
    * 서비스를 위해서 한 번 가져온 SQL을 필요에 따라 수정할 수 있게 하는 기능
  * 기본적으로 SqlService를 구현해서 DAO에 서비스를 제공해주는 오브젝트가
  <br>다른 책임을 가진 오브젝트와 협력해서 동작하도록 만들어야 함.
  <br>-> 변경 가능한 기능은 전략 패턴을 적용해 별도의 오브젝트로 분리
  * SqlService의 구현 클래스가 변경 가능한 책임을 가진 `SqlReader`와 `SqlRegistry` 2가지 타입의 오브젝트를 사용하게 만든다.
    * SqlReader: SqlService에서 받은 읽기 요청을 SQL 리소스를 이용해서 처리하는 오브젝트
    * SqlRegistry: SqlService에서 받은 SQL 등록/조회 요청을 SqlReader를 이용해서 처리하는 오브젝트
  * SqlReader가 읽어오는 SQL 정보는 다시 SqlRegistry에 전달해서 등록되게 해야 함.
  <br>-> **SqlReader에게 SqlRegistry 전략을 제공해주면서 이를 이용해 SQL 정보를 SqlRegistry에 저장하라고 요청**
  ```java
  // 리스트 7-27 SqlService 구현 클래스 코드
  Map<String, String> sqls = sqlReader.readSql(); // Map이라는 구체적인 전송 타입을 강제하게 됨.
  sqlRegistry.addSqls(sqls);
  
  // 리스트 7-28 변경된 SqlService 코드
  // 불필요하게 SqlService 코드를 통해 특정 포맷으로 변환한 SQL 정보를 주고받을 필요 없이
  // SqlReader가 직접 SqlRegistry에 SQL 정보를 등록
  sqlReader.readSql(sqlRegistry); // SQL을 저장할 대상인 sqlRegistry 오브젝트를 전달
  
  // 리스트 7-29 등록 기능을 제공하는 SqlResgitry 메서드
  interface SqlRegistry {
  	// SqlReader는 읽어들인 SQL을 이 메서드를 이용해서 레지스트리에 저장
	void registerSql(String key, String sql);
	...
  }
  ```
  * SqlReader는 내부에 갖고 있는 SQL 정보를 형식을 갖춰서 돌려주는 대신, SqlRegistry에 필요에 따라 등록을 요청할 때만 활용
  * SqlRegistry가 일종의 콜백 오브젝트처럼 사용됨.
  * SqlReader 입장에서는 SqlRegistry 인터페이스를 구현한 오브젝트를 런타임 시에 메서드 파라미터로 제공받아 사용하는 구조
  <br>-> 일종의 코드에 의한 수동 DI
  * SqlRegistry는 SqlService에게 등록된 SQL을 검색해서 돌려주는 기능을 제공
  <br>-> SqlRegistry는 SqlService의 의존 오브젝트
  ```
  SqlService ---(SQL을 가져오도록 요청)---> SqlReader ---(SQL 등록)---> SqlRegistry
             ---(SQL 검색)---> SqlRegistry
  ```

```java
// 리스트 7-30 SqlRegistry 인터페이스
package springbook.user.sqlservice;
...
public interface SqlRegistry {
	void registerSql(String key, String sql); // SQL을 키와 함께 등록
	String findSql(String key) throws SqlNotFoundException; // 키로 SQL을 검색. 실패 시 예외를 던짐
}

// 리스트 7-31 SqlReader 인터페이스
public interface SqlReader {
	// SQL을 외부에서 가져와 SqlRegistry에 등록
	// 다양한 예외가 발생할 수 있지만 대부분 복구 불가능한 예외이므로 예외 선언 X
	void read(SqlRegistry sqlRegistry);
}
```
