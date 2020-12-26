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
