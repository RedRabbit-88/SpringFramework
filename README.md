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

* 환경과 상황에 따라서 기술이 바뀌고, 그에 따라 다른 API를 사용하고 다른 스타일의 접근 방법을 따라야 한다?
<br>-> 매우 피곤하며 비효율적! 그래서 추상화라는 개념이 필요!


### 5.1 사용자 레벨 관리 기능 추가

* UserDao를 다수의 회원이 가입할 수 있는 인터넷 서비스의 사용자 관리 모듈에 적용해보자.
  * 사용자의 레벨은 BASIC, SILVER, GOLD 3가지 중 하나
  * 사용자가 처음 가입하면 BASIC 레벨이 되며, 이후 활동에 따라 한 단계씩 업그레이드
  * 가입 후 50회 이상 로그인 하면 BASIC에서 SILVER 레벨이 된다.
  * SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다.
  * 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행됨.

### 5.1.1 필드 추가

* Level enum (이늄)
  * DB에서 상태값으로 관리하기에는 문자열이라 번거로움
  * 상수값으로 사용자 레벨을 관리하는 건 직관적이지 않고 위험
  * 내부에는 DB에 저장할 int 타입의 값을 갖고 있지만 겉으로는 Level 타입을 사용

```java
// 리스트 5-3 사용자 레벨용 이늄
package springbook.user.domain;

public enum Level {
	BASIC(1), SILVER(2), GOLD(3); // 3개의 이늄 오브젝트 정의

	private final int value;
		
	Level(int value) { // DB에 값을 넣어줄 생성자
		this.value = value;
	}

	public int intValue() { // 값을 가져오는 메서드
		return value;
	}
	
	public static Level valueOf(int value) { // 값으로부터 Level 타입 오브젝트를 가져오는 스태틱 메서드
		switch(value) {
		case 1: return BASIC;
		case 2: return SILVER;
		case 3: return GOLD;
		default: throw new AssertionError("Unknown value: " + value);
		}
	}
}
```

```java
// 리스트 5-4 User에 추가된 필드
package springbook.user.domain;

public class User {
	...
	Level level;
	int login;
	int recommend;
	
	public User() {
	}
	
	public User(String id, String name, String password, Level level,
			int login, int recommend) {
		this.id = id;
		this.name = name;
		this.password = password;
		this.level = level;
		this.login = login;
		this.recommend = recommend;
	}
	...
	public Level getLevel() {
		return level;
	}

	public void setLevel(Level level) {
		this.level = level;
	}
	...
}

// 리스트 5-5 수정된 테스트 픽스처
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserDaoTest {
	...
	@Before
	public void setUp() {
		this.user1 = new User("gyumee", "박성철", "springno1", Level.BASIC, 1, 0);
		this.user2 = new User("leegw700", "이길원", "springno2", Level.SILVER, 55, 10);
		this.user3 = new User("bumjin", "박범진", "springno3", Level.GOLD, 100, 40);
	}
	...
}
```

* 추가된 필드를 위해 UserDaoJdbc 수정이 필요

```java
// 리스트 5-9 추가된 필드를 위한 UserDaoJdbc의 수정 코드
public class UserDaoJdbc implements UserDao {
	...
	private RowMapper<User> userMapper = 
		new RowMapper<User>() {
				public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				User user = new User();
				user.setId(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				user.setLevel(Level.valueOf(rs.getInt("level"))); // int 타입 값을 받아서 Level로 변환
				user.setLogin(rs.getInt("login"));
				user.setRecommend(rs.getInt("recommend"));
				return user;
			}
		};
	...
  
	// DB에 입력할 때는 Level을 int 타입으로 변경해서 입력
	public void add(User user) {
		this.jdbcTemplate.update(
				"insert into users(id, name, password, level, login, recommend) " +
				"values(?,?,?,?,?,?)", 
					user.getId(), user.getName(), user.getPassword(), 
					user.getLevel().intValue(), user.getLogin(), user.getRecommend());
	}
}
```


