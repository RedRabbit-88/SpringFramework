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
```java
// 리스트 6-8 목 오브젝트 설정이 필요한 테스트 코드 수정
@Test
public void upgradeLevels() throws Exception {
	...
	MockMailSender mockMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
}
```
