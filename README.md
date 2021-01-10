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
  * **타깃 오브젝트에 종속되지 않으며 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트**
  * InvocationHandler를 구현했을 때와 달리 MethodInterceptor를 구현할 때는 타깃 오브젝트가 등장하지 않음.
  <br>-> MethodInterceptor는 메서드 정보와 타깃 오브젝트가 담긴 MethodInvocatino 오브젝트가 전달됨.
  * `MethodInvocation`
    * 일조의 콜백 오브젝트로, `proceed()` 메서드를 실행하면 타깃 오브젝트의 메서드를 내부적으로 실행
    * MethodInvocation 구현 클래스는 일종의 공유 가능한 템플릿처럼 동작!
    <br>-> JDK 다이내믹 프록시 코드와의 가장 큰 차이점이자 ProxyFactoryBean의 장점
  * ProxyFactoryBean은 작은 단위의 템플릿/콜백 구조를 이용해서 적용함.
  <br>-> **템플릿 역할을 하는 MethodInvocation을 싱글톤으로 두고 공유 가능**
  * `addAdvice()`
    * MethodInterceptor 설정 시 수정자 대신 사용하는 메서드
    * MethodInterceptor가 Advice 인터페이스를 상속하는 서브 인터페이스기 때문에 addAdvice 메서드 사용
    * MethodInterceptor에 여러 개의 어드바이스를 추가하는 기능
    <br>-> 부가기능 추가 시 프록시, 프록시 팩토리빈도 추가해줘야 했던 문제 해결
  * 인터페이스를 제공받지 않아도 ProxyFactoryBean에 있는 인터페이스 자동검출 기능을 사용하여
  <br>타깃 오브젝트가 구현하고 있는 인터페이스 정보를 알아내고 이를 구현하는 프록시를 생성함.

* **포인트컷**: 부가기능 적용 대상 메서드 선정 방법
  * **메서드 선정 알고리즘을 담은 오브젝트**
  * 여러 프록시가 공유하는 MethodInterceptor에 특정 프록시에만 적용되는 패턴을 넣으면 문제가 됨.
  * MethodInterceptor에는 재사용 가능한 순수한 부가기능 제공 코드만 남기고 프록시에 부가기능 적용 메서드를 넣으면?
    * InvocationHandler가 타깃과 메서드 선정 알고리즘 코드에 의존하게 됨! 확장이 불가능!
    * DI를 통해 빈으로 분리할 수는 있지만 빈으로 구성된 InvocationHandler 오브젝트는 특정 타깃을 위한 프록시에 제한됨.

* 어드바이스와 포인트 컷은 모두 프록시에 DI로 주입되서 사용됨.
<br>여러 프록시에서 공유가 가능하도록 만들어지기 때문에 싱글톤 빈으로 등록 가능.

* 어드바이스는 JDK 다이내믹 프록시의 InvocationHandler와 달리 직접 타깃을 호출하지 않음.

* 어드바이스가 부가기능을 부여하는 중에 타깃 메서드의 호출이 필요하면 프록시로부터 전달받은
<br>MethodInvocation 타입 콜백 오브젝트의 `proceed()` 메서드를 호출하면 됨.
  * Invocation 콜백은 타깃 오브젝트의 레퍼런스를 갖고 있고 이를 이용해 타깃 메서드를 직접 호출
    * 어드바이스가 일종의 템플릿
    * 타깃 호출기능을 갖고 있는 MethodInvoation 오브젝트가 콜백

* 어드바이스도 독립적인 싱글톤 빈으로 등록하고 DI를 주입해서 여러 프록시가 사용할 수 있게 생성 가능.

* **어드바이저 = 포인트컷(메서드 선정 알고리즘) + 어드바이스(부가기능)**

```java
// 리스트 6-42 포인트컷까지 적용한 ProxyFactoryBean
@Test
public void pointcutAdvisor() {
	ProxyFactoryBean pfBean = new ProxyFactoryBean();
	pfBean.setTarget(new HelloTarget());
	
	// 메서드 이름을 비교해서 대상을 선정하는 알고리즘을 제공하는 포인트컷 생성
	NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
	// 이름 비교조건 설정
	pointcut.setMappedName("sayH*"); 
	
	// 포인트컷과 어드바이스를 Advisor로 묶어서 한 번에 추가
	pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
	
	Hello proxiedHello = (Hello) pfBean.getObject();
	
	assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
	assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
	assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby")); 
}
```


### 6.4.2 ProxyFactoryBean 적용

* TransactionAdvice
<br>MethodInterceptor라는 Advice 서브 인터페이스를 구현해서 생성.
```java
// 리스트 6-43 트랜잭션 어드바이스
package springbook.user.service;
...
public class TransactionAdvice implements MethodInterceptor {
	PlatformTransactionManager transactionManager;

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	// 타깃을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받음
	// 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능
	public Object invoke(MethodInvocation invocation) throws Throwable {
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			// 콜백을 호출해서 타깃의 메서드를 실행. 호출 전후로 필요한 부가기능을 넣을 수 있음.
			Object ret = invocation.proceed();
			this.transactionManager.commit(status);
			return ret;
		} catch (RuntimeException e) { // JDK 다이내믹 프록시와 달리 예외를 포장하지 않고 그대로 전달.
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
```

