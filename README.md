# 스프링 프레임워크 학습 (토비의 스프링 3.1)


## 1장 오브젝트와 의존관계
https://github.com/RedRabbit-88/SpringFramework/wiki/1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84#1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84


## 2장 테스트
https://github.com/RedRabbit-88/SpringFramework/wiki/2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8#2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8


## 3장 템플릿

* 개방 폐쇄 원칙
<br>코드에서 어떤 부분은 변경을 통해 그 기능이 다양해지고 확장하려는 성질이 있고
<br>어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있음을 말해준다.

* 템플릿
<br>바뀌는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분은
<br>자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법


### 3.1 다시 보는 초난감 DAO


### 3.1.1 예외처리 기능을 갖춘 DAO

* 정상적인 JDBC 코드의 흐름을 따르지 않고 중간에 어떤 이유로든 예외가 발생했을 경우에도 사용한 리소스를 반드시 반환하도록 만들어야 함.

```java
// 리스트 3-1 JDBC API를 이용한 DAO 코드인 deleteAll()
public void deleteAll() throws SQLException {
	Connection c = dataSource.getConnection();
	
	// 여기서 예외가 발생하면 바로 메서드 실행이 중단된다.
  	// try-catch-finally 문으로 예외처리가 필요.
	PreparedStatement ps = c.prepareStatement("delete from users");
	ps.executeUpdate();

	ps.close();
	c.close();
}
```

* 리소스 반환과 close()
  * `close()` 메서드는 리소스를 반환한다는 의미
  * Connection과 PreparedStatement는 보통 풀(pool) 방식으로 운영됨.
  * 미리 정해진 풀 안에 제한된 수의 리소스를 만들어두고 필요할 때 이를 할당하고, 반환하면 다시 풀에 채워넣는 방식

* 보통은 try-catch-finally 문으로 예외처리를 진행함.
  * `close()` 메서드도 예외가 발생할 수 있으므로 예외처리가 필요.
  * `ResultSet`의 경우도 예외가 발생할 수 있으므로 예외처리가 필요.
  <br>-> 너무 난잡한 try-catch-finally 문이 남용될 수 있음.


### 3.2 변하는 것과 변하지 않는 것


### 3.2.1 JDCB try/catch/finally 코드의 문제점

* 만약에 예외처리가 제대로 되지 않아 `close()` 메서드가 실행되지 않을 경우 풀로 리소스가 반환되지 않을 수 있음.
<br>-> 당장은 문제가 발생하지 않겠지만 지속적으로 시스템을 사용 시 언젠가 문제 발생할 수 있음.

* try/catch/finally문으로 해결은 가능하나 너무 난잡하게 반복될 수 있음.

* 예외상황을 처리하는 코드는 테스트하기가 매우 어렵고 모든 DAO 메서드에 대해 이런 테스트를 일일이 한다는 건 매우 번거롭다.


### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용

```java
// 리스트 3-4 개선할 deleteAll() 메서드
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	
	try {
		c = dataSource.getConnection();
		
		// 변하는 부분
		ps = c.prepareStatement("delete from users");
		
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null) {
			try {
				ps.close();
			} catch (SQLException) {}
		}
		if (c != null) {
			try {
				c.close();
			} catch (SQLException) {}
		}
	}
}
```

* 메서드 추출
  * 변하는 부분을 메서드로 추출하는 건 별로 유용하지 않다.
  * 템플릿 메서드 패턴을 적용하면 상속을 통해 기능을 확장해서 사용함.
  <br>-> DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 하는 단점!

```java
// 리스트 3-6 변하는 부분을 메서드로 추출한 후의 deleteAll()
public void deleteAll() throws SQLException {
	...
	try {
		c = dataSource.getConnection();
		
		// 변하는 부분을 메서드로 추출하고 변하지 않는 부분에서 호출하도록 만들었다.
		ps = makeStatement(c);
		
		ps.executeUpdate();
	} catch (SQLException e)
	...
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
	PreparedStatement ps;
	ps = c.prepareStatement("delete from users");
	return ps;
}

// 리스트 3-7 makeStatement()를 구현한 UserDao 서브 클래스
// 템플릿 메서드 패턴으로의 접근은 제한이 많음.
// DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다!
public class UserDaoDeleteAll extends UserDao {
	protected PreparedStatement makeStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```

* 전략 패턴의 적용
  * 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만든다.
  * 컨텍스트의 contextMethod(): deleteAll()에서 변하지 않는 부분

* deleteAll()의 컨텍스트
  * DB 커넥션 가져오기
  * PreparedStatement를 만들어줄 외부 기능 호출하기
  <br>-> 전략 패턴에서 말하는 전략
  * 전달받은 PreparedStatement 실행하기
  * 예외가 발생하면 이를 다시 메서드 밖으로 던지기
  * 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기
  <br>-> 이 부분을 컨텍스트 메서드로 분류한다!

