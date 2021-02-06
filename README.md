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


### 7.1.2 SQL 제공 서비스

* SQL과 DI 설정정보가 섞여있으면 보기에도 지저분하고 관리하기에도 좋지 않다.

* 스프링 설정파일로부터 생성된 오브젝트와 정보는 애플리케이션 재시작 전에는 변경이 매우 어려움.
<br>-> **DAO가 사용할 SQL을 제공하는 기능을 독립시켜야 함!**

* SQL 서비스 인터페이스
  * 가장 먼저 할 일은 SQL 서비스의 인터페이스를 설계하는 것
  * SQL에 대한 키 값을 전달하면 그에 해당하는 SQL을 돌려주도록 설계
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
    * **SqlReader**
    <br>SqlService에서 받은 읽기 요청을 SQL 리소스를 이용해서 처리하는 오브젝트
    * **SqlRegistry**
    <br>SqlService에서 받은 SQL 등록/조회 요청을 SqlReader를 이용해서 처리하는 오브젝트
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


### 7.2.5 자기참조 빈으로 시작하기

* 다중 인터페이스 구현과 간접 참조
  * 클래스의 코드는 인터페이스에 대해서만 알고 있고, 인터페이스를 통해서만 의존 오브젝트에 접근
  * XmlSqlService를 3개의 인터페이스를 통해 구현하도록 변경
    * SqlReader, SqlService, SqlRegistry 3개의 인터페이스를 구현
    * XmlSqlService 내부에 SqlReader와 SqlRegistry를 넣고 이를 이용해서 접근
```java
// 리스트 7-32 SqlService의 DI 코드
public class XmlSqlService implements SqlService {
	// 의존 오브젝트를 DI 받을 수 있도록 인터페페이스 타입의 프로퍼티로 선언
	private SqlReader sqlReader;
	private SqlRegistry sqlRegistry;
	
	public void setSqlReader(SqlReader sqlReader) {
		this.sqlReader = sqlReader;
	}
	
	public void setSqlRegistry(SqlRegistry sqlRegistry) {
		this.sqlRegistry = sqlRegistry;
	}
}

// 리스트 7-33 SqlRegistry의 구현 부분
public class XmlSqlService implements SqlService, SqlRegistry {
	...
	// sqlMap은 SqlRegistry 구현의 일부가 되며 외부에서 직접 접근 불가
	private Map<String, String> sqlMap = new HashMap<>();
	
	public String findSql(String key) throws SqlNotFoundException {
		String sql = sqlMap.get(key);
		if (sql == null) throw new SqlNotFoundException(key + "에 대한 SQL을 찾을 수 없습니다.");
		else return sql;
	}
	
	// HashMap이라는 저장소를 사용하는 구체적인 구현방법에서 독립될 수 있도록 인터페이스 메서드로 접근
	public void registerSql(String key, String sql) {
		sqlMap.put(key, sql);
	}
	...
}

// 리스트 7-34 SqlReader의 구현 부분
public class XmlSqlService implements SqlService, SqlRegistry, SqlReader {
	...
	// sqlmapFile은 SqlReader 구현의 일부
	// SqlReader 구현 메서드를 통해서만 접근하도록 설정
	private String sqlmapFile;
	
	public void setSqlmapFile(String sqlmapFile) {
		this.sqlmapFile = sqlmapFile;
	}
	
	// loadSql()에 있던 코드를 SqlReader로 가져옴
	// 초기화를 위해 무엇을 할 것인가와 SQL을 어떻게 읽는지를 분리
	public void read(SqlRegistry sqlRegistry) {
		String contextPath = Sqlmap.class.getPackage().getName();
		try {
			JAXBContext context = JAXBContext.newInstance(contextPath);
			Unmarshaller unmarshaller = context.createUnmarshaller();
			InputStream is = UserDao.class.getResourceAsStream(sqlmapFile);
			Sqlmap sqlmap = (Sqlmap)unmarshaller.unmarshal(is);
			for (SqlType sql : sqlmap.getSql()) {
				// SQL 저장 로직 구현에 독립적인 인터페이스 메서드를 통해 읽어들인 SQL과 키를 전달
				sqlRegistry.registerSql(sql.getKey(), sql.getValue());
			}
		} catch (JAXBException e) {
			throw new RuntimeException(e);
		}
	}
	...
}

// 리스트 7-35 SqlService 인터페이스 구현 부분
public class XmlSqlService implements SqlService, SqlRegistry, SqlReader {
	...
	// loadSql() 초기화 메서드
	// sqlReader에게 sqlRegistry를 전달하면서 SQL을 읽어서 저장해두도록 
	@PostConstruct
	public void loadSql() {
		this.sqlReader.read(this.sqlRegistry);
	}
	
	public String getSql(String key) throws SqlRetrievalFailureException {
		try {
			return this.sqlRegistry.findSql(key);
		} catch(SqlNotFoundException e) {
			throw new SqlRetrievalFailureException(e);
		}
	}
}
```

* 자기참조 빈 설정
  * SqlService의 메서드에서 SQL을 읽을 때는 SqlReader 인터페이스를 통해서,
  <br>SQL을 찾을 때는 SqlRegistry 인터페이스를 통해 간접적으로 접근
  * 책임과 관심사가 복잡하게 얽혀 있어서 확장이 힘들고 변경에 취약한 구조의 클래스를
  <br>유연한 구조로 만들려고 할 때 처음 시도해볼 수 있는 방법
```java
<bean id="sqlService" class="springbook.user.sqlservice.XmlSqlService">
	<property name="sqlReader" ref="sqlService" /> // 프로퍼티는 자기 자신을 참조 가능
	<property name="sqlRegistry" ref="sqlService" />
	<property name="sqlmapFile" value="sqlmap.xml" />
</bean>
```


### 7.2.6 디폴트 의존관계

* 확장 가능한 기반 클래스
  * SqlRegistry와 SqlReader를 이용하는 가장 간단한 SqlService 구현 클래스 생성
  * XmlSqlService 코드에서 의존 인터페이스와 구현코드를 제거
  * BaseSqlService를 sqlService 빈으로 등록하고 SqlReader와 SqlRegistry를 구현한 클래스 역시 빈으로 등록해서 DI
  * DI를 적용했으니 언제든지 BaseSqlService의 코드에는 영향을 주지 않은 채 SqlReader와 SqlRegistry의 구현 클래스는 자유롭게 변경해서 기능 확장 가능
```java
// 리스트 7-37 SqlReader와 SqlRegistry를 사용하는 SqlService 구현 클래스
public class BaseSqlService implements SqlService {
	// BaseSqlService를 상속받아서 사용하는 서브클래스에서 접근 가능하도록 protected로 선언
	protected SqlReader sqlReader;
	protected SqlRegistry sqlRegistry;
		
	public void setSqlReader(SqlReader sqlReader) {
		this.sqlReader = sqlReader;
	}

	public void setSqlRegistry(SqlRegistry sqlRegistry) {
		this.sqlRegistry = sqlRegistry;
	}

	@PostConstruct
	public void loadSql() {
		this.sqlReader.read(this.sqlRegistry);
	}

	public String getSql(String key) throws SqlRetrievalFailureException {
		try {
			return this.sqlRegistry.findSql(key);
		} catch(SqlNotFoundException e) {
			throw new SqlRetrievalFailureException(e);
		}
	}
}

// 리스트 7-38 HashMap을 이용하는 SqlRegistry 클래스
public class HashMapSqlRegistry implements SqlRegistry {
	private Map<String, String> sqlMap = new HashMap<String, String>();

	public String findSql(String key) throws SqlNotFoundException {
		String sql = sqlMap.get(key);
		if (sql == null)  throw new SqlRetrievalFailureException(key + "을 이용해서 SQL을 찾을 수 없습니다.");
		else return sql;
	}

	public void registerSql(String key, String sql) { sqlMap.put(key, sql);	}
}

// 리스트 7-39 JAXB를 사용하는 SqlReader 클래스
public class JaxbXmlSqlReader implements SqlReader {
	private String sqlmapFile; // sqlmapFile은 SqlReader의 특정 구현 방법에 종속되는 프로퍼티로 설정

	public void setSqlmapFile(String sqlmapFile) { this.sqlmapFile = sqlmapFile; }

	public void read(SqlRegistry sqlRegistry) {
		...
	}
}

// 리스트 7-40 SqlReader와 SqlRegistry의 독립적인 빈 설정
<!-- sql service -->
<bean id="sqlService" class="springbook.user.sqlservice.BaseSqlService">
	// 독립시킨 reader, registry을 참조하도록 ref 수정
	<property name="sqlReader" ref="sqlReader" />
	<property name="sqlRegistry" ref="sqlRegistry" />
</bean>

<bean id="sqlReader" class="springbook.user.sqlservice.JaxbXmlSqlReader">
	<property name="sqlmapFile" value="sqlmap.xml" />
</bean>

<bean id="sqlRegistry" class="springbook.user.sqlservice.HashMapSqlRegistry">
</bean>
```