* 스프링 XML 설정파일
  * 어드바이스를 빈으로 등록하고 트랜잭션 기능 적용을 위해 transactionManager를 DI
  * 트랜잭션 적용 메서드 선정을 위해 포인트컷을 빈으로 등록
  * 어드바이스와 포인트컷을 담을 어드바이저를 빈으로 등록
  * ProxyFactoryBean을 등록하면서 프로퍼티에 타깃 빈과 어드바이저 빈을 지정
```java
// 리스트 6-44 트랜잭션 어드바이스 빈 설정
<bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
	<property name="transactionManager" ref="transactionManager" />
</bean>

// 리스트 6-45 포인트컷 빈 설정
<bean id="transactionPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
	<property name="mappedName" value="upgrade*" />
</bean>

// 리스트 6-46 어드바이저 빈 설정
<bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
	<property name="advice" ref="transactionAdvice" />
	<property name="pointcut" ref="transactionPointcut" />
</bean>

// 리스트 6-47 ProxyFactoryBean 설정
<bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
	<property name="target" ref="userServiceImpl" />
	<property name="interceptorNames">
		<list>
			<value>transactionAdvisor</value>
		</list>
	</property>
</bean>
```
```java
// 리스트 6-48 ProxyFactoryBean을 이용한 트랜잭션 테스트
@Test 
@DirtiesContext
public void upgradeAllOrNothing() {
	TestUserService testUserService = new TestUserService(users.get(3).getId());
	testUserService.setUserDao(userDao);
	testUserService.setMailSender(mailSender);
	
	// userService 빈을 스프링의 ProxyFactoryBean으로 변환
	ProxyFactoryBean txProxyFactoryBean = context.getBean("&userService", ProxyFactoryBean.class);
	txProxyFactoryBean.setTarget(testUserService);
	// FactoryBean 타입이므로 동일하게 getObject()를 이용해서 프록시를 가져옴.
	UserService txUserService = (UserService) txProxyFactoryBean.getObject();
	...
}
```

* ProxyFactoryBean은 스프링의 DI와 템플릿/콜백 패턴, 서비스 추상화 등의 기법이 모두 적용됨.
<br>-> 독립적이며, 여러 프록시가 공유할 수 있는 어드바이스와 포인트컷으로 확장 기능 분리 가능.


### 6.5 스프링 AOP

* 서비스 오브젝트에서 분리해낸 부가기능은 투명한 부가기능 형태로 제공되어야 함.
<br>-> 투명하다? 부가기능을 적용한 후에도 기존 설계와 코드에는 영향을 주지 않는 것


### 6.5.1 자동 프록시 생성

* 부가기능의 적용이 필요한 타깃 오브젝트마다 거의 비슷한 내용의 ProxyFactoryBean 빈 설정정보를 추가해주는 문제 존재.
<br>-> 소스 수정은 아니어도 매번 설정을 복사해서 붙이고 target 프로퍼티의 내용을 수정해야 함.

* 중복 문제의 접근 방법
  * 다이내믹 프록시라는 런타임 코드 자동생성 기법을 이용
  * 개발자가 일일이 인터페이스 메서드를 구현하는 프록시 클래스를 만들어서 위임과 부가기능의 코드를 넣지 않아도 됨.
  * 변하지 않는 타깃으로의 위임과 부가기능 적용 여부 판단 부분 -> 다이내믹 프록시 기술로 해결
  * 변하는 부가기능 코드 -> 별도로 만들어서 다이내믹 프록시 생성 팩토리에 DI

* 빈 후처리기를 이용한 자동 프록시 생성기
  * 스프링은 OCP의 가장 중요한 요소인 *유연한 확장*이라는 개념을 스프링 컨테이너에 다양한 방법으로 적용
  * 스프링은 컨테이너로서 제공하는 기능 중 변하지 않는 핵심부분 외에는 대부분 확장할 수 있도록 *확장 포인트를 제공*
  * `BeanPostProcessor` 인터페이스를 구현해서 만든 빈 후처리기
  <br>-> 스프링 빈 오브젝트로 만들어지고 난 후, 빈 오브젝트를 다시 가공할 수 있게 해줌.
  * `DefaultAdvisorAutoProxyCreator`
    * 어드바이저를 이용한 자동 프록시 생성기
    * 빈 후처리기 자체를 빈으로 등록해서 스프링에 적용
    * **스프링은 빈 후처리기가 빈으로 등록되어 있으면 빈 오브젝트가 생성될 때마다 빈 후처리기에 보내서 후처리 작업을 요청**
    * 스프링이 설정을 참고해서 만든 오브젝트가 아닌 다른 오브젝트를 빈으로 등록도 가능.