### 5.1.2 사용자 수정 기능 추가

* 수정 기능 테스트 추가
  * 추가된 Level, Login, Recommend가 정상적으로 작동하는지 확인하도록 테스트 메서드 보완
```java
// 리스트 5-10 사용자 정보 수정 메서드 테스트
@Test
public void update() {
	dao.deleteAll();
	dao.add(user1);
	
	// 픽스처에 들어 있는 정보를 변경해서 수정 메서드 호출
	user1.setName("오민규");
	user1.setPassword("springno6");
	user1.setLevel(Level.GOLD);
	user1.setLogin(1000);
	user1.setRecommend(999);
	dao.update(user1);
	
	User user1update = dao.get(user1.getId());
	checkSameUser(user1, user1update);
}
```

* UserDao와 UserDaoJdbc 수정
  * 각각 update() 메서드를 추가 및 구현
```java
// 리스트 5-11 update() 메서드 추가
public interface UserDao {
	...
	public void update(User user);
}

// 리스트 5-12 사용자 정보 수정용 update() 메서드
public void update(User user) {
	this.jdbcTemplate.update(
		"update users set name = ?, password = ?, level = ?, login = ?, " +
		"recommend = ? where id = ? ", user.getName(), user.getPassword(), 
		user.getLevel().intValue(), user.getLogin(), user.getRecommend(), user.getId());
}
```

* 수정 테스트 보완
  * 기존의 update() 테스트는 수정할 row의 데이터가 변경되었는지만 확인함.
  * 수정하지 않아야 할 row의 내용이 그대로 남아있는지도 확인 필요.
  * 보완방법
    * JdbcTemplate의 update()가 돌려주는 리턴값을 확인(업데이트한 row의 갯수)
    * 테스트를 보강해서 원하는 사용자 외의 정보는 변경되지 않았음을 확인
```java
// 리스트 5-13 보완된 update() 테스트
@Test
public void update() {
	dao.deleteAll();
		
	dao.add(user1);		// 수정할 사용자
	dao.add(user2);		// 수정하지 않을 사용자
		
	user1.setName("오민규");
	user1.setPassword("springno6");
	user1.setLevel(Level.GOLD);
	user1.setLogin(1000);
	user1.setRecommend(999);
		
	dao.update(user1);
		
	User user1update = dao.get(user1.getId());
	checkSameUser(user1, user1update);
	User user2same = dao.get(user2.getId());
	checkSameUser(user2, user2same);
}
```


### 5.1.3 UserService.upgradeLevels()

* 사용자 관리 로직은 어디에 두는 것이 좋을까?
  * UserDao에 놓기에는 적당하지 않음
  * DAO는 데이터 관리를 담당하지 비즈니스 로직을 두는 곳이 아님.
  <br>-> **비즈니스 로직을 담을 클래스를 추가! (UserService)**

* UserService 클래스
  * 비즈니스 로직을 담당하는 클래스
  * UserDao의 구현 클래스가 변경되도 영향받지 않도록 해야함.
  * 데이터 액세스 로직이 변경되었다고 비즈니스 로직 코드를 수정하면 안 됨.
  <br>-> **DAO의 인터페이스를 사용하고 DI를 적용해야 한다.**

* UserService 클래스와 빈 등록
```java
// 리스트 5-14 UserService 클래스
public class UserService {
	private UserDao userDao;

	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}

// 리스트 5-15 userService 빈 설정
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
</bean>

<bean id="userDao" class="springbook.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
</bean>
```

* UserServiceTest 테스트 클래스
  * 테스트 클래스에서 UserService 사용을 위해 `@AutoWired` 적용
```java
// 리스트 5-16 UserServiceTest 클래스
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
	@Autowired UserService userService;
}
```