```java
// 리스트 3-8 StatementStrategy 인터페이스
package springbook.user.dao;
...
public interface StatementStrategy {
	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}

// 리스트 3-9 deleteAll() 메서드의 기능을 구현한 StatementStrategy 전략 클래스
package springbook.user.dao;
...
public class DeleteAllStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}

// 리스트 3-10 전략 패턴을 따라 DeleteAllStatement가 적용된 deleteAll() 메서드
public void deleteAll() throws SQLException {
	...
	try {
		c = dataSource.getConnection();
		
		// StatementStrategy 인터페이스를 이용해서 전략 패턴을 활용
		// 컨텍스트가 특정 구현 클래스인 DeleteAllStatement를 알고 있다는 건
		// 전략 패턴, OCP에도 잘 들어맞는다고 보기 어려움.
		StatementStrategy strategy = new DeleteAllStatement();
		ps = strategy.makePreparedStatement(c);
		
		ps.executeUpdate();
	} catch (SQLException e)
	...
}
```

* DI 적용을 위한 클라이언트/컨텍스트 분리
  * 전략 패턴에 따르면 컨텍스트가 어떤 전략을 사용하게 할 것인가는 컨텍스트를 사용하는 **앞단의 클라이언트가 결정**
  * 클라이언트가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 컨텍스트에 전달
  * 컨텍스트는 전달받은 Strategy 구현 클래스의 오브젝트를 사용

```java
// 리스트 3-11 메서드로 분리한 try/catch/finally 컨텍스트 코드
// StatementStrategy stmt: 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;

	try {
		c = dataSource.getConnection();

		ps = stmt.makePreparedStatement(c);
	
		ps.executeUpdate();
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
		if (c != null) { try {c.close(); } catch (SQLException e) {} }
	}
}

// 리스트 3-12 클라이언트 책임을 담당한 deleteAll() 메서드
public void deleteAll() throws SQLException {
	// 선정한 전략 클래스의 오브젝트 생성
	StatementStrategy st = new DeleteAllStatement();
	// 컨텍스트 호출. 전략 오브젝트 전달
	jdbcContextWithStatementStrategy(st);
}
```

* 마이크로 DI
  * DI의 가장 주요한 개념
  <br>제 3자의 도움을 통해 두 오브젝트 사이의 유연한 관계가 설정되도록 만드는 것.
  * 일반적으로 DI는 의존관계에 있는 두 개의 오브젝트, 이 관계를 설정해주는 DI 컨테이너, 그리고 이를 사용하는 클라이언트 사이에서 발생.
  * 때로는 클라이언트가 DI 컨테이너의 책임을 같이 지고 있을 수도 있음. 또는 클라이언트와 전략(의존 오브젝트)가 결합될 수도 있음.
  * 마이크로 DI: DI의 장점을 단순화해서 IoC 컨테이너의 도움 없이 코드 내에서 적용한 경우


### 3.3 JDBC 전략 패턴의 최적화

* 개선된 DAO 메서드는 전략 패턴의 클라이언트로서 컨텍스트에 해당하는 `jdbcContextWithStatementStrategy()` 메서드에 적절한 전략,
<br>즉 바뀌는 로직을 제공해주는 방법으로 사용할 수 있다.
  * 컨텍스트: PreparedStatement를 실핸하는 JDBC의 작업 흐름
  * 전략: PreparedStatement를 생성하는 것


### 3.3.1 전략 클래스의 추가 정보

* `deleteAll()`과 `add()` 두 군데에서 모두 PreparedStatement를 실행하는 JDBC try/catch/finally 컨텍스트를 공유해서 사용 가능

```java
// 리스트 3-14 User 정보를 생성자로부터 제공받도록 만든 AddStatement
package springbook.user.dao;
...
public class AddStatement implements StatementStrategy {
	User user;
	
	public AddStatement(User user) {
		this.user = user;
	}
	
	public PreparedStatement makePreparedStatement(Connection c) {
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());
					
		return ps;
	}
}

// 리스트 3-15 user 정보를 AddStatement에 전달해주는 add() 메서드
public void add(User user) throws SQLException {
	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```


### 3.3.2 전략과 클라이언트의 동거

* 현재까지 구조의 2가지 문제점
  * DAO 메서드마다 새로운 StatementStrategy 구현 클래스를 만들어야 한다.
  <br>기존 UserDao 때보다 클래스 파일의 개수가 많이 늘어난다.
  * DAO 메서드에서 StatementStrategy에 전달한 User와 같은 부가적인 정보가 있는 경우
  <br>이를 위해 오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만들어야 함.

* 로컬 클래스
  * StatementStrategy 전략 클래스를 매번 독립된 파일로 만들지 말고 UserDao 클래스 안에 내부 클래스로 정의
  * 클래스가 내부 클래스이기 때문에 자신이 선언된 곳의 정보에 접근할 수 있음.

* 중첩 클래스의 종류
  * 중첩 클래스: 다른 클래스 내부에 정의되는 클래스
    * 스태틱 클래스(static class): 독립적으로 오브젝트로 만들어질 수 있음
    * 내부 클래스(inner class): 자신이 정의된 클래스와 오브젝트 안에서만 만들어질 수 있음
  * 내부 클래스의 분류
    * 멤버 내부 클래스(member inner class): 멤버 필드처럼 오브젝트 레벨에 정의
    * 로컬 클래스(local class): 메서드 레벨에 정의
    * 익명 내부 클래스(anonymous inner class): 이름을 갖지 않는 익명 클래스