* 디폴트 의존관계를 갖는 빈 만들기
  * 디폴트 의존관계
  <br>외부에서 DI 받지 않는 경우 기본적으로 자동 적용되는 의존관계
  ```java
  // 리스트 7-41 생성자를 통한 디폴트 의존관계 설정
  public class DefaultSqlService extends BaseSqlService {
 	public DefaultSqlService() {
		// 생성자에서 디폴트 의존 오브젝트를 직접 만들어서 스스로 DI
		setSqlReader(new JaxbXmlSqlReader());
		setSqlRegistry(new HashMapSqlRegistry());
	}
  }
  
  // 리스트 7-42 디폴트 의존관계 빈의 설정
  <bean id="sqlService" class="springbook.user.sqlservice.DefaultSqlService" />
  ```
  * 위의 설정으로 테스트를 돌려보면 실패!
    * DefaultSqlService 내부에서 생성하는 JaxbXmlSqlReader의 sqlmapFile 프로퍼티가 비어있기 때문
    * 디폴트 의존 오브젝트로 직접 넣어줄 때는 프로퍼티를 외부에서 직접 지정할 수 없다.
  * sqlmapFile을 DefaultSqlService의 프로퍼티로 지정?
    * JaxbXmlSqlReader는 디폴트 의존 오브젝트에 불과해서 적합하지 않음.
    * 반드시 필요하지도 않은 sqlmapFile을 프로퍼티로 등록하는 건 적합하지 않음.
  * 관례적으로 사용할 만한 이름을 정해서 디폴트로 지정!
  ```java
  // 리스트 7-43 디폴트 값을 갖는 JaxbXmlSqlReader
  public class JaxbXmlSqlReader implements SqlReader {
	private final String DEFAULT_SQLMAP_FILE = "sqlmap.xml";
	private String sqlmapFile = DEFAULT_SQLMAP_FILE;
	
	// wqlmapFile 프로퍼티를 지정하면 지정된 파일이 사용되고, 아니라면 디폴트로 넣은 파일이 사용됨.
	public void setSqlmapFile(String sqlmapFile) { this.sqlmapFile = sqlmapFile; }
  }
  ```
  * **DI를 사용한다고 해서 항상 모든 프로퍼티 값을 설정에 넣고 모든 의존 오브젝트를 빈으로 일일이 지정할 필요는 없음**
  * 자주 사용되는 의존 오브젝트는 미리 지정한 디폴트 의존 오브젝트를 설정 없이도 사용할 수 있게 만드는 것도 좋은 방법.
  * **DefaultSqlService는 SqlService를 바로 구현한 것이 아니라 BaseSqlService를 상속했다는 점이 중요**
    * DefaultSqlService는 BaseSqlService의 sqlReader와 sqlRegistry 프로퍼티를 그대로 갖고 있음
    * 원한다면 언제든지 일부 또는 모든 프로퍼티를 변경 가능
  ```java
  // 리스트 7-44 디폴트 의존 오브젝트 대신 사용할 빈 선언
  <bean id="sqlService" class="springbook.user.sqlservice.DefaultSqlService">
  	<property name="sqlRegistry" ref=ultraSuperFastSqlRegistry" />
  </bean>
  ```
  * 디폴트 의존 오브젝트 사용법의 단점
    * 설정을 통해 다른 구현 오브젝트를 사용하게 해도 DefaultSqlService는 생성자에서 일단 디폴트 구현 오브젝트를 다 생성함.
    * 불필요한 오브젝트를 생성하는 건 리소스를 불필요하게 소모함
    * @PostContruct를 이용해서 프로퍼티가 설정되지 않았을 경우에만 디폴트 오브젝트를 만들도록 구성


### 7.3 서비스 추상화 적용


* JaxbXmlSqlReader 개선과제
  * 자바에는 JAXB 외에도 다양한 XML과 자바오브젝트를 매핑하는 기술이 존재
  <br>-> 필요에 따라 다른 기술로 손쉽게 바꿔서 사용할 수 있게 해야 함.
  * XML 파일을 좀 더 다양한 소스에서 가져올 수 있게 만든다.


### 7.3.1 OXM 서비스 추상화

* 실전에서 자주 사용되는 XML과 자바오브젝트 매핑 기술
  * Castor XML: 설정파일이 필요없는 인트로스펙션 모드를 지원하기도 하는 매우 간결하고 가벼운 바인딩 프레임워크
  * JiBX: 뛰어난 퍼포먼스를 자랑하는 XML 바인딩 기술
  * XmlBeans: 아파치 XML 프로젝트의 하나. XML의 정보셋을 효과적으로 제공
  * Xstream: 관례를 이용해서 설정이 없는 바인딩을 지원하는 XML 바인딩 기술

* OXM (Object-XML Mapping)
<br>XML과 자바오브젝트를 매핑해서 상호 변환해주는 기술

* OXM 프레임워크와 기술들은 기능 면에서 상호 호환성이 있음
<br>JAXB를 포함해서 다석 가지 기술 모두 사용 목적이 동일하기 때문에 유사한 기능과 API를 제공

* 스프링은 트랜잭션, 메일 전송뿐 아니라 OXM에 대해서도 서비스 추상화 기능을 제공함.
  * 스프링이 제공하는 OXM 추상 계층의 API를 이용해 XML 문서와 오브젝트 사이의 변환을 처리 시
  <br>코드 수정 없이도 OXM 기술을 자유롭게 바꿔서 적용 가능

* OXM 서비스 인터페이스
  * Unmarshaller 인터페이스
    * SqlReader를 대체하는 인터페이스
    * XML 파일에 대한 정보를 담은 Source 타입의 오브젝트를 받아
    <br>설정에서 지정한 OXM 기술을 이용해 자바오브젝트 트리로 변환하고 루트 오브젝트를 돌려줌
```java
// 리스트 7-45 Unmarshaller 인터페이스
package org.springframework.oxm; // spring-oxm 모듈 안에 정의되어 있음
...
import javax.xml.transform.Source;

public interface Unmarshaller {
	// 해당 클래스로 언마샬이 가능한지 확인해줌. 별로 사용할 일 없음.
	boolean supports(Class<?> class);

	// source를 통해 제공받은 XML을 자바오브젝트 트리로 변환해서 그 루트 오브젝트를 돌려줌.
	// 매핑 실패 시 추상화된 예외를 던진다. 서브클래스에 좀 더 세분화되어 있음.
	Object unmarshal(Source source) throws IOException, XmlMappingException;
}
```

* JAXB 구현 테스트
  * `Jaxb2Marshaller`: JAXB를 이용하도록 만들어진 Unmarshaller 구현 클래스
    * Jaxb2Marshaller 클래스는 Unmarshaller 인터페이스와 Marshaller 인터페이스를 모두 구현
  ```java
  // 리스트 7-46 JAXB용 Unmarshaller 빈 설정
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans-2.0.xsd">

	<bean id="unmarshaller" class="org.springframework.oxm.jaxb.Jaxb2Marshaller">
		<property name="contextPath" value="springbook.user.sqlservice.jaxb" />
	</bean>
  </beans>
  ```
  * 추상 인터페이스인 Unmarshaller의 `unmarshal()` 메서드를 호출하면 Jaxb2Marshaller 빈이 알아서 작업 수행
  ```java
  package springbook.learningtest.spring.oxm;
  ...
  import org.springframework.oxm.Unmarshaller;
  import javax.xml.transform.stream.StreamSource;

  @RunWith(SpringJUnit4ClassRunner.class)
  @ContextConfiguration
  public class OxmTest {
	@Autowired
	Unmarshaller unmarshaller;
	
	@Test
	public void unmarshallSqlMap() throws XmlMappingException, IOException  {
		Source xmlSource = new StreamSource(getClass().getResourceAsStream("sqlmap.xml"));
		// 어떤 OXM 기술이든 언마샬은 이 한 줄이면 끝.
		Sqlmap sqlmap = (Sqlmap)this.unmarshaller.unmarshal(xmlSource);
		
		List<SqlType> sqlList = sqlmap.getSql();		
		assertThat(sqlList.size(), is(3));
		assertThat(sqlList.get(0).getKey(), is("add"));
		...
		assertThat(sqlList.get(2).getValue(), is("delete"));
	}
  }
  ```

* Castor 구현 테스트
  * 서비스 추상화는 로우레벨의 기술을 필요에 따라 변경해서 사용하더라도 일관된 애플리케이션 코드를 유지할 수 있게 해줌.
```java
// 리스트 7-48 Castor용 매핑정보
<?xml version="1.0"?>
<!DOCTYPE mapping PUBLIC "-//EXOLAB/Castor Mapping DTD Version 1.0//EN" "http://castor.org/mapping.dtd">
<mapping>
    <class name="springbook.user.sqlservice.jaxb.Sqlmap">
        <map-to xml="sqlmap" />
        <field name="sql"
               type="springbook.user.sqlservice.jaxb.SqlType"
               required="true" collection="arraylist">
            <bind-xml name="sql" node="element" />
        </field>
    </class>
    <class name="springbook.user.sqlservice.jaxb.SqlType">
        <map-to xml="sql" />
        <field name="key" type="string" required="true">
            <bind-xml name="key" node="attribute" />
        </field>
        <field name="value" type="string" required="true">
            <bind-xml node="text" />
        </field>
    </class>
</mapping>

// 리스트 7-49 Castor 기술을 사용하는 언마샬러 적용
<bean id="unmarshaller" class="org.springframework.oxm.castor.CastorMarshaller"> // Unmarshaller 인터페이스를 Castor API를 이용해서 구현한 클래스
	<property name="mappingLocation" value="springbook/learningtest/spring/oxm/mapping.xml" />
</bean>
```


### 7.3.2 OXM 서비스 추상화 적용

* OxmSqlService 생성
  * 스프링의 OXM 서비스 추상화 기능을 이용하는 SqlService
  * SqlRegistry: DI할 수 있게 구성
  * SqlReader: 스프링의 OXM 언마샬러를 이용하도록 OxmSqlService 내에 고정

