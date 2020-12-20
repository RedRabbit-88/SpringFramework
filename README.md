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

* **개방 폐쇄 원칙 (OCP: Open-Closed Principle)**
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
  
  
### 1.4 제어의 역전 (IoC - Inversion of Control)

### 1.4.1 오브젝트 팩토리

* 팩토리 (factory)
<br>객체의 생성 방법을 결정하고 그렇게 만들어진 오브젝트를 돌려주는 것
<br>추상 팩토리 패턴, 팩토리 메서드 패턴과는 다르니 혼동하지 말 것!

```java
// 리스트 1-14 UserDao의 생성 책임을 맡은 팩토리 클래스
public class UserDaoFactory {
  // UserDao 타입의 오브젝트를 어떻게 만들고 준비시킬지 결정
  public UserDao userDao() {
    ConnectionMaker connectionMaker = new DConnectionMaker();
    UserDao dao = new UserDao(connectionMaker);
    return dao;
  }
}

// UserDaoTest는 UserDao가 어떻게 만들어지고 초기화되는지 신경쓰지 않고
// UserDaoFactory로부터 UserDao 오브젝트를 받아서 사용
public class UserDaoTest {
  public static void main(String[] args) throws ClassNotFoundException, SQLException {
    UserDao dao = new UserDaoFactory().userDao();
    ...
  }
}
```

* UserDao는 변경이 필요 없이 연결방식을 변경 및 확장할 수 있다.

* 애플리케이션의 컴포넌트 역할을 하는 오브젝트와, 애플리케이션의 구조를 결정하는 오브젝트를 분리함.


### 1.4.2 오브젝트 팩토리의 활용

```java
public class UserDaoFactory {
  // UserDao 타입의 오브젝트를 어떻게 만들고 준비시킬지 결정
  public UserDao userDao() {
    return new UserDao(connectionMaker());
  }
  
  public AccountDao accountDao() {
    return new AccountDao(connectionMaker());
  }
  
  // 분리해서 중복을 제거한 ConnectionMaker 타입 오브젝트 생성 코드
  public ConnectionMaker connectionMaker() {
    return new DConnectionMaker();
  }
}
```


### 1.4.3 제어권의 이전을 통한 제어관계 역전

* 일반적인 프로그램의 흐름
  * main() 메서드에서 다음에 사용할 오브젝트 결정
  * 결정한 오브젝트를 생성하고, 만들어진 오브젝트에 있는 메서드를 호출
  * 오브젝트 메서드 안에서 다음에 사용할 것을 결정하고 호출
  * 모든 종류의 작업을 **사용하는 쪽에서 제어하는 구조**

* 제어의 역전
  * 프로그램의 제어 흐름 구조가 뒤바뀌는 것
  * 프로그램의 시작을 담당하는 main() 같은 엔트리 포인트를 제외하면
  <br>위임 받은 제어 권한을 갖는 특별한 오브젝트에 의해 결정되고 생성됨.

* 서블릿의 경우 서블릿에 대한 제어 권한을 가진 컨테이너가 적절한 시점에
<br>서블릿 클래스의 오브젝트를 만들고 그 안의 메서드를 호출

* 추상 UserDao를 상속한 서브클래스는 getConnection()을 구현하지만 언제 어떻게 사용될지 자신은 모름.
<br>제어권을 상위 템플릿 메서드에 넘기고 자신은 필요할 때 호출되어 사용됨.

* 프레임워크는 보통 프레임워크 위에 개발한 클래스를 등록해두고,
<br>프레임워크가 흐름을 주도하는 중에 개발자가 만든 애플리케이션 코드를 사용하도록 만드는 방식

* 프레임워크에는 분명한 베어의 역전 개념이 적용되어 있어야 한다.


### 1.5 스프링의 IoC

### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC

* 스프링의 핵심: **애플리케이션 컨텍스트 (Application Context) = 빈 팩토리 (Bean Factory)**

* 빈 (Bean)
  * 스피링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트
  * 자바빈 또는 엔터프라이즈 자바빈에서 말하는 빈과 비슷한 오브젝트 단위의 애플리케이션 컴포넌트
  * 스프링 컨테이너가 생성과 관계설정, 사용 등을 제어해주는 **제어의 역전이 적용된 오브젝트**

* 빈 팩토리와 애플리케이션 컨텍스트의 차이
  * 빈 팩토리: 빈을 생성하고 관계를 설정하는 **IoC의 기본 기능에 초점**
  * 애플리케이션 컨텍스트: 애플리케이션 전반의 모든 구성요소의 제어작업을 담당하는 **IoC 엔진**
  <br>-> 설정정보를 담고 있는 무엇인가를 가져와 이를 활용하는 범용적인 IoC 엔진