```java
// 리스트 3-16 add() 메서드 내의 로컬 클래스로 이전한 AddStatement
public void add(User user) throws SQLException {
	class AddStatement implements StatementStrategy {
		User user;
	
		public AddStatement(User user) {
			this.user = user;
		}
	
		public PreparedStatement makePreparedStatement(Connection c) {
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
			ps.setString(1, user.getId());
			ps.setString(2, user.getName());
			ps.setString(3, user.getPassword());
					
			return ps;
		}
	}
	
	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}

// 리스트 3-17 add() 메서드의 로컬 변수를 직접 사용하도록 수정한 AddStatement
public void add(final User user) throws SQLException {
	class AddStatement implements StatementStrategy {
		public PreparedStatement makePreparedStatement(Connection c) {
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
			// 로컬 클래스의 코드에서 외부의 메서드 로컬 변수에 직접 접근 가능
			ps.setString(1, user.getId());
			ps.setString(2, user.getName());
			ps.setString(3, user.getPassword());
					
			return ps;
		}
	}
	
	// 생성자 파라미터로 user를 전달하지 않아도 됨.
	StatementStrategy st = new AddStatement();
	jdbcContextWithStatementStrategy(st);
}
```

* 익명 내부 클래스
  * 내부 클래스를 만들지 않고 익명 내부 클래스로 적용 가능
  * 이럴 경우 내부 클래스 코드보다 간결하게 표현 가능

```java
// 리스트 3-19 메서드 파라미터로 이전한 익명 내부 클래스
public void add(final User user) throws SQLException {
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {			
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
				ps.setString(1, user.getId());
				ps.setString(2, user.getName());
				ps.setString(3, user.getPassword());
				
				return ps;
			}
		}
	);
}

// 리스트 3-20 익명 내부 클래스를 적용한 deleteAll() 메서드
public void deleteAll() throws SQLException {
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement("delete from users");
			}
		}
	);
}
```


### 3.4 컨텍스트와 DI


### 3.4.1 JdbcContext의 분리

* 전략 패턴의 구조로 보는 UserDao
  * 클라이언트: UserDao의 메서드
  * 전략: 익명 내부 클래스로 만들어지는 로직
  * 컨텍스트: jdbcContextWithStatementStrategy() 메서드
  <br>컨텍스트 메서드는 UserDao 내의 PreparedStatement를 실행하는 기능을 가진 메서드에서 공유 가능

```java
// 리스트 3-21 JDBC 작업 흐름을 분리해서 만든 JdbcContext 클래스
package springbook.user.dao;
...
public class JdbcContext {
	// DataSource 타입 빈을 DI 받을 수 있게 준비
	DataSource dataSource;
	
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
	
	// JdbcContext 클래스 안으로 옮겼으므로 메서드 이름도 수정함.
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
		Connection c = null;
		PreparedStatement ps = null;

		try {
			c = dataSource.getConnection();

			ps = stmt.makePreparedStatement(c);
		
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
			if (c != null) { try {c.close(); } catch (SQLException e) {} }
		}
	}
}

// 리스트 3-22 JdbcContext를 DI 받아서 사용하도록 만든 UserDao
public class UserDao {
	private JdbcContext jdbcContext;
		
	public void setJdbcContext(JdbcContext jdbcContext) {
		this.jdbcContext = jdbcContext;
	}
	
	public void add(final User user) throws SQLException {
		// DI 받은 JdbcContext의 컨텍스트 메서드를 사용하도록 수정
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() {...}
		);
	}

	public void deleteAll() throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() {...}
		);
	}
}
```

* UserDao는 이제 JdbcContext에 의존하고 있음. 하지만 JdbcContext의 경우 인터페이스가 아닌 구체 클래스임.
<br>**스프링의 DI는 기본적으로 인터페이스를 사이에 두고 의존 클래스를 바꿔서 사용하도록 하는 게 목적**

```java
// 리스트 3-23 JdbcContext 빈을 추가하도록 수정한 설정파일
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
			http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
	
	<bean id="userDao" class="springbook.user.dao.UserDao">
		<property name="dataSource" ref="dataSource" /> // UserDao 내에 아직 JdbcContext를 적용하지 않은 메서드 존재
		<property name="jdbcContext" ref="jdbcContext" />
	</bean>
	
	// 추가된 JdbcContext 타입 빈
	<bean id="jdbcContext" class="springbook.user.dao.JdbcContext">
		<property name="dataSource" ref="dataSource" />
	</bean>
	
	<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource">
		...
	</bean>
</beans>
```


### 3.4.2 JdbcContext의 특별한 DI

* UserDao와 JdbcContext 사이에는 인터페이스를 사용하지 않고 DI를 적용함.
<br>런타임 시에 DI 방식으로 외부에서 오브젝트를 주입해주는 방식을 사용하긴 했지만
<br>클래스 레벨에서 의존관계를 맺었기 때문에 의존 오브젝트의 구현 클래스 변경은 불가능.

