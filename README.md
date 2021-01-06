# 스프링 프레임워크 학습 (토비의 스프링 3.1)


## 1장 오브젝트와 의존관계
https://github.com/RedRabbit-88/SpringFramework/wiki/1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84#1%EC%9E%A5-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EC%9D%98%EC%A1%B4%EA%B4%80%EA%B3%84


## 2장 테스트
https://github.com/RedRabbit-88/SpringFramework/wiki/2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8#2%EC%9E%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8


## 3장 템플릿
https://github.com/RedRabbit-88/SpringFramework/wiki/3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF#3%EC%9E%A5-%ED%85%9C%ED%94%8C%EB%A6%BF


## 4장 예외
https://github.com/RedRabbit-88/SpringFramework/wiki/4%EC%9E%A5-%EC%98%88%EC%99%B8#4%EC%9E%A5-%EC%98%88%EC%99%B8


## 5장 서비스 추상화

https://github.com/RedRabbit-88/SpringFramework/wiki/5%EC%9E%A5-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%83%81%ED%99%94#5%EC%9E%A5-%EC%84%9C%EB%B9%84%EC%8A%A4-%EC%B6%94%EC%83%81%ED%99%94


## 6장 AOP


### 6.1 트랜잭션 코드의 분리

* 스피링이 제공하는 트랜잭션 인터페이스를 썼음에도 비즈니스 코드와 트랜잭션 코드가 같이 존재하는 게 문제!
  * 트랜잭션 경계설정 코드와 비즈니스 로직 코드 간에 서로 주고받는 정보가 없음.
  * 성격이 다를 뿐 아니라 서로 주고받는 것도 없는, 완벽하게 독립적인 코드


### 6.1.1 메서드 분리

```java
// 릿트 6-2 비즈니스 로직과 트랜잭션 경계설정의 분리
public void upgradeLevels() {
	TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
	try {
		// 트랜잭션 관리 기능과 비즈니스 로직 코드를 분리
		upgradeLevelsInternal();
		this.transactionManager.commit(status);
	} catch (RuntimeException e) {
		this.transactionManager.rollback(status);
		throw e;
	}
}

// 분리된 비즈니스 로직 코드
public void upgradeLevelsInternal() {
	List<User> users = userDao.getAll();
	for (User user : users) {
		if (canUpgradeLevel(user)) {
			upgradeLevel(user);
		}
	}
}
```


### 6.1.2 DI를 이용한 클래스의 분리

* 트랜잭션 코드를 어떻게든 UserService 밖으로 분리하면 UserService 클래스를 직접 사용하는 클라이언트는
<br>트랜잭션 기능이 분리된 UserService를 사용하게 됨.

* DI의 기본 아이디어는 실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간접으로 접근하는 것.

* Client(UserServiceTest) -> UserService(Interface) <- UserServiceImpl / UserServiceTx
  * UserService를 인터페이스로 만들고 이걸 구현한 UserServiceImpl/UserServiceTx를 사용
  * 트랜잭션 기능은 UserServiceTx으로 분리시키고 클라이언트에서는 UserServiceTx를 호출
  * UserServiceTx에서 UserServiceImpl의 비즈니스 로직 코드를 호출
```java
// 리스트 6-3 UserService 인터페이스
public interface UserService {
	void add(User user);
	void upgradeLevels();
}

// 리스트 6-4 트랜잭션 코드를 제거한 UserService 구현 클래스
public class UserServiceImpl implements UserService {
	...
	private UserDao userDao;
	private MailSender mailSender;
	
	public void upgradeLevels() {
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
	}
	...
}

// 리스트 6-6 위임 기능을 가지고 트랜잭션을 관리하는 UserServiceTx
public class UserServiceTx implements UserService {
	UserService userService;
	PlatformTransactionManager transactionManager;

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setUserService(UserService userService) {
		this.userService = userService;
	}

	public void add(User user) {
		this.userService.add(user);
	}

	public void upgradeLevels() {
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			// UserServiceImpl의 메서드를 호출
			userService.upgradeLevels();

			this.transactionManager.commit(status);
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}

// 리스트 6-7 트랜잭션 오브젝트가 추가된 설정파일
// UserService는 UserServiceTx를 사용하도록 설정
// UserServiceTx는 UserServiceImpl을 사용하도록 설정
<bean id="userService" class="springbook.user.service.UserServiceTx">
	<property name="transactionManager" ref="transactionManager" />
	<property name="userService" ref="userServiceImpl" />
</bean>

<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
	<property name="userDao" ref="userDao" />
	<property name="mailSender" ref="mailSender" />
</bean>
```

