# 스프링 프레임워크 학습 (토비의 스프링 3.1)


## 1장 오브젝트와 의존관계
https://github.com/RedRabbit-88/SpringFramework/wiki/1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84#1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84


## 2장 테스트
https://github.com/RedRabbit-88/SpringFramework/wiki/2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8#2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8


## 3장 템플릿
https://github.com/RedRabbit-88/SpringFramework/wiki/3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF#3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF


## 4장 예외


### 4.1 사라진 SQLException

```java
// JdbcTemplate 적용 전
public void deleteAll() throws SQLException {
  this.jdbcContext.executeSql("delete from users");
}

// JdbcTemplate 적용 후: throws SQLException이 사라짐
public void deleteAll() {
  this.jdbcTemplate.executeSql("delete from users");
}
```


### 4.1.1 초난감 예외처리

```java
// 리스트 4-1 초난감 예외처리 코드 1
// 예외가 발생했는데 아무런 처리도 하지 않는 건 정말 위험!
try {
  ...
} catch (SQLException e) { // 예외를 잡고 아무것도 하지 않음.
}

// 리스트 4-2 초난감 예외처리 코드 2
// 모니터링하지 않는 이상 발견하기 어려운 예외 처리
} catch (SQLException e) {
  System.out.println(e);
}

// 리스트 4-3 초난감 예외처리 코드 3
// 모니터링하지 않는 이상 발견하기 어려운 예외 처리
} catch (SQLException e) {
  e.printStackTrace();
}
```

* 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 통보되어야 함.

```java
// 리스트 4-5 초난감 예외처리 4
// 메서드 선언에 throws Exception을 기계적으로 붙이는 개발방법 -> 의미 있는 정보를 얻을 수 없음.
public void method1() throws Exception {
  method2();
  ...
}

public void method2() throws Exception {
  method3();
  ...
}

public void method3() throws Exception {
  ...
}
```


### 4.1.2 예외의 종류와 특징

* Error
  * `java.lang.Error` 클래스의 서브클래스들
  * **시스템에 뭔가 비정상적인 상황이 발생했을 경우에 사용**
  <br>-> 애플리케이션 코드에서 처리하려하지 말 것.
  * 시스템 레벨에서 특별한 작업을 하는게 아니라면 애플리케이션에서는 처리 불필요.
  <br>ex) OutOfMemoryError, ThreadDeath

* Exception과 체크 예외
  * `java.lang.Exception` 클래스와 서브클래스들
  * **개발자들이 만든 애플리케이션 코드의 작업 중에 예외상황이 발생했을 경우에 사용**
  * 체크 예외(checked exception)과 언체크 예외(unchecked exception)
    * 체크 예외: Exception 클래스의 서브클래스면서 RuntimeException 클래스를 상속하지 않은 것
    * 언체크 예외: Exception 클래스의 서브클래스면서 RuntimeException 클래스를 상속한 것
  * 일반적인 예외 = 체크 예외
  * 체크 예외가 발생할 수 있는 메서드를 사용할 경우 반드시 예외 처리 코드도 같이 작성해야 함.
  <br>ex) IOException, SQLException

* RuntimeException과 언체크/런타임 예외
  * `java.lang.RuntimeException` 클래스를 상속한 예외
  * **프로그램의 오류가 있을 때 발생하도록 의도된 것들**
  <br>ex) NullPointerException, IllegalArgumentException


### 4.1.3 예외처리 방법

* 예외 복구
  * **예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓는 방법**
  * 예외가 처리됐으면 비록 기능적으로는 사용자에게 예외상황으로 표시되도
  <br>애플리케이션은 정상적으로 설계된 흐름을 따라 진행되어야 함.

```java
// 리스트 4-6 재시도를 통해 예외를 복구하는 코드
int maxretry = MAX_RETRY;
while (maxretry -- > 0) {
  try {
  ... // 예외 발생 가능한 코드
  return;
  } catch (SomeException e) {
    // 로그 출력. 정해진 시간만큼 대기
  } finally {
    // 리소스 반납. 정리 작업
  }
}
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생
```

* 예외처리 회피
  * **예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버리는 방법**
  * `throws`문으로 선언해서 예외가 발생하면 알아서 던져지게 하거나
  <br>`catch`문으로 예외를 잡은 후에 로그를 남기고 다시 예외를 던지는 방법
  * `JdbcContext`나 `JdbcTemplate`이 사용하는 콜백 오브젝트는 `ResultSet`, `PreparedStatement`
  <br>등을 이용해서 작업하다 발생하는 SQLException을 자신이 처리하지 않고 템플릿으로 던짐.
  * 콜백 메서드는 모두 `throws SQLException`이 붙어있음.
  * 예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분영해야 함.

