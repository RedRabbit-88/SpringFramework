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
  * 변하는 부분을 메서드로 추출 -> 별로 유용하지 않다.
  * 템플릿 메서드 패턴을 적용 -> 상속을 통해 기능을 확장해서 사용

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

* deleteAll()의 컨텍스트
  * DB 커넥션 가져오기
  * PreparedStatement를 만들어줄 외부 기능 호출하기
  <br>-> 전략 패턴에서 말하는 전략
  * 전달받은 PreparedStatement 실행하기
  * 예외가 발생하면 이를 다시 메서드 밖으로 던지기
  * 모든 경우에 만들어진 PreparedStatement와 Connection을 적절히 닫아주기

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

```java
// 리스트 3-11 메서드로 분리한 try/catch/finally 컨텍스트 코드
// stmt: 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터
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

* 개선된 DAO 메메서드는 저략 패턴의 클라이언트로서 컨텍스트에 해당하는 `jdbcContextWithStatementStrategy()` 메서드에 적절한 전략,
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
<br>**스프링의 DI는 기본적으로 인터페이스를 사의에 두고 의존 클래스를 바꿔서 사용하도록 하는 게 목적**

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