* 트랜잭션 분리에 따른 테스트 수정
  * `@Autowired` 애노테이션 변경이 필요함.
    * 기존에는 UserService 클래스 타입의 빈을 지정
    * 수정한 스프링의 설정파일에는 UserService라는 인터페이스 타입을 가진 두 개의 빈이 존재
    * `@Autowired UserService userService` -> id가 userService인 빈이 주입됨.
  * 목 오브젝트를 이용해 수동 DI를 적용하는 테스트라면 어떤 클래스의 오브젝트인지 분명히 해야 함.
    * `@Autowired UserServiceImpl userServiceImpl`의 방식으로 적용해야 함.
  * `upgradeLevels()' 테스트 메서드 수정
    * MailSender의 목 오브젝트 설정이 UserSerivce 인터페이스를 통해서는 불가
    * UserServiceImpl을 통해 설정하도록 변경 필요
  * `upgradeAllOrNothing()` 테스트 메서드 수정
    * TestUserService가 트랜잭션 기능은 빠진 UserServiceImpl을 상속하도록 변경 필요
    * 트랜잭션 롤백의 확인을 위해 강제로 예외를 발생시킬 위치가 UserServiceImpl에 있기 때문
    * `static class TestUserService extends UserServiceImpl`로 변경 필요
```java
// 리스트 6-8 목 오브젝트 설정이 필요한 테스트 코드 수정
@Test
public void upgradeLevels() throws Exception {
	...
	MockMailSender mockMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
}

// 리스트 6-9 분리된 테스트 기능이 포함되도록 수정한 upgradeAllOrNothing()
@Test
public void upgradeAllOrNothing() {
	// UserService 인터페이스를 이용하던 걸 TestUserService 클래스르 이용하도록 변경
	TestUserService testUserService = new TestUserService(users.get(3).getId());
	testUserService.setUserDao(userDao);
	testUserService.setMailSender(mailSender);
	
	// 트랜잭션 기능을 분리한 UserServiceTx는 예외 발생용으로 수정할 필요가 없으니 그대로 사용
	UserServiceTx txUserService = new UserServiceTx();
	txUserService.setTransactionManager(transactionManager);
	txUserService.setUserService(testUserService); // TestUserService를 수동 DI시킴
	 
	userDao.deleteAll();			  
	for(User user : users) userDao.add(user);
	
	try {
		// 트랜잭션 기능을 분리한 오브젝트를 통해 예외 발생용 TestUserService가 호출되야 함.
		txUserService.upgradeLevels();   
		fail("TestUserServiceException expected"); 
	}
	catch(TestUserServiceException e) { 
	}
	
	checkLevelUpgraded(users.get(1), false);
}
```

* 트랜잭션 경계설정 코드 분리의 장점
  * 비즈니스 로직을 담당하는 UserServiceImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용은 신경쓰지 않아도 됨.
  * 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있음.


### 6.2 고립된 단위 테스트

* 작은 단위의 테스트가 좋은 이유?
<br>테스트가 실패했을 때 그 원인을 찾기 쉽다.

* 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위 테스트가 주는 장점을 얻기 힘들다


### 6.2.1 복잡한 의존관계 속의 테스트

* UserServiceTest
  * UserService (테스트 대상)
    * UserDaoJdbc -> XXDataSource -> DB
    * DSTransactionManager
    * JavaMailSenderImpl -> JavaMail -> Mail Server

* UserServiceTest가 테스트하고자 하는 단위는 UserService
  * 하지만 UserService는 UserDao, TransactionManager, MailSender라는 3가지 의존관계를 갖고 있음.
  * 따라서 의존관계를 갖는 오브젝트들이 테스트가 진행되는 동안에 같이 실행됨!
  * UserService를 테스트하는 것처럼 보이지만 사실은 연관된 모든 로직과 환경을 같이 테스트하는 셈.


### 6.2.2 테스트 대상 오브젝트 고립시키기

* MailSender에서 적용한 것처럼 테스트를 위한 대역을 사용해서 오브젝트를 고립시킬 필요가 있음.
  * 트랜잭션과 기술들은 `UserServiceTx`에서 관리하도록 함.
  * `UserServiceImpl`에서 `MockUserDao`와 `MockMailSender`를 이용해서 테스트 오브젝트를 고립.

```java
// 리스트 6-12 UserDao 오브젝트
static class MockUserDao implements UserDao { 
	private List<User> users; // 레벨 업그레이드 후보 User 오브젝트 목록
	private List<User> updated = new ArrayList<User>(); // 업그레이드 대상 오브젝트를 저장해둘 목록
	
	private MockUserDao(List<User> users) {
		this.users = users;
	}

	public List<User> getUpdated() {
		return this.updated;
	}

	// 스텁 기능 제공
	public List<User> getAll() {  
		return this.users;
	}

	// 목 오브젝트 기능 제공
	public void update(User user) {  
		updated.add(user);
	}
	
	// 테스트에 사용되지 않는 메서드
	public void add(User user) { throw new UnsupportedOperationException(); }
	public void deleteAll() { throw new UnsupportedOperationException(); }
	public User get(String id) { throw new UnsupportedOperationException(); }
	public int getCount() { throw new UnsupportedOperationException(); }
}