* DI 개념을 충실히 따르자면, 인터페이스를 사이에 둬서 클래스 레벨에서는 의존관계가 고정되지 않게 하고
<br>런타임 시에 의존할 오브젝트와의 관계를 다이내믹하게 주입해주는게 맞음.

* 스프링의 DI는 넓게 보자면 객체의 생성과 관계설정에 대한 제어권한을 오브젝트에서 제거하고 외부로 위임했다는 IoC라는 개념을 포괄함.
<br>-> 이런 의미에서는 JdbcContext를 스프링을 이용해 UserDao에서 사용한 건 DI라고 볼 수 있음.

* JdbcContext를 UserDao와 DI 구조로 만들어야 하는 이유
  * JdbcContext가 스프링 컨테이너의 싱글톤 레지스트리에서 관리되는 싱글톤 빈이 되기 때문.
    * JdbcContext는 자체로 변경되는 상태정보를 갖고 있지는 않지만 싱글톤으로 등록되어 공유되는게 이상적
  * **JdbcContext가 DI를 통해 다른 빈에 의존하고 있기 때문.**
    * JdbcContext는 dataSource 프로퍼티를 통해 DataSource 오브젝트를 주입받고 있음.
    * DI를 위해서는 주입되는 오브젝트와 주입받는 오브젝트 양쪽 모두 스프링 빈으로 등록되어야 함.
    * JdbcContext는 다른 빈을 DI 받기 위해서라도 스프링 빈으로 등록되어야 함.

* 코드를 이용하는 수동 DI
  * JdbcContext를 스프링 빈으로 등록해서 UserDao에 DI 하는 대신 사용 가능한 방법도 있음.
    * UserDao 내부에서 직접 DI를 적용.
    * 이럴 경우 싱글톤으로 관리될 수 없다. 사용 시마다 생성과 초기화를 해줘야 함.
  * DI 컨테이너를 통해 DI 받고 싶지 않을 경우
    * JdbcContext에 대한 제어권을 갖고 생성과 관리를 담당하는 UserDao에게 DI까지 위임.

```java
// 리스트 3-24 jdbcContext 빈을 제거한 설정파일
<beans>
	<bean id="userDao" class="springbook.user.dao.UserDao">
		<property name="dataSource" ref="dataSource" />
	</bean>
	
	<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource" >
	...
	</bean>
</beans>

// 리스트 3-25 JdbcContext 생성과 DI 작업을 수행하는 setDataSource() 메서드
public class UserDao {
	...
	private JdbcContext jdbcContext;
	
	// 수정자 메서드이면서 JdbcContext에 대한 생성, DI 작업을 동시에 수행
	public void setDataSource(DataSource dataSource) {
		// JdbcContext 생성 (IoC)
		this.jdbcContext = new JdbcContext();
		// 의존 오브젝트 주입 (DI)
		this.jdbcContext.setDataSource(dataSource);
		// 아직 JdbcContext를 적용하지 않은 메서드를 위해 저장
		this.dataSource = dataSource;
	}
}
```

* `setDataSource()` 메서드는 DI 컨테이너가 DataSource 오브젝트를 주입해줄 때 호출됨.
<br>이 때 JdbcContext에 대한 수동 DI 작업을 진행하면 됨.
  * JdbcContext의 오브젝트를 만들어서 인스턴스 변수에 저장
  * JdbcContext에 UserDao가 DI 받은 DataSource 오브젝트를 주입

* 굳이 인터페이스를 두지 않아도 될 만큼 긴밀한 관계를 갖는 DAO 클래스와 JdbcContext를
<br>어색하게 따로 빈으로 분리하지 않고 내부에서 직접 만들어 사용하면서도 다른 오브젝트에 대한 DI 적용 가능


### 3.5 템플릿과 콜백

* 템플릿/콜백 패턴
  * 전략 패턴의 기본 구조에 익명 내부 클래스를 활용한 방식
    * 템플릿: 전략 패턴의 컨텍스트
    * 콜백: 익명 내부 클래스로 만들어지는 오브젝트
  * 복잡하지만 바뀌지 않는 일정한 패턴을 갖는 작업 흐름이 존재하고
  <br>그 중 일부분만 바꿔서 사용해야 하는 경우에 적합한 구조


### 3.5.1 템플릿/콜백의 동작원리

* 템플릿: 고정된 작업 흐름을 가진 코드를 재사용

* 콜백: 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트

* 템플릿/콜백 패턴의 콜백은 보통 단일 메서드 인터페이스를 사용
<br>템플릿의 작업 흐름 중 특정 기능을 위해 한 번 호출되는 경우가 일반적이기 때문
<br>콜백은 일반적으로 하나의 메서드를 가진 인터페이스를 구현한 익명 내부 클래스로 만들어진다.

* 콜백 인터페이스의 메서드에는 보통 파라미터가 존재.
<br>템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용됨.
<br>ex) JdbcContext 클래스에서 템플릿인 workWithStatementStrategy 메서드에서
<br>콜백의 메서드인 makePreparedStatement()을 실행할 때 Connection 오브젝트를 파라미터로 넘겨줌.

