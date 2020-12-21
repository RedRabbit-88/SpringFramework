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


### 1.6.1 싱글톤 레지스트리오서의 애플리케이션 컨텍스트

* 애플리케이션 컨텍스트는 싱글톤을 저장하고 관리하는 싱글톤 레지스트리(Singleton Registry)다.

* 스피링은 왜 싱글톤으로 빈을 만들까?
<br>스프링이 주로 적용되는 대상이 자바 엔터프라이즈 기술을 사용하는 서버환경이기 때문.
<br>클라이언트에서 다수의 요청이 들어올 때 싱글톤이 아니라면 어마어마한 부하가 걸림.

* 싱글톤 패턴의 한계
  * private 생성자를 갖고 있기 때문에 상속할 수 없다.
  * 싱글톤은 테스트하기가 힘들다.
  * 서버환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못한다.
  * 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다.

* 싱글톤 레지스트리
  * 스프링이 제공하는 싱글톤 형태의 오브젝트를 만들고 관리하는 기능
  * 평범한 자바 클래스를 싱글톤으로 활용 가능
  * 싱글톤 방식으로 사용된 애플리케이션 클래스라도 public 생성자를 가질 수 있다.
  * 싱글톤 패턴과 달리 스프링이 지지하는 객체지향 설계 방식과 원칙, 디자인 패턴을 적용하는데 제약 없음.


### 1.6.2 싱글톤과 오브젝트의 상태

* 싱글톤이 멀티스레드 환경에서 서비스 형태의 오브젝트로 사용되는 경우에는
<br>상태 정보를 내부에 갖고 있지 않음 무상태(stateless) 방식으로 반들어져야 한다.

* 다중 사용자의 요청을 한꺼번에 처리하는 스레드들이 동시에 싱글톤 오브젝트의 인스턴스 변수를 수정하는 건 매우 위험

* 자신이 사용하는 다른 싱글톤 빈을 저장하려는 용도라면 인스턴스 변수를 사용해도 좋다.
<br>그 외의 경우에는 사용 금지!!


### 1.6.3 스프링 빈의 스코프

* 빈의 스코프(Scope): 빈이 생성되고, 존재하고, 적용되는 범위

* 스프링 빈의 기본 스코프는 싱글톤
<br>컨테이너 내에 한 개의 오브젝트만 만들어짐. 강제로 제거하지 않는 한 스프링 컨테이너가 존재하는 동안 유지됨.


### 1.7 의존관계 주입(DI: Dependency Injection)


### 1.7.1 제어의 역전(IoC)과 의존관계 주입

* IoC라는 용어만으로는 스프링이 제공하는 IoC 방식의 핵심을 정확히 표현하지 못함.
<br>따라서 의존관계 주입이라는 좀 더 의도가 명확히 드러나는 이름을 사용


### 1.7.2 런타임 의존관계 설정

* 의존관계란? 2개의 클래스 또는 모듈이 있을 때 누가 누구에게 의존하는 관계라는 것을 정의

* 의존관계에는 방향성이 있다.
  * B가 A에 의존하고 있다면 A가 변경되었을 때 B는 영향을 받는다.
  * 반대로 A는 B가 변경되어도 영향을 받지 않는다.

* 의존 오브젝트 (Dependent Object)
  * 런타임 시에 의존관계를 맺는 대상, 즉 실제 사용대상인 오브젝트

* 의존관계 주입
<br>구체적인 의존 오브젝트와 그것을 사용할 주체, 보통 클라이언트라고 부르는 오브젝트를 런타임 시에 연결해주는 작업
  * 클래스 모델이나 코드에는 런타임 시점의 의존관계가 드러나지 않는다.
  <br>-> 인터페이스에만 의존하고 있어야 함.
  * 런타임 시점의 의존관계는 컨테이너나 팩토리 같은 제 3의 존재가 결정
  * 의존관계는 사용할 오브젝트에 대한 레퍼런스를 외부에서 제공(주입)해줌으로써 만들어짐.

* 의존관계 주입의 핵심
<br>설계 시점에는 알지 못했던 두 오브젝트의 관계를 맺도록 도와주는 제 3의 존재가 있다.
<br>ex) 애플리케이션 컨텍스트, 빈 팩토리, IoC 컨테이너

* DaoFactory는 두 오브젝트 사이의 런타임 의존관계를 설정해주는 의존관계 주입 작업을 주도하는 존재.
<br>-> DI 컨테이너