* 멤버 클래스를 참조하는 통합 클래스
  * SqlReader 타입의 의존 오브젝트를 사용하되 이를 스태틱 멤버 클래스로 내장하고 자신만 사용할 수 있게 구성
    * 의존 오브젝트를 자신만이 사용하도록 독점하는 구조로 만드는 방법
    * 유연성은 조금 희생하더라도 내부적으로 낮은 결합도를 유지한 채로 응집도가 높은 구현을 만들 때 유용
    * OxmSqlService가 내부적으로 OxmSqlReader를 갖고 있음
  * OxmSqlService와 OxmSqlReader는 구조적으로는 강하게 결합되어 있지만 논리적으로는 명확하게 분리되는 구조
  ```java
  // 리스트 7-50 OxmSqlService 기본 구조
  package springbook.user.sqlservice;
  ...
  public class OxmSqlService implements SqlService {
  	private final OxmSqlReader oxmSqlReader = new OxmSqlReader();
	...
	
	// private 멤버 클래스로 정의해서 톱레벨 클래스인 OxmSqlService만 사용 가능하도록
	private class OxmSqlReader implements SqlReader {
		...
	}
  }
  ```
  * 스프링의 OXM 서비스 추상화를 사용하면 언마샬러를 빈으로 등록해야 함.
  <br>-> 자꾸 늘어나는 빈의 개수와 반복되는 비슷한 DI 구조가 불편!
  * BaseSqlService를 확장해서 디폴트 설정을 두는 방법으로 빈의 개수를 줄이고 설정을 단순하게 할 수는 있음.
  <br>-> 디폴트로 내부에서 만드는 오브젝트의 프로퍼티를 외부에서 지정해주기가 힘들다!
  * OxmSqlReader는 외부에 노출되지 않기 때문에 OxmSqlService에 의해서만 생성되며 스스로 빈으로 등록 불가능
  <br>-> 자신이 DI를 통해 제공받아야 하는 프로퍼티가 있다면 OxmSqlService의 공개된 프로퍼티를 통해 간접적으로 DI
  ```java
  // 리스트 7-51 내부 오브젝트의 프로퍼티를 전달해주는 코드
  public class OxmSqlService implements SqlService {
	private final OxmSqlReader oxmSqlReader = new OxmSqlReader();
	...
	
	// OxmSqlService의 프로퍼티를 통해 DI 받은 것을 멤버 클래스인 OxmSqlReader로 전달
	public void setUnmarshaller(Unmarshaller unmarshaller) {
		this.oxmSqlReader.setUnmarshaller(unmarshaller);
	}
	
	public void setSqlmapFile(String sqlmapFile) {
		this.oxmSqlReader.setSqlmapFile(sqlmapFile);
	}
	
	private class OxmSqlReader implements SqlReader {
		private Unmarshaller unmarshaller;
		private String sqlmapFile;
		...
	}
  }
  
  // 리스트 7-52 완성된 OxmSqlService 클래스
  public class OxmSqlService implements SqlService {
	private final BaseSqlService baseSqlService = new BaseSqlService();
	
	private final OxmSqlReader oxmSqlReader = new OxmSqlReader();
	// oxmSqlReader와 달리 단지 디폴트 오브젝트로 만들어진 프로퍼피. 필요에 따라 DI를 통해 교체 가능
	private SqlRegistry sqlRegistry = new HashMapSqlRegistry();
	
	public void setSqlRegistry(SqlRegistry sqlRegistry) {
		this.sqlRegistry = sqlRegistry;
	}
	
	public void setUnmarshaller(Unmarshaller unmarshaller) {
		this.oxmSqlReader.setUnmarshaller(unmarshaller);
	}
	
	public void setSqlmapFile(String sqlmapFile) {
		this.oxmSqlReader.setSqlmapFile(sqlmapFile);
	}

	@PostConstruct
	public void loadSql() {
		this.oxmSqlReader.read(this.sqlRegistry);
	}

	public String getSql(String key) throws SqlRetrievalFailureException {
		try { return this.sqlRegistry.findSql(key); }
		catch(SqlNotFoundException e) { throw new SqlRetrievalFailureException(e); }
	}
	
	private class OxmSqlReader implements SqlReader {
		private Unmarshaller unmarshaller;
		private final static String DEFAULT_SQLMAP_FILE = "sqlmap.xml";
		private String sqlmapFile = DEFAULT_SQLMAP_FILE;

		public void setUnmarshaller(Unmarshaller unmarshaller) {
			this.unmarshaller = unmarshaller;
		}

		public void setSqlmapFile(String sqlmapFile) {
			this.sqlmapFile = sqlmapFile;
		}

		public void read(SqlRegistry sqlRegistry) {
			try {
				Source source = new StreamSource(UserDao.class.getResourceAsStream(this.sqlmapFile));
				// OxmSqlService를 통해 전달받은 OXM 인터페이스 구현 오브젝트를 가지고 언마샬링 수행
				Sqlmap sqlmap = (Sqlmap)this.unmarshaller.unmarshal(source);
				
				for(SqlType sql : sqlmap.getSql()) {
					sqlRegistry.registerSql(sql.getKey(), sql.getValue());
				}
			} catch (IOException e) {
				throw new IllegalArgumentException(this.sqlmapFile + "을 가져올 수 없습니다.", e);
			}
		}
	}
  }
  
  // 리스트 7-53 OXM을 적용한 SqlService 설정
  <bean id="sqlService" name="springbook.user.sqlservice.OxmSqlService">
  	<property name="unmarshaller" ref="unmarshaller" />
  </bean>
  
  <bean id="unmarshaller" class="org.springframework.oxm.jaxb.Jaxb2Marshaller">
  	<property name="contextPath" value="springbook.user.sqlservice.jaxb" />
  </bean>
  ```

* 위임을 이용한 BaseSqlService의 재사용
  * `loadSql()`, `getSql()`이라는 SqlService의 핵심 메서드 구현코드가 BaseSqlService와 동일
  * 프로퍼티 설정을 통한 초기화 작업을 제외하면 두 가지 작업의 코드는 BaseSqlService, OxmSqlService 양쪽에 중복됨.
  * `loadSql()`, `getSql()`의 구현 로직은 BaseSqlService에 두고 OxmSqlService는 일종의 설정과 기본 구성을
  <br>변경해주기 위한 어댑터 개념으로 BaseSqlService 앞에 두는 설계가 가능
  * 위임을 위해서는 2개의 빈을 등록하고 클라이언트의 요청을 받는 빈이 주요 내용을 뒤의 빈에 전달하는 구조로 설계
  * OxmSqlService는 OXM 기술에 특화된 SqlReader를 멤버로 내장, 필요한 설정을 한 번에 지정할 수 있는 확장구조만을 갖고 있음.
  * 실제 SqlReader와 SqlService를 이용해 SqlService의 기능을 구현하는 일은 내부에 BaseSqlService를 만들어서 위임.
```java
// 리스트 7-54 BaseSqlService로의 위임을 적용한 OxmSqlService
public class OxmSqlService implements SqlService {
	// SqlService의 실제 구현 부분을 위임할 대상인 BaseSqlService를 인스턴스 변수로 정의
	private final BaseSqlService baseSqlService = new BaseSqlService();
	...

	// OxmSqlService의 프로퍼티를 통해서 초기화된 SqlReader와 SqlRegistry를
	// 실제 작업을 위임할 대상인 baseSqlService에 주입
	@PostConstruct
	public void loadSql() {
		this.baseSqlService.setSqlReader(this.oxmSqlReader);
		this.baseSqlService.setSqlRegistry(this.sqlRegistry);
		
		// SQL을 등록하는 초기화 작업을 baseSqlService에 위임
		this.baseSqlService.loadSql();
	}
	
	// SQL을 찾아오는 작업도 baseSqlService에 위임
	public String getSql(String key) throws SqlRetrievalFailureException {
		return this.baseSqlService.getSql(key);
	}
	...
}
```


### 7.3.3 리소스 추상화

* OxmSqlReader, XmlSqlReader의 공통적인 문제
<br>SQL 매핑 정보가 담긴 XML 파일 이름을 프로퍼티로 외부에서 지정은 가능하지만 클래스패스에 존재하는 파일로 제한됨.

* 리소스
  * 스프링은 자바에 존재하는 일관성 없는 리소스 접근 API를 추상화함.
  <br>-> `Resource`라는 추상화 인터페이스를 정의
  * 다른 서비스 추상화 오브젝트와는 달리 **Resource는 스프링에서 빈이 아니라 값으로 취급됨.**
  * OXM이나 트랜잭션처럼 서비스를 제공해주는 것이 아닌 단순한 정보를 가진 값으로 지정됨.
```java
// 리스트 7-55 Resource 인터페이스
package org.springframework.core.io;
...
public interface Resource extends InputStreamSource {
  	// 리소스의 존재, 읽기 가능여부, 입력 스트림이 열려있는지를 확인 가능
	boolean exists();
	boolean isReadable();
	boolean isOpen();
	
	// JDK의 URL, URI, File 형태로 전환 가능한 리소스에 사용
	URL getURL() throws IOException;
	URI getURI() throws IOException;
	File getFile() throws IOException;
	
	Resource createRelative(String relativePath) throws IOException;
	
	// 리소스에 대한 이름과 부가적인 정보를 제공
	long lastModified() throws IOException;
	String getFilename();
	String getDescription();
}
```

* 리소스 로더
  * 리소스의 종류와 위치를 정의한 문자열을 실제 Resource 타입 오브젝트로 변환해주는 `ResourceLoader`를 제공
  * 접두어가 없는 경우 리소스 로더의 구현 방식에 따라 리소스를 가져오는 방식이 달라짐.
  * 접두어를 붙여주면 리소스 로더의 종류와 상관없이 접두어가 의미하는 위치와 방법을 이용해 리소스를 읽어옴.
  |접두어|예|설명|
  |-|-|-|
  |file:|`file:/C:/temp/file.txt`|파일 시스템의 C:temp 폴더에 있는 file.txt를 리소스로 생성|
  |classpath:|`classpath:file.txt`|클래스패스의 루트에 존재하는 file.txt 리소스에 접근|
  |없음|`WEB-INF/test.dat`|접두어가 없는 경우 ResourceLoader 구현에 따라 리소스의 위치가 결정됨|
  |http:|`http://www.myserver.com/test.dat`|HTTP 프로토콜을 사용해 접근할 수 있는 웹상의 리소스를 지정
  * ResourceLoader의 대표적인 예 -> **스프링의 애플리케이션 컨텍스트**
  <br>Application Context는 ResourceLoader 인터페이스를 상속
  * 스프링이 제공하는 빈으로 등록 가능한 클래스에 파일을 지정해주는 프로퍼티가 존재한다면 거의 모두 Resource 타입