* 자동 프록시 생성 빈 후처리기
<br>-> 스프링이 생성하는 빈 오브젝트의 일부를 프록시로 포장하고, 프록시를 빈으로 대신 등록 가능
  1. DefaultAdvisorAutoProxyCreator 빈 후처리기 등록되어 있으면
  <br>스프링은 **빈 오브젝트를 만들 때마다 후처리기에 빈을 보냄.**
  2. 빈으로 등록된 모든 어드바이저 내의 포인트컷을 이용해 전달받은 빈이 **프록시 전달대상인지 확인**
  3. 프록시 적용대상이면 내장된 프록시 생성기에 전달받은 빈의 프록시를 만들게 하고, **만든 프록시에 어드바이저를 연결**
  4. 빈 후처리기는 프록시가 생성되면 **빈 오브젝트 대신 프록시 오브젝트를 컨테이너에게 돌려줌.**
  5. 컨테이너는 최종적으로 빈 후처리기가 돌려준 오브젝트를 빈으로 등록하고 사용

* 확장된 포인트컷
  * 메서드 이름뿐만 아니라 적용대상 클래스여부도 확인해서 적용대상을 검토
  * NameMatchMethodPointcut은 메서드 선별 기능만 가진 특별한 포인트 컷
  * Pointcut 선정 기능을 모두 적용한다면 아래 순서로 동작
    1. 프록시를 적용할 클래스인지를 먼저 확인
    2. 적용 대상 클래스인 경우에는 어드바이스를 적용할 메서드인지 확인
```java
// 리스트 6-49 두 가지 기능을 정의한 Pointcut 인터페이스
public interface Pointcut {
	ClassFilter getClassFilter(); // 프록시를 적용할 클래스인지 확인
	MethodMatcher getMethodMatcher(); // 어드바이스를 적용할 메서드인지 확인
}

// 리스트 6-50 확장 포인트컷 테스트
@Test
public void classNamePointcutAdvisor() {
	NameMatchMethodPointcut classMethodPointcut = new NameMatchMethodPointcut() {  
		public ClassFilter getClassFilter() { // 익명 내부 클래스 방식으로 클래스를 정의
			return new ClassFilter() {
				public boolean matches(Class<?> clazz) {
					// 클래스 이름이 HelloT로 시작하는 것만 선정
					return clazz.getSimpleName().startsWith("HelloT");
				}
			};
		}
	};
	classMethodPointcut.setMappedName("sayH*"); // sayH로 시작하는 메서드만 적용

	checkAdviced(new HelloTarget(), classMethodPointcut, true); // 적용 클래스 O

	class HelloWorld extends HelloTarget {};
	checkAdviced(new HelloWorld(), classMethodPointcut, false); // 적용 클래스 X
	
	class HelloToby extends HelloTarget {};
	checkAdviced(new HelloToby(), classMethodPointcut, true); // 적용 클래스 O
}

private void checkAdviced(Object target, Pointcut pointcut, boolean adviced) { 
	ProxyFactoryBean pfBean = new ProxyFactoryBean();
	pfBean.setTarget(target);
	pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
	Hello proxiedHello = (Hello) pfBean.getObject();
	
	if (adviced) {
		assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY")); // 메서드 선정 방식을 통해 어드바이스 적용
		assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY")); // 메서드 선정 방식을 통해 어드바이스 적용
		assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
	}
	else { // 클래스 이름이 적용대상이 아닐 때는 어드바이스 적용 후보에서 아예 제외
		assertThat(proxiedHello.sayHello("Toby"), is("Hello Toby"));
		assertThat(proxiedHello.sayHi("Toby"), is("Hi Toby"));
		assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
	}
}
```


### 6.5.2 DefaulAdvisorAutoProxyCreator의 적용

```java
// 리스트 6-51 클래스 필터가 포함된 포인트컷
public class NameMatchClassMethodPointcut extends NameMatchMethodPointcut {
	// 프로퍼티로 받은 클래스 이름을 이용해서 필터를 만든다.
	public void setMappedClassName(String mappedClassName) {
		this.setClassFilter(new SimpleClassFilter(mappedClassName));
	}
	
	static class SimpleClassFilter implements ClassFilter {
		String mappedName;
		
		private SimpleClassFilter(String mappedName) {
			this.mappedName = mappedName;
		}

		public boolean matches(Class<?> clazz) {
			// 와일드카드(*)가 들어간 문자열 비교를 지원하는 스프링의 유틸리티 메서드
			return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
		}
	}
}
```

* DefaultAdvisorAutoProxyCreator는 등록된 빈 중에서 Advisor 인터페이스를 구현한 것을 모두 찾음.
<br>그리고 생성되는 모든 빈에 대해 어드바이저의 포인트컷을 적용해보면서 프록시 적용 대상을 선정
  * `<bean class="org.springframework.aop.framwork.autoproxy.DefaultAdvisorAutoProxyCreator" />
    * id가 없이 클래스만 적용
    * 다른 빈에서 참조되거나 코드에서 빈 이름으로 조회될 필요가 없는 빈이면 id가 없어도 무방

```java
<bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
	<property name="transactionManager" ref="transactionManager" />
</bean>

<bean id="transactionPointcut" class="springbook.user.service.NameMatchClassMethodPointcut">
	<property name="mappedClassName" value="*ServiceImpl" /> // 클래스 이름 확인
	<property name="mappedName" value="upgrade*" /> // 메서드 이름 확인
</bean>

<bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
	<property name="advice" ref="transactionAdvice" /> // 어드바이스 적용
	<property name="pointcut" ref="transactionPointcut" /> // 포인트컷 적용
