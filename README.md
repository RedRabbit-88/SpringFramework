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

