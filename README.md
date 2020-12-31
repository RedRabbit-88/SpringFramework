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