</bean>

// 리스트 6-53 프록시 팩토리 빈을 제거한 후의 빈 설정
// 더 이상 명시적인 프록시 팩토리 빈을 등록하지 않기 때문에 userServiceImpl을 userService로 변경
<bean id="userService" class="springbook.user.service.UserServiceImpl">
	<property name="userDao" ref="userDao" />
	<property name="mailSender" ref="mailSender" />
</bean>
```

* 자동 프록시 생성기를 사용하는 테스트
  * @Autowired를 통해 컨텍스트에서 가져오는 UserService 타입 오브젝트는
  <br>UserServiceImpl 오브젝트가 아닌 트랜잭션이 적용된 프록시여야 함.
  * 자동 프록시 생성기를 적용한 후에는 더 이상 가져올 ProxyFactoryBean 같은 팩토리 빈이 없음.
  * 예외상황을 위한 테스트 대상도 빈으로 등록해줘야 함.
```java
// 리스트 6-54 수정한 테스트용 UserService 구현 클래스
public class UserServiceTest {
	...
	// 포인트컷의 클래스 필터에 선정되도록 이름 변경
	static class TestUserServiceImpl extends UserServiceImpl {
		private String id = "madnite1"; // users(3).getId() 테스트 픽스처 user(3)의 ID를 고정
		
		protected void upgradeLevel(User user) {
			if (user.getId().equals(this.id)) throw new TestUserServiceException();  
			super.upgradeLevel(user);  
		}
	}
}

// 리스트 6-55 테스트용 UserService의 등록
<bean id="testUserService"
	class="springbook.user.service.UserServiceTest$TestUserServiceImpl" // 스태틱 멤버 클래스는 $로 지정
	parent="userService" /> // 프로퍼티 정의를 포함해서 userService 빈의 설정을 상속받음
```

* 클래스 이름에 사용한 $ 기호
  * 스태틱 멤버 클래스를 지정할 때 사용
  * 특정 테스트 클래스에서만 사용되는 클래스는 스태틱 멤버 클래스로 정의하는 것이 편리

* parent 애프리뷰트
  * 다른 빈 설정의 내용을 상속 받을 수 있음.
  * 클래스는 물론, 프로퍼티 설정도 모두 상속받음

```java
// 리스트 6-56 testUserService 빈을 사용하도록 수정된 테스트
public class UserServiceTest {
	@Autowired UserService userService; // 디폴트로 사용될 타깃 오브젝트 빈
	// 예외적인 상황에 사용될 타깃 오브젝트 빈
	// 같은 타입의 빈이 두 개 존재하기 때문에 필드 이름을 기준으로 주입될 빈이 결정됨.
	@Autowired UserService testUserService;
	...
	