* 자바에서 오브젝트에 무엇인가를 넣어주려면?
  * 메서드를 실행하면서 파라미터로 오브젝트의 레퍼런스를 전달해주는 방법 뿐.
  * 가장 손쉽게 파라미터 전달이 가능한 메서드는 생성자.

```java
// 리스트 1-25 의존관계 주입을 위한 코드
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	// 실제 주입되는 건 DConnectionMaker와 같은 오브젝트
	public UserDao(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
}
```

* DI는 자신이 사용할 오브젝트에 대한 선택과 생성 제어권을 외부로 넘기고 자신은 수동적으로 주입받은 오브젝트를 사용함.


### 1.7.3 의존관계 검색과 주입

* 의존관계 검색 (Dependency Lookup)
  * 의존관계를 맺을 때 의존관계를 검색해서 적용하기 때문에 검색으로도 불림.
  * 물론 자신이 어떤 클래스의 오브젝트를 이용할지는 결정하지 않음.
  ex) 애플리케이션 컨텍스트의 `getBean()` 메서드

* 의존관계 검색 방식에서 검색하는 오브젝트는 의존관계 주입과 달리 자신이 스프링의 빈일 필요가 없다.

* 의존관계 주입에서는 UserDao와 ConnectionMaker 사이에 DI가 적용되려면 UserDao도 반드시 컨테이너가 만드는 빈 오브젝트여야 함.
<br>DI를 원하는 오브젝트는 먼저 자기 자신이 컨테이너가 관리하는 빈이 되어야 한다.

* 주입받는 메서드 파라미터가 이미 특정 클래스 타입으로 고정되어 있다면 DI가 일어날 수 없다.
<br>DI를 이용하려면 메서드 파라미터를 인터페이스로 이용해야 한다.


### 1.7.4 의존관계 주입의 응용

* 의존관계 주입의 장점
  * 코드에는 런타임 클래스에 대한 의존관계가 나타나지 않음
  * 인터페이스를 통해 결합도가 낮은 코드를 생성
  * 다른 책임을 가진 사용 의존관계에 있는 대상이 바뀌거나 변경되더라도 자신은 영향을 받지 않음
  * 변경을 통한 다양한 확장 방법에는 자유롭다.

* 기능 구현의 교환
  * 데이터베이스 연결을 변경 시 관련된 연결 소스 전체를 바꾸지 않고 빈 팩토리만 수정해서 해결 가능

```java
// 리스트 1-28 개발용 ConnectionMaker 생성 코드
@Bean
public ConnectionMaker connectionMaker() {
	return new LocalDBConnectionMaker();
}

// 리스트 1-29 운영용 ConnectionMaker 생성 코드
@Bean
public ConnectionMaker connectionMaker() {
	return new ProductionDBConnectionMaker();
}
```

* 부가기능 추가
  * 부가 기능을 추가할 때 인터페이스를 구현한 클래스에 추가함으로써 기존 소스 수정을 최소화할 수 있음

```java
// 리스트 1-30 연결횟수 카운팅 기능이 있는 클래스
// CountingConnectionMaker의 오브젝트가 DI 받을 오브젝트 역시 ConnectionMaker 인터페이스를 구현한 오브젝트다.
public class CountingConnectionMaker implements ConnectionMaker {
	int counter = 0;
	private ConnectionMaker realConnectionMaker;
	
	public CountingConnectionMaker(ConnectionMaker realConnectionMaker) {
		this.realConnectionMaker = realConnectionMaker;
	}
	
	public Connection makeConnection() throws ClassNotFoundException, SQLException {
		this.counter++;
		return realConnectionMaker.makeConnection();
	}
	
	public int getCounter() {
		return this.counter;
	}
}

// 리스트 1-31 CountingConnectionMaker 의존관께가 추가된 DI 설정용 클래스
@Configuration
public class CountingDaoFactory {
	@Bean
	public UserDao userDao() {
		// 모든 DAO는 여전히 connectionMaker()에서 만들어지는 오브젝트를 DI 받는다.
		return new UserDao(connectionMaker());
	}
	
	@Bean
	public ConnectionMaker connectionMaker() {
		return new CountingConnectionMaker(realConnectionMaker());
	}
	
	@Bean
	public ConnectionMaker realConnectionMaker() {
		return new DConnectionMaker();
	}
}

// 리스트 1-32 CountingConnectionMaker에 대한 테스트 클래스
public class UserDaoConnectionCountingTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ApplicationContext context = new AnnotationConfigApplicationContext(CountingDaoFactory.class);
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		/*
		 * DAO 사용 코드
		 */
		// DL(의존관계 검색)을 사용하면 이름을 이용해 어떤 빈이든 가져올 수 있다.
		CountingConnectionMaker ccm = context.getBean("connectionMaker", CountingConnectionMaker.class);
		System.out.println("Connection counter : " + ccm.getCounter());
	}
}
```