* upgradeLevels() 메서드 생성 및 테스트
  * 요구사항에 맞게 upgradeLevels() 메서드를 작성
  * 테스트를 위해 setUp() 메서드에 테스트 데이터 생성
```java
// 리스트 5-20 사용자 레벨 업그레이드 테스트
@Test
public void upgradeLevels() throws Exception {
	userDao.deleteAll();
	for(User user : users) userDao.add(user);
		
	userService.upgradeLevels();
		
	checkLevel(users.get(0), Level.BASIC);
	checkLevel(users.get(1), Level.SILVER);
	checkLevel(users.get(2), Level.SILVER);
	checkLevel(users.get(3), Level.GOLD);
	checkLevel(users.get(4), Level.GOLD);
}

private void checkLevel(User user, Level expectedLevel) {
	User userUpdate = userDao.get(user.getId());
	assertThat(userUpdate.getLevel(), is(expectedLevel));
}
```


### 5.1.4 UserService.add()

* User를 처음 생성할 때 Level.BASIC으로 설정하는 로직은 어디에 담아야 할까?
  * UserDaoJdbc.add() 메서드는 적합하지 않다. 비즈니스 로직을 DAO에 심으면 안 됨.
  * 사용자 관리 로직을 담고 있는 UserService가 적절함.


### 5.1.5 코드 개선

* 작성된 코드를 살펴볼 떄 중점적으로 봐야할 것들
  * 코드에 중복된 부분은 없는가?
  * 코드가 무엇을 하는 건지 이해하기 불편하지 않은가?
  * 코드가 자신이 있어야 할 자리에 있는가?
  * 앞으로 변경이 일어난다면 어떤 것이 있을 수 있고, 그 변화에 쉽게 대응할 수 있게 작성되었는가?

* upgradeLevels() 리팩토링
  * 난잡한 if/else-if/else 구문을 canUpgradeLevel() 메서드로 분리
  * upgradeLevel() 메서드를 알기 쉽게 변경
  * 각 오브젝트와 메서드가 각각 자기 몫의 책임을 맡아 일을 하는 구조로 변경
  * 객체지향적인 코드는 다른 오브젝트의 데이터를 가져와서 작업하지 않고
  <br>데이터를 갖고 있는 다른 오브젝트에게 작업을 요청함.

```java
// 리스트 5-23 기본 작업 흐름만 남겨준 upgradeLevels()
public void upgradeLevels() throws Exception {
	List<User> users = userDao.getAll();
	for (User user : users) {
		if (canUpgradeLevel(user)) {
			upgradeLevel(user);
		}
	}
}

// 리스트 5-26 업그레이드 순서를 담고 있도록 수정한 Level
public enum Level {
	GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);  
	
	private final int value;
	private final Level next; // 다음 단계의 레벨 정보를 스스로 갖고 있도록 수정
	
	Level(int value, Level next) {  
		this.value = value;
		this.next = next; 
	}
	
	public int intValue() {
		return value;
	}
	
	public Level nextLevel() { 
		return this.next;
	}
	
	public static Level valueOf(int value) {
		switch(value) {
		case 1: return BASIC;
		case 2: return SILVER;
		case 3: return GOLD;
		default: throw new AssertionError("Unknown value: " + value);
		}
	}
}

// 리스트 5-27 User의 레벨 업그레이드 작업용 메서드
package springbook.user.domain;

public class User {
	...
	public void upgradeLevel() {
		Level nextLevel = this.level.nextLevel();	
		if (nextLevel == null) { 								
			throw new IllegalStateException(this.level + "은 업그레이드가 불가 합니다.");
		}
		else {
			this.level = nextLevel;
		}	
	}
}

// 리스트 5-28 간결해진 UserService의 upgradeLevel() 메서드
public class UserService {
	...
	protected void upgradeLevel(User user) {
		user.upgradeLevel();
		userDao.update(user);
	}
}

// 리스트 5-30 개선한 upgradeLevels() 테스트
@Test
public void upgradeLevels() throws Exception {
	userDao.deleteAll();
	for(User user : users) userDao.add(user);
		
	userService.upgradeLevels();
		
	checkLevelUpgraded(users.get(0), false);
	checkLevelUpgraded(users.get(1), true);
	checkLevelUpgraded(users.get(2), false);
	checkLevelUpgraded(users.get(3), true);
	checkLevelUpgraded(users.get(4), false);
}

private void checkLevelUpgraded(User user, boolean upgraded) {
	User userUpdate = userDao.get(user.getId());
	if (upgraded) {
		assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
	}
	else {
		assertThat(userUpdate.getLevel(), is(user.getLevel()));
	}
}
```