	@Test 
	public void upgradeAllOrNothing() { // 스프링 컨텍스트의 빈 설정을 변경하지 않으므로 @DirtiesContext 제거
		userDao.deleteAll();			  
		for(User user : users) userDao.add(user);
		
		try {
			testUserService.upgradeLevels(); // 예외용 타깃 오브젝트 사용
			fail("TestUserServiceException expected"); 
		}
		catch(TestUserServiceException e) { 
		}
		
		checkLevelUpgraded(users.get(1), false);
	}
}
```

* 빈 후처리기를 이용해서 프록시 자동생성기를 적용시 확인해야 할 점들
  * 부가 기능이 적용되어야 하는 빈에 제대로 적용됐는지 확인 필요
  * 아무 빈에나 부가기능이 적용된 것은 아닌지 확인 필요
  <br>-> 포인트컷 빈의 클래스 이름 패턴을 변경해서 테스트


### 6.5.3 포인트컷 표현식을 이용한 포인트컷

* 더 복잡하고 세밀한 기준을 이용해 클래스나 메서드를 선정하려면 어떻게 해야할까?
<br>-> 필터나 매처에서 클래스와 메서드의 메타정보를 제공 받으니 어떤 식이든 가능!
  * 리플렉션 API는 코드를 작성하기가 번거로움
  * 리플렉션 API를 이용해 메타정보를 비교하는 방법은 조건이 달라질 때마다 포인트컷 수정 필요

* 포인트컷 표현식
  * `AspectJExpressionPointcut` 클래스를 적용
  * 클래스와 메서드의 선정 알고리즘을 포인트컷 표현식을 이용해 한 번에 적용 가능

* 포인트컷 표현식 문법 = **AspectJ 포인트컷 표현식**
  * AspectJ 포인트컷 표현식은 포인트컷 지시자를 이용해 작성
  <br>-> 대표적으로 `execution()`을 사용
  * `execution([접근제한자 패턴] 타입패턴 [타입패턴.]이름패턴 (타입패턴 | ".", ...) [throws 예외패턴])`
    * [접근제한자 패턴]: public, private와 같은 접근제한자. 생략 가능
    * 타입패턴: 리턴값의 타입패턴
    * [타입패턴.]: 패키지와 클래스 이름에 대한 패턴. 생락 가능. 연결시 . 사용
    * 이름패턴: 메서드 이름패턴
    * (타입패턴 | ".", ...): 파라미터의 파입 패턴을 순서대로 넣을 수 있음
    * [throws 예외 패턴]: 예외 이름 패턴
  * 메서드 시그니처와 동일한 방식으로 입력한다고 보면 됨.

```java
// 리스트 6-61 메서드 시그니처를 이용한 포인트컷 표현식 테스트
@Test
public void methodSignaturePointcut() throws SecurityException, NoSuchMethodException {
	AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
	// Target 클래스 minus() 메서드 시그니처
	pointcut.setExpression("execution(public int " +
		"springbook.learningtest.spring.pointcut.Target.minus(int,int)) " +
		"throws java.lang.RuntimeException)");
	
	// 클래스 필터와 메서드 매처를 가져와 각각 비교
	assertThat(pointcut.getClassFilter().matches(Target.class) &&
			   pointcut.getMethodMatcher().matches(
				  Target.class.getMethod("minus", int.class, int.class), null), is(true));
	
	assertThat(pointcut.getClassFilter().matches(Target.class) &&
			   pointcut.getMethodMatcher().matches(
				  Target.class.getMethod("plus", int.class, int.class), null), is(false));

	assertThat(pointcut.getClassFilter().matches(Bean.class) &&
			pointcut.getMethodMatcher().matches(
					Target.class.getMethod("method"), null), is(false));
}
```

* 포인트컷 표현식 종류
  * `execution(int minus(int, int));`
  <br>int 리턴 타입. minus라는 메서드명, 2개의 int 파라미터를 가진 모든 메서드를 허용하는 포인트컷 표현식
  * `execution(* minus(int, int));`
  <br>위와 동일하지만 리턴 타입을 상관하지 않는 포인트컷 표현식
  * `execution(* *(..));`
  <br>리턴 타입, 메서드명, 파라미터에 상관없이 모든 메서드에 적용되는 포인트컷 표현식

* 포인트컷 표현식을 이용하는 포인트컷 적용
  * AspectJ 포인트컷 표현식은 메서드를 선정하는데 편리하게 쓸 수 있는 강력한 표현식 언어
  * `bean()`: 빈의 이름으로 적용 대상을 선정하는 포인터컷 지시자
  * 특정 애노테이션이 타입, 메서드, 파라미터에 적용되어 있는 것을 보고 메서드를 선정하는 포인트컷도 생성 가능
  <br>`@annotation(org.springframework.transaction.annotation.Transactional)`
  <br>-> @Transactional 애노테이션이 적용된 메서드를 선정
  * 포인트컷 표현식을 사용하면 로직이 짧은 문자열에 담기기 때문에 클래스나 코드를 추가할 필요 없음
  <br>-> 코드와 설정이 단순해짐!
  * 반면 문자열로 된 표현식이므로 런타임 시점까지 문법의 검증이나 기능 확인이 되지 않는다는 단점도 존재.

```java
// 리스트 6-65 포인트컷 표현식을 사용한 빈 설정
// 클래스 이름에 적용되는 패턴은 클래스 이름 패턴이 아닌 타입 패턴
// TestUserService 클래스는 비록 이름은 다르지만 타입을 따져보면
// TestUserService 클래스이자 슈퍼 클래스인 UserServiceImpl, 구현 인터페이스인 UserService 3가지가 모두 적용
// TestUserService 클래스로 정의된 빈은 UserServiceImpl 타입이기 때문에 타입 패턴 조건을 충족
<bean id="transactionPointcut" class="org.springframework.aop.aspectj.AspectJExpressionPointcut">
	<property name="expression" value="execution(* *..*ServiceImpl.upgrade*(..))" />