* 템플릿/콜백의 작업 흐름
  * 클라이언트(UserDao)의 역할은 템플릿 안에서 실행될 로직을 담은 콜백 오브젝트를 만들고, 콜백이 참조할 정보를 제공하는 것.
  <br>만들어진 콜백은 클라이언트가 템플릿의 메서드를 호출할 때 파라미터로 전달됨.
  * 템플릿(workWithStatementStrategy())은 정해진 작업 흐름을 따라 작업을 진행하다가 내부에서 생성한
  <br>참조정보를 가지고 콜백 오브젝트의 메서드를 호출. (StatementStrategy)
  * 콜백은 클라이언트 메서드에 있는 정보와 템플릿이 제공한 참조정보를 이용해서 작업을 수행하고 그 결과를 다시 템플릿에 돌려줌.
  * 템플릿은 콜백이 돌려준 정보를 사용해서 작업을 마저 수행. 경우에 따라 최종 결과를 클라이언트에 다시 돌려주기도 함.

* 클라이언트가 템플릿 메서드를 호출하면서 콜백 오브젝트를 전달하는 것은 메서드 레벨에서 일어나는 DI.

* 일반적인 DI라면 템플릿에 인스턴스 변수를 만들어두고 사용할 의존 오브젝트를 수정자 메서드로 받아서 사용

* 템플릿/콜백 방식에서는 매번 메서드 단위로 사용할 오브젝트를 새롭게 전달받음.
<br>콜백 오브젝트가 내부 클래스로서 자신을 생성한 클라이언트 메서드 내의 정보를 직접 참조.

* 템플릿/콜백 방식은 전략 패턴과 DI의 장점을 익명 내부 클래스 사용 전략과 결합한 독특한 활용법


### 3.5.2 편리한 콜백의 재활용

* DAO 메서드에서 익명 내부 클래스를 사용하기 때문에 상대적으로 코드 작성 및 읽기가 불편함.

```java
// 리스트 3-26 익명 내부 클래스를 사용한 클라이언트 코드
public void deleteAll() throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
		// 변하지 않는 콜백 클래스 정의와 오브젝트 생성
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				// 변하는 SQL 문장
				return c.prepareStatement("delete from users");
			}
		}
	);
}

// 리스트 3-27 변하지 않는 부분을 분리시킨 deleteAll() 메서드
// 콜백 클래스 정의와 오브젝트 생성 부분을 분리시키고 SQL만 다르게 받는 방식
public void deleteAll() throws SQLException {
	// 변하는 SQL 문장
	executeSql("delete from users");
}

private void executeSql(final String query) throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement(query);
			}
		}
	);
}
```

* 콜백과 템플릿의 결합
  * 일반적으로 성격이 다른 코드들은 가능한 한 분리하는 편이 나음.
  * 하나의 목적을 위해 서로 긴밀하게 연관되어 있는 코드들이면 한 군데 모여 있는 게 유리함.

```java
// 리스트 3-28 JdbcContext로 옮긴 executeSql() 메서드
public class JdbcContext {
	...
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException {
	...
	}
	
	// 메서드 접근자는 public으로 변경하여 외부에서 접근 허용
	public void executeSql(final String query) throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() {
				public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
					return c.prepareStatement(query);
				}
			}
		);
	}
}

// 리스트 3-29 JdbcContext로 옮긴 executeSql()을 사용하는 deleteAll() 메서드
ublic void deleteAll() throws SQLException {
	this.jdbcContext.executeSql("delete from users");
}
```


### 3.5.3 템플릿/콜백의 응용

* 템플릿/콜백 패턴이 스프링에서만 사용할 수 있거나 스프링이 독점적으로 제공하는 기술은 아니지만,
<br>스프링만큼 이 패턴을 적극적으로 활용하는 프레임워크는 없다.

* 스프링을 사용하는 개발자라면 당연히 스프링이 제공하는 템플릿/콜백 기능을 잘 사용할 수 있어야 함.
<br>템플릿/콜백이 필요한 곳이 있으면 직접 만들어서 사용할 줄도 알아야 함.

* 고정된 작업 흐름을 갖고 있으면서 여기저기서 자주 반복되는 코드가 있다면,
<br>중복되는 코드를 분리할 방법을 생각해보는 습관을 기르자.
  1. 중복된 코드를 메서드로 분리하는 걸 시도
  2. 일부 작업을 필요에 따라 바꿔 사용해야 한다면 인터페이스를 사이에 두고 분리해서
  <br>전략 패턴을 적용하고 DI로 의존관계를 관리하도록 만든다.
  3. 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면
  <br>템플릿/콜백 패턴을 적용하는 것을 고려

* 테스트와 try/catch/finally
  * 파일을 하나 열어서 모든 라인의 숫자를 더한 합을 만드는 코드