// 리스트 6-13 MockUserDao를 사용해서 만든 고립된 테스트
@Test 
public void upgradeLevels() throws Exception {
	// 고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성
	UserServiceImpl userServiceImpl = new UserServiceImpl(); 
	
	// 목 오브젝트로 만든 UserDao를 직접 DI
	MockUserDao mockUserDao = new MockUserDao(this.users);  
	userServiceImpl.setUserDao(mockUserDao);

	MockMailSender mockMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
	
	userServiceImpl.upgradeLevels();

	List<User> updated = mockUserDao.getUpdated(); // MockUserDao로부터 업데이트 결과를 가져옴.
	// 업데이트 횟수와 정보를 확인
	assertThat(updated.size(), is(2));  
	checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER); 
	checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);
	
	List<String> request = mockMailSender.getRequests();
	assertThat(request.size(), is(2));
	assertThat(request.get(0), is(users.get(1).getEmail()));
	assertThat(request.get(1), is(users.get(3).getEmail()));
}

// id와 level을 확인하는 간단한 헬퍼 메서드
private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
	assertThat(updated.getId(), is(expectedId));
	assertThat(updated.getLevel(), is(expectedLevel));
}
```

* 고립된 테스트를 수행 시 테스트 수행 성능이 향상됨.
  * UserServiceImpl과 테스트를 도와주는 2개의 목 오브젝트 외 불필요한 의존 오브젝트와 서비스를 제거.
  * DB Connection, Transaction 등등을 모두 제거.


### 6.2.3 단위 테스트와 통합 테스트

* 단위 테스트
<br>테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를
<br>사용하지 않도록 고립시켜서 테스트 하는 것

* 통합 테스트
<br>두 개 2상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나,
<br>또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트

* 단위/통합 테스트 가이드라이
  * 항상 단위 테스트를 먼저 고려
  * 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고
  <br>필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 생성.
  * 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 생성.
  * DAO 같이 단위 테스트로 만들기 어려운 코드도 존재.
  <br>DAO는 DB까지 연동하는 테스트로 만드는 편이 효과적 (DB 연결, SQL 테스트 등)
  * DAO 테스트는 DB라는 외부 리소스를 사용하기 때문에 통합 테스트로 분류.
  * 여러 개의 단위가 의존관계를 가지고 동작할 떄를 위한 통합 테스트는 필요.
  * 단위 테스트를 만들기가 너무 복잡하다고 판단되는 코드는 처음부터 통합 테스트를 고려.
  * 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트


### 5.2.4 목 프레임워크

* 단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트의 사용이 필수적

* 단위 테스트가 많은 장점이 있고 가장 우선시해야하지만 작성이 번거로움.

* Mockito 프레임워크
  * 목 프레임워크를 사용하면 목 클래스를 일일이 준비할 필요가 없다.
```java
// 목 오브젝트를 생성해줌.
UserDao mockUserDao = mock(UserDao.class);
// mockUserDao.getAll()이 호출됐을 때 users 리스트를 리턴해주라는 선언
when(mockUserDao.getAll()).thenReturn.this.users);
// User 타입의 오브젝트를 파라미터로 받으며 update() 메서드가 두 번 호출됐는지 확인하라는 선언
verify(mockUserDao, times(2)).update(any(User.class));
```

* Mockito 목 오브젝트 사용방법
  1. 인터페이스를 이용해 목 오브젝트를 생성 (필수)
  2. 목 오브젝트가 리턴할 값이 있으면 이를 지정해줌.
  3. 테스트 대상 오브젝트에 DI해서 목 오브젝트가 테스트 중에 사용되게 만든다. (필수)
  4. 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메서드가 호출됐는지,
  <br>어떤 값을 가지고 몇 번 호출됐는지를 검증.

```java
// 리스트 6-14 Mockito를 적용한 테스트 코드
@Test
public void mockUpgradeLevels() throws Exception {
	UserServiceImpl userServiceImpl = new UserServiceImpl();
	
	UserDao mockUserDao = mock(UserDao.class); // 다이내믹한 목 오브젝트 생성
	when(mockUserDao.getAll()).thenReturn(this.users); // 메서드 리턴 값 설정
	userServiceImpl.setUserDao(mockUserDao); // DI
	
	// 리턴 값이 없는 메서드는 더 쉽게 DI 가능
	MailSender mockMailSender = mock(MailSender.class);
	userServiceImpl.setMailSender(mockMailSender);
	
	userServiceImpl.upgradeLevels();
	
	// 목 오브젝트가 제공하는 검증 기능을 통해 테스트
	verify(mockUserDao, times(2)).update(any(User.class)); // times(): 메서드 호출 횟수 점검
	verify(mockUserDao, times(2)).update(any(User.class)); // any(): 파라미터 내용은 무시하고 확인
	verify(mockUserDao).update(users.get(1));
	assertThat(users.get(1).getLevel(), is(Level.SILVER));
	verify(mockUserDao).update(users.get(3));
	assertThat(users.get(3).getLevel(), is(Level.GOLD));
	
	ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
	verify(mockMailSender, times(2)).send(mailMessageArg.capture()); // 파라미터를 정밀하게 검사하기 위해 캡쳐
	List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
	assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
	assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```


### 6.3 다이내믹 프록시와 팩토리 빈


### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴

* 단순히 확장성을 고려해서 한 가지 기능을 분리한다면? **전략 패턴을 사용하면 됨**
<br>-> 전략 패턴으로는 트랜잭션 기능의 구현 내용을 분리해냈을 뿐. 적용여부는 소스에 남아있음.

* 트랜잭션 기능을 소스에서 분리시키기 위해 `UserServiceImpl`, `UserServiceTx`로 분류함.
<br>-> 부가기능 외에 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임해줘야 함. (UserServiceImpl)

* 클라이언트 -> 프록시 -> 타깃
  * 각각을 연결할 때 클래스 직접 접근이 아닌 핵심기능 인터페이스를 통해 접근
  * **프록시(proxy)**
  <br>마치 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것.
  <br>ex) `UserServiceTx`
  * **타깃(target), 실체(real subject)**
  <br>프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트
  <br>ex) `UserServiceImpl`

* 프록시의 사용 목적에 따른 구분
  * 클라이언트가 타깃에 접근하는 방법을 제어하기 위해서
  * 타깃에 부가적인 기능을 부여해주기 위해서

* 데코레이터 패턴
  * **타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴**
  * 컴파일 시점에는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않음.
  * 데코레이터 패턴에서는 프록시가 1개로 제한되지 않음
  * 같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용 가능
  <br>ex) 클라이언트 -> 라인넘버 데코레이터 -> 컬러 데코레이터 -> 페이징 데코레이터 -> 소스코드 출력 기능(타깃)
  * 자바 IO 패키지의 `InputStream`과 `OutputStream` 구현 클래스가 대표적인 데코레이터 패턴
  <br>`InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));
  * 데코레이터 패턴은 인터페이스를 통해 위임하는 방식이기 때문에
  <br>어느 데코레이터에서 타깃으로 연결될지 코드 레벨에서는 미리 알 수 없음.