* `@Configuration`
<br>스프링이 빈 팩토리를 위한 오브젝트 설정을 담당하는 클래스로 인식할 수 있게 하는 애노테이션

* `@Bean`
<br>스프링이 오브젝트를 만들어주는 메서드를 인식할 수 있게 하는 애노테이션

```java
// 리스트 1-18 스프링 빈 팩토리가 사용할 설정정보를 담은 DaoFactory 클래스
package springbook.user.dao;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// 애플리케이션 컨텍스트 또는 빈 팩토리가 사용할 설정정보라는 표시
@Configuration
public class DaoFactory {
	// 오브젝트 생성을 담당하는 IoC용 메서드라는 표시
	@Bean
	public UserDao userDao() {
		UserDao dao = new UserDao(connectionMaker());
		return dao;
	}

	@Bean
	public ConnectionMaker connectionMaker() {
		ConnectionMaker connectionMaker = new DConnectionMaker();
		return connectionMaker;
	}
}

// 리스트 1-19 애플리케이션 컨텍스트를 적용한 UserDaoTest
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		// 애플리케이션 컨텍스트는 ApplicationContext 타입의 오브젝트다.
		ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
		
		// getBean은 ApplicationContext가 관리하는 오브젝트를 요청하는 메서드
		// "userDao": ApplicationContext에 등록된 빈의 이름
		UserDao dao = context.getBean("userDao", UserDao.class);
		...
	}
}
```

* `getBean()`은 기본적으로 Object 타입으로 리턴해서 매번 캐스팅을 해줘야 하는 부담 존재.
<br>제네릭 메서드 방식을 적용해 두 번째 파라미터에 리턴 타입을 주면 캐스팅 불필요.


### 1.5.2 애플리케이션 컨텍스트에 동작방식

* 오브젝트 팩토리(UserDaoFactory)에 대응되는 것이 스프링의 애플리케이션 컨텍스트(= IoC 컨테이너 / 스프링 컨테이너)

* 애플리케이션 컨텍스트는 애플리케이션에서 IoC를 적용해서 관리할 모든 오브젝트에 대한 생성과 관계설정을 담당

* `@Configuration`이 붙은 DaoFactory는 애플리케이션 컨텍스트가 활용하는 IoC 설정정보

* getBean() 메서드 작동방식
  * 클라이언트가 애플리케이션 컨텍스트의 getBean() 메서드를 호출
  * 애플리케이션 컨텍스트의 빈 목록에서 요청한 이름이 있는지 확인
  * 요청한 이름이 있다면 빈을 생성하는 메서드를 호출해서 오브젝트 생성
  * 생성한 오브젝트를 클라이언트에 리턴

* 애플리케이션 컨텍스트를 사용했을 때의 장점
  * 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다.
  * 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다.
  * 애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다.


### 1.5.3 스프링 IoC의 용어 정리

* **빈 (Bean)**
  * 스프링이 IoC 방식으로 관리하는 오브젝트 (= managed object)
  * 스프링이 직접 그 생성과 제어를 담당하는 오브젝트만을 빈이라 말한다.

* **빈 팩토리 (Bean Factory)**
  * 스프링의 IoC를 담당하는 핵심 컨테이너
  * 빈을 등록, 생성, 조회하고 돌려주고, 그 외에 부가적인 빈을 관리하는 기능을 담당

* **애플리케이션 컨텍스트 (Application Context)**
  * 빈 팩토리를 확장한 IoC 컨테이너
  * 스프링이 제공하는 애플리케이션 지원 기능을 모두 포함
  * ApplicationContext는 BeanFactory를 상속함

* **설정정보/메타정보 (Configuration metadata)**
  * 애플리케이션 컨텍스트 또는 빈 팩토리가 IOC를 적용하기 위해 사용하는 메타정보
  * IoC 컨테이너에 의해 관리되는 애플리케이션 오브젝트를 생성하고 구성할 때 사용

* **컨테이너 또는 IoC 컨테이너 (Container)**
  * 애플리케이션 컨텍스트나 빈 팩토리를 의미

* **스프링 프레임워크**
  * IoC 컨테이너, 애플리케이션 컨텍스트를 포함해서 스프링이 제공하는 모든 기능을 통틀어 말할 때 사용


### 1.6 싱글톤 레지스트리와 오브젝트 스코프


