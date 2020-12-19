# 스프링 프레임워크 학습 (토비의 스프링 3.1)

## 1장 오브젝트와 의존관계

* 스프링은 자바를 기반으로 한 기술 -> 객체지향 프로그래밍을 지향함
<br>따라서 스프링이 가장 관심을 두는 대상은 오브젝트(객체)다.

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
  
### 1.2.1 관심사의 분리
  
* 개발자가 객체를 설계할 때 가장 염두에 두어야 하는 사항
<br>-> 미래에 어떻게 대비할 것인가?
<br>-> 변경이 일어날 때 작업을 최소화하고 문제를 일으키지 않는 방법? **분리와 확장**
  
* 모든 변경과 발전은 한 번에 한 가지에 집중해서 일어난다.
  
* **관심사의 분리**
<br>관심이 같은 것끼리는 모으고, 관심이 다른 것은 분리시킨다.
<br>-> **최대한 모듈화를 진행하라!**

### 1.2.2 커넥션 만들기의 추출

* **메서드 추출(extract method) 기법**
<br>공통의 기능을 담당하는 메서드로 중복된 코드를 뽑아내는 것.

* **리팩토링**
<br>기존의 코드를 외부의 동작방식에는 변화 없이 내부 구조만 변경해서 재구성하는 작업 또는 기술

```java
// 리스트 1-4 getConnection() 메서드를 추출해서 중복을 제거
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

### 1.2.3 DB 커넥션 만들기의 독립

```java
// 리스트 1-5 상속을 통한 확장이 제공되는 UserDao
public abstract class UserDao {
  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = getConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = getConnection();
    ...
  }

  // 구현 코드는 제거되고 추상 메서드로 변경됨.
  private abstract Connection getConnection() throws ClassNotFoundException, SQLException;
}

public class NUserDao extends UserDao {
  // 상속을 통해 확장된 getConnection() 메서드
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // N사 DB Connection 생성코드
  }
}

public class DUserDao extends UserDao {
  public Connection getConnection() throws ClassNotFoundException, SQLException {
    // D사 DB Connection 생성코드
  }
}
```

* **디자인 패턴**
<br>소프트웨어 설계 시 특정 상황에서 자주 만나는 문제를 해결하기 위해 사용할 수 있는 재사용 가능한 솔루션
  * 간단히 패턴 이름을 언급하는 것만으로 설계의도와 해결책을 함께 설명 가능
  * 주로 객체지향 설계에 관한 것이며 **클래스 상속**과 **오브젝트 합성**이 대표적임
  * 각 패턴을 제대로 적용하기 위해 패턴의 목적/의도를 파악하는게 중요

* **템플릿 메서드 패턴 (Template Method Pattern)**
<br>슈퍼클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메서드나 오버라이딩이 가능한
<br>protected 메서드 등으로 만든 뒤 서브 클래스에서 이런 메서드를 필요에 맞게 구현해서 사용하는 방법
  * 상속을 통해 슈퍼클래스의 기능을 확장할 때 사용하는 가장 대표적인 방법
  * 변하지 않는 기능은 슈퍼클래스에, 자주 변경 및 확장되는 기능은 서브클래스에 생성

* **팩토리 메서드 패턴 (Factory Method Pattern)**
<br>템플릿 메서드 패턴과 마찬가지로 상속을 통해 기능을 확장하는 패턴
<br>오브젝트 생성 방법을 슈퍼클래스의 기본 코드에서 독립시키는 방법
<br>**팩토리 메서드**: 서브클래스에서 오브젝트 생성 방법과 클래스를 결정할 수 있도록 미리 정의해둔 메서드
  * 슈퍼클래스 코드에서는 서브클래스에서 구현할 메서드를 호출해서 필요한 타입의 오브젝트(주로 인터페이스)를 가져와 사용


### 1.3 DAO의 확장

### 1.3.1 클래스의 분리


```java
// 리스트 1-6 독립된 SimpleConnectionMaker를 사용하게 만든 UserDao
public class UserDao {
  private SimpleConnectionMaker simpleConnectionMaker;
  
  public UserDao() {
    // 한 번만 만들어 인스턴스 변수에 저장하고 메소드에서 사용
    simpleConnectionMaker = new SimpleConnectionMaker();
  }
  
  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = simpleConnectionMaker.makeNewConnection();
    ...
  }
}

// 리스트 1-7 독립시킨 DB 연결 기능인 SimpleConnectionMaker
// 더 이상 상속할 필요가 없으니 추상클래스로 만들 필요 없음
public class SimpleConnectionMaker {
  public Connection makeNewConnection() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring", "book");
    