### 5.2 트랜잭션 서비스 추상화


### 5.2.1 모 아니면 도

* DB Connection을 이용한 작업을 할 때는 관련된 SQL들이 모두 하나의 트랜잭션으로 묶여있어야 함.

* 트랜잭션 테스트용 UserService 확장 클래스를 어떻게 만들까?
<br>-> 별도의 클래스보다 UserService 클래스를 상속받아 일부 메서드를 오버라이딩.
<br>-> 단, 상속을 위한 메서드는 private가 아닌 protected 접근자로 변경이 필요.
```java
// 리스트 5-36 예외 발생 시 작업 취소 여부 테스트
@Test
public void upgradeAllOrNothing() throws Exception {
	UserService testUserService = new TestUserService(users.get(3).getId());  
	testUserService.setUserDao(this.userDao); // UserDao를 수동으로 DI
	testUserService.setDataSource(this.dataSource);
	 
	userDao.deleteAll();			  
	for(User user : users) userDao.add(user);
	
	try {
		// TestUserService 업그레이드 작업 중 예외 발생시킴.
		testUserService.upgradeLevels();   
		fail("TestUserServiceException expected"); 
	}
	catch(TestUserServiceException e) { 
	}
	
	// 예외가 발생하기 전에 레벨 변경이 있었던 사용자의 레벨이 처음 상태로 변경됐는지 확인
	// 트랜잭션으로 관리되지 않았기 때문에 테스트 실패
	checkLevelUpgraded(users.get(1), false);
}
```


### 5.2.2 트랜잭션 경계설정

* SQL을 처리할 때는 하나의 트랜잭션이 완료된 후에 commit이나 rollback을 진행해야 함.

* **트랜잭션의 경계 설정 (transaction demarcation)**
<br>`setAutoCommit(false)`로 트랜잭션의 시작을 선언하고 `commit()` 또는 `rollback()`으로 트랜잭션을 종료하는 방법

* **로컬 트랜잭션 (local transaction)**
<br>하나의 DB 커넥션 안에서 만들어지는 트랜잭션
```java
// 리스트 5-37 트랜잭션을 사용한 JDBC 코드
Connection c = dataSource.getConnection();

c.setAutoCommit(false); // 트랜잭션 시작
try {
	PreparedStatement st1 = c.prepareStatement("update users ...");
	st1.executeUpdate();
	
	PreparedStatement st2 = c.prepareStatement("update users ...");
	st2.executeUpdate();
	
	c.commit(); // 트랜잭션 커밋
} catch (Exception) {
	c.rollback() // 트랜잭션 롤백
}
```

* 만약 DAO에 트랜잭션 경계 설정 로직이 들어간다면 템플릿 메서드 호출 시마다 DB 커넥션이 생성됨.
  * 여러 작업을 하나의 트랜잭션으로 묶을 수 없음!
  * 어떤 일련의 작업이 하나의 트랜잭션으로 묶이려면 DB 커넥션도 하나만 사용해야 함.

* 비즈니스 로직 내의 트랜잭션 경계설정
  * 여러 작업을 하나로 묶으려면 UserService에서 트랜잭션 경계설정을 담당해야 함.