```java
// 리스트 4-7 예외처리 회피 1
public void add() throws SQLException {
  // JDBC API
}

// 리스트 4-8 예외처리 회피 2
public void add() throws SQLException {
  try {
    // JDBC API
  } catch (SQLException e) {
    // 로그 출력
    throw e;
  }
}
```

* 예외 전환 (exception translation)
  * 예외 회피와 비슷하게 예외를 복구해서 정상적인 상태로는 만들 수 없기 때문에 예외를 메서드 밖으로 던짐.
  * **발생한 예외를 그대로 던지는게 아닌 예외상황에 적절한 예외로 전환해서 던짐.**
  <br>-> 의미를 분명하게 해줄 수 있는 예외로 바꿔주기 위해서
  * 예외를 처리하기 쉽고 단순하게 만들기 위해 포장하는 방법도 존재
  <br>-> 예외처리를 강제하는 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우에 사용

```java
// 리스트 4-9 예외 전환 기능을 가진 DAO 메서드
public void add(User user) throws DuplicateUserIdException, SQLException {
  try {
    // JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
    // 그런 기능을 가진 다른 SQLException을 던지는 메서드를 호출하는 코드
  } catch (SQLException e) {
    // ErrorCode가 MySQL의 "Duplicate Entry(1062)"면 예외 전환
    if (e.getErrorCde() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw DuplicateUserIdException();
    else
      throw e; // 그 외의 경우는 SQLException 그대로
  }
}

// 리스트 4-12 예외 포장
try {
  OrderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
  Order order = orderHome.findByPrimaryKey(Integer id);
} catch (NamingException ne) {
  throw new EJBException(re); // EJBException: RuntimeException 클래스를 상속한 런타임 예외
} catch (SQLException se) {
  throw new EJBException(re);
} catch (RemoteException re) {
  throw new EJBException(re);
}
```


### 4.1.4 예외처리 전략

* 체크 예외는 복구할 가능성이 있는 예외적인 상황이기 때문에 자바는 이를 처리하는 catch블록이나 throws 선언을 강제하고 있음.

* 자바 엔터프라이즈 서버환경은 수많은 사용자가 동시에 요청을 보내고 각 요청이 독립적인 작업으로 취급됨.
<br>-> 하나의 요청을 처리하는 중에 예외가 발생하면 해당 작업만 중단

* 애플리케이션 차원에서 예외상황을 미리 파악하고, 예외가 발생하지 않도록 차단하는 게 좋음.

* 대응이 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 나음.

* 최근에 등장하는 표준 스텍 또는 오픈소스 프레임워크에서는
<br>API가 발생시키는 예외를 체크 예외 대신 언체크 예외로 정의하는 게 일반적.

```java
// 리스트 4-13 아이디 중복 시 사용하는 예외
public class DuplicateUserIdException extends RuntimeException {
  public DuplicateUserIdException(Throwable cause) {
    super(cause);
  }
}

// 리스트 4-14 예외처리 전략을 적용한 add()
public void add(User user) throws DuplicateUserIdException, SQLException {
  try {
    // JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
    // 그런 기능을 가진 다른 SQLException을 던지는 메서드를 호출하는 코드
  } catch (SQLException e) {
    if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
      throw new DuplicateUserIdException(e); // 예외 전환
    else
      throw new RuntimeException(e); // 예외 포장
  }
}
```

* 리스트 4-14
  * 중복 ID 상황 처리를 위해서는 DuplicateUserIdException을 사용
  * 그 외의 경우에는 RuntimeException을 통해 언체크 예외로 처리

* **애플리케이션 예외**
<br>시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고,
<br>반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외

* 예시: 사용자가 요청한 금액을 은행계좌에서 출금하는 기능을 가진 메서드
  * 방법1: 정상적인 출금처리를 했을 경우와 잔고 부족이 발생했을 경우에 각각 다른 종류의 리턴값을 돌려주는 방법
    * 리턴값으로 결과를 확인하고, 예외상황을 체크하면 불편
    * 예외상황에 대한 리턴 값을 명확하게 코드화하고 잘 관리하지 않으면 혼란 발생
    * 결과 값을 확인하는 조건문이 자주 등장
  * 방법2: 정상적인 흐름을 따르는 코드는 그대로 두고, 잔고 부족과 같은 예외 상황에서는 비즈니스적인 의미를 띈 예외를 던지는 방법
    * 사용하는 예외는 의도적으로 체크 예외로 생성