```java
// 리스트 3-30 파일의 숫자 합을 계산하는 코드의 테스트
public class CalcSumTest {	
	@Test public void sumOfNumbers() throws IOException {
		Calculator calculator = new Calculator();
		int sum = calculator.calcSum(getClass().getResource("numbers.txt").getPath());
		assertThat(sum, is(10));
	}
}

// 리스트 3-32 try/catch/finally를 적용한 calcSum() 메서드
public class Calculator {
	public Integer calcSum(String filepath) throws IOException {
		BufferedReader br = null;
		
		try {
			// 한 줄씩 읽기 편하게 BufferedReader로 파일을 가져온다.
			br = new BufferedReader(new FileReader(filepath));
			Integer sum = 0;
			String line = null;
		
			// 마지막 라인까지 한 줄씩 읽어가면서 숫자를 더한다.
			while(line = br.readLine()) != null) {
				sum += Integer.valueOf(line);
			}
		
			return sum;
		} catch (IOException e) {
			System.out.println(e.getMessage());
		} finally {
			// BufferedReader 오브젝트가 생성되기 전에 예외가 발생할 수 있으므로 null 체크
			if (br != null) {
				try { br.close(); }
				catch (IOException e) { System.out.println(e.getMessage()); }
			}
		}
	}
}
```

* 중복의 제거와 템플릿/콜백 설계
  * 파일에 있는 모든 숫자의 곱을 계산하는 기능이 추가될 경우 소스를 모두 복사해서 사용할 것인가? NO!!
  <br>-> 템플릿/콜백 패턴을 적용!
  * 템플릿/콜백을 적용할 때는 템플릿과 콜백의 결계를 정하고 템플릿이 콜백에서, 콜백이 템플릿에게
  <br>각자 전달하는 내용이 무엇인지 파악하는게 가장 중요.
  <br>-> 파악된 내용을 기반으로 콜백의 인터페이스를 정의해야 하기 때문

```java
// 리스트 3-33 BufferedReader를 전달받는 콜백 인터페이스
public interface BufferedReaderCallBack {
	Integer doSomethingWithReader(BufferedReader br) throws IOException;
}

// 리스트 3-34 BufferedReaderCallBack을 사용하는 템플릿 메서드
public Integer fileReadTemplate(String filepath, BufferedReaderCallback callback) throws IOException {
	BufferedReader br = null;
		
	try {
		br = new BufferedReader(new FileReader(filepath));
		// 콜백 오브젝트 호출
		// 템플릿에서 만든 컨텍스트 정보인 BufferedReader를 전달해주고 콜백의 작업 결과를 받아둔다.
		int ret = callback.doSomethingWithReader(br);
		return ret;
	} catch (IOException e) {
		System.out.println(e.getMessage());
		throw e;
	} finally {
		if (br != null) {
			try { br.close(); }
			catch (IOException e) { System.out.println(e.getMessage()); }
		}
	}
}

// 리스트 3-35 템플릿/콜백을 적용한 calcSum() 메서드
public Integer calcSum(String filepath) throws IOException {
	BufferedReaderCallBack sumCallback =
	new BufferedReaderCallback() {
		public Integer doSomethingWithReader(BufferedReader br) throws IOException {
			Integer sum = 0;
			String line = null;
			while(line = br.readLine()) != null) {
				sum += Integer.valueOf(line);
			}
			return sum;
		}
	};
	// sumCallback 이란 익명 내부 클래스를 이용해서 fileReadTemplate를 호출
	// fileReadTemplate에서 callback.doSomethingWithReader()를 이용해서 콜백 실행
	return fileReadTemplate(filepath, sumCallback);
}

// 리스트 3-36 새로운 테스트 메서드를 추가한 CalSumTest
public class CalcSumTest {
	Calculator calculator;
	String numFilePath;
	
	// 테스트 효율을 위해 미리 픽스처로 만들어둠.
	@Before public void setUp() {
		this.calculator = new Calculator();
		this.numFilePath = getClass().getResource("numbers.txt").getPath();
	}
	
	@Test public void sumOfNumbers() throws IOException {
		assertThat(calculator.calcSum(this.numFilePath), is(10));
	}
	
	@Test public void multiplyOfNumbers() throws IOException {
		assertThat(calculator.calcMultiply(this.numFilePath), is(24));
	}
}

// 리스트 3-37 곱을 계산하는 콜백을 가진 calcMultiply() 메서드
public Integer calcMultiply(String filepath) throws IOException {
	BufferedReaderCallBack multiplyCallback =
	new BufferedReaderCallback() {
		public Integer doSomethingWithReader(BufferedReader br) throws IOException {
			Integer multiply = 1;
			String line = null;
			while(line = br.readLine()) != null) {
				multiply *= Integer.valueOf(line);
			}
			return multiply;
		}
	};
	// multiplyCallback 이란 익명 내부 클래스를 이용해서 fileReadTemplate를 호출
	// fileReadTemplate에서 callback.doSomethingWithReader()를 이용해서 콜백 실행
	return fileReadTemplate(filepath, multiplyCallback);
}
```

* 템플릿/콜백의 재설계
  * `calcSum()`, `calcMultiply()` 모두 숫자를 계산하는 부분 말고는 중복되는 부분이 많음.
  * 해당 부분만 변경될 수 있도록 분리할 필요가 있음.