* Connection 오브젝트를 UserService에서 UserDao로 던져주면 어떨까?
  * Connection 정보가 드러나고 만약 DB 커넥션이 변경되면 일일이 다 바꿔줘야 함!
  * DB 커넥션을 비롯한 리소스의 깔끔한 처리를 가능하게 했던 JdbcTemplate 사용 불가
  * UserService 메서드에 Connection 파라미터가 더 추가됨.
  * Connection 파라미터가 UserDao 인터페이스 메서드에 추가되면 더 이상 데이터 액세스 기술에 독립적이지 않음.
  * DAO 메서드에 Connection 파라미터를 받게 하면 테스트 코드에도 영향을 끼침.

* 스프링은 트랜잭션 동기화 (transaction synchronization) 방법을 제안함.
  * UserService에서 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관
  * 이후 호출되는 DAO의 메서드에서는 저장된 Connection을 가져다가 사용하도록 함.
  * 트랜잭션 동기화 저장소는 작업 스레드마다 독립적으로 Connection 오브젝트를 저장하고 관리
  <br>-> 다중 사용자를 처리하는 서버의 멀티스레드 환경에서도 OK!
```java
public void upgradeLevels() throws Exception {
	// 트랜잭션 동기화 관리자를 이용해 동기화 작업을 초기화
	TransactionSynchronizationManager.initSynchronization();  
	// DB 커넥션을 생성하고 트랜잭션을 시작. 이후의 DAO 작업은 모두 이 트랜잭션으로 처리
	Connection c = DataSourceUtils.getConnection(dataSource); 
	c.setAutoCommit(false);
	
	try {									   
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
		c.commit(); // 정상적으로 작업을 마치면 트랜잭션 커밋
	} catch (Exception e) {    
		c.rollback(); // 예외가 발생하면 롤백
		throw e;
	} finally {
		// 스프링 유틸리티 메서드를 이용해 Connection을 안전하게 close
		DataSourceUtils.releaseConnection(c, dataSource);	
		// 동기화 작업 종료 및 정리
		TransactionSynchronizationManager.unbindResource(this.dataSource);  
		TransactionSynchronizationManager.clearSynchronization();  
	}
}
```

### 5.2.4 트랜잭션 서비스 추상화

* 기술과 환경에 종속되는 트랜잭션 경계설정 코드
  * 한 개 이상의 DB로의 작업을 하나의 트랜잭션으로 만드는 건 JDBC의 Connection을 이용한 트랜잭션 방식인 로컬 트랜잭션으로는 불가능
  * 별도의 트랜잭션 관리자를 통해 트랜잭션을 관리하는 글로벌 트랜잭션(global transaction) 방식을 사용해야 함.
  * JTA(Java Transaction API)를 이용하면 글로벌 트랜잭션 적용이 가능하나 UserService의 코드를 수정해야 함.

* 트랜잭션 API의 의존관계 문제와 해결책
  * 트랜잭션 경계설정을 담당하는 코드는 일정한 패턴을 갖는 유사한 구조
  * 여러 기술의 사용법에 공통점이 있다면 추상화 가능
  * 추상화란?
  <br>하위 시스템의 공통점을 뽑아내서 분리시키는 것
  * DB에서 제공하는 DB 클라이언트 라이브러리와 API는 원래 전혀 호환이 되지 않음.
  * JDBC라는 추상화 기술이 있기 때문에 DB 종류와 관계없이 데이터 액세스 코드 작성 가능

* 스프링의 트랜잭션 서비스 추상화
  * 스프링은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공
  * `PlatformTransactionManager`: 트랜잭션 경계설정을 위한 추상 인터페이스
  * `DataSourceTransactionManager`: JDBC의 로컬 트랜잭션을 이용할 경우 구현하는 클래스
  * `getTransaction()`: 트랜잭션 오브젝트 생성. 필요에 따라 DB 커넥션도 가져옴.