</bean>
```


### 6.5.4 AOP란 무엇인가?

* 비즈니스 로직을 담은 UserService에 트랜잭션을 적용해온 과정 정리
  * 트랜잭션 서비스 추상화
    * 트랜잭션 경계설정 코드를 비즈니스 로직을 담은 코드에 추가 시 특정 트랜잭션 기술에 종속되는 코드가 되는 문제 존재
    * 트랜잭션 적용이라는 추상적인 작업 내용은 유지한 채로 구체적인 구현 방법을 자유롭게 바꿀 수 있도록 서비스 추상화 기법을 적용
    * **인터페이스와 DI를 통해 기능을 분리**
  * 프록시와 데코레이션 패턴
    * 추상화를 통해 코드에서는 제거했지만 비즈니스 로직 코드에는 트랜잭션을 적용하고 있다는 사실은 드러나는 문제 존재
    * 비즈니스 로직 클래스와 동일한 인터페이스를 구현한 **프록시와 DI를 통해 데코레이터 패턴을 적용**
    * 클라이언트 -> 데코레이터(프록시 / 트랜잭션 처리 코드) -> 타깃 클래스
    * 비즈니스 로직 코드를 트랜잭션과 같이 성격이 다른 코드에서부터 분리
    <br>-> 독립적으로 로직을 검증하는 고립된 단위 테스트 생성 가능
  * 다이내믹 프록시와 프록시 팩토리 빈
    * 트랜잭션 기능을 부여하지 않아도 되는 메서드조차 프록시로서 위임 기능이 필요하기 때문에 일일이 다 구현해야되는 문제 존재
    * 프록시 클래스 없이도 프록시 오브젝트를 런타임 시에 만들어주는 JDK 다이내믹 프록시 기술을 적용
    * 프록시 기술을 추상화한 스프링의 **프록시 팩토리 빈**을 이용해서 다이내믹 프록시 생성 방법에 DI를 도입
    * 스프링 프록시 팩토리 빈 사용 시 **어드바이스와 포인트컷을 프록시에서 분리**하여 여러 프록시에 공유 가능
  * 자동 프록시 생성 방법과 포인트 컷
    * 트랜잭션 적용 대상이 되는 빈마다 일일이 프록시 팩토리 빈을 설정해줘야 하는 문제 존재
    * 스프링 컨테이너의 **빈 생성 후처리 기법**을 활용해 컨테이너 초기화 시점에서 자동으로 프록시를 생성하는 방법 도입
    * 포인트컷을 분리하여 부가기능을 어디에 적용할지 결정하도록 구성
    * **AspectJ 포인트컷 표현식**을 활용하여 간단한 설정으로 적용 대상을 선택할 수 있게 구성
  * 부가기능의 모듈화
    * 관심사가 같은 코드를 분리해 한데 모으는 것은 SW 개발의 가장 기본이 되는 원칙
    * 트랜잭션 경계설정 기능은 다른 모듈의 코드에 부가적으로 부여되는 기능
    * 트랜잭션 같은 부가기능은 핵심기능과 같은 방식으로는 모듈화하기가 매우 어려움.
    * 독립적인 방식으로 존재해서는 적용되기 어려움. 타깃이 존재해야만 의미가 있음.

* AOP: 애스펙트 지향 프로그래밍 (= 관점 지향 프로그래밍)
  * 애스팩트 (Aspect)
  <br>애플리케이션의 핵심기능을 담고 있지는 않지만, 애플리케이션을 구성하는 중요한 한 가지 요소이고,
  <br>핵심기능에 부가되어 의미를 갖는 특별한 모듈
  * 애스펙트 = 어드바이스 + 포인트컷
  * 런타임 시 각 부가기능 애스펙트는 필요한 위치에 다이내믹하게 참여
  * 애스펙트 지향 프로그래밍 (Aspect Oriented Programming)
  <br>애플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 애스펙트라는 모듈로 만들어서 설계/개발하는 방법
  * AOP는 OOP를 돕는 보조적인 기술이지 OOP를 대체하는 기술은 아님.
  * AOP는 결국 애플리케이션을 다양한 측면에서 독립적으로 모델링하고 설계하고 개발할 수 있도록 만들어줌.


### 6.5.5 AOP 적용기술

* 프록시를 이용한 AOP
  * AOP의 핵심은 프록시를 이용한다는 것
  * 프록시로 만들어서 DI로 연결된 빈 사이에 적용. 타깃의 메서드 호출 과정에 참여해서 부가기능을 제공
  * 스프링 AOP는 자바의 기본 JDK와 스프링 컨테이너 외에는 특별한 기술이나 환경을 요구하지 않음
  * 어드바이스가 적용되는 대상은 오브젝트의 메서드
  * 어드바이스가 구현하는 `MethodInterceptor` 인터페이스
  <br>프록시로부터 메서드 요청정보를 전달받아 타깃 오브젝트의 메서드를 호출
  * 독립적으로 개발한 부가기능 모듈을 다양한 타깃 오브젝트의 메서드에 다이내믹하게 적용해주기 위해 프록시를 사용!

* 바이트코드 생성과 조작을 통한 AOP
  * AspectJ는 프록시를 사용하지 않는 대표적인 AOP 기술
  * AspectJ는 타깃 오브젝트를 뜯어고쳐서 부가기능을 직접 넣어주는 직접적인 방법을 사용
  <br>-> 컴파일된 타깃의 클래스 파일 자체를 수정하거나 클래스가 JVM에 로딩되는 시점에 바이트코드를 조작
  * AspectJ가 바이트코드 수정방식을 사용하는 이유?
    * 스프링과 같은 DI 컨테이너의 도움을 받아서 자동 프록시 생성 방식을 사용하지 않아도 AOP 적용 가능
    * 프록시 방식보다 훨씬 강력하고 유연한 AOP가 가능


### 6.5.6 AOP의 용어

* 타깃
  * 부가기능을 부여할 대상
  * 핵심 기능을 담은 클래스 혹은 다른 부가기능을 제공하는 프록시 오브젝트

* 어드바이스
  * 타깃에게 제공할 부가기능을 담은 모듈
  * 오브젝트 혹은 메서드 레벨에서 정의 가능

* 조인 포인트
  * 어드바이스가 적용될 수 있는 위치
  * 스프링의 프록시 AOP에서 조인 포인트는 메서드의 실행 단계뿐

* 포인트 컷
  * 어드바이스를 적용할 조인 포인트를 선별하는 작업 또는 그 기능을 정의한 모듈
  * 포인트컷 표현식은 메서드의 실행이라는 의미인 execution으로 시작

* 프록시
  * 클라이언트와 타깃 사이에 투명하게 존재하면서 부가기능을 제공하는 오브젝트
  * DI를 통해 타깃 대신 클라이언트에 주입되며, 클라이언트의 메서드 호출을 대신 받아서 타깃에 위임해주며 부가기능 구현

* 어드바이저
  * 포인트컷과 어드바이스를 하나씩 갖고 있는 오브젝트
  * AOP의 가장 기본이 되는 모듈
  * 스프링 AOP에서만 사용되는 특별한 용어. 일반적인 AOP에서는 사용되지 않음

* 애스펙트
  * OOP의 클래스와 같이 AOP의 기본 모듈
  * 한 개 또는 그 이상의 포인트컷과 어드바이스의 조합. 보통 싱글톤 형태의 오브젝트로 존재
  * 스프링의 어드바이저는 아주 단순한 애스펙트


### 6.5.7 AOP 네임스페이스

* 스프링 AOP를 적용하기 위해 추가한 어드바이저, 포인트컷 같은 빈들은
<br>스프링 컨테이너에 의해 자동으로 인식되어 특별한 작업을 위해 사용됨.
<br>스프링 프록시 방식 AOP를 적용하려면 최소한 4가지 빈을 등록해야 함.
  * 자동 프록시 생성기
    * 스프링의 `DefaulAdvisorAutoProxyCreator` 클래스를 빈으로 등록
    * 다른 빈을 DI하지도 않고 자신도 DI되지 않으며 독립적으로 존재 = id도 불필요
  * 어드바이스
    * 부가기능을 구현한 클래스를 빈으로 등록
    * `TransactionAdvice`는 AOP 관련 빈 중에서 유일하게 직접 구현한 클래스를 사용
  * 포인트컷
    * 스프링의 `AspectJExpressionPointcut`을 빈으로 등록
    * expression 프로퍼티에 포인트컷 표현식을 넣어줌
  * 어드바이저
    * 스프링의 `DefaulPointcutAdvisor` 클래스를 빈으로 등록

* AOP 네임스페이스
  * 스프링은 AOP와 관련된 태그를 정의해둔 aop 스키마를 제공
```java
// 리스트 6-66 aop 네임스페이스 선언
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
						http://www.springframework.org/schema/aop 
						http://www.springframework.org/schema/aop/spring-aop-3.0.xsd>
	...
