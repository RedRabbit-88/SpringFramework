# 스프링 프레임워크 학습 (토비의 스프링 3.1)

## 1장 오브젝트와 의존관계
https://github.com/RedRabbit-88/SpringFramework/wiki/1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84#1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84

## 2장 테스트

* 스프링이 개발자에게 제공하는 가장 중요한 가치?
<br>**객체지향 & 테스트**

* 변화하는 애플리케이션에 대응하는 방법
  * 확장과 변화를 고려한 **객체지향적 설계**와 그것을 효과적으로 담아낼 수 있는 **Ioc/DI와 같은 기술**
  * 만들어진 코드를 확신할 수 있게 해주고, 변화에 유연하게 대처할 수 있는 자신감을 주는 **테스트 기술**


### 2.1 UserDaoTest 다시 보기


### 2.1.1 테스트의 유용성

* 초기 난잡한 UserDao에서 점점 스프링을 적용했을 때 어떻게 동일한 기능을 수행함을 보장할 수 있는가?
<br>-> 테스트가 필요!

* 테스트란?
<br>예상하고 의도했던 대로 코드가 정확히 동작하는지를 확인해서, 만든 코드를 확신할 수 있게 해주는 작업


### 2.1.2 UserDaoTest의 특징

```java
// 리스트 2-1 main() 메서드로 작성된 테스트
// main() 메서드를 이용해 테스트를 수행. UserDao를 직접 호출해서 사용함.
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("whiteship");
		user.setName("백기선");
		user.setPassword("married");

		dao.add(user);
			
		System.out.println(user.getId() + " 등록 성공");
		
		User user2 = dao.get(user.getId());
		System.out.println(user2.getName());
		System.out.println(user2.getPassword());
			
		System.out.println(user2.getId() + " 조회 성공");
	}
}
```

* 웹을 통한 DAO 테스트 방법의 문제점
  * 웹 화면을 통해 값을 입력하고, 기능을 수행하고, 결과를 확인하는 방법은 가장 흔히 쓰이는 방법이지만
  <br>DAO에 대한 테스트로는 단점이 너무 많다.
  * 테스트를 하기 위해 너무 많은 소스를 구현해야 한다.
  * 만약 에러가 발생해도 DAO 기능에서의 에러 외에 다른 에러들이 발생할 확률이 높다.
  <br>-> **따라서, 작은 단위의 테스트를 추구해야 한다.**

* 작은 단위의 테스트 = 단위 테스트 (Unit Test)
  * 테스트하고자 하는 대상이 명확하다면 그 대상에만 집중해서 테스트하는 것이 바람직함.
  * 테스트의 관심이 다르다면 테스트 대상을 분리하고 집중할 필요가 있음.
  * 하나의 관심에 집중해서 효율적으로 테스트할 만한 범위로 테스트할 필요가 있음.

* 단위 테스트를 하는 이유?
  * 개발자가 설계하고 만든 코드가 원래 의도한 대로 동작하는지를 개발자 스스로 빨리 확인받기 위해서
  * 확인의 대상과 조건이 간단하고 명확할수록 좋다.

* 테스트는 자동으로 수행되도록 코드로 만들어지는 것이 중요
  * 테스트가 수작업으로 진행되보다는 코드로 만들어져 자동으로 수행되는 것이 좋다.
  * 애플리케이션을 구성하는 클래스 안에 테스트 코드를 포함시키기 보다 별도의 테스트 클래스를 구성하는 것이 좋음.
  * 장점? 자주 반복할 수 있다.

* 개발을 마친 후에 테스트를 진행하려고 하면 어렵다.
<br>-> 테스트를 염두에 두고 개발을 진행할 것!


### 2.1.3 UserDaoTest의 문제점

* UserDaoTest의 문제점
  * 수동 확인 작업의 번거로움
    * 테스트 수행은 코드에 의해 진행되지만, 결과는 사람이 확인해야해서 자동화된 테스트라 하기는 어렵다.
  * 실행 작업의 번거로움
    * 만약 DAO가 수백 개가 되고 그에 대한 main() 메서드도 그만큼 만들어진다면 수백 회의 main() 실행을 진행해야 함.


### 2.2 UserDaoTest 개선


### 2.2.1 테스트 검증의 자동화

* 테스트 결과의 분류
  * 테스트 성공
  * 테스트 실패
    * 테스트 진행 중 에러 발생으로 인한 실패 (테스트 에러)
    * 테스트 작업 중 에러는 발생하지 않았지만 결과값이 기대치와 다름 (테스트 실패)


### 2.2.2 테스트의 효율적인 수행과 결과 관리

* 테스트 지원 도구인 JUnit 자바 테스팅 프레임워크를 활용

* JUnit은 프레임워크
  * 개발자가 만든 클래스의 오브젝트를 생성하고 실행하는 일은 프레임워크에 의해 진행
  * main() 메서드나 오브젝트 생성 코드가 불필요

* **JUnit 프레임워크가 요구하는 조건 두 가지**
  * 메서드가 public으로 선언되어야 함.
  * 메서드에 `@Test` 애노테이션을 붙여줘야 함.

```java
// 리스트 2-5 JUnit을 적용한 UserDaoTest
import static org.hamcrest.CoreMatchers.is;
import static org.junit.Assert.assertThat;

public class UserDaoTest {
	// @Test: JUnit에게 테스트용 메서드임을 알려줌
	// 테스트용 메서드는 public으로 선언되어야 함.
	@Test 
	public void andAndGet() throws SQLException {
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("gyumee");
		user.setName("박성철");
		user.setPassword("springno1");

		dao.add(user);
			
		User user2 = dao.get(user.getId());
		
		// assertThat() 스태틱 메서드: if 조건문을 표현가능
		// 첫 번째 파라미터의 값을 뒤에 나오는 매처(Matcher) 조건으로 비교해서 일치하면 다음으로, 아니면 테스트 실패
		// is(): 매처의 일종으로 equals()로 비교하는 기능과 동일
		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
	}
	
	public static void main(String[] args) {
		// JUnit을 이용해 테스트를 실행
		JUnitCore.main("springbook.user.dao.UserDaoTest");
	}
}
```


### 2.3 개발자를 위한 테스팅 프레임워크 JUnit

* 스프링을 학습하고 제대로 활용하기 위해서 JUnit은 필수!
<br>스프링의 핵심 기능 중 하나인 스프링 테스트 모듈도 JUnit을 이용함.


### 2.3.1 JUnit 테스트 실행 방법