* 프록시 패턴
  * **프록시를 사용하는 방법 중에서 타깃에 대한 접근방법을 제어하려는 목적을 가진 경우**
  * 프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않음.
  <br>**대신 클라이언트가 타깃에 접근하는 방식을 변경.**
  * 프록시 패턴을 사용하는 경우
    * 타깃 오브젝트를 생성하기가 복잡하거나 당장 필요하지 않은 경우 필요한 시점까지 오브젝트는 생성 안 하는 게 좋음.
    * 하지만 타깃 오브젝트에 대한 레퍼런스가 미리 필요한 경우에 적용
    * 클라이언트에 타깃에 대한 레퍼런스를 넘길 때 실제 오브젝트 대신에 프록시를 넘겨줌.
  * 원격 오브젝트를 이용하는 경우에도 프록시를 사용하면 편리함.
    * 각종 리모팅 기술을 이용해 다른 서버에 존재하는 오브젝트를 사용해야 하는 경우
    * 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트는 로컬에 존재하는 오브젝트를 쓰는 것처럼 프록시 사용.
  * 프록시 패턴은 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것.
  <br>ex) 클라이언트 -> **접근제어 프록시** -> 컬러 데코레이터 -> 페이징 데코레이터 -> 소스코드 출력 기능(타깃)


### 6.3.2 다이내믹 프록시

* 프록시의 구성과 프록시 작성의 문제점
  * 프록시의 2가지 기능
    * 타깃과 같은 메서드를 구현하고 있다가 메서드가 호출되면 타깃 오브젝트로 위임
    * 지정된 요청에 대해서는 부가기능을 수행
  * 프록시를 만들기가 번거로운 이유
    * 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거로움.
    * 부가기능 코드가 중복될 가능성이 많음.

* 리플렉션 (Reflection)
  * 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어 줌.
  * 자바의 모든 클래스는 그 클래스 자체의 구성정보를 담은 Class 타입의 오브젝트를 갖고 있음.
  * `클래스이름.class`라고 하거나 오브젝트의 `getClass()` 메서드를 호출하면
  <br>클래스 정보를 담은 Class 타입의 오브젝트를 가져올 수 있음.
  <br>`Method lengthMethod = String.class.getMethod("length"); // String의 length() 메서드
  * `invoke()` 메서드
  <br>메서드를 실행시킬 대상 오브젝트와 파라미터 목록을 받아서 메서드를 호출한 뒤에
  <br>그 결과를 Object 타입으로 리턴
  <br>`int length = lengthMethod.invoke(name); // int length = name.length();`