</beans>

// 리스트 6-67 aop 네임스페이스를 적용한 AOP 설정 빈
<aop:config> // AOP 설정을 담는 부모 태그. 필요에 따라 AspectJAdvisorAutoProxyCreator를 빈으로 등록
	// expression의 표현식을 프로퍼티로 가진 AspectJExpressionPointcut을 빈으로 등록
	<aop:pointcut id="transactionPointcut" expression="execution(* *..*ServiceImpl.upgrade*(..))" />
	// advice와 pointcut의 ref를 프로퍼티로 갖는 DefaultBeanFactoryPointcutAdvisor를 등록
	<aop:advisor advice-ref="transactionAdvice" pointcut-ref="transactionPointcut" />
</aop:config>
```

* 어드바이저 내장 포인트컷
  * AspectJ 포인트컷 표현식을 활용하는 포인트컷은
  <br>String으로 된 표현식을 담은 expression 프로퍼티 하나만 설정해주면 사용 가능
  * 포인트컷은 어드바이저에 참조돼야만 사용됨.
  * 하나의 포인트컷을 여러 개의 어드바이저에서 공유하려 할 경우 포인트컷을 독립적인 `<aop:pointcut>` 태그로 등록
  * 포인트컷을 내장하는 경우 `<aop:advisor>` 태그 하나로 2개의 빈이 등록됨.
```java
// 리스트 6-68 포인트컷을 내장한 어드바이저 태그
<aop:config>
	<aop:advisor advice-ref="transactionAdvice" pointcut="execution(* *..*ServiceImpl.upgrade*(..))" />