```java
// 리스트 7-56 ResourceLoader 인터페이스
package org.springframework.core.io;
  
public interface ResourceLoader {
  	Resource getResource(String location); // location에 담긴 스트링 정보를 적절한 Resource로 변환
	...
}
```

* Resource를 이용해 XML 파일 가져오기
  * Resource 타입은 소스와 무관하게 `getInputStream()` 메서드를 이용해 스트림으로 가져올 수 있음.
  * Resource를 사용할 때는 **Reousrce 오브젝트가 실제 리소스는 아니라는 점을 주의**
  <br>-> Resource는 리소스에 접근할 수 있는 추상화된 핸들러
  * 코드에서 클래스패스 리소스를 바로 지정하고 싶다면 `ClassPathResource`를 사용해 오브젝트를 생성
  * 문자열로 지정할 때는 클래스 로더가 인식할 수 있는 문자열로 표현
```java
public class OxmSqlService implements SqlService {
  	// 이름과 타입을 모두 변경하여 유연성 확보
	public void setSqlmap(Resource sqlmap) {
		this.oxmSqlReader.setSqlmap(sqlmap);
	}
	...
	private class OxmSqlReader implements SqlReader {
		// SQL 매핑정보 소스의 타입을 Resource로 변경
		// Resource 구현 클래스인 ClassPathResource를 사용
		private Resource sqlmap = new ClassPathResource("sqlmap.xml", UserDao.class);

		public void setSqlmap(Resource sqlmap) {
			this.sqlmap = sqlmap;
		}

		public void read(SqlRegistry sqlRegistry) {
			try {
				// 리소스의 종류에 상관없이 getInputStream()으로 가져올 수 있음
				Source source = new StreamSource(sqlmap.getInputStream());
				...
			} catch (IOException e) {
				throw new IllegalArgumentException(this.sqlmap.getFilename() + " Read Error", e);
			}
		}
		...
	}
}

// 리스트 7-58 classpath: 접두어를 이용해 지정한 리소스
<bean id="sqlService" class="springbook.user.sqlservice.OxmSqlService">
	<property name="unmarshaller" ref="unmarshaller" /> 
	<property name="sqlmap" value="classpath:/springbook/user/dao/sqlmap.xml" />
</bean>
  
// 리스트 7-59 file: 접두어를 이용해 지정한 리소스
<bean id="sqlService" class="springbook.user.sqlservice.OxmSqlService">
	<property name="unmarshaller" ref="unmarshaller" /> 
	<property name="sqlmap" value="file:/opt/resources/sqlmap.xml" />
</bean>
  
// 리스트 7-60 HTTP로 접근 가능한 리소스
<bean id="sqlService" class="springbook.user.sqlservice.OxmSqlService">
	<property name="unmarshaller" ref="unmarshaller" /> 
	<property name="sqlmap" value="http://www.epril.com/resources/sqlmap.xml" />
</bean>
```


### 7.4 인터페이스 상속을 통한 안전한 기능확장

* 현재까지 만든 SqlService 클래스들은 초기에 리소스에서 SQL 정보를 읽어오고 이를 메모리에 두고 사용
<br>-> SQL 매핑파일 정보를 변경한다고 해도 메모리상의 SQL 정보가 갱신되지 않음!
<br>-> 변경내용을 반영하려면 서버를 재시작하거나 애플리케이션을 리로딩해야 함.


### 7.4.1 DI와 기능의 확장

* DI를 의식하는 설계가 중요
  * 인터페이스를 활용해서 단일 책임 원칙을 적용했기에 소스 수정 및 확장이 용이
  * 스프링 DI는 사용할 오브젝트를 직접 만드는 대신 프로퍼티로 정의하고 XML 빈 설정을 이용해서 주입하는 것
  * DI를 적용하려면 최소 2개 이상의 의존관계를 갖는 오브젝트가 필요
  <br>-> 의존 오브젝트는 자유롭게 확장될 수 있다는 점을 염두에 둬야 함

* DI와 인터페이스 프로그래밍
  * DI를 적용할 때는 가능한 한 인터페이스를 사용하게 해야 함.
  * 첫 번째 이유는 다형성을 통해 하나의 인터페이스를 여러 개로 구현할 수 있기 때문
  * 두 번째 이유는 인터페이스 분리 원칙을 통해 클라이언트와 의존 오브젝트 사이의 관계를 명확하게 해줄 수 있기 때문
  * **인터페이스 분리 원칙 (Interface Segregation Principle)**
  <br>오브젝트가 그 자체로 충분히 응집도가 높은 작은 단위로 설계됐더라도, **목적과 관심이 각기 다른 클라이언트가 있다면 인터페이스를 통해 적절히 분리**


### 7.4.2 인터페이스 상속

* 하나의 오브젝트가 구현하는 인터페이스를 여러 개 만들어서 구분하는 이유?
<br>-> 오브젝트의 기능이 발전하는 과정에서 다른 종류의 클라이언트가 등장하기 때문

* 인터페이스 분리 원칙이 주는 장점
  * 모든 클라이언트가 자신의 관심에 따른 접근 방식을 불필요한 간섭 없이 유지 가능
  * 기존 클라이언트에 영향을 주지 않은 채로 오브젝트의 기능을 확장하거나 수정 가능

* SQL 수정 기능을 추가하기 위해 SqlRegistry를 상속받는 인터페이스를 구성
  * 기존의 BaseSqlService는 신규 생성한 인터페이스를 사용할 필요 없이 기존의 SqlRegistry를 사용하면 됨.
  * SQL 업데이트 작업이 필요한 새로운 클라이언트만 신규 인터페이스를 사용
```java
// 리스트 7-62 SQL 수정 기능을 가진 확장 인터페이스
package springbook.issuetracker.sqlservice;
...
public interface UpdatableSqlRegistry extends SqlRegistry {
	public void updateSql(String key, String sql) throws SqlUpdateFailureException;
	
	public void updateSql(Map<String, String> sqlmap) throws SqlUpdateFailureException;
}

// 리스트 7-63 MyUpdatableSqlRegistry의 의존관계
<bean id="sqlService" class="springbook.user.sqlservice.BaseSqlService">
	...
	<property name="sqlRegistry" ref="sqlRegistry" />
</bean>

<bean id="sqlRegistry" class="springbook.user.sqlservice.MyUpdatableSqlRegistry" />

<bean id="sqlAdminService" class="springbook.user.sqlservice.SqlAdminService">
	...
	<property name="updatableSqlRegistry" ref="sqlRegistry" />
</bean>
```


### 7.5 DI를 이용해 다양한 구현 방법 적용하기


### 7.5.1 ConcurrentHashMap을 이용한 수정 가능 SQL 레지스트리

* 운영 중인 시스템에서 사용 중인 정보를 실시간으로 변경할 때는 *동시성 문제가 가장 먼저 고려되어야 함.*

* HashMapRegistry는 JDK의 HashMap을 사용
  * 멀티스레드 환경에서 안전하지 않음.
  * 멀티스레드 환경에서 안전하게 조작하려면 Collections.synchronizedMap() 등을 사용해야 하지만 부하가 큼.
  * ConcurrentHashMap은 데이터 조작 시 전체 데이터에 대해 락을 걸지 않고 조회는 락을 아예 사용하지 않음.
```java
// 리스트 7-65 ConcurrentHashMap을 이용한 SQL 레지스트리 테스트
public class ConcurrentHashMapSqlRegistryTest {
	UpdatableSqlRegistry sqlRegistry;
	
	@Before
	public void setUp() {
		sqlRegistry = new ConcurrentHashMapSqlRegistry();
		// 각 테스트 메서드에서 사용할 초기 SQL 정보를 미리 등록
		sqlRegistry.registerSql("KEY1", "SQL1");
		sqlRegistry.registerSql("KEY2", "SQL2");
		sqlRegistry.registerSql("KEY3", "SQL3");
	}
	
	@Test
	public void find() {
		checkFindResult("SQL1", "SQL2", "SQL3");
	}

	// 반복적으로 검증하는 부분은 별도의 메서드로 분리
	private void checkFindResult(String expected1, String expected2, String expected3) {
		assertThat(sqlRegistry.findSql("KEY1"), is(expected1));		
		assertThat(sqlRegistry.findSql("KEY2"), is(expected2));		
		assertThat(sqlRegistry.findSql("KEY3"), is(expected3));		
	}
	
	// 주어진 키에 해당하는 SQL를 찾을 때 에러가 발생하는지 확인
	@Test(expected= SqlNotFoundException.class)
	public void unknownKey() {
		sqlRegistry.findSql("SQL9999!@#$");
	}

	// 하나의 SQL을 변경하는 기능에 대한 테스트
	@Test
	public void updateSingle() {
		sqlRegistry.updateSql("KEY2", "Modified2");		
		checkFindResult("SQL1", "Modified2", "SQL3");
	}
	
	@Test
	public void updateMulti() {
		Map<String, String> sqlmap = new HashMap<String, String>();
		sqlmap.put("KEY1", "Modified1");
		sqlmap.put("KEY3", "Modified3");
		
		sqlRegistry.updateSql(sqlmap);		
		checkFindResult("Modified1", "SQL2", "Modified3");
	}

	// 존재하지 않는 키의 SQL을 변경하려고 시도할 때 에러가 발생하는지 확인
	@Test(expected=SqlUpdateFailureException.class)
	public void updateWithNotExistingKey() {
		sqlRegistry.updateSql("SQL9999!@#$", "Modified2");
	}
}

// 리스트 7-66 ConcurrentHashMap을 사용하는 SQL 레지스트리
public class ConcurrentHashMapSqlRegistry implements UpdatableSqlRegistry {
	private Map<String, String> sqlMap = new ConcurrentHashMap<String, String>();

	public String findSql(String key) throws SqlNotFoundException {
		String sql = sqlMap.get(key);
		if (sql == null)  throw new SqlNotFoundException(key);
		else return sql;
	}

	public void registerSql(String key, String sql) { sqlMap.put(key, sql);	}

	public void updateSql(String key, String sql) throws SqlUpdateFailureException {
		if (sqlMap.get(key) == null) {
			throw new SqlUpdateFailureException(key);
		}
		
		sqlMap.put(key, sql);
	}

	public void updateSql(Map<String, String> sqlmap) throws SqlUpdateFailureException {
		for(Map.Entry<String, String> entry : sqlmap.entrySet()) {
			updateSql(entry.getKey(), entry.getValue());
		}
	}
}

// 리스트 7-67 ConcurentHashMapSqlRegistry를 적용한 설정
<bean id="sqlService" class="springbook.user.sqlservice.OxmSqlService">
	<property name="unmarshaller" ref="unmarshaller" /> 
	<property name="sqlRegistry" ref="sqlRegistry" /> // 디폴트로 준비된 HashMapSqlRegistry 대신 외부에 지정한 레지스트리 사용하도록 명시적으로 지정
</bean>

<bean id="sqlRegistry" class="springbook.user.sqlservice.updatable.ConcurrentHashMapSqlRegistry">
</bean>
```


