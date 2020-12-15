# 스프링 프레임워크 학습 (토비의 스프링 3.1)

## 1장 오브젝트와 의존관계

* 스프링은 자바를 기반으로 한 기술 -> 객체지향 프로그래밍을 지향함

* 스프링이 가장 관심을 두는 대상? 오브젝트(객체)

* DAO (Data Access Object)
<br>DB를 사용해 데이터를 조회하거나 조작하는 기능을 전담하도록 만든 오브젝트

* 자바빈 (JavaBean)
  * 원래는 비주얼 툴에서 조작 가능한 컴포넌트를 뜻함.
  * 현재는 **디폴트 생성자, 프로퍼티**를 이용해 만들어진 오브젝트를 뜻함.
  <br>프로퍼티: 자바빈이 노출하는 이름을 가진 속성

* JDBC를 이용하는 작업의 일반적인 순서
  * DB 연결을 위한 Connection을 가져옴
  * SQL을 담은 Statement 혹은 PreparedStatement를 생성해서 실행
  * 조회의 경우 SQL의 실행 결과를 ResultSet으로 받아서 정보 저장용 오브젝트에 저장
  * 작업 중 생성된 리소스는 작업을 마친 후 닫아줌.
  <br>ex) Connection, Statement, ResultSet
  * JDBC API가 만들어내는 예외를 잡아서 직접 처리하거나 throws를 이용하여 메서드 밖으로 던짐.
  
### 1.2 DAO의 분리
  
#### 1.2.1 관심사의 분리
  
* 개발자가 객체를 설계할 때 가장 염두에 두어야 하는 사항
<br>-> 미래에 어떻게 대비할 것인가?
<br>-> 변경이 일어날 때 작업을 최소화하고 문제를 일으키지 않는 방법? **분리와 확장**
  
* 모든 변경과 발전은 한 번에 한 가지에 집중해서 일어난다.
  
* **관심사의 분리**
<br>관심이 같은 것끼리는 모으고, 관심이 다른 것은 분리시킨다.
<br>-> **최대한 모듈화를 진행하라!**

* **메서드 추출(extract method) 기법**
<br>공통의 기능을 담당하는 메서드로 중복된 코드를 뽑아내는 것.

* **리팩토링**
<br>기존의 코드를 외부의 동작방식에는 변화 없이 내부 구조만 변경해서 재구성하는 작업 또는 기술

```java
public void add(User user) throws ClassNotFoundException, SQLException {
  Connection c = getConnection();
  ...
}

public User get(String id) throws ClassNotFoundException, SQLException {
  Connection c = getConnection();
  ...
}

// 중복 사용되는 getConnection을 분리시킴.
// 향후 Connection 정보가 변경되도 이 메서드만 수정하면 됨!
private Connection getConnection() throws ClassNotFoundException, SQLException {
  Class.forName("com.mysql.jdbc.Driver");
  Connection c = DriverManager.getConnection(
    "jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring", "book");
  return c;
}
```