* 다이내믹 프록시 적용
  * **다이내믹 프록시**
  <br>프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트
  * 타깃의 인터페이스와 같은 타입으로 만들어짐.
  * 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용 가능
  * 프록시 팩토리에게 인터페이스 정보만 제공하면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동생성.
  * 프록시로서 필요한 부가기능 제공 코드는 직접 작성해야 함.
  <br>-> 부가기능은 `InvocationHandler`를 구현한 오브젝트에 담는다.
  * `public Object invoke(Object proxy, Method method, Object[] args)`
    * `Method method`: 리플렉션의 Method 인터페이스
    * `Object[] args`: 메서드를 호출할 때 전달되는 파라미터
  * 다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서
  <br>InvocationHandler 구현 오브젝트의 invoke() 메서드로 넘김.
  <br>-> 타깃 인터페이스의 모든 메서드 요청이 하나의 메서드로 집중되기 떄문에 중복되는 기능을 효과적으로 제공 가능
```java
// 리스트 6-23 InvocationHandler 구현 클래스
public class UppercaseHandler implements InvocationHandler {
	Hello target;
	
	// 다이내믹 프록시로부터 전달받은 요청을
	// 다시 타깃 오브젝트에 주입해야 하기 때문에 타깃 오브젝트를 주입받아 둔다.
	public UppercaseHandler(Hello target) {
		this.target = target;
	}
	
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		String ret = (String)method.invoke(target, args); // 타깃으로 위임. 인터페이스의 메서드 호출에 모두 적용됨.
		return ret.toUppserCase(); // 부가기능 제공
	}
}

// 리스트 6-24 프록시 생성
// 파라미터로 제공한 Hello.class 인터페이스를 구현한 클래스의 오브젝트이기 때문에 Hello 타입으로 캐스팅 가능
Hello proxiedHello = (Hello)Proxy.newProxyInstance(
	getClass().getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용될 클래스 로더
	new Class[] { Hello.class }, // 구현할 인터페이스. 한 번에 하나 이상의 인터페이스 구현 가능
	new UppercaseHandler(new HelloTarget()); // 부가기능과 위임 코드를 담은 InvocationHandler
```

* `InvocationHandler`는 타깃의 종류에 상관없이도 적용이 가능함.

```java
// 리스트 6-25 확장된 UppercaseHandler
public class UppercaseHandler implements InvocationHandler {
	Object target;
	
	// 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정
	public UppercaseHandler(Object target) {
		this.target = target;
	}
	
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		Object ret = method.invoke(target, args);
		if (ret instanceof String) { // 호출된 메서드의 리턴타입이 String일 경우에만 부가기능 제공
			return ((String)ret).toUppserCase();
		} else {
			return ret;
		}
	}
}
```


### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능

* 트랜잭션 InvocationHandler
  * 롤백 적용 예외는 `InvocationTargetException`을 적용해야 함.
  * 리플렉션 메서드 `Method.invoke()` 사용 시에는 타깃 오브젝트에서 발생한 예외가 포장됨.
  
```java
// 리스트 6-27 다이내믹 프록시를 위한 트랜잭션 부가기능
public class TransactionHandler implements InvocationHandler {
	Object target; // 부가기능을 제공할 타깃 오브젝트. 어떤 타입의 오브젝트에도 적용 가능.
	PlatformTransactionManager transactionManager; // 트랜잭션 기능을 제공하는 트랜잭션 매니저
	String pattern; // 트랜잭션을 적용할 메서드 이름 패턴

	public void setTarget(Object target) {
		this.target = target;
	}

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setPattern(String pattern) {
		this.pattern = pattern;
	}

	// 트랜잭션 적용 대상 메서드를 선별해서 트랜잭션 경계설정 기능을 부여
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		if (method.getName().startsWith(pattern)) {
			return invokeInTransaction(method, args);
		} else {
			return method.invoke(target, args);
		}
	}

	private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			Object ret = method.invoke(target, args); // 트랜잭션을 시작하고 타깃 오브젝트의 메서드를 호출
			this.transactionManager.commit(status); // 예외가 발생하지 않았으면 트랜잭션 커밋
			return ret;
		} catch (InvocationTargetException e) { // 예외가 발생하면 트랜잭션 롤백
			this.transactionManager.rollback(status);
			throw e.getTargetException();
		}
	}
}

// 리스트 6-28 다이내믹 프록시를 이용하는 테스트
@Test
public void upgradeAllOrNothing() throws Exception {
	...
	TransactionHandler txHandler = new TransactionHandler();
	// 트랜잭션 핸들러가 필요한 정보와 오브젝트를 DI
	txHandler.setTarget(testUserService);
	txHandler.setTransactionManager(transactionManager);
	txHandler.setPattern("upgradeLevels");
	
	// UserService 인터페이스 타입의 다이내믹 프록시 생성
	UserService txUserService = (UserService)Proxy.newProxyInstance(
		getClass().getClassLoader(), new Class[] { UserService.class }, txHandler);
	...
}
```


### 6.3.4 다이내믹 프록시를 위한 팩토리 빈

