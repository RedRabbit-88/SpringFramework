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