```java
// 리스트 3-38 라인별 작업을 정의한 콜백 인터페이스
public interface LineCallback {
	Integer doSomethingWithLine(String line, Integer value);
}

// 리스트 3-39 LineCallback을 사용하는 템플릿
// int initVal: 계산 결과를 저장할 변수의 초기값
public Integer lineReadTemplate(String filepath, LineCallback callback, int initVal) throws IOException {
	BufferedReader br = null;
	try {
		br = new BufferedReader(new FileReader(filepath));
		Integer res = initVal;
		String line = null;
		// while 루프 안에서 콜백을 호출
		while((line = br.readLine()) != null) {
			res = callback.doSomethingWithLine(line, res);
		}
		return res;
	}
	catch(IOException e) {
		System.out.println(e.getMessage());
		throw e;
	}
	finally {
		if (br != null) {
			try { br.close(); } 
			catch(IOException e) { System.out.println(e.getMessage()); }
		}
	}
}

// 리스트 3-40 lineReadTemplate을 사용하도록 수정한 calSum(), calMultiply() 메서드
public Integer calcSum(String filepath) throws IOException {
	LineCallback sumCallback = 
	new LineCallback() {
		public Integer doSomethingWithLine(String line, Integer value) {
			return value + integer.valueOf(line);
		}
	};
	// sumCallback 이란 익명 내부 클래스를 이용해서 lineReadTemplate를 호출
	// lineReadTemplate를 callback.doSomethingWithLine()를 이용해서 콜백 실행
	return lineReadTemplate(filepath, sumCallback, 0);
}

public Integer calcMultiply(String filepath) throws IOException {
	LineCallback multiplyCallback = 
	new LineCallback() {
		public Integer doSomethingWithLine(String line, Integer value) {
			return value * integer.valueOf(line);
		}
	};
	return lineReadTemplate(filepath, multiplyCallback, 1);
}
```

* 제네릭스를 이용한 콜백 인터페이스
  * 지금까지는 템플릿과 콜백이 만들어내는 결과가 Integer로 고정되어 있음.
  <br>-> 제네릭스를 활용하면 다양한 타입으로 활용 가능.

```java
// 리스트 3-41 타입 파라미터를 적용한 LineCallback
public interface LineCallback<T> {
	T doSomethingWithLine(String line, T value);
}

// 리스트 3-42 타입 파라미터를 추가해서 제네릭 메서드로 만든 lineReadTemplate()
public <T> T lineReadTemplate(String filepath, LineCallback<T> callback, T initVal) throws IOException {
	BufferedReader br = null;
	try {
		br = new BufferedReader(new FileReader(filepath));
		T res = initVal; // 제네릭스로 변경
		String line = null;
		// while 루프 안에서 콜백을 호출
		while((line = br.readLine()) != null) {
			res = callback.doSomethingWithLine(line, res);
		}
		return res;
	}
	catch(IOException e) { ... }
	finally { ... }
}

// 리스트 3-43 문자열 연결 기능 콜백을 이용해 만든 concatenate() 메서드
public String concatenate(String filepath) throws IOException {
	LineCallback<String> concatenateCallback = 
	new LineCallback<String>() {
		public String doSomethingWithLine(String line, Stringvalue) {
			return value + line;
		}
	};
	// 템플릿 메서드의 T는 모두 String이 된다.
	return lineReadTemplate(filepath, concatenateCallback, "");
}

// 리스트 3-44 concatenate() 메서드에 대한 테스트
@Test public void concatenateStrings() throws IOException {
	assertThat(calculator.concatenate(this.numFilePath), is("1234"));
}
```


### 3.6 스프링의 JdbcTemplate

* 스프링은 JDBC를 이용하는 DAO에서 사용할 수 있도록 준비된 다양한 템플릿과 콜백을 제공

* `JdbcTemplate`
<br>스프링이 제공하는 JDBC 코드용 기본 템플릿

```java
// 리스트 3-45 JdbcTemplate의 초기화를 위한 코드
public class UserDao {
	private JdbcTemplate jdbcTemplate;
	private DataSource dataSource;
		
	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
			
		this.dataSource = dataSource;
	}
	...
}
```


### 3.6.1 update()

* JdbcContext vs. JdbcTemplate
  * 인터페이스: `StatementStrategy` -> `PreparedStatementCreator`
  * 메서드: `makePreparedStatement()` -> `createPreparedStatement`

```java
public void deleteAll() throws SQLException {
	// 리스트 3-46 JdbcTemplate을 적용
	this.jdbcTemplate.update(new PreparedStatementCreator() {
		@Override
		public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
			return con.prepareStatement("delete from users");
		}
	});
	
	// 리스트 3-47 내장 콜백을 사용하는 update()
	this.jdbcTemplate.update("delete from users");
}

// 내장 콜백을 사용하는 update()로 만든 add() 메서드
public void add(final User user) throws SQLException {
	this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)",
					user.getId(), user.getName(), user.getPassword());
}
```


### 3.6.2 queryForInt()

* `query()` 메서드
<br>`PreparedStatementCreator` 콜백과 `ResultSetExtractor` 콜백을 파라미터로 받음