### 7.5.2 내장형 데이터베이스를 이용한 SQL 레지스트리 만들기

* 내장형 DB
<br>애플리케이션에 내장돼서 애플리케이션과 함께 시작되고 종료되는 DB

* 스프링의 내장형 DB 지원 기능
  * 내장형 DB는 애플리케이션 내에서 DB 기동, 초기화 SQL 스크립트 등의 초기화 작업이 필요
  * 스프링은 내장형 DB 지원 기능을 제공
  	EmbeddedDatabase db;
	...
	
	* 내장형 DB 초기화 작업은 내장형 DB 빌더를 통해 사용
  * 애플리케이션에서 `shutdown()` 메서드를 통해 DB 종료
	
* 내장형 DB 빌더 학습 테스트
  * 내장형 DB는 애플리케이션을 통해 DB가 시작될 때마다 매번 테이블을 새롭게 생성
  <br>-> 지속적으로 사용가능한 테이블 생성 SQL 스크립트가 필요
  * 내장형 DB 빌더 실행절차
    1. DB 엔진을 생성하고 초기화 스크립트 실행해서 테이블, 초기 데이터 준비
    2. DB에 접근할 수 있는 Connection을 생성하는 DataSource 오브젝트를 돌려줌
  * DataSource를 통해 DB Connection을 가져오는 것부터 JdbcTemplate을 활용하는 등 사용방법은 일반 DB와 거의 동일

* `EmbeddedDatabaseBuilder`: 스프링이 제공하는 내장형 DB 빌더
```java
new EmbeddedDatabaseBuilder() // 빌더 오브젝트 생성
	.setType(내장형DB종류) // EmbeddedDatabaseType의 HSAL, DERBY, H2 중 하나 선택
	.addScript(초기화에 사용할 DB 스크립트의 리소스) // 하나 이상 지정 가능
	...
	.build(); // 작업을 수행하고 오브젝트 리턴

// 리스트 7-70 내장형 DB 학습 테스트
public class EmbeddedDbTest {
	EmbeddedDatabase db;
	...
	
	@Before
	public void setUp() {
		db = new EmbeddedDatabaseBuilder()
		.setType(HSQL)
		.addScript("classpath:/springbook/learningtest/spring/embeddebddb/schema.sql")
		.addScript("classpath:/springbook/learningtest/spring/embeddebddb/data.sql")
		.build();
	}
	
	// 매 테스트를 진행하고 DB를 종료
	@After
	public void tearDown() {
		db.shutdown();
	}
}
```

* 내장형 DB를 이용한 SqlRegistry 만들기
  * EmbeddedDatabaseBuilder는 직접 빈으로 등록해도 바로 사용불가
    * 적절한 메서드를 호출해주는 초기화 코드가 필요
    * 초기화 코드가 필요하다면 팩토리 빈으로 생성
  * `jdbc` 태그를 통해 내장형 DB와 관련된 빈을 설정하고 등록
```java
// 리스트 7-71 HSQL 내장형 DB 설정 예
<jdbc:embedded-database id="embeddedDatabase" type="HSQL">
	<jdbc:script location="classpath:schema.sql" />
</jdbc:embedded-database>
```

* UpdatableSqlRegistry 테스트 코드의 재사용
  * ConcurrentHashMapSqlRegistry / EmbeddedDbSqlRegistry
  <br>-> 둘 다 UpdatableSqlRegistry 인터페이스를 구현하며 테스트 내용이 중복됨!
  * ConcurrentHashMapSqlRegistry 테스트 코드를 EmbeddedDbSqlRegistry가 공유하도록 변경
  <br>-> JUnit4.x를 사용하는 테스트 클래스는 상속 구조로 생성 가능
```java
// 리스트 7-73 테스트 코드에서 ConcurrentHashMapSqlRegistry에 의존하는 부분
public class ConcurrentHashMapSqlRegistryTest {
	UpdatableSqlRegistry sqlRegistry;
	
	@Before
	public void setUp() {
		sqlRegistry = new ConcurrentHashMapSqlRegistry();
		...
	}
	...
}

// 리스트 7-74 UpdatableSqlRegistry에 대한 테스트 추상 클래스
public abstract class AbstractUpdatableSqlRegistryTest {
	UpdatableSqlRegistry sqlRegistry;
	
	@Before
	public void setUp() {
		sqlRegistry = createUpdatableSqlRegistry();
		...
	}
	
	// 테스트 픽스쳐를 생성하는 부분만 추상 메서드로 만들어두고 서브클래스에서 구현
	abstract protected UpdatableSqlRegistry createUpdatableSqlRegistry();
	...
	
	// 다른 @Test 메서드들
}

// 리스트 7-75 변경된 ConcurrentHashMapSqlRegistryTest
// 해당 클래스에는 @Test 메서드가 보이지 않지만 슈퍼클래스의 @Test 테스트 메서드를 모두 상속받아서 활용
public class ConcurrentHashMapSqlRegistryTest extends AbstractUpdatableSqlRegistryTest {
	protected UpdatableSqlRegistry createUpdatableSqlRegistry() {
		return new ConcurrentHashMApSqlRegistry();
	}
}
```

* XML 설정을 통한 내장형 DB의 생성과 적용
```java
// 리스트 7-77 jdbc 네임스페이스 선언
<beans xmlns="http://www.springframework.org/schema/beans"
	...
	xmlns:jdbc="http://www.springframework.org/schema/jdbc"
	xsi:schemaLocation="http://www.springframework.org/schema/tx/spring-tx-3.0.xsd 
			http://www.springframework.org/schema/jdbc...">

// 리스트 7-78 내장형 DB 등록
<jdbc:embedded-database id="embeddedDatabase" type="HSQL">
	<jdbc:script location="classpath:springbook/user/sqlservice/updatable/sqlRegistrySchema.sql"/>
</jdbc:embedded-database>

// 리스트 7-79 EmbeddedDBSqlRegistry 클래스를 이용한 빈 등록
<bean id="sqlService" class="springbook.user.sqlservice.OxmSqlService">
	<property name="unmarshaller" ref="unmarshaller" /> 
	<property name="sqlRegistry" ref="sqlRegistry" />
</bean>

<bean id="sqlRegistry" class="springbook.user.sqlservice.updatable.EmbeddedDbSqlRegistry">
	<property name="dataSource" ref="embeddedDatabase" />
</bean>
```


### 7.5.3 트랜잭션 적용

* SimpleJdbcTemplate만 사용하면 트랜잭션이 미적용됨.
  * 트랜잭션 경계가 DAO 밖에 있고 범위가 넓은 경우는 AOP를 이용
  * 제한된 오브젝트 내에서는 트랜잭션 추상화 API를 사용

* 코드를 이용한 트랜잭션 적용
  * PlatformTransactionManager보다 TransactionTemplate을 사용
    * 트랜잭션을 프록시간 공유할 필요 없음
    * 빈으로 등록되지 않고 내부적으로만 사용
```java
// 리스트 7-81 트랜잭션 기능을 가진 EmbeddedDBSqlRegistry
public class EmbeddedDbSqlRegistry implements UpdatableSqlRegistry {
	SimpleJdbcTemplate jdbc;
	// JdbcTemplate과 트랜잭션을 동기화해주는 트랜잭션 템블릿. 멀티스레드 환경 사용 가능
	TransactionTemplate transactionTemplate;
	
	public void setDataSource(DataSource dataSource) {
		jdbc = new SimpleJdbcTemplate(dataSource);
		transactionTemplate = new TransactionTemplate(new DataSourceTransactionManager(dataSource));
		transactionTemplate.setIsolationLevel(TransactionTemplate.ISOLATION_READ_COMMITTED);
	}
	...
	// 파라미터는 익명 내부 클래스로 만들어지는 콜백 오브젝트 안에서 사용하므로 final로 선언
	public void updateSql(final Map<String, String> sqlmap) throws SqlUpdateFailureException {
		transactionTemplate.execute(new TransactionCallbackWithoutResult() {
			protected void doInTransactionWithoutResult(TransactionStatus status) {
				for(Map.Entry<String, String> entry : sqlmap.entrySet()) {
					updateSql(entry.getKey(), entry.getValue());
				}
			}
		});
	}
}
```


### 7.6 스프링 3.1의 DI