```java
// 리스트 4-15 애플리케이션 예외를 사용한 코드
try {
  BigDecimal balance = account.withdraw(amount);
  ...
  // 정상적인 처리 결과를 출력하도록 진행
} catch (InsufficientBalanceException e) { // 체크 예외
  // InsufficientBalanceException에 담긴 인출 가능한 잔고금액 정보를 가져옴
  BigDecimal availFunds = e.getAvailFunds();
  ...
  // 잔고 부족 안내 메시지를 준비하고 이를 출력하도록 진행
}
```


### 4.1.5 SQLException은 어떻게 됐나?

* SQLException은 복구 가능한 예외인가? 99% 복구 불가능
<br>-> 언체크/런타임 예외를 통한 예외처리 전략을 적용해야 함.

* JdbcTemplate 템플릿과 콜백 안에서 발생하는 모든 SQLException은 런타임 예외인 `DataAccessException`으로 포장해서 던져짐.


### 4.2 예외 전환

* 예외 전환의 목적 2가지
  * 런타임 예외로 포장해서 불필요한 catch/throws를 줄여주는 것
  * 로우레벨의 예외를 좀 더 의미 있고 추상화된 예외로 바꿔서 던져주는 것

* 스프링의 JdbcTemplate이 던지는 `DataAccessException`은 런타임 예외로 SQLException을 포장해주는 역할


### 4.2.1 JDBC의 한계

* JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API 형태로 정의해놓고,
<br>각 DB 업체가 JDBC 표준을 따라 만들어진 드라이버를 제공하게 해 줌.
  * DB마다 내부 구현은 다르지만 Connection, Statement, ResultSet 등의 표준 인터페이스를 통해 기능을 제공

* 첫 번째 문제: JDBC 코드에서 사용하는 SQL
  * 대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공
  * 작성된 비표준 SQL은 DAO에 들어가고, DAO는 특정 DB에 종속적인 코드가 됨.
  * 해결책
    * 호환 가능한 표준 SQL만 사용하는 방법 -> 말도 안됨!
    * DB별로 별도의 DAO를 만들거나 SQL을 외부에 독립시켜서 DB에 따라 변경해서 사용하는 방법

* 두 번째 문제: SQLException
  * DB마다 SQL만 다른 것이 아니라 에러의 종류와 원인도 제각각
  <br>-> JDBC는 데이터 처리 중에 발생한 다양한 예외를 SQLException에 담아버림.
  <br>`if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)` -> MySQL 전용코드. 다른 DB 호환 불가!
  * SQLException은 예외가 발생했을 떄의 DB 상태를 담은 SQL 상태정보를 부가적으로 제공
    * `getSQLState()` 메서드로 예외상황에 대한 상태정보를 가져올 수 있음.
    * 문제는 DB의 JDBC 드라이버에서 SQLException을 담을 상태코드를 정확하게 만들어주지 않음.


### 4.2.2 DB 에러 코드 매핑을 통한 전환

* SQL 상태 코드
  * JDBC 드라이버를 만들 때 들어가는 것
  * 같은 DB라고 하더라도 드라이버를 만들 떄마다 달라질 수 있음

* DB 에러 코드
  * DB에서 직접 제공
  * 버전이 올라가더라도 어느 정도 일관성이 유지됨.

```java
// 리스트 4-18 중복 키 예외의 전환
public void add() throws DuplicateUserIdException { // 애플리케이션 레벨의 체크 예외
  try {
    // jdbcTemplate을 이용해 User를 add하는 코드
  } catch (DuplicateKeyException e) {
    // 로그를 남기는 등의 필요한 작업
    throw new DuplicateUserIdException(e); // 예외 전환 시 원인이 되는 예외를 중첩하는 게 좋음.
  }
}
```


### 4.2.3 DAO 인터페이스와 DataAccessException 계층구조

* `DataAccessException`은 JDBC의 SQLException을 전환하는 용도로만 만들어진 게 아니라
<br>JDBC 외의 자바 데이터 액세스 기술에서 발생하는 예외에도 적용 가능.

* `DataAccessException`은 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어줌.

* DAO를 굳이 따로 만들어서 사용하는 이유?
  * 데이터 액세스 로직을 담은 코드를 성격이 다른 코드에서 분리해놓기 위해서
  * 전략 패턴을 적용해 구현 방법을 변경해서 사용할 수 있게 만들기 위해서

```java
public void add(User user) throws SQLException; // JDBC용. 데이터 액세스 기술을 바꾸면 사용 불가
public void add(User user) throws PersistentException; // JPA용
public void add(User user) throws HibernateException; // Hibernate용
public void add(User user) throws JdoException; // JDO용
```