* DI의 대상이 되는 다이내믹 프록시 오브젝트는 스프링의 빈으로 등록할 방법이 없음!
  * 스프링의 빈의 기본적으로 클래스 이름과 프로퍼티로 정의됨.
  * Class의 newInstance() 메서드는 해당 클래스의 파라미터가 없는 생성자를 호출하고
  <br>그 결과 생성되는 오브젝트를 돌려주는 리플렉션 API.
  <br>`Date now = (Date)Class.forName("java.util.Date").newInstance();`
  * 스프링은 내부적으로 리플렉션 API를 이용해서 빈 정의에 나오는 **클래스 이름을 가지고 빈 오브젝트를 생성**
  <br>-> 다이내믹 프록시 오브젝트는 이런 식으로 프록시 오브젝트가 생성되지 않는다!
  <br>-> 다이내믹 프록시는 Proxy 클래스의 newProxyInstance()라는 스태틱 팩토리 메서드를 통해서만 생성 가능!

* 팩토리 빈
  * 스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈
  <br>-> 스프링의 `FactoryBean` 인터페이스를 구현하는 방법으로 생성 가능
  
```java
// 리스트 6-29 FactoryBean 인터페이스
package org.springframework.beans.factory;

public interface FactoryBean<T> {
	T getObject() throws Exception; // 빈 오브젝트를 생성해서 돌려줌.
	Class<? extends T> getObjectType(); // 생성되는 오브젝트의 타입을 알려줌.
	boolean isSingleton(); // getObject()가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려줌.
}

// 리스트 6-30 생성자를 제공하지 않는 클래스
public class Message {
	String text;
	
	// 생성자 private로 선언되어 있어서 외부에서 접근 불가
	private Message(String text) {
		this.text = text;
	}
	
	public String getText() {
		return text;
	}
	
	// 생성자 대신 사용할 수 있는 스태틱 팩토리 메서드를 제공
	public static Message newMessage(String text) {
		return new Message(text);
	}
}

// 리스트 6-31 Message의 팩토리 빈 클래스
public class MessageFactoryBean implements FactoryBean<Message> {
	String text;
	
	// 오브젝트를 생성할 때 필요한 정보를 팩토리 빈의 프로퍼티로 설정해서 대신 DI 받을 수 있게 함.
	// 주입된 정보는 오브젝트 생성 중 사용됨.
	public void setText(String text) {
		this.text = text;
	}

	// 실제 빈으로 사용할 오브젝트를 직접 생성
	// 코드를 이용하기 때문에 복잡한 방식의 오브젝트 생성과 초기화 작업도 가능.
	public Message getObject() throws Exception {
		return Message.newMessage(this.text);
	}

	public Class<? extends Message> getObjectType() {
		return Message.class;
	}

	// getObject() 메서드가 돌려주는 오브젝트가 싱글톤인지를 알려줌.
	// 이 팩토리 빈은 매번 요청할 때마다 새로운 오브젝트를 만드므로 false로 설정
	public boolean isSingleton() {
		return true;
	}
}
```

* 팩토리 빈의 설정 방법
  * 여타 빈 설정과 다른 점
    * message 빈 오브젝트의 타입이 class 애트리뷰트에 정의된 MessageFactoryBean이 아니라 Message 타입임.
    * Message 빈의 타입은 getObjectType() 메서드가 돌려주는 타입으로 결정됨.

```java
// 리스트 6-32 팩토리 빈 설정
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
	<property name="text" value="Factory Bean" />
</bean>

// 리스트 6-33 팩토리 빈 테스트
RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration // 설정파일 이름을 지정하지 않으면 클래스이름 + "-context.xml"이 디폴트로 사용됨.
public class FactoryBeanTest {
	@Autowired
	ApplicationContext context;
	
	@Test
	public void getMessageFromFactoryBean() {
		Object message = context.getBean("message");
		assertThat(message, is(Message.class)); // 타입 확인
		assertThat(((Message)message).getText(), is("Factory Bean"));
	}
	
	@Test
	public void getFactoryBean() throws Exception {
		Object factory = context.getBean("&message"); // "&"를 앞에 붙여주면 팩토리 빈 자체를 돌려줌.
		assertThat(factory, is(MessageFactoryBean.class));
	}
}
```

* 다이내믹 프록시를 만들어주는 팩토리 빈
  * Proxy의 newProxyInstance() 메서드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는
  <br>일반적인 방법으로는 스프링의 빈으로 등록이 불가
  * 대신 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수 있음.

