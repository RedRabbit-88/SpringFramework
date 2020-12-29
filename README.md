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
    if (e.getErrorCOde() == MysqlErrorNumbers.ER_DUP_ENTRY)
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