### 1.7.5 메서드를 이용한 의존관계 주입

* 생성자가 아닌 일반 메서드를 이용해 의존 오브젝트와의 관계를 주입하는 방식 2가지
  * 수정자 메서드를 이용한 주입 (Setter)
  * 일반 메서드를 이용한 주입: 여러 개의 파라미터를 받을 수 있지만 개수가 많아지고 비슷한 타입이 많으면 실수할 수 있음.
  <br>-> 스프링은 전통적으로 수정자 메서드를 가장 많이 사용해 옴.

* 수정자 메서드 DI를 사용할 때는 메서드의 이름을 잘 결정하는게 중요.

```java
// 리스트 1-33 수정자 메서드 DI 방식을 사용한 UserDao
public class UserDao {
	private ConnectionMaker connectionMaker;
	
	public void setConnectionMaker(ConnectionMaker connectionMaker) {
		this.connectionMaker = connectionMaker;
	}
	...
}

// 리스트 1-34 수정자 메서드 DI를 사용하는 팩토리 메서드
@Bean
public UserDao userDao() {
	UserDao userDao = new UserDao();
	userDao.setConnectionMaker(connectionMaker());
	return userDao;
}
```


### 1.8 XML을 이용한 설정


### 1.8.1 XML 설정

* 스프링의 애플리케이션 컨텍스트는 XML에 담긴 DI 정보를 활용할 수 있다.

* DI 정보가 담긴 XML 파일은 `<beans>`를 루트 엘리먼트로 사용한다.
  * `@Configuration` = `<beans>`
  * `@Bean` = `<bean>`

* 하나의 `@Bean` 메서드를 통해 얻을 수 있는 빈의 DI 정보
  * 빈의 이름: `@Bean` 메서드 이름
  * 빈의 클래스: 빈 오브젝트를 어떤 클래스를 이용해서 만들지를 정의
  * 빈의 의존 오브젝트: 빈의 생성자나 수정자 메서드를 통해 의존 오브젝트를 넣어준다.

* connectionMaker() 전환

||자바 코드 설정정보|XML 설정정보|
|--|--|--|
|빈 설정파일|`@Configuration`|`<beans>`|
|빈의 이름|`@Bean methodName()`|`<bean id="methodName"`|
|빈의 클래스|`return new BeanClass();`|`class="a.b.c...BeanClass">`|

* class 애트리뷰트에 넣을 클래스 이름은 패키지까지 모두 포함해야 함.

```java
// 리스트 1-35 connectionMaker() 메서드의 <bean> 태그 전환
@Bean
public ConnectionMaker connectionMaker() {
	return new DConnectionMaker();
}

<bean id="connectionMaker" class="springbook...DConnectionMaker"/>
```

* userDao() 전환
  * 수정자 메서드는 프로퍼티(property)가 된다.
  * **name**은 **프로퍼티의 이름**. 프로퍼티 이름으로 수정자 메서드를 알 수 있음.
  * **ref**는 수정자 메서드를 통해 **주입해줄 오브젝트의 빈 이름**

```java
// 리스트 1-36 userDao 빈 설정
userDao.setConnectionMaker(connectionMaker());
<property name="connectionMaker" ref="connectionMaker" />

<bean id="userDao" class="springbook.dao.UserDao">
	<property name="connectionMaker" ref="connectionMaker" />
</bean>

// 리스트 1-38 빈의 이름과 참조 ref의 변경
<beans>
	<bean id="myConnectionMaker" class="springbook.user.dao.DConnectionMaker" />
	
	<bean id="userDao" class="springbook.user.dao.UserDao">
		<property name="connectionMaker" ref="myConnectionMaker" />
	</bean>
</beans>

// 리스트 1-39 같은 인터페이스 타입의 빈을 여러 개 정의한 경우
<beans>
	<bean id="localDBConnectionMaker" class="springbook.user.dao.LocalDBConnectionMaker" />
	<bean id="testDBConnectionMaker" class="springbook.user.dao.TestDBConnectionMaker" />
	<bean id="productionDBConnectionMaker" class="springbook.user.dao.ProductionDBConnectionMaker" />
	
	<bean id="userDao" class="springbook.user.dao.UserDao">
		<property name="connectionMaker" ref="localDBConnectionMaker" />
	</bean>
</beans>
```