```java
public class TxProxyFactoryBean implements FactoryBean<Object> {
	// TransactionHandler를 생성할 때 필요
	Object target;
	PlatformTransactionManager transactionManager;
	String pattern;
	// 다이내믹 프록시를 생성할 때 필요.
	// UserService 외의 인터페이스를 가진 타깃에도 적용 가능
	Class<?> serviceInterface;
	
	public void setTarget(Object target) {
		this.target = target;
	}

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setPattern(String pattern) {
		this.pattern = pattern;
	}

	public void setServiceInterface(Class<?> serviceInterface) {
		this.serviceInterface = serviceInterface;
	}

	// FactoryBean 인터페이스 구현 메서드
	// DI 받은 정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시를 구현
	public Object getObject() throws Exception {
		TransactionHandler txHandler = new TransactionHandler();
		txHandler.setTarget(target);
		txHandler.setTransactionManager(transactionManager);
		txHandler.setPattern(pattern);
		return Proxy.newProxyInstance(
			getClass().getClassLoader(),new Class[] { serviceInterface }, txHandler);
	}

	// 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에 따라 달라짐
	// 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용 가능
	public Class<?> getObjectType() {
		return serviceInterface;
	}

	// 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 뜻.
	public boolean isSingleton() {
		return false;
	}
}

// 리스트 6-36 UserService에 대한 트랜잭션 프록시 팩토리 빈
<bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
	<property name="target" ref="userServiceImpl" />
	<property name="transactionManager" ref="transactionManager" />
	<property name="pattern" value="upgradeLevels" />
	<property name="serviceInterface" value="springbook.user.service.UserService" />
</bean>
```

* 트랜잭션 프록시 팩토리 빈 테스트
  * UserServiceTest의 add() 메서드는 @Autowired로 가져온 userService의 빈을 사용하기 때문에
  <br>TxProxyFactoryBean 팩토리 빈이 생성하는 다이내믹 프록시를 통해 UserService 기능을 사용
  * upgradeAllOrNothing() 메서드의 경우 예외 발생 시 트랜잭션이 롤백됨을 확인하려면
  <br>비즈니스 로직 코드를 수정한 `TestUserService` 오브젝트를 타깃 오브젝트 대신 사용해야 함.
  <br>-> **빈으로 등록된 TxProxyFactoryBean을 직접 가져와서 프록시를 생성**

```java
// 리스트 6-37 트랜잭션 프록시 팩토리 빈을 적용한 테스트
public class UserServiceTest {
	...
	@Autowired ApplicationContext context; // 팩토리 빈을 가져오려면 필요
	...
 
	@Test
	@DirtiesContext // 수동 DI를 사용하기 때문에 사용
	public void upgradeAllOrNothing() throws Exception {
		TestUserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(userDao);
		testUserService.setMailSender(mailSender);
		
		// 팩토리 빈 자체를 가져와야 하므로 빈 이름에 "&"를 반드시 넣어줘야 함.
		TxProxyFactoryBean txProxyFactoryBean = 
			context.getBean("&userService", TxProxyFactoryBean.class);
		txProxyFactoryBean.setTarget(testUserService); // 테스트용 타깃 주입
		// 변경된 타깃 설정을 이용해서 트랜잭션 다이내믹 프록시 오브젝트를 다시 생성.
		UserService txUserService = (UserService) txProxyFactoryBean.getObject();
				 
		userDao.deleteAll();			  
		for(User user : users) userDao.add(user);
		
		try {
			txUserService.upgradeLevels();   
			fail("TestUserServiceException expected"); 
		}
		catch(TestUserServiceException e) { 
		}
		
		checkLevelUpgraded(users.get(1), false);
	}
	...
}
```


### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계

* 한 번 부가기능을 가진 프록시를 생성하는 프록시 빈을 만들어두면 타깃의 타입에 상관없이 재사용 가능.

* 프록시 팩토리 빈의 재사용
  * TxProxyFactoryBean은 코드의 수정 없이도 다양한 클래스에 적용 가능
  <br>-> 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록하면 됨.
  * 프록시 팩토리 빈을 이용하면 프록시 기법을 아주 빠르고 효과적으로 적용 가능
  
```java
// 리스트 6-38 트랜잭션 없는 서비스 빈 설정
<bean id="coreService" class="complex.module.CoreServiceImpl">
	<property name="coreDao" ref="coreDao" />
</bean>

// 리스트 6-39 아이디를 변경한 CoreService 빈
<bean id="coreServiceTarget" class="complex.module.CoreServiceImpl">
	<property name="coreDao" ref="coreDao" />
</bean>

// 리스트 6-40 CoreService에 대한 트랜잭션 프록시 팩토리 빈
<bean id="coreService" class="springbook.service.TxProxyFactoryBean">
	<property naem="target" ref="coreServiceTarget" /> // 타깃 지정
	<property name="transactionManager" ref="transactionManager" />
	<property name="pattern" value="" />
	<property name="serviceInterface" ref="complex.module.CoreService" /> // 프록시가 구현할 인터페이스
</bean>
```

* 프록시 팩토리 빈 방식의 장점
  * 데코레이터 패턴이 적용된 프록시를 적극적으로 활용하지 못하는 2가지 문제점
    * 프록시를 적용할 대상이 구현하고 있는 인터페이스를 구현하는 프록시 클래스를 일일이 만들어야 하는 번거로움
    * 부가적인 기능이 여러 메서드에 반복적으로 나타나게 돼서 코드 중복의 문제가 발생
  * 프록시 팩토리 빈 방식은 이 2가지 문제점을 모두 해결!
    * 다이내믹 프록시에 팩토리 빈을 이용한 DI를 적용하면 번거로운 다이내믹 프록시 생성 코드 제거 가능
    * 하나의 핸들러 메서드를 구현하는 것만으로 여러 메서드에 부가기능 부여 가능