```java
// 리스트 4-45 스프링의 트랜잭션 추상화 API를 적용한 upgradeLevels()
public void upgradeLevels() {
	// JDBC 트랜잭션 추상 오브젝트 생성
	PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
	
	// 트랜잭션 시작
	TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
	try {
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
		transactionManager.commit(status); // 트랜잭션 커밋
	} catch (RuntimeException e) {
		transactionManager.rollback(status); // 트랜잭션 롤백
		throw e;
	}
}
```

* 트랜잭션 기술 설정의 분리
  * 다른 트랜잭션 처리 방식으로 변경하려면 UserService의 소스 변경이 불가피.
  * `HibernateTransactionManager`: 하이버네이트 트랜잭션
  * `JPATransactionManager`: JPA 트랜잭션
  * `JTATransactionManager`: JTA 트랜잭션
  * 컨테이너를 통해 외부에서 제공받게 하는 스프링 DI 방식으로 변경해야 함.

* **스프링의 빈으로 등록 시 주의사항**
  * 싱글톤으로 만들어져 여러 스레드에서 동시에 사용해도 괜찮은지 검토
  * 멀티스레드 환경에서 안전하지 않은 클래스를 빈으로 등록하면 심각한 문제 발생

```java
// 리스트 5-46 트랜잭션 매니저를 빈으로 분리시킨 UserService
public class UserService {
	...
	private PlatformTransactionManager transactionManager;

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void upgradeLevels() {
		// DI 받은 트랜잭션 매니저를 공유해서 사용. 멀티스레드 환경에서도 안전.
		TransactionStatus status = 
			this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
			List<User> users = userDao.getAll();
			for (User user : users) {
				if (canUpgradeLevel(user)) {
					upgradeLevel(user);
				}
			}
			this.transactionManager.commit(status);
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
	...
}

// 리스트 5-47 트랜잭션 매니저 빈을 등록한 설정파일
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
	<property name="transactionManager" ref="transactionManager" />
</bean>
	
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource" />  
</bean>

// 리스트 5-48 트랜잭션 매니저를 수동 DI하도록 수정한 테스트
public class UserServiceTest {
	...
	@Autowired PlatformTransactionManager transactionManager;
	
	@Test
	public void upgradeAllOrNothing() {
		UserService testUserService = new TestUserService(users.get(3).getId());  
		testUserService.setUserDao(this.userDao);
		testUserService.setTransactionManager(this.transactionManager);
		...
	}
	...
}
```


### 5.3 서비스 추상화와 단일 책임 원칙

* 수직, 수평 계층구조와 의존관계
  * 기술과 서비스에 대한 추사오하 기법을 이용 시 특정 기술환경에 종속되지 않는 코드 생성 가능
  <br>-> 같은 계층에서 수평적인 분리
  * 트랜잭션의 추상화
  <br>애플리케이션의 비즈니스 로직과 그 하위에서 동작하는 로우레벨의 트랜잭션 기술이라는
  <br>아예 다른 계층의 특성을 갖는 코드를 분리한 것.
  * `PlatformTransactionManager` 인터페이스를 통한 추상화 계층을 사이에 두고 사용하기 때문에 구체적인 트랜잭션 기술에 독립적.
  * **DI의 가치는 관심, 책심, 성격이 다른 코드를 깔끔하게 분리하는 것.**

* 단일 책임 원칙 (Single Responsibility Principle)
  * 하나의 모듈은 한 가지 책임을 가져야 한다는 의미
  * UserService에서 JDBC Connection의 메서드를 직접 사용하는 트랜잭션 코드가 있을 경우
  <br>-> 비즈니스 로직과 트랜잭션 관리 2가지의 책임을 갖고 있음.
  <br>-> 트랜잭션 서비스 추상화 방식을 도입하고나면 비즈니스 로직 관리만 하는 단일 책임.