* `ResultSetExtractor`
<br>`PreparedStatement`의 쿼리를 실행해서 얻은 `ResultSet`을 전달받는 콜백
  * 템플릿이 제공하는 `ResultSet`을 이용해 원하는 값을 추출해서 템플릿에 전달
  * 템플릿은 나머지 작업을 수행한 뒤에 그 값을 `query()` 메서드의 리턴값으로 돌려줌.

```java
// 리스트 3-49 JdbcTemplate을 이용해 만든 getCount()
// getCount(): SQL 쿼리를 실행하고 ResultSet을 통해 결과 값을 가져오는 코드
public int getCount() {
	return this.jdbcTemplate.query(new PreparedStatementCreator() {
		@Override
		public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
			return con.prepareStatement("select count(*) from users");
		}
	}, new ResultSetExtractor<Integer>() {
		@Override
		public Integer extractData(ResultSet rs) throws SQLException, DataAccessException {
			rs.next();
			return rs.getInt(1);
		}		
	});
}

// 리스트 3-50 queryForInt()를 사용하도록 수정한 getCount()
public int getCount() {
	return this.jdbcTemplate.queryForInt("select count(*) from users");
}
```


### 3.6.3 queryForObject()

* `get()` 메서드를 변환하려면 `ResultSet`의 결과를 User 오브젝트를 만들어 프로퍼티에 넣어줘야 함.

* `RowMapper` 콜백
<br>`ResultSet`의 row 하나를 매핑하기 위해 사용되며 여러 번 호출될 수 있음.

* `ResultSetExtractor`는 `ResultSet`을 한 번 전달받아 알아서 추출 작업을 모두 진행하고 최종 결과만 리턴

* `queryForObject()` 메서드
  * 1번째 파라미터: `PreparedStatement`를 만들기 위한 SQL
  * 2번째 파라미터: 1번에 바인딩할 값들

```java
// 리스트 3-51 queryForObject()와 RowMapper를 적용한 get() 메서드
public User get(String id) {
	return this.jdbcTemplate.queryForObject("select * from users where id = ?",
		new Object[] {id}, // SQL에 바인딩할 파라미터 값. 가변인자 대신 배열을 사용
		new RowMapper<User>() { // ResultSet 한 row의 결과를 오브젝트에 매핑해주는 RowMapper 콜백
			public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				User user = new User();
				user.setId(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				return user;
			}
		});
}
```

* `queryForObject()`는 SQL을 실행하면 1개의 row만 얻을 것이라고 기대함.
<br>`ResultSet`의 `next()`를 실행해서 첫 번째 row로 이동시킨 후에 `RowMapper` 콜백을 호출

* `queryForObject()` 이용 시 조회 결과가 없는 예외상황은 어떻게 처리할까? **필요없음**
  * `queryForObject()`는 SQL을 실행해서 받은 row의 개수가 하나가 아니라면 예외를 던짐.
  <br>-> `EmptyResultDataAccessException`


### 3.6.4 query()

* 여러 row를 테이블에서 가져올 때는 오브젝트가 아닌 오브젝트의 컬렉션으로 만듬.

```java
// 리스트 3-52 getAll()에 대한 테스트
@Test
public void getAll()  {
	dao.deleteAll();
	
	List<User> users0 = dao.getAll();
	assertThat(users0.size(), is(0));
		
	dao.add(user1); // Id: gyumee
	List<User> users1 = dao.getAll();
	assertThat(users1.size(), is(1));
	checkSameUser(user1, users1.get(0));
		
	dao.add(user2); // Id: leegw700
	List<User> users2 = dao.getAll();
	assertThat(users2.size(), is(2));
	checkSameUser(user1, users2.get(0));  
	checkSameUser(user2, users2.get(1));
		
	dao.add(user3); // Id: bumjin
	List<User> users3 = dao.getAll();
	assertThat(users3.size(), is(3));
	checkSameUser(user3, users3.get(0));  
	checkSameUser(user1, users3.get(1));  
	checkSameUser(user2, users3.get(2));  
}

// User 오브젝트의 내용을 비교하는 검증 코드. 반복 사용을 위해 분리.
private void checkSameUser(User user1, User user2) {
	assertThat(user1.getId(), is(user2.getId()));
	assertThat(user1.getName(), is(user2.getName()));
	assertThat(user1.getPassword(), is(user2.getPassword()));
}
```

* `query()`의 리턴 타입은 `List<T>`
<br>`query()`는 제네릭 메서드로 타입은 파라미터로 넘기는 `RowMapper<T>` 콜백 오브젝트에서 결정됨.

```java
// 리스트 3-53 getAll() 메서드
public List<User> getAll() {
	// query() 템플릿은 SQL을 실행해서 얻은 ResultSet의 모든 row를 열람하면서 row마다 RowMapper 콜백을 호출
	return this.jdbcTemplate.query("select * from users order by id",
		// RowMapper는 현재 row의 내용을 User 타입 오브젝트에 매핑해서 돌려줌
		new RowMapper<User>() { // RowMapper<User>: 리턴 타입이 User로 결정됨.
			public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				// 현재 row의 내용을 User 타입 오브젝트에 매핑해서 돌려줌.
				User user = new User();
				user.setId(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				return user;
			}
		});
}
```