    return c;
  }
}
```

* 분리는 잘 됐으나 UserDao의 코드가 SimpleConnectionMaker에 종속되어 있어 상속을 사용했을 때처럼
<br>UserDao 코드의 수정 없이 DB 커넥션 생성 기능을 변경할 방법이 없다.
<br>만약 변경하려면 생성자의 아래 구문을 바꿔야 함.
```java
simpleConnectionMaker = new SimpleConnectionMaker();
```

### 1.3.2 인터페이스의 도입

* 1.3.1에서 발생하는 문제점들을 해결하기 위해 인터페이스를 도입

* 두 개의 클래스가 서로 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리를 만들어 줌.
<br>인터페이스는 기능만 정의되어 있고 구현 방법은 나타나 있지 않다.
<br>따라서, 인터페이스를 이용해서 자유로운 확장이 가능하다.

```java
// 리스트 1-8 ConnectionMaker 인터페이스
public interface ConnectionMaker {
  public abstract Connection makeConnection() throws ClassNotFoundException, SQLException;
}

// 리스트 1-9 ConnectionMaker 구현 클래스
public class DConnectionMaker implements ConnectionMaker {
  public Connection makeConnection() throws ClassNotFoundException, SQLException {
    Class.forName("com.mysql.jdbc.Driver");
    Connection c = DriverManager.getConnection(
      "jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring", "book");
    return c;
  }
}

// 리스트 1-10 ConnectionMaker 인터페이스를 이용하도록 개선한 UserDao
public class UserDao {
  private ConnectionMaker connectionMaker;
	
  // 하지만 생성자에 클래스 이름이 존재!
  // 완벽한 분리가 되지 않았다.
  public UserDao() {
    this.connectionMaker = new DConnectionMaker();
  }
}
```


### 1.3.3 관계설정 책임의 분리

* 1.3.2의 UserDao에는 어떤 ConnectionMaker 구현 클래스를 사용할지 결정하는 코드가 남아있다.
<br>인터페이스를 이용한 분리에도 **UserDao 변경 없이는 DB 커넥션 기능 확장이 자유롭지 않음.**
  * UserDao와 UserDao에서 사용한 ConnectionMaker의 특정 구현 클래스 사이의 관계설정 분리에 대한 관심

* UserDao의 클라이언트 오브젝트 (UserDaoTest)
<br>UserDao와 ConnectionMaker 구현 클래스의 관계를 결정해주는 기능을 분리해서 두기에 적절

* 외부에서 만든 오브젝트를 전달받으려면?
<br>-> 메서드 파라미터나 생성자 파라미터를 전달받으면 된다.

```java
// 리스트 1-12 관계설정 책미이 추가된 UserDao 클라이언트임 main() 메서드
public class UserDaoTest {
  public static void main(String[] args) throws ClassNotFoundException, SQLException {
    // UserDao가 사용한 ConnectionMaker 구현 클래스를 결정하고 오브젝트를 생성
    ConnectionMaker connectionMaker = new DConnectionMaker();
    
    // 1. UserDao 생성
    // 2. 사용한 ConnectionMaker 타입의 오브젝트 제공. 결국 두 오브젝트 사이의 의존관계 설정 효과
    UserDao dao = new UserDao(connectionMaker);
  }
}

public class UserDao {
  private ConnectionMaker connectionMaker;
	
  // UserDaoTest에서 전달받은 구현 클래스에 따라 기능이 설정됨.
  public UserDao(ConnectionMaker simpleConnectionMaker) {
    this.connectionMaker = simpleConnectionMaker;
  }
  
  public void add(User user) throws ClassNotFoundException, SQLException {
    Connection c = this.connectionMaker.makeConnection();
    ...
  }

  public User get(String id) throws ClassNotFoundException, SQLException {
    Connection c = this.connectionMaker.makeConnection();
    ...
  }
}
```


### 1.3.4 원칙과 패턴

* **개방 폐쇄 원칙 (OCP: Open-Closed Principle**
  * 클래스나 모듈은 **확장에는 열려 있어야 하고 변경에는 닫혀 있어야 한다.**
  * 깔끔한 설계를 위해 적용 가능한 객체지향 설계 원칙 중 하나

* 높은 응집도와 낮은 결합도가 중요
  * 높은 응집도: 변화가 일어날 때 모듈의 많은 부분이 바뀌는 경우
  * 낮은 결합도: 하나의 오브젝트가 변경이 일어날 때에 관계를 맺고 있는 다른 오브젝트에 영향이 적은 경우

* 전략 패턴 (Strategy Pattern)
<br>자신의 기능 맥락(Context)에서 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고,
<br>이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용할 수 있게 하는 디자인 패턴
<br>ex) 개선한 UserDaoTest-UserDao-ConnectionMaker 구조
  * *컨텍스트*를 사용하는 *클라이언트*는 컨텍스트가 사용할 *전략*을 컨텍스트의 생성자 등을 통해 제공
  <br>컨텍스트 - UserDao
  <br>클라이언트 - UserDaoTest
  <br>전략 - ConnectionMaker를 구현한 클래스
  
  
  ### 1.4 제어의 역전 (IoC)
  
  