* XML 문서의 구조를 정의하는 방법은 DTD(Document Type Definition)와 스키마(schema)가 존재
<br>특별한 이유가 없다면 DTD보다는 스키마를 사용하는 편이 좋음.


### 1.8.2 XML을 이용하는 애플리케이션 컨텍스트

* **GenericXmlApplicationContext**
  * XML에서 빈의 의존관계 정보를 이용하는 IoC/DI 작업에 사용됨.
  * 생성자 파라미터로 XML 파일의 클래스패스를 지정해서 사용.
  * 클래스패스뿐 아니라 다양한 소스로부터 설정파일을 읽어올 수 있음.

```java
// 리스트 1-40 XML 설정정보를 담은 applicationContext.xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
	<bean id="myConnectionMaker" class="springbook.user.dao.DConnectionMaker">
		<property name="driverClass" value="com.mysql.jdbc.Driver" />
		<property name="url" value="jdbc:mysql://localhost/springbook?characterEncoding=UTF-8" />
		<property name="username" value="spring" />
		<property name="password" value="book" />
	</bean>

	<bean id="userDao" class="springbook.user.dao.UserDao">
		<property name="connectionMaker" ref="myConnectionMaker" />
	</bean>
</beans>

// GenericXmlApplicationContext 예시
ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
```

* **ClassPathXmlApplicationContext**
  * XML 파일을 클래스패스에서 가져올 때 사용할 수 있는 편리한 기능이 추가된 버전

```java
// GenericXmlApplicationContext를 사용할 경우: 클래스패스를 전부 적어야 함.
ApplicationContext context = new GenericXmlApplicationContext("springbook/user/dao/daoContext.xml");

// ClassPathXmlApplicationContext를 사용할 경우: 2번째 파라미터의 패키지와 동일한 위치에 있는 것을 가져옴.
ApplicationContext context = new ClassPathXmlApplicationContext("daoContext.xml", UserDao.class);
```


### 1.8.3 DataSource 인터페이스로 변환

* 기존에 만든 ConnectionMaker 인터페이스의 기능을 모두 포함한 DataSource 인터페이스가 존재.

```java
// 리스트 1-42 DataSource를 사용하는 UserDao
public class UserDao {
	private DataSource dataSource;
	
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	public void add(User user) throws SQLException {
		Connection c = this.dataSource.getConnection();
		...
	}
}

// DataSource를 사용하도록 DaoFactory 변경
@Configuration
public class DaoFactory {
	@Bean
	public DataSource dataSource() {
		SimpleDriverDataSource dataSource = new SimpleDriverDataSource ();

		// DB 연결정보를 수정자 메서드를 통해 넣어준다.
		// 이런 방식으로 오브젝트 레벨에서 DB 연결방식을 변경할 수 있음.
		dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
		dataSource.setUrl("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8");
		dataSource.setUsername("spring");
		dataSource.setPassword("book");

		return dataSource;
	}

	@Bean
	public UserDao userDao() {
		UserDao userDao = new UserDao();
		userDao.setDataSource(dataSource());
		return userDao;
	}
}

```


### 1.8.4 프로퍼티 값의 주입

* value 애트리뷰트를 이용해서 레퍼런스가 아닌 단순 값을 주입할 수 있음.

```java
// 리스트 1-46 코드를 통한 DB 연결정보 주입
dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
dataSource.setUrl("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8");
dataSource.setUsername("spring");
dataSource.setPassword("book");

// 리스트 1-47 XML을 이용한 DB 연결정보 설정
// value 애트리뷰트에 들어가는 건 다른 빈의 이름이 아닌 수정자 메서드의 파라미터로 사용되는 String
<property name="driverClass" value="com.mysql.jdbc.Driver.class" />
<property name="url" value="jdbc:mysql://localhost/springbook?characterEncoding=UTF-8" />
<property name="username" value="spring" />
<property name="password" value="book" />
```

* 스프링이 프로퍼티의 value를 수정자 메서드의 파라미터 타입을 참고해서 적절한 형태로 변환해 줌.
<br>스프링은 value에 지정한 텍스트 값을 적절한 자바 타입으로 변환해준다.