* 자바 코드의 메타정보를 이용하는 프로그래밍 방식이 도입됨
  * 일반적인 자바 코드와 달리 메타 데이터로 취급되는 경우가 존재
  * 리플렉션 API를 이용해 애노테이션의 메타정보를 조회하고 설정된 값을 가져와서 사용하는 방식
  * 애노테이션은 프레임워크가 참조하는 메타정보로 사용되기에 IoC 방식에 적용하기 좋음
  * 장점: XML에 비해 애노테이션은 작성할 코드의 양이 적음. 텍스트 방식이 아니기 때문에 오타 예방 가능
  * 단점: 애노테이션은 자바 코드에 존재하므로 변경할 때마다 클래스를 컴파일해야 함.
```java
// XML 방식
<x:special target="type" class="com.mycompany.myproject.MyClass" />

// 애노테이션 방식
package com.mycompany.myproject;

@Special
public class MyClass {
	...
}
```

* 정책과 관례를 이용한 프로그래밍이 방식이 도입됨.
  * 코드 없이도 미리 약속한 규칙 또는 관례를 따라 프로그래밍이 동작 ex) 애노테이션
  * 자바 코드로 모든 작업 과정을 작성할 때보다 코드의 양이 줄어듬.
  * 단, 프로그래밍 언어나 API 외에 미리 정의된 규칙과 관례를 이해해야 함.
  * 코드는 간결해지지만 정책을 잘못 알고 있을 경우 의도와 다른 코드가 작성될 수 있음.


### 7.6.1 자바 코드를 이용한 빈 설정

* 애노테이션과 자바 코드로 XML을 대체
  * 스프링 테스트 컨텍스트를 사용하지 않는 UserTest 같은 단위테스트는 변경 불필요
  * 스프링 프레임워크와 DI 정보를 사용하는 UserDaoTest, UserServiceTest는 변경 필요

* `@ContextConfiguration`
  * 스프링 테스트가 테스트용 DI 정보를 어디서 참조해야할지 지정하는 애노테이션
  * DI 정보로 사용될 자바 클래스를 만들고 `@Configuration` 애노테이션을 지정
```java
// 리스트 7-82 XML 파일을 사용하는 UserDaoTest
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserDaoTest {
	...
}

// 리스트 7-85 TestApplicationContext를 테스트 컨텍스트로 사용하도록 만든 UserDaoTesti
@Configuration
@ImportResource("/test-applicationContext.xml")
public class TestApplicationContext {
}

// 리스트 7-84 TestApplicationContext를 테스트 컨텍스트로 사용하도록 변경한 UserDaoTest
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes=TestApplicationContext.class)
public class UserDaoTest {
	...
}
```

* `<context:annotation-config />` 삭제
  * 해당 태그는 `@PostConstruct`를 붙인 메서드가 빈이 초기화된 후에 자동으로 실행되도록 사용
  * 컨텍스트를 가져오는 클래스에 `@Configuration`이 있으면 삭제 가능
  <br>-> 컨테이너가 직접 `@PostConstruction` 애노테이션을 처리하는 빈 후처리기를 등록해줌.

* `<bean>` 전환
  * `<bean>`으로 정의된 DI 정보는 `@Bean`이 붙은 애노테이션과 1:1 매핑됨.
  * `@Bean`이 붙은 public 메서드로 변경
  * 리턴값의 경우 빈을 주입받아서 사용하는 다른 빈이 어떤 타입으로 사용 중인지에 따라 달라짐
```java
// 리스트 7-87 XML을 이용한 dataSource 빈의 정의
<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
	<property name="driverClass" value="com.mysql.jdbc.Driver" />
	<property name="url" value="jdbc:mysql://localhost/springbook?characterEncoding=UTF-8" />
	<property name="username" value="spring" />
	<property name="password" value="book" />
</bean>

// 리스트 7-88 자바 코드로 작성한 dataSource 빈
@Bean
public DataSource dataSource() {
	// 빈 메서드 내부에서는 빈의 구현 클래스에 맞는 프로퍼티 값 주입이 필요
	SimpleDriverDataSource dataSource = new SimpleDriverDataSource();
	
	dataSource.setDriverClass(Driver.class); // 클래스 타입의 드라이버를 사용
	dataSource.setUrl("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8");
	dataSource.setUsername("spring");
	dataSource.setPassword("book");
	
	// 리턴은 확장성을 위해 인터페이스를 리턴함
	return dataSource;
}

// 리스트 7-89 XML로 정의한 transactionManager 빈
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource" />  
</bean>

// 리스트 7-90 자바 코드로 정의한 transactionManager 빈
@Bean
public PlatformTransactionManager transactionManager() {
	DataSourceTransactionManager tm = new DataSourceTransactionManager();
	tm.setDataSource(dataSource());
	return tm;
}
```

* testUserService 빈 전환
  * `parent` 정의를 이용해서 동일한 userService의 프로퍼티를 상속받음
  <br>-> `@Bean` 메서드로 전환 시에는 userService의 빈 설정을 참고해서 프로퍼티를 모두 지정해야 함.
  * `<bean>`으로 빈을 지정할 때는 리플렉션 API를 사용하기 때문에 private 접근제한자도 사용 가능
  <br>-> 직접 자바 코드에서 참조할 때는 접근제한자를 public으로 지정해야 함.
  * XML에서는 자바 코드로 작성한 빈을 `<property>`를 이용해서 참조 가능
  <br>-> 자바 코드에서 XML에 정의된 빈을 참조하려면 `@Autowired`가 붙은 필드를 선언해서 사용
```java
// 7-91 XML의 빈 정의
<bean id="userDao" class="springbook.user.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
	<property name="sqlService" ref="sqlService" />
</bean>

<bean id="userService" class="springbook.user.service.UserServiceImpl">
	<property name="userDao" ref="userDao" />
	<property name="mailSender" ref="mailSender" />
</bean>

<bean id="testUserService" class="springbook.user.service.UserServiceTest$TestUserService" parent="userService" />

<bean id="mailSender" class="springbook.user.service.DummyMailSender" />

// 애노테이션으로 변환한 빈 정의
@Autowired
SqlService sqlService; // XML에 정의된 sqlService 빈을 참조

@Bean
public UserDao userDao() {
	UserDaoJdbc dao = new UserDaoJdbc();
	dao.setDataSource(dataSource());
	dao.setSqlService(this.sqlService);
	return dao;
}

@Bean
public UserService userService() {
	UserServiceImpl service = new UserServiceImpl();
	service.setUserDao(userDao());
	service.setMailSender(mailSender());
	return service;
}


@Bean
public UserService testUserService() {
	TestUserService testService = new TestUserService();
	// userService와 동일하게 설정
	testService.setUserDao(userDao());
	testService.setMailSender(mailSender());
	return testService;
}

@Bean
public MailSender mailSender() {
	return new DummyMailSender();
}
```

* sqlService, sqlRegistry, unmarshaller 빈 전환
  * `@Autowired`: 필드 "타입"을 기준으로 XML에서 빈을 참조해서 주입
  * `@Resource`: 필드 "이름"을 기준으로 XML에서 빈을 참조해서 주입
```java
// SQL 서비스를 위한 3개의 <bean>
<bean id="sqlService" class="springbook.user.sqlservice.OxmSqlService">
	<property name="unmarshaller" ref="unmarshaller" />
	<property name="sqlRegistry" ref="sqlRegistry" />
</bean>

<bean id="sqlRegistry" class="springbook.user.sqlservice.updatable.EmbeddedDbSqlRegistry">
	<property name="dataSource" ref="embeddedDatabase" />
</bean>

<bean id="unmarshaller" class="org.springframework.oxm.jaxb.Jaxb2Marshaller">
	<property name="contextPath" value="springbook.user.sqlservice.jaxb" />
</bean>

// SQL 서비스를 위한 3개의 @Bean 메서드
@Bean
public SqlService sqlService() {
	OxmSqlService sqlService = new OxmSqlService();
	sqlService.setUnmarshaller(unmarshaller());
	sqlService.setSqlRegistry(sqlRegistry());
	return sqlService;
}

@Resource // 필드 "이름"을 기준으로 XML에서 빈을 참조
Database embeddedDatabase;

@Bean
public SqlRegistry sqlRegistry() {
	EmbeddedDbSqlRegistry sqlRegistry = new EmbeddedDbSqlRegistry();
	sqlRegistry.setDataSource(this.embeddedDatabase);
	return sqlRegistry;
}

@Bean
public Unmarshaller unmarshaller() {
	Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
	marshaller.setContextPath("springbook.user.sqlservice.jaxb");
	return marshaller;
}
```

* `<jdbc:embedded-database>`
  * type에 지정한 내장형 DB 생성
  * `<jdbc:script>`로 지정한 스크립트로 초기화
  * DataSource 타입 DB의 커넥션 오브젝트를 빈으로 등록
* `<tx:annotation-driven />`
  * 스프링 3.0에서는 아래 4가지 클래스를 빈으로 등록해야 했음.
    * org.springframework.aop.framework.autoproxy.InfrastructureAdvisorAutoProxyCreator
    * org.springframework.transaction.annotation.AnnotationTransactionAttributeSource
    * org.springframework.transaction.interceptor.TransactionInterceptor
    * org.springframework.transaction.interceptor.BeanFactoryTransactionAttributeSourceAdvisor
  * 스프링 3.1에서는 `@EnableTransactionManagement` 애노테이션을 사용
```java
// XML에 남은 2개의 빈
<jdbc:embedded-database id="embeddedDatabase" type="HSQL">
	<jdbc:script location="classpath:springbook/user/sqlservice/updatable/sqlRegistrySchema.sql" />
</jdbc:embedded-database>

<tx:annotation-driven />

// 내장형 DB 빈을 생성하는 @Bean 메서드
@Bean
public DataSource embeddedDatabase() {
	return new EmbeddedDatabaseBuilder()
		.setName("embeddedDatabase")
		.setType(HSQL)
		.addScript("classpath:springbook/user/sqlservice/updatable/sqlRegistrySchema.sql")
		.build();
}

// embeddedDatabase() 메서드를 사용해서 빈을 가져오도록 수정한 sqlRegistry()
@Bean
public SqlRegistry sqlRegistry() {
	EmbeddedDbSqlRegistry sqlRegistry = new EmbeddedDbSqlRegistry();
	sqlRegistry.setDataSource(embeddedDatabase()); // embeddedDatabase() 빈을 사용
	return sqlRegistry;
}
```