* 단일 책임 원칙의 장점
  * 어떤 변경이 필요할 때 수정 대상이 명확해짐.
  * 책임/관심이 다른 코드를 분리하고, 서로 영향을 주지 않게 추상화 기법을 도입하고,
  <br>애플리케이션 로직과 기술/환경을 분리하는 등의 작업은 엔터프라이즈 애플리케이션에 필수
  <br>-> 이를 위한 핵심적인 도구가 스프링이 제공하는 DI
  * 단일 책임 원칙을 잘 지키는 코드를 만드려면 **인터페이스를 도입하고 이를 DI로 연결**


### 5.4 메일 서비스 추상화


### 5.4.1 JavaMail을 이용한 메일 발송 기능

* JavaMail 메일 발송
  * `javax.mail` 패키지에서 제공하는 `JavaMail` 클래스를 이용해서 메일 전송 가능
```java
// 리스트 5-49 레벨 업그레이드 작업 메서드 수정
protected void upgradeLevel(User user) {
	user.upgradeLevel();
	userDao.update(user);
	sendUpgradeEMail(user);
}
```


### 5.4.2 JavaMail이 포함된 코드의 테스트

* 서버가 잘 준비되어 있다면 메일 전송은 문제없으나, 테스트 시마다 메일을 전송해야 하는가?
<br>테스트 시에는 테스트용으로 준비된 메일 서버를 이용하는 방법을 고려

* 굳이 메일 전송을 할 이유가 없다면 `JavaMail`을 사용하지 말고 별도로 구성한 테스트용 클래스를 사용


### 5.4.3 테스트를 위한 서비스 추상화

```java
// 리스트 5-53 메일 전송 기능을 가진 오브젝트를 DI 받도록 수정한 UserService
public class UserService {
	...
	private MailSender mailSender;

	public void setMailSender(MailSender mailSender) {
		this.mailSender = mailSender;
	}
	
	private void sendUpgradeEMail(User user) {
		SimpleMailMessage mailMessage = new SimpleMailMessage();
		mailMessage.setTo(user.getEmail());
		mailMessage.setFrom("useradmin@ksug.org");
		mailMessage.setSubject("Upgrade 안내");
		mailMessage.setText("사용자님의 등급이 " + user.getLevel().name());
		
		this.mailSender.send(mailMessage);
	}
	...
}

// 리스트 5-54 메일 발송 오브젝트의 빈 등록
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
	<property name="transactionManager" ref="transactionManager" />
	<property name="mailSender" ref="mailSender" />
</bean>

<bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
	<property name="host" value="mail.server.com" />
</bean>
```

* 테스트용 메일 발송 오브젝트
  * 불필요한 테스트 메일이 발송되지 않도록 Dummy 클래스를 이용해서 구성
```java
// 리스트 5-55 아무런 기능이 없는 MailSender 구현 클래스
public class DummyMailSender implements MailSender {
	public void send(SimpleMailMessage mailMessage) throws MailException {
	}

	public void send(SimpleMailMessage[] mailMessage) throws MailException {
	}
}

// 리스트 5-56 테스트용 UserService를 위한 메일 전송 오브젝트의 수동 DI
public class UserServiceTest {
	...
	@Autowired MailSender mailSender;
	
	@Test
	public void upgradeAllOrNothing() {
		...
		testUserService.setMailSender(this.mailSender);
		...
	}
}
```

* 테스트와 서비스 추상화
  * 서비스 추상화란?
  <br>트랜잭션과 같이 기능은 유사하나 사용 방법이 다른 로우레벨의 다양한 기술에 대해
  <br>추상 인터페이스와 일관성 있는 접근 장법을 제공해주는 것.
  * 메일 발송 작업에 트랜잭션 개념을 적용하는 법
    * 방법1: 업그레이드 대상 사용자를 발견할 때마다 메일을 전송하지 않고 별도의 목록에 발송대상 저장
    * 방법2: MailSender를 확장해서 메일 전송에 트랜잭션 개념을 적용


### 5.4.4 테스트 대역