* 프록시 팩토리 빈의 한계
  * 프록시를 통해 타깃에 부가기능을 제공하는 것은 메서드 단위로 일어나는 일
  <br>-> 한 번에 여러 클래스에 공통적인 부가기능을 제공하는 일은 지금까지 방법으로는 불가능
  * 하나의 타깃에 여러 개의 부가기능을 적용하려 할 때도 문제
  <br>-> XML로 가능은 하지만 설정 파일이 급격히 복잡해진다!
  * TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어진다.
  <br>-> 동일한 기능을 제공하는 코드임에도 타깃 오브젝트가 달라지면 새로운 Handler 오브젝트를 만들어야 함.


### 6.4 스프링의 프록시 팩토리 빈


### 6.4.1 ProxyFactoryBean

* 스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공
  * 생성된 프록시는 스프링의 빈으로 등록돼야 함.
  * 스프링은 프록시 오브젝트를 생성해주는 기술을 추상화한 팩토리 빈을 제공

* `ProxyFactoryBean`
  * 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈
  * 순수하게 프록시를 생성하는 작업만을 담당
  * 프록시를 통해 제공해줄 부가기능은 별도의 빈에 둘 수 있음

* `MethodInterceptor`
  * ProxyFactoryBean이 생성하는 프록시에서 사용할 부가기능을 구현하는 인터페이스
  * InvocationHandler와 다른점
    * InvocationHandler의 Invoke() 메서드는 타깃 오브젝트에 대한 정보를 제공하지 않음.
    <br>-> 타깃은 InvocationHandler가 구현한 클래스가 직접 알고 있어야 함.
    * MethodInterceptor의 Invoke() 메서드는 ProxyFactoryBean으로부터 타깃 오브젝트 정보도 제공받음.
    <br>-> MethodInterceptor는 타깃 오브젝트에 상관없이 독립적으로 생성 가능
    <br>-> 타깃이 다른 여러 프록시에서 함께 사용할 수 있고 싱글톤 빈으로 등록 가능

```java
// 리스트 6-41 스프링 ProxyFactoryBean을 이용한 다이내믹 프록시 테스트

```
public class DynamicProxyTest {
	@Test
	public void simpleProxy() {
		// JDK 다이내믹 프록시 생성
		Hello proxiedHello = (Hello)Proxy.newProxyInstance(
				getClass().getClassLoader(), 
				new Class[] { Hello.class},
				new UppercaseHandler(new HelloTarget()));
		...
	}
	
	@Test
	public void proxyFactoryBean() {
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(new HelloTarget()); // 타깃 설정
		pfBean.addAdvice(new UppercaseAdvice()); // 부가 기능을 담은 어드바이스를 추가. 여러 개 추가 가능

		Hello proxiedHello = (Hello) pfBean.getObject(); // FactoryBean이므로 getObject()로 생성된 프록시 가져옴.
		
		assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
		assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
		assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
	}
	
	static class UppercaseAdvice implements MethodInterceptor {
		public Object invoke(MethodInvocation invocation) throws Throwable {
			String ret = (String)invocation.proceed(); // 리플렉션의 메서드와 달리 타깃 오브젝트 전달 불필요.
			return ret.toUpperCase();
		}
	}
	
	// 타깃과 프록시가 구현할 인터페이스
	static interface Hello {
		String sayHello(String name);
		String sayHi(String name);
		String sayThankYou(String name);
	}
	
	// 타깃 클래스
	static class HelloTarget implements Hello {
		public String sayHello(String name) { return "Hello " + name; }
		public String sayHi(String name) { return "Hi " + name; }
		public String sayThankYou(String name) { return "Thank You " + name; }
	}
}

* **어드바이스**: 타깃이 필요 없는 순수한 부가기능
  * InvocationHandler를 구현했을 때와 달리 MethodInterceptor를 구현할 때는 타깃 오브젝트가 등장하지 않음.
  <br>-> MethodInterceptor는 메서드 정보와 타깃 오브젝트가 담긴 MethodInvocatino 오브젝트가 전달됨.
  * `MethodInvocation`
    * 일조의 콜백 오브젝트로, `proceed()` 메서드를 실행하면 타깃 오브젝트의 메서드를 내부적으로 실행
    * MethodInvocation 구현 클래스는 일종의 공유 가능한 템플릿처럼 동작!
    <br>-> JDK 다이내믹 프록시 코드와의 가장 큰 차이점이자 ProxyFactoryBean의 장점
  * ProxyFactoryBean은 작은 단위의 템플릿/콜백 구조를 이용해서 적용함.
  <br>-> **템플릿 역할을 하는 MethodInvocation을 싱글톤으로 두고 공유 가능**
  * 