### 7.6.2 빈 스캐닝과 자동와이어링

* `@Autowired`를 이용한 자동와이어링
  * 스프링은 `@Autowired`가 붙은 수정자 메서드가 있으면 **파라미터 타입을 보고 주입 가능한 빈 모두 검색**
    * 주입 가능한 타입의 빈이 하나라면 스프링이 수정제 메서드를 호출해서 넣어줌.
    * 주입 가능한 타입의 빈이 2개 이상이면 그 중 프로퍼티와 동일한 이름의 빈을 사용
  * 주어진 오브젝트를 그대로 필드에 저장하는 수정자 메서드라면 필드에 `@Autowired` 적용
  * 만약 다른 오브젝트를 생성해서 저장해야 할 경우 메서드에 `@Autowired` 적용
  * DI 관련 코드를 줄일 수 있지만 설정정보를 통해 빈들 사이의 의존관계를 파악하기가 어려움
```java
public class UserDaoJdbc implements UserDao {
	// dataSource 수정자에 @Autowired 사용
	@Autowired
	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
	}
	
	// sqlService 필드에 @Autowired 사용. 리플렉션 API를 이용해서 DI
	@Autowired
	private SqlService sqlService;
	
	// @Autowired 주입으로 변경된 userDao()
	@Bean
	public UserDao userDao() {
		return new UserDaoJdbc();
	}
}
```

* `@Component`를 이용한 자동 빈 등록
  * 클래스에 부여되는 애노테이션
  * `@Component`가 붙은 클래스는 빈 스캐너를 통해 자동으로 빈으로 등록됨.
  * 빈으로 등록될 후보 클래스에 붙여주는 일종의 마커(marker)
  * 해당 애노테이션을 붙이면 프로젝트 내의 모든 클래스패스를 다 검색해서 부하가 많이 걸림
  <br>-> `@ComponentScan`을 이용해서 스캔 범위를 지정
  * `@ComponentScan`
    * basePackages: 기준 패키지를 지정할 때 사용. 지정한 패키지 아래의 모든 서브패키지를 다 검색
  * `@Component`가 붙은 클래스의 이름 대신 다른 이름을 사용하고 싶으면 애노테이션에 이름을 지정
  <br>`@Component("userDao")`
```java
// @Component를 적용한 UserDaoJdbc
@Component
public class UserDaoJdbc implements UserDao {
}

// 어플리케이션 컨텍스트 클래스에서 userDao() 메서드 제거
@Autowired UserDao userDao;

@Bean
public UserService userService() {
	UserServiceImpl service = new UserServiceImpl();
	service.setUserDao(this.userDao);
	service.setMailSender(mailSender());
	return service;
}


@Bean
public UserService testUserService() {
	TestUserService testService = new TestUserService();
	testService.setUserDao(this.userDao);
	testService.setMailSender(mailSender());
	return testService;
}

// @ComponentScan 적용
@Configuration
@EnableTransactionManagement
@ComponentScan(basePackages="springbook.user")
public class TestApplicationContext {
}
```

* 메타 애노테이션
  * 애노테이션의 정의에 부여된 애노테이션
  * 여러 개의 애노테이션에 공통된 속성을 적용하고자 할 때 사용
  * 애노테이션은 `@interface` 키워드를 이용해서 정의
```java
// @Component 메타 애노테이션을 가진 애노테이션 정의
@Component
public @interface SnsConnector {
	...
}

// @SnsConnector를 부여해서 자동 빈 등록 대상으로 지정
@SnsConnector
public class FacebookConnector {
}
```

* `@Repository`
  * DAO 빈을 자동등록 대상으로 지정
  * 스프링은 DAO 기능을 제공하는 클래스에는 해당 애노테이션을 이용하도록 권장
```java
@Repository
public class UserDaoJbc implements UserDao {
}
```

* `@Service`
  * 비즈니스 로직을 담은 서비스 계층의 빈을 자동등록 대상으로 지정
  * 스프링은 서비스 기능을 제공하는 클래스에는 해당 애노테이션을 이용하도록 권장
```java
@Service("userService")
public class UserServiceImpl implements UserService {
}
```


### 7.6.3 컨텍스트 분리와 @Import

* 테스트용 컨텍스트 분리
  * 일반적인 빈과 테스트용 빈은 따로 분리하는게 좋음
  * DI 설정정보를 분리하려면 DI 설정 클래스를 추가하고 관련된 빈 설정 애노테이션, 필드, 메서드를 이동
  * 테스트용 빈과 애플리케이션 빈의 설정정보를 분리했다면 스캔 대상의 위치도 분리하는 게 좋음
  * 테스트용 빈은 설정정보에 내용이 나타나는 게 좋음
```java
// 분리한 테스트 DI 정보
@Configuration
public class TestAppContext {
	// userDao 빈은 @Repository를 적용해서 자동으로 빈으로 등록됨
	@Autowired UserDao userDao;
	
	// testUserService 빈은 userDao와 mailSender 빈에 의존
	@Bean
	public UserService testUserService() {
		// TestUserService는 UserServiceImpl을 상속했으므로 자동 와이어링 대상
		TestUserService testService = new TestUserService();
		testService.setUserDao(this.userDao);
		testService.setMailSender(mailSender());
		return testService;
	}
	
	@Bean
	public MailSender mailSender() {
		return new DummyMailSender();
	}
}

// 자동 와이어링을 활용하도록 간략하게 바꾼 테스트 DI 정보
@Configuration
public class TestAppContext {
	@Bean
	public UserService testUserService() {
		return new TestUserService();
	}
	
	@Bean
	public MailSender mailSender() {
		return new DummyMailSender();
	}
}

// 테스트 컨텍스트의 DI 설정 클래스 정보 수정
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes={TestAppContext.class, AppContext.class})
public class UserDaoTest {
}
```

* `@Import`
  * SQL 서비스는 독립적인 모듈처럼 취급하는 게 좋음
  <br>-> 다른 애플리케이션을 구성하는 빈과 달리 독립적으로 개발되거나 변경될 가능성이 높기 때문
  * 애플리케이션이 동작할 때 반드시 필요한 정보라면, 기존의 AppContext와 연결되는 게 좋음
  <br>-> AppContext는 메인 설정정보로, SqlServiceContext는 AppContext에 포함되는 보조 설정정보로 사용!
```java
// SQL 서비스 빈 설정을 위한 SqlServiceContext 클래스
@Configuration
public class SqlServiceContext {
	@Bean
	public SqlService sqlService() {
		OxmSqlService sqlService = new OxmSqlService();
		sqlService.setUnmarshaller(unmarshaller());
		sqlService.setSqlRegistry(sqlRegistry());
		return sqlService;
	}

	@Bean
	public SqlRegistry sqlRegistry() {
		EmbeddedDbSqlRegistry sqlRegistry = new EmbeddedDbSqlRegistry();
		sqlRegistry.setDataSource(embeddedDatabase());
		return sqlRegistry;
	}

	@Bean
	public Unmarshaller unmarshaller() {
		Jaxb2Marshaller marshaller = new Jaxb2Marshaller();
		marshaller.setContextPath("springbook.user.sqlservice.jaxb");
		return marshaller;
	}

	@Bean
	public DataSource embeddedDatabase() {
		return new EmbeddedDatabaseBuilder()
			.setName("embeddedDatabase")
			.setType(HSQL)
			.addScript("classpath:springbook/user/sqlservice/updatable/sqlRegistrySchema.sql")
			.build();
	}
}

// @Import 적용
@Configuration
@EnableTransactionManagement
@ComponentScan(basePackages="springbook.user")
@Import(SqlServiceContext.class) // SqlServiceContext를 보조 설정정보로 사용
public class AppContext {
}
```


### 7.6.4 프로파일

* 운영용 설정정보는 테스트와 별도로 만들어서 관리가 필요
  * 실행환경에 따라 설정정보를 각각 설정하는 건 불편
  * 테스트 환경에서는 AppContext, TestAppContext를 적용하고 운영 환경에서는 AppContext, ProductionContext를 적용하는 식

* `@Profile`과 `@ActiveProfiles`
  * 스프링은 환경에 따라 빈 설정정보가 달라져야하는 경우 간단히 설정정보를 구성할 수 있는 방법을 제공
  <br>-> 프로파일을 구성해 놓고 실행 시점의 어떤 프로파일의 빈 설정을 사용할지 지정
  * 프로파일을 적용 시 하나의 설정 클래스만 이용해서 환경에 따라 다른 빈 설정 구성 가능
  * **프로파일은 설정 클래스 단위로 지정**
  * 프로파일이 지정되어 있지 않은 빈 설정은 default 프로파일로 취급되며 항상 적용됨.
  * `@Profile`이 붙은 설정 클래스는 `@ActiveProfiles`에 프로파일 이름이 들어있지 않으면 무시됨.
  <br>-> 프로파일이 일종의 필터처럼 적용됨!
```java
// @Profile을 지정한 TestAppContext
@Configuration
@Profile("test")
public class TestAppContext { ... }


// @Import에 모든 설정 클래스 추가
@Configuration
@EnableTransactionManagement
@ComponentScan(basePackages="springbook.user")
@Import(SqlServiceContext.class, TestAppContext.class, ProductionAppContext.class)
public class AppContext { ... }


// 활성 프로파일을 지정한 UserServiceTest
@RunWith(SprintJUnit4ClassRunner.class)
@ActiveProfiles("test") // @Profile이 test인 컨텍스트만 적용
@ContextConfiguration(classes=AppContext.class) // 클래스 하나만 지정해서 모든 컨텍스트를 포함 가능
public class UserServiceTest { ... }
```

