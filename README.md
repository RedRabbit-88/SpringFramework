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
		} 
		catch(SqlNotFoundException e) {
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
  * 다른 서비스 추상화 오브젝트와는 달리 **Resource는 스프링에서 빈이 아니라 값으로 취급됨.**
  * OXM이나 트랜잭션처럼 서비스를 제공해주는 것이 아닌 단순한 정보를 가진 값으로 지정됨.

* 리소스 로더
  * 리소스의 종류와 위치를 정의한 문자열을 실제 Resource 타입 오브젝트로 변환해주는 `ResourceLoader`를 제공
  ```java
  // 리스트 7-56 ResourceLoader 인터페이스
  package org.springframework.core.io;
  
  public interface ResourceLoader {
  	Resource getResource(String location); // location에 담긴 스트링 정보를 적절한 Resource로 변환
	...
  }
  ```
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

* Resource를 이용해 XML 파일 가져오기
  * Resource 타입은 소스와 무관하게 `getInputStream()` 메서드를 이용해 스트림으로 가져올 수 있음.
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
  ```
  * Resource를 사용할 때는 **Reousrce 오브젝트가 실제 리소스는 아니라는 점을 주의**
  <br>-> Resource는 리소스에 접근할 수 있는 추상화된 핸들러
  * 코드에서 클래스패스 리소스를 바로 지정하고 싶다면 `ClassPathResource`를 사용해 오브젝트를 생성
  * 문자열로 지정할 때는 클래스 로더가 인식할 수 있는 문자열로 표현
  ```java
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