* 인터페이스로 메서드 구현은 추상화했지만 구현 기술마다 던지는 예외가 다르기 떄문에 메서드의 선언이 달라진다!

* 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서
<br>`DataAccessException` 계층구조 안에 정리해놓음.

* `DataAccessException`
<br>자바의 주요 데이터 액세스 기술에서 발생할 수 있는 대부분의 예외를 추상화
  * `InvalidDataAccessResourceUsageException`
  <br>-> 데이터 액세스 기술을 부정확하게 사용했을 때
  * `ObjectOptimisticLockingFailureException`
  <br>-> 낙관적인 락킹(optimistic locking)이 발생한 경우

* 낙관적인 락킹
<br>같은 정보를 2명 이상의 사용자가 동시에 조회하고 순차적으로 업데이트할 때
<br>뒤늦게 업데이트한 것이 먼저 업데이트한 것을 덮어쓰지 않도록 막아주는 기능.

* JdbcTemplate과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들 경우
<br>-> 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있음.


### 4.2.4 기술에 독립적인 UserDao 만들기

* 확장성을 위해 UserDao 인터페이스 구분
  * UserDao 클래스에서 DAO의 기능을 사용하려는 클라이언트들이 필요한 것만 추출
  * `setDataSource()` 메서드는 UserDao 구현 방법에 따라 달라질 수 있으므로 추가하면 안 됨!

```java
// 리스트 4-20 UserDao 인터페이스
public interface UserDao {
  void add(User user);
  User get(String id);
  List<User> getAll();
  void teleAll();
  int getCount();
}

// 인터페이스를 구현하여 JDBC를 사용하는 UserDaoJdbc 생성
public class UserDaoJdbc implements UserDao {
  ...
}

// 리스트 4-21 빈 클래스 변경
<bean id="userDao" class="springbook.dao.UserDaoJdbc">
  <property name="dataSource" ref="dataSource" />
</bean>
```

* UserDaoTest의 UserDao 인스턴스 변수는 따로 변경할 필요 없음.
<br>-> UserDao는 UserDaoJdbc가 구현한 인터페이스이므로 아무 문제없음.

```java
// 리스트 4-22 DataAccessException에 대한 테스트
@Test(expected=DataAccessException.class)
public void duplicateKey() {
  dao.deleteAll();
  
  dao.add(user1);
  dao.add(user1);
}
```

* DataAccessException 활용 시 주의사항
  * DuplicateKeyException은 JDBC를 이용하는 경우에만 발생
  * JPA, 하이버네이트를 사용할 때는 다른 예외가 던져짐.
    * JDBC는 SQLException에 담긴 DB의 에러코드를 바로 해석
    * JPA 등은 각 기술이 재정의한 예외를 가져와 스프링이 최종적으로 DataAccessException으로 변환

```java
// 리스트 4-24 SQLException 전환 기능의 학습 테스트
@Test
public void sqlExceptionTranslate() {
  dao.deleteAll();
  
  try {
    dao.add(user1);
    dao.add(user1);
  } catch (DuplicateKeyException ex) {
    SQLException sqlEx = (SQLException)ex.getRootCause();
    // 코드를 이용한 SQLException의 전환
    SQLExceptionTranslator set = new SQLErrorCOdeSqlExceptionTranslator(this.dataSource);
    
    assertThat(set.translate(null, null, sqlEx), is(DuplicateKeyException.class));
  }
}
```


### 4.3 정리

* 예외를 잡아서 아무런 조치를 취하지 않거나 의미 없는 throws 선언을 남발하는 건 위험

* 예외는 복구하거나 예외처리 오브젝트로 의도적으로 전달하거나 적절한 예외로 전환해야 함.

* 2가지의 예외 전환 방법이 존재
  * 좀 더 의미 있는 예외로 변경
  * try/catch를 피하기 위해 런타임 예외로 포장

* 복구할 수 없는 예외는 가능한 한 빨리 런타임 예외로 전환하는 것이 바람직함.

* 애플리케이션의 로직을 담기 위한 예외는 체크 예외로 만든다.

* JDBC의 SQLException은 대부분 복구할 수 없는 예외이므로 런타임 예외로 포장

* SQLException의 에러 코드는 DB에 종속됨. DB에 독립적인 예외로 전환할 필요가 있음.

* 스프링의 DataAccessException을 통해 DB에 독립적으로 적용 가능한 추상화된 런타임 예외 계층을 제공

* DAO를 데이터 액세스 기술에서 독립시키려면 아래 방법 적용 필요
  * 인터페이스 도입
  * 런타임 예외 전환
  * 기술에 독립적인 추상화된 예외로 전환