* 컨테이너의 빈 등록 정보 확인
  * 스프링 컨테이너는 모두 BeanFactory라는 인터페이스를 구현
  * `DefaultListableBeanFactory`: 대부분의 컨테이너들이 빈을 등록/관리할 때 사용하는 클래스
```java
// 등록된 빈 내역을 조회하는 테스트 메서드
@Autowired DefaultListableBeanFactory bf;
@Test
public void beans() {
	for (String n : bf.getBeanDefinitionNames()) {
		System.out.println(n + " \t " + bf.getBean(n).getClass().getName());
	}
}
```

* 중첩 클래스를 이용한 프로파일 적용
  * 파일이 많아지면 전체 구성을 알아보기가 힘듦.
  * static 중첩 클래스를 이용해서 구조는 유지한 채 소스 코드의 위치를 통합
```java
// 프로파일을 적용한 AppContext 설정 클래스
@Configuration
@EnableTransactionManagement
@ComponentScan(basePackages="springbook.user")
@Import(SqlServiceContext.class)
public class AppContext {
	@Bean
	public DataSource dataSource() {
		SimpleDriverDataSource dataSource = new SimpleDriverDataSource();
		dataSource.setDriverClass(Driver.class);
		dataSource.setUrl("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8");
		dataSource.setUsername("spring");
		dataSource.setPassword("book");
		return dataSource;
	}

	@Bean
	public PlatformTransactionManager transactionManager() {
		DataSourceTransactionManager tm = new DataSourceTransactionManager();
		tm.setDataSource(dataSource());
		return tm;
	}
	
	// static 중첩 클래스로 넣은 @Configuration 클래스는 스프링이 자동으로 포함시킴.
	// @Import에 따로 표시할 필요 없음!
	@Configuration
	@Profile("production")
	public static class ProductionAppContext {
		@Bean
		public MailSender mailSender() {
			JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
			mailSender.setHost("localhost");
			return mailSender;
		}
	}
	
	@Configuration
	@Profile("test")
	public static class TestAppContext {
		@Bean
		public UserService testUserService() {
			return new TestUserService();
		}
		
		@Bean
		public MailSender mailSender() {
			return new DummyMailSender();
		}
	}
}
```


### 7.6.5 프로퍼티 소스

* AppContext에 있는 테스트 환경에 종속되는 dataSource의 DB 연결정보를 분리할 필요가 있음
<br>-> 별도 프로퍼티 소스에 DB 연결정보를 저장!

* `@PropertySource`
  * 사용할 프로퍼티 등록을 위한 애노테이션. 프로퍼티 소스에 등록된 정보를 가져옴
  * 프로퍼티에 들어갈 DB 연결정보는 텍스트로 된 이름과 값의 쌍으로 구성
  * `@PropertySource`로 등록한 리소스로부터 가져오는 프로퍼티 값은 `Environment` 타입의 오브젝트에 저장됨.
  <br>-> `getProperty()` 메서드를 이용해서 프로퍼티값 조회 가능
```java
// database.properties 파일
db.driverClass=com.mysql.jdbc.Driver
db.url=jdbc:mysql://localhost/springbook?characterEncoding=UTF-8
db.username=spring
db.password=book


// 환경 오브젝트로부터 프로퍼티 값을 가져오도록 수정한 dataSource() 메서드
@Configuration
@EnableTransactionManagement
@ComponentScan(basePackages="springbook.user")
@Import(SqlServiceContext.class)
@PropertySource("/database.properties")
public class AppContext {
	...
	@Autowired Environment env;
	
	@Bean
	public DataSource dataSource() {
		SimpleDriverDataSource ds = new SimpleDriverDataSource();
		
		try {
			ds.setDriverClass((Class<? extends java.sql.Driver>)
				Class.forName(env.getProperty("db.driverClass")));
		} catch (ClassNotFoundException e) {
			throw new RuntimeException(e);
		}
		ds.setUrl(env.getProperty("db.url"));
		ds.setUserName(env.getProperty("db.username"));
		ds.setPassword(env.getProperty("db.password"));
		
		return ds;
	}
}
```

* `PropertySourcesPlaceholderConfigurer`
  * Environment 오브젝트 대신 프로퍼티 값을 직접 DI 받을 때 사용
  * `@Value`를 통해 값을 주입하는 방식으로 적용
  * PropertySourcesPlaceholderConfigurer는 프로퍼티 소스로부터 가져온 값을 `@Value` 필드에 주입하는 기능을 제공
```java
// @Value를 이용한 프로퍼티 값 주입
@PropertySource("/database.properties")
public class AppContext {
	@Value("${db.driverClass}") Class<? extends Driver> driverClass;
	@Value("${db.url}") String url;
	@Value("${db.username}") String username;
	@Value("${db.password}") String password;
}


// 프로퍼티 소스를 이용한 치환자 설정용 빈
// 반드시 static으로 선언해야 함!
@Bean
public static PropertySourcesPlaceholderConfigurer placeholderConfigurer() {
	return new PropertySourcesPlaceholderConfigurer();
}


// @Value 필드를 사용하도록 수정한 dataSource() 메서드
@Bean
public DataSource dataSource() {
	SimpleDriverDataSource ds = new SimpleDriverDataSource();
	ds.setDriverClass(this.driverClass);
	ds.setUrl(this.url);
	ds.setUsername(this.username);
	ds.setPassword(this.password);
	
	return ds;
}
```


### 7.6.6 빈 설정의 재사용과 @Enable*

* 빈 설정자
  * SQL 서비스를 재사용 가능한 독립적인 모듈로 만들어야 향후 확장이 가능
  * SqlServiceContext에서 SQL 매핑파일의 위치를 지정하는 작업을 분리
  * `@Configuration` 애노테이션이 달린, 빈 설정으로 사용되는 AppContext 같은 클래스도 스프링에서는 하나의 빈
  * `@Configuration`은 `@Component`를 메타 애노테이션으로 갖고 있는 자동 빈 등록용 애노테이션
```java
// SqlMapConfig 인터페이스
import org.springframework.core.io.Resource;
public interface SqlMapConfig {
	Resource getSqlMapResource();
}


// SqlMapConfig 인터페이스를 구현한 클래스
public class UserSqlMapConfig implements SqlMapConfig {
	@Override
	public Resource getSqlMapResource() {
		return new ClassPathResource("sqlmap.xml", UserDao.class);
	}
}


// SqlMapConfig 타입 빈에 의존하게 만든 SqlServiceContext
@Configuration
public class SqlServiceContext {
	@Autowired SqlMapConfig sqlMapConfig;
	
	@Bean
	public SqlService sqlService() {
		OxmSqlService sqlService = new OxmSqlService();
		sqlService.setUnmarshaller(unmarshaller());
		sqlService.setSqlRegistry(sqlRegistry());
		sqlService.setSqlmap(this.sqlMapConfig.getSqlMapResource()); // SqlMapConfig 타입 빈에 의존
		return sqlService;
	}
	...
}


// sqlMapConfig 빈 설정
public class AppContext {
	...
	
	// sqlMapConfig 빈은 SqlConfigService 빈에 @Autowired를 통해 주입돼서 사용됨.
	@Bean
	public SqlMapConfig sqlMapConfig() {
		return new UserSqlMapConfig();
	}
}


// SqlMapConfig를 구현하게 만든 AppContext
// sqlMapConfig() 메서드는 제거
public class AppContext implements SqlMapConfig {
	...
	
	@Override
	public Resource getSqlMapResource() {
		return new ClassPathResource("sqlmap.xml", UserDao.class);
	}
}
```


* @Enable* 애노테이션
  * SqlServiceContext는 이제 SqlService 라이브러리 모듈에 포함돼서 재사용이 가능
  * SqlService가 필요한 애플리케이션은 아래 작업 진행
    * 메인 설정 클래스에서 `@Import`로 SqlServiceContext 빈 설정을 추가
    * SqlMapConfig를 구현해 SQL 매핑 파일의 위치를 지정
  * `@Component`를 `@Repository`, `@Service`처럼 의미있는 애노테이션으로 만들어서 사용했듯이
  <br>`@Import`도 @Enable* 애노테이션으로 만들어서 사용 가능
```java
// @Import를 메타 애노테이션으로 넣은 애노테이션 정의
@Import(value=SqlServiceContext.class)
public @interface EnableSqlService {
}


// @EnableSqlService 적용
@Configuration
@ComponentScan(basePackages="springbook.user")
@EnableTransactionManagement
@EnableSqlService // SqlService를 사용하겠다는 의미
@PropertySource("/database.properties")
public class AppContext implements SqlMapConfig { ... }
```


### 7.7 정리

* SQL처럼 변경될 수 있는 텍스트로 된 정보는 **외부 리소스에 만들고 가져오게 구성**

* 성격이 다른 코드가 섞여 있는 클래스는 코드를 인터페이스로 분리할 것
<br>다른 인터페이스에 속한 기능은 인터페이스를 통해 접근하고, 간단히 **자기참조 빈으로 의존관계를 만들어 검증**

* 자주 사용되는 의존 오브젝트는 디폴트로 미리 정의할 것

* XML과 오브젝트 매핑은 스프링의 **OXM 추상화 기능**을 활용

* 특정 의존 오브젝트를 고정시켜 기능을 특화하려면 멤버 클래스로 구성

* 외부의 파일이나 리소스를 사용하는 코드에서는 스프링의 **리소스 추상화**와 **리소스 로더**를 사용

* DI를 의식하면서 코드를 작성하면 객체지향 설계에 도움이 됨.

* DI에는 인터페이스를 사용해서 인터페이스 분리 원칙을 준수할 것

* 클라이언트에 따라 인터페이스를 분리 시, 신규 인터페이스 생성 혹은 인터페이스 상속으로 설계

* 애플리케이션에 내장 DB를 사용 시에는 스프링의 내장형 DB 추상화 기능과 전용 태그를 사용