</aop:config>
```


### 6.6 트랜잭션 속성


### 6.6.1 트랜잭션 정의

* 트랜잭션 전파 (Transaction Propagation)
  * 트랜잭션의 경계에서 이미 진행 중인 트랜잭션이 있을 때 또는 없을 때 어떻게 동작할 것인가를 결정하는 방식

* 트랜잭션 전파 속성
  * PROPAGATION_REQUIRED
    * 가장 많이 사용되는 트랜잭션 전파 속성
    * 진행 중인 트랜잭션이 없으면 새로 시작하고, 이미 시작된 트랜잭션이 있으면 이에 참여
    * `DefaultTransactionDefinition`의 트랜잭션 전파 속성
  * PROPAGATION_REQUIRES_NEW
    * 항상 새로운 트랜잭션을 시작
    * 독립적인 트랜잭션이 보장돼야 하는 코드에 적용 가능
  * PROPAGATION_NOT_SUPPORTED
    * 트랜잭션 없이 동작하도록 설정
    * 특별한 메서드만 트랜잭션 속성에서 제외하려 할 때 사용
    * 모든 메서드에 AOP가 적용되게 하고, 특정 메서드의 트랜잭션 전파 속성만 설정해서 트랜잭션 없이 동작하도록 설정

* 격리수준 (Isolation level)
  * 모든 DB 트랜잭션은 격리수준을 갖고 있어야 함.
  * 적절하게 격리수준을 조정해서 가능한 한 많은 트랜잭션을 동시에 진행시키면서도 문제가 발생하지 않게 하는 제어가 필요
  * 기본적으로 DB에 설정되어 있지만 JDBC 드라이버나 DataSource 등에서 재설정 가능.
  * 필요하다면 트랜잭션 단위로 격리수준 조정 가능

* 제한시간 (Timeout)
  * 트랜잭션을 수행하는 제한시간을 설정 가능
  * `DefaultTransactionDefinition`의 기본 설정은 제한시간이 없음.
  * PROPAGATION_REQUIRED, PROPAGATION_REQUIRES_NEW와 같이 사용해야 의미가 있음.

* 읽기전용 (Read only)
  * 트랜잭션 내에서 데이터 조작 시도를 막아줄 수 있음.


### 6.6.2 트랜잭션 인터셉터와 트랜잭션 속성

* 메서드 별로 다른 트랜잭션 정의를 적용하려면 어드바이스의 기능을 확장해야 함.

* TransactionInterceptor 어드바이스
  * TransactionAdvice와 작동 방식은 동일
  * 트랜잭션 정의를 메서드 이름 패턴을 이용해서 다르게 지정할 수 있는 방법을 추가로 제공
  * 프로퍼티 타입: PlatformTransactionManager, Properties
  * `TransactionAttribute`
    * Properties 타입의 2번째 프로퍼티
    * 트랜잭션 속성을 정의한 프로퍼티
  * TransactionInterceptor에는 2가지 종류의 예외 처리 방식이 존재
    * 런타임 예외가 발생하면 트랜잭션은 롤백
    * 타깃 메서드가 런타임 예외가 아닌 체크 예외를 던지는 경우 의미 있는 리턴으로 판단하고 트랜잭션 커밋
  * TransactionAttribute에 `rollback()` 속성을 둬서 기본 원칙과 다른 예외처리가 가능하게해 줌.
  <br>이를 활용하면 특정 체크 예외의 경우는 트랜잭션을 롤백하고 특정 런타임 예외에 대해서는 트랜잭션 커밋 가능

* 메서드 이름 패턴을 이용한 트랜잭션 속성 지정
  * 트랜잭션 속성은 문자열로 정의 가능
  <br>PROPAGATION_NAME, ISOLATION_NAME, readOnly, timout_NNNN, -Exception1, +Exception2
    * PROPAGATION_NAME: 트랜잭션 전파 방식. PROPAGATION_ 으로 시작 **(필수)**
    * ISOLATION_NAME: 격리수준. ISOLATION_ 으로 시작
    * readOnly: 읽기전용 항목. 디폴트는 읽기 전용이 아님
    * timeout_NNNN: 제한시간. timeout_ 으로 시작하고 **초 단위 시간**을 뒤에 붙인다.
    * -Exception1: 체크 예외 중에서 롤백 대상으로 추가할 것들. 한 개 이상 등록 가능
    * +Exception2: 런타임 예외지만 롤백시키지 않을 예외들. 한 개 이상 등록 가능
```java
// 리스트 6-71 트랜잭션 속성 정의 예
<bean id='transactionAdvice" class="org.springframework.transaction.interceptor.TransactionInterceptor">
	<property name="transactionManager" ref="transactionManager" />
	<property name=transactionAttributes">
		<props>
			<prop key="get*">PROPAGATION_REQUIRED,readOnly,timeout_30</prop>
			<prop key="upgrade*">PROPAGATION_REQUIRED_NEW,ISOLATION_SERIALIZABLE</prop>
			<prop key="*">PROPAGATION_REQUIRED</prop>
		</props>
	</property>
</bean>
```

* 리스트 6-71 분석
  * `<prop key="get*">`: 이름이 get으로 시작하는 모든 메서드에 대한 속성
  * `<prop key="upgrade*">`: 이름이 upgrade로 시작하는 메서드는 항상 독립적인 트랜잭션으로 실행
  * `<prop key="*">`: 그 외의 모든 메서드에 대한 속성 지정

* tx 네임스페이스를 이용한 설정 방법
```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:aop="http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx"
	xsi:schemaLocation="http://www.springframework.org/schema/beans 
						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
						http://www.springframework.org/schema/aop 
						http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
						http://www.springframework.org/schema/tx 
						http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">
	...
	// 트랜잭션 매니저의 빈 아이디가 transactionManager라면 transaction-manager 속성 생략 가능
	<tx:advice id="transactionAdvice" transaction-manager="transactionManager">
		<tx:attributes>
			<tx:method name="get*" propagation="REQUIRED" read-only="true" timeout="30" />
			<tx:method name="upgrade*" propagation="REQUIRED_NEW" isolation="SERIALIZABLE" />
			<tx:method name="*" propagation="REQUIRED" />
		</tx:attributes> 
	</tx:advice>
	...
</beans>
```


### 6.6.3 포인트컷과 트랜잭션 속성의 적용 전략

* 포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 몇 가지 전략
  * 트랜잭션 포인트컷 표현식은 타입 패턴이나 빈 이름을 이용한다.
