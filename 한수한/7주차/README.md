# 7주차 서비스 추상화 01

## 5.1 사용자 레벨 관리 기능 추가

현재 UserDao는 CRUD만 가능, 비즈니스 로직 존재 X

비즈니스 로직 추가

- 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나다.
- 사용자가 처음 가입하면 BASIC 레벨이 된다.
- 가입 후 50회 이상 로그인을 하면 BASIC에서 SILVER 레벨이 된다
- SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다
- 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.

### 5.1.1 필드 추가

**Level 이늄**

---

레벨 저장할 Enum 필드 추가

[ 사용자 레벨용 이늄 ]

```java
package springbook.user.domain;
...
public enum Level {
	BASIC(1), SILVER(2), G0LD(3); // 세 개의 이늄오브젝트 정의

	private final int value;

	Level(int value) {—> db에 저장할 값을 넣어줄 생성자를 만들어둔다.
		this.value = value;
	}

	public int intValue() { //―> 값을 가져오는 메소드
		return value；
	}

	public static Level valueOf(int value) {
		/*
			값으로부터 Level 타입 오브젝트를
			가져오도록 만든 스태틱 메소드
		*/
		switch(value) {
			case 1: return BASIC;
			case 2: return SILVER;
			case 3: return GOLD;

			default:throw new AssertionError("Unknown value： " + value);
		}
	}
}

```

**User 필드 추가**

---

Level 타입의 변수, 로그인 횟수, 추천수 추가

[ User에 추가된 필드 ]

```java
public class User {
	Level level;
	int login;
	int recommend;

	public Level getLevel() {
		return level;
	}

	public void setLevel(Level level) {
		this.level = level;
	}
	...
}
```

DB에도 필드 추가

![Untitled](imgs/Untitled.png)

**UserDaoTest 수정**

---

UserDaoJdbc와 테스트에도 필드 추가

[ 테스트 픽스처에 추가된 필드 넣기 ]

```java
public class UserDaoTest {
	@Before
	public void setUp() {
		this.user1 = new User ("gyumee", "박성철", "springnol", Level.BASIC, 1, 0);
		this.user2 = new User("leegw700", "이길원", "springnoZ", Level.SILVER, 55, 10);
		this.user3 = new User("bumjin", "박범진", "springnol", Level.GOLD, 100, 40);
	}
	...
}
```

[ 추가된 필드를 파라미터로 포함하는 생성자 ]

```java
class User {
	public User(String id, String name. String password, Level level,
		int login, int recommend) {
		this.id = id;
		this.name = name;
		this.password = password;
		this.level = level;
		this.login = login;
		this.recommend = recommend;
	}
	...
}
```

[ 새로운 필드를 포함하는 User 필드 값 검증 메소드 ]

```java
private void checkSameUser(User user1, User user2) {
	assertThat(user1 .getId(), is(user2.getld()));
	assertThat(user1.getName(), is(user2.getName()));
	assertThat(user1 .getPassword(), is(user2.getPassword()));
	assertThat(user1 .getLevel(), is(user2.getLevel()));
	assertThat(user1.getLogin(), is(user2.getLogin()));
	assertThat(user1.getRecommend(), is(user2.getRecommend()));
}
```

User 필드가 바껴도 addAndGet() 코드는 바뀌지 않도록 변경

[ checkSameUser() 메소드를 사용하도록 만든 addAndGet() 메소드 ]

```java
@Test public void addAndGet() {
	...
	User userget1 = dao.get(user1.getId());
	checkSameUser(userget1, user1);

	User userget2 = dao.get(user2.getId());
	checkSameUser(userget2, user2);
}
```

**UserDaoJdbc 수정**

---

[ 추가된 필드를 위한 UserDaoJdbc의 수정 코드 ]

```java
public class UserDaoJdbc implements UserDao {
	private RowMapper<User> userMapper =
		new RowMapper<User>() {
			public User mapRow(ResultSet rs, int rowNum) throws SQLException {
				User user = new User();
				user.setld(rs.getString("id"));
				user.setName(rs.getString("name"));
				user.setPassword(rs.getString("password"));
				user.setLevel(Level.valueOf(rs.getlnt("level")));
				user.setLogin(rs.getInt("loign"));
				user.setRecommend(rs.getlnt("recommend"));
				return user;
		}
	};

	public void add(User user) {
		this.jdbcTemplate.update(
			"insert into users(id, name, password, level, login, recommend) "+
			"value(?,?,?,?,?,?)", user.getId(), user.getName(),
			user.getPassword(), user.getLevel().intValue(),
			user.getLogin(), user.getRecommend());
	}
```

User의 Level은 DB와 호환이 안되므로 intValue()메소드 사용

```java
user.getLevel().intValue()
```

setLevel시에도 Level의 valueOf 메소드 사용

```java
user.setLevel(Level.valueOf(rs.getlnt("level")));
```

위 테스트는 아래 문장때문에 오류가 발생

```java
user.setLogin(rs.getInt("loign"));
```

미리 테스트 작업을 통해 이러한 오류를 발견하는 것이 중요함

### **5.1.2 사용자 수정 기능 추가**

**수정 기능 테스트 추가**

---

[ 사용자 정보 수정 메소드 테스트 ]

```java
@Test
public void update() {
	dao.deleteAll();
	dao.add(user1);

	// 픽스처에 있는 정보 변경
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

**UserDao와 UserDaoJdbc 수정**

---

UserDao interface에 update() 메소드 추가

```java
public interface UserDao {
	...
	public void update(User user1);
}
```

UserDaoJdbc의 update() 메소드 구현

[ 사용자 정보 수정용 update() 메소드 ]

```java
public void update(User user) {
	this.jdbcTemplate.update(
		"update users set name = ?, password = ?, level = ?, login = ?, "+
		"recommend = ? where id = ? ", user.getName(), user.getPassword(),
		user.getLevel().intValue(), user.getLogin(), user.getRecommend(),
		user.getId());
}
```

**수정 테스트 보완**

---

수정하지 않아야 할 로우의 내용이 그대로 남아 있는지는 확인해주지 못
한다는 문제가 있음

- where문을 빼버려도 테스트가 성공해버림…

해결방법

1.  update() 리턴 값 확인
    1. 영향받은 row의 수 리턴
2.  테스트 보강
    1. 사용자 두 명 등록 후 하나만 수정한 뒤 비교

[ 보완된 update() 테스트 ]

```java
@Test
public void updateO {
	dao.deleteAll();

	dao.add(user1); // 수정할 사용자
	dao.add(user2); // 수정하지 않을 사용자

	user1.setName("오민규");
	user1.setPassword("springno6");
	user1.setLevel(Level.GOLD);
	user1.setLogin(1000);
	user1.setRecommend(999);
	dao.update(user1);

	User user1update = dao.get(user1.getId());
	checkSamellser(user1, user1update);

	User user2same = dao.get(user2.getId());
	checkSameUser(user2, user2same);
}
```

### 5.1.3 UserService.upgradeLevels()

레벨 관리 기능 구현

UserDao의 getAll() 메소드로 사용자를 다 가져외서 사용자별로 레벨 업그레이드 작업을 진행하면서 UserDao의 update()를 호출

사용자 관리 로직을 서비스 클래스에 넣음

- Dao는 비즈니스 로직이 아닌 DB관련 기능한 제공할 것이기 때문

UserService 클래스 생성

- UserDao DI받음

UserServiceTest 클래스 생성

![Untitled](imgs/Untitled%201.png)

**UserService 클래스와 빈 등록**

---

[ UserService 클래스 ]

```java
package springbook.user.service;
...
public class UserService {
	UserDao userDao;

	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}
```

[ UserService 빈 설정 ]

```xml
<bean id="UserService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
</bean>

<bean id="userDao" class="springbook.dao.UserDaoJdbc")
	<property name="dataSource" ref="dataSource" />
</bean>
```

**UserServiceTest 클래스**

---

[ UserServiceTest 클래스 ]

```java
package springbook.user.service;
@RunWith(SpringJUnit4ClassRunner.class)
@Contextconfiguration(locations="/test-applicationContext.xml")
public class UserServiceTest {
	@Autowired
	UserService userservice;
}
```

[ userservice 빈의 주입을 확인하는 테스트 ]

```java
@test
public void bean(){
	assertThat(this.userService, is(notNullValue()));
}
```

**upgradeLevels() 메소드**

---

[ 사용자 레벨 업그레이드 메소드 ]

```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
	for(User user : users) {
		Boolean changed = null; // 레벨의 변화가있는지를 확인하는플래그
		if (user.getLevel() == Level.BASIC && user.getLogin() >= 50) {
			user.setLevel(Level.SILVER);
			changed = true;
		}
		else if (user.getLevel() == Level.SILVER && user.getRecommend() >= 30) {
			user.setLevel(Level.GOLD);
			changed = true; //레벨 변경 플래그 설정
		}
		else if (user.getLevel() == Level.GOLD) { changed = false; }
		else { changed = false; } // 일치하는 조건이 없으면 변경 없음

		if (changed) { userDao.update(user); }
	}
}
```

**upgradeLevels() 테스트**

---

Level이 다양하고 확인해야 할 것이 많으므로 픽스처가 많이 필요

- 리스트로 픽스처 생성

[ 리스트로 만든 테스트 픽스처 ]

```java
class UserServiceTest {
	List<User> users; // 테스트 픽스처
	@Before
	public void setUp()
		users = Arrays.asList(
			new User("bum jin", "박범진","p1", Level.BASIC, 49, 0),
			new User("joytouch", "강명성", "p2", Level.BASIC, 50, 0),
			new User("erwins", "신승한", "p3", Level.SILVER, 60, 29),
			new User("madnite1", "이상호", "p4", Level.SILVER, 60, 30),
			new User("green", "오민규", "p5", Level.GOLD, 100, 100)
		);
	}
```

BASIC과 SILVER 레벨의 사용자는 각각 두 개씩 등록

로그인 횟수와 추천 횟수가 각각 기준 값인 50, 30 이상이 되면 SILVER와 GOLD로 업그레이드

- **테스트 데이터 경계가 되는 값 전후로 선택**

[ 사용자 레벨 업그레이드 테스트 ]

```java
@Test
public void upgradeLevels() {
	userDao.deleteAll();
	for(User user : users) userDao.add(user);

	UserService.upgradeLevels();

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

### 5.1.5 UserService.add()

처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 함 적용

- 비즈니스 로직이니 Service 클래스에 위임

[ add() 메소드의 테스트 ]

```java
©Test
public void add() {
	userDao.deleteAll(),

	User userWithLevel = users.get(4); // GOLD 레벨
	User userWithoutLevel = users.get(0);
	userWithoutLevel.setLevel(null);

	UserService.add(userWithLevel);
	UserService.add(userWithoutLevel);

	User userWithLevelRead = userDao.get(userWithLevel.getId());
	User userWithoutLevelRead = userDao.get(userWithoutLevel.getldO);

	assertThat(userWithLevelRead.getLevel(), is(userWithLevel.getLevel()));
	assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
}
```

[ 사용잔신규 등록 로직르 담르 add() 메소드 ]

```java
public void add(User user) {
	if (user.getLevel() == null) user.setLevel(Level.BASIC);
	userDao.add(user);
}
```

테스트가 조금 복잡한 것이 흠

간단한 비즈니스 로직을 담은 코드를 테스트하기 위해 DAO와 DB까지 모두 동원되는 점이 조금 불편

- 깔끔하고 간단히 만드는 방법 존재

### 5.1.5 코드 개선

코드 개선을 위한 질문

- 코드 중복은 없는지
- 코드가 무엇을 하는지 이해하기 불편한지
- 코드가 있어야 할 자리에 있는지
- 변화에 쉽게 대응할 수 있게 작성되었는지

**upgradeLevels() 메소드 코드의 문제점**

---

1.  if/else if/else 블록들이 읽기 불편

    - 성격이 다른 여러 가지 로직이 한데 섞여있기 때문

    ![Untitled](imgs/Untitled%202.png)

    ①: 현재 레벨이 무엇인지 파악하는 로직

    ②: 업그레이드 조건을 담은 로직

    ③: 다음 단계의 레벨, 업그레이드를 위한 작업 포함

    ④: 그 자체로는 의미가 없음

    - ⑤의 작업이 필요한지 알려주는 임시 플래그 설정

2.  현재 레벨과 업그레이드 조건을 동시에 비교하는 부분의 문제

**upgradeLevels() 리팩토링**

---

레벨을 업그레이드하는 작업의 기본 흐름만 먼저 만듦

[ 기본 작업 흐름만 남겨둔 upgradeLevels() ]

```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
	for(User user : users) {
		if (canUpgradeLevel(user)) {
			upgradeLevel(user);
		}
	}
}
```

[ 업그레이드 가능 확인 메소드 ]

```java
private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
	switch(currentLevel) {
		case BASIC: return (user.getLoginO >= 50);
		case SILVER: return (user.getRecommend() >= 30);
		case GOLD: return false;
		default: throw new IllegalArgumentException("Unknown Level: " + currentLevel);
	}
}
```

[ 레벨 업그레이드 작업 메소드 ]

```java
private void upgradeLevel(User user) {
	if (user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
	else if (user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
	userDao.update(user);
}
```

- 다음 단계 확인 로직, 필드 변경 로직이 함께 존재
- 예외상황 처리 없음

레벨의 순서와 다음 단계 레벨이 무엇인지를 결정하는 일은 Level에 위임

[ 업그레이드 순서를 담고 있도록 수정한 Level ]

```java
package springbook.user.domain;
...
public enum Level {
	BASIC(1,SILVER), SILVER(2,GOLD), G0LD(3,null); // 세 개의 이늄오브젝트 정의

	private final int value;
	private final Level next;

	Level(int value, Level next) {—> db에 저장할 값을 넣어줄 생성자를 만들어둔다.
		this.value = value;
		this.next = next;
	}

	public int intValue() { //―> 값을 가져오는 메소드
		return value；
	}

	public int nextValue() { //―> 값을 가져오는 메소드
		return next;
	}

	public static Level valueOf(int value) {
		/*
			값으로부터 Level 타입 오브젝트를
			가져오도록 만든 스태틱 메소드
		*/
		switch(value) {
			case 1: return BASIC;
			case 2: return SILVER;
			case 3: return GOLD;

			default:throw new AssertionError("Unknown value： " + value);
		}
	}
}

```

사용자 정보가 바뀌는 부분을 UserService 메소드에서 User로 옮기기

[ User의 레벨 업그레이드 작업용 메소드 ]

```java
public void upgradeLevel() {
	Level nextLevel = this.level.nextLevel();
	if (nextLevel == null) {
		throw new IllegalStateException(this.level + "은 업그레이드가 불가능합니다");
	}
	else {
		this.level = nextLevel;
	}
}
```

[ 간결해진 upgradeLevel() ]

```java
private void upgradeLevel(User user) {
	user.upgradeLevel();
	userDao.update(user);
}
```

**User 테스트**

---

User에 로직 추가했으므로 테스트

```java
package springbook.user.service；
public class UserTest {
	User user;
	@Before
	public void setUp() {
		user = new User();
	}

	@Test()
	public void upgradeLevel() {
		Level[] levels = Level.values();
		for(Level level : levels) {
			if (level.nextLevel() == null) continue;
			user.setLevel(level);
			user.upgradeLevel();
			assertThat(user.getLevel(), is(level.nextLevel())));
		}
	}

	@Test(expected=IllegalStateException.class)
	public void cannotUpgradeLevel() {
		Level[] levels = Level.values();
		for(Level level : levels) {
			if (level.nextLevel() != null) continue;
			user.setLevel(level);
			user.upgradeLevel();
		}
	}
}
```

**UserServiceTest 개선**

---

```java
@Test
public void upgradeLevels() {
	userDao.deleteAll();
	for(User user : users) userDao.add(user);

	UserService.upgradeLevels();

	checkLevel(users.get(0), false);
	checkLevel(users.get(1), true);
	checkLevel(users.get(2), false);
	checkLevel(users.get(3), true);
	checkLevel(users.get(4), false);
}

private void checkLevel(User user, boolean upgraded) {
	User userUpdate = userDao.get(user.getId());
	if (upgraded) {
		assertThat(userUpdate.getLevel(), is(user.getLevel().nextLevel()));
	}else{
		assertThat(userUpdate.getLevel(), is(user.getLevel()));
	}
}
```

코드에 나타난 중복 제거

로그인 횟수, 추천횟수가 중복

```java
 case BASIC： return (user.getLogin() >= 50); // UserService
 new User("joytouch", "강명성", "p2", Level.BASIC, 50, 0) // UserServiceTest
```

[ 상수의 도입 ]

```java
public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final int MIN_RECCOMEND_FOR_GOLD = 30;

private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
	switch(currentLevel) {
		case BASIC: return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
		case SILVER: return (user.getRecommend() >= MIN_RECCOMEND_FOR_GOLD);
		case GOLD: return false;
		default: throw new IllegalArgumentException("Unknown Level: " +
			currentLevel);
	}
}
```

[ 상수를 사용하도록 만든 테스트 ]

```java
import static springbook.user.service.UserService.MIN_LOGCOUNT_FOR_SILVER;
import static springbook.user.service.UserService.MIN_RECCOMEND_FOR_GOLD;

class UserServiceTest {
	List<User> users; // 테스트 픽스처
	@Before
	public void setUp()
		users = Arrays.asList(
			new User("bum jin", "박범진","p1", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER-1, 0),
			new User("joytouch", "강명성", "p2", Level.BASIC, MIN_LOGCOUNT_FOR_SILVER, 0),
			new User("erwins", "신승한", "p3", Level.SILVER, 60, MIN_RECCOMEND_FOR_GOLD-1),
			new User("madnite1", "이상호", "p4", Level.SILVER, 60, MIN_RECCOMEND_FOR_GOLD),
			new User("green", "오민규", "p5", Level.GOLD, 100, Integer.MAX_VALUE)
		);
	}

```

연말 이벤트나 새로운 서비스 홍보기간 중에는 레벨 업그레이드 정책을 다르게 적용할 필요가 있을 수도 있다

- 사용자 업그레이드 정책을 UserService에서 분리하는 방법을 고려

[ 업그레이드 정책 인터페이스 ]

```java
public interface UserLevelUpgradePolicy {
	boolean canUpgradeLevel(User user);
	void upgradeLevel(User user);
}
```

## 5.2 트랜잭션 서비스 추상화

### 5.2.1 모 아니면 도

기존 사용자 레벨 업그레이드 코드 작동 방식 확인

- 작업 중간 예외 발생시 결과

예외가 던져지는 상황을 의도적으로 생성

**테스트용 UserService 대역**

---

UserService를 대신해서 테스트의 목적에 맞게 동작하는 클래스를 만들어 사용

간단히 UserService를 상속해서 테스트에 필요한 기능을 추가하도록 일부 메소드를 오버라이딩

테스트용이므로 UserServiceTest 내부에 스태틱 클래스로 만듦

5개의 테스트용 사용자 정보 중에서 두 번째와 네 번째가 업그레이드

네 번째 사용자를 처리하는 중에 예외를 발생

upgradeLevel() 오버라이딩

- 접근 제한 private → protected

[ UserService의 테스트용 대역 클래스 ]

```java
static class TestUserService extends UserService {
	private String id;

	private TestUserService(String id) {
		this.id = id;
	}

	 protected void upgradeLevel(User user) {
		 if (user.getId().equals(this, id)) throw new TestUserServiceException();
		 super.upgradeLevel(user);
	}
}
```

[ 테스트용 예외 ]

```java
static class TestUserServiceException extends RuntimeException {
}
```

**강제 예외 발생을 통한 테스트**

---

[ 예외 발생 시 작업 취소 여부 테스트 ]

```java
@Test
public void upgradeAllorNothing() {
	UserService testUserService = new TestUserService(users.get(3).getId());
	testUserService.setUserDao(this.userDao);
	userDao.deleteAll();
	for(User user : users) userDao.add(user);

	try {
		testUserService.upgradeLevels();
		fail("TestUserServiceException expected");
	} catch(TestUserServiceException e) { }

	checkLevelUpgraded(users.get(1)), false); // 예외 발생 전으로 돌아갔는지 확인
}
```

테스트는 실패함

- 중간에 예외가 발생해도 그 전 작업들은 변경된 채로 유지가 됨

**테스트 실패 원인**

---

트랜잭션 안에서 동작하지 않았기 때문

## 5.2.2 트랜잭션 경계설정

DB는 트랜잭션 지원

여러 개의 SQL이 사용되는 작업을 하나의 트랜잭션으로 취급해야 하는 경우 다수

**트랜잭션 롤백**

- 한 작업이 취소되면 그 전에 했던 모든 작업 취소

**트랜잭션 커밋**

- 모든 SQL 작업이 성공적으로 마무리되어 작업 확정

**JDBC 트랜잭션의 트랜잭션 경계설정**

---

```java
Connection c = dataSource.getConnection();

c.setAutoCommit(false); // 트랜잭션 시작

try{
	...
	c.commit();
}catch(Exception e){
	c.rollback();
}

...
```

JDBC의 트랜잭션

- 하나의 Connection을 가져와 사용하다가 닫는 사이에 일어남

**트랜잭션의 경계설정**

- setAutoCommit(false)로 트랜잭션의 시작을 선언하고 commit()또는 rollback()으로 트랜잭션 종료

**로컬 트랜잭션**

- 하나의 DB 커넥션 안에서 만들어지는 트랜잭션

**UserService와 UserDao 트랜잭션 문제**

---

트랜잭션 경계설정 코드가 존재하지 않음

UserDao는 각 메소드마다 하나씩의 독립적 인 트랜잭션으로 실행

![Untitled](imgs/Untitled%203.png)

여러 번 DB에 업데이트를 해야 하는 작업을 하나의 트랜잭션으로 만들기 위해선 어떻게 해야할까?

**비즈니스 로직 내의 트랜잭션 경계설정**

---

경계설정 작업을 UserService 쪽으로 가져와야 함

- upgradeLevels() 메소드의 시작과 함께 트랜잭션이 시작하고 메소드를 빠져나올 때 트랜잭션이 종료돼야 하기 때문

DB 커넥션도 이 메소드 안에서 만들고, 종료시킬 필요가 있음

[ upgradeLevels() 트랜잭션 경계설정 구조 ]

```java
public void upgradeLevels() throws Exception {
	(1) DB Connection 생성
	(2) 트랜잭션 시작
	try {
		(3) DAO 메소드 호출
		(4) 트랜잭션 커밋
	}
	catch(Exception e) {
		(5) 트랜잭션 롤백
		throw e;
	}
	finally {
		(6) DB Connection 종료
	}
}
```

DAO 메소드를 호출할 때마다 Connection 오브젝트를 파라미터로 전달해야함

[ Connection 오브젝트를 파라미터로 전달받는 UserDao 메소드 ]

```java
public interface UserDao {
	public void add(Connection c, User user);
	public User get(Connection c, String id);
	...
	public void update(Connection c, User user);
}
```

[ Connection을 공유하도록 수정한 UserService 메소드 ]

```java
class UserService {
 public void upgradeLevels() throws Exception {
	 Connection c = ...;
	 try{
		 ...
		 upgradeLevel(c,user);
		 ...
	 }
	 ...
	}

	protected void upgradeLevel(Connection c, User user) {
		user.upgradeLevel();
		userDao.update(c,user);
	}
}
```

**UserService 트랜잭션 경계설정의 문제점**

---

문제점

1. JdbcTemplate 활용 X
2. DAO, Service에 Connection 파라미터 추가
3. UserDao는 더 이상 데이터 액세스 기술에 독립적일 수가 없음
   1. JPA나 하이버네이트로 UserDao의 구현 방식을 변경하면 Connection이 아니라 EntityManager나 Session 오브젝트를 전달받아야함
   2. 즉, DB에 따라 메소드 시그니처가 변경되어야함

### 5.2.3 트랜잭션 동기화

스프링을 통한 해결

**Connection 파라미터 제거**

---

**독립적인 트랜잭션 동기화**

- 스프링이 제안하는 방법
- 트랜잭션을 시작하기 위해 만든 Connection 오브젝트를 특별한 저장소에 보관
- 호출되는 DAO의 메소드에서는 저장된 Connection을 가져다가 사용
- JdbcTemplate이 트랜잭션 동기화 방식을 이용하도록 하는 것

![Untitled](imgs/Untitled%204.png)

(1) UserService는 Connection을 생성

(2) 트랜잭션 동기화 저장소에 저장후 Connection의 setAutoCommit(false)를 호출해 트랜잭션을 시작

(3) 첫 번째 update() 메소드가 호출

(4) 트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인후 가져옴

(5) 가져온 Connection을 이용해 PreparedStatement를 만들어 수정 SQL을 실행

- JdbcTemplate은 Connection을 닫지 않은 채로 작업을 마침

(6)~(11) 실행

(12) Connection의 commit()을 호출해서 트랜잭션을 완료

(13) 저장소가 더 이상 Connection 오브젝트를 저장해두지 않도록 이를 제거

- rollback시에도 동일

**트랜잭션 동기화 적용**

---

[ 트랜잭션 동기화 방식을 적용한 UserService ]

```java
private DataSource dataSource;

public void setDataSource(DataSource dataSource) {
	 this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception {
	// 동기화 작업 초기화
	TransactionSynchronizationManager.initSynchronization();
	Connection c = DataSourceUtils.getConnection(dataSource);
	c.setAutoCommit(false);

	try {
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
		c.commit();
	} catch (Exception e) {
		c.rollback();
		throw e;
	} finally {
		DataSourceUtils.releaseConnection(c,dataSource);
		TransactionSynchronizationManager.unbindResource(this.dataSource);
		TransactionSynchronizationManager.clearSynchronization();
	}
}
```

TransactionSynchronizationManager

- 스프링이 제공하는 트랜잭션 동기화 관리 클래스

DataSourceUtils

- getConnection()
  - Connection 오브젝트 생성
  - 트랜잭션 동기화에 사용하도록 저장소에 바인딩

**트랜잭션 테스트 보완**

---

[ 동기화가 적용된 UserService에 따라 수정된 테스트 ]

```java
@Autowired DataSource dataSource;

@Test
public void upgradeAllOrNothing() throws Exception {
	UserService testUserService = new TestUserService(users.get(3).getId());
	testUserService.setUserDao(this.userDao);
	testUserService.setDataSource(this.dataSource);
	...
```

[ dataSource 프로퍼티를 추가한 userService 빈 설정 ]

```xml
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
	<property name="dataSource" ref="dataSource" />
</bean>
```

**JdbcTemplate과 트랜잭션 동기화**

---

트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인후 가져옴

### 5.2.4 트랜잭션 서비스 추상화

**기술과 환경에 종속되는 트랜잭션 경계설정 코드**

---

하나의 트랜잭션 안에서 여러 개의 DB에 데이터를 넣는 작업을 해야 할 필요가 발생

로컬 트랜잭션으로는 불가능

글로벌 트랜잭션 사용해야함

자바는 이를 지원하기 위해 JTA(Java Transaction API) 제공

![Untitled](imgs/Untitled%205.png)

[ JTA를 이용한 트랜잭션 코드 구조 ]

```java
Initialcontext ctx = new InitialContext();
UserTransaction tx = (UserTransaction)ctx.lookup(USER_TX_JNDI_NAME);

tx.begin();
Connection c = dataSource.getConnection();
try {
	// 데이터 액세스 코드
	tx.commit();
} catch (Exception e) {
	tx.rollback();
	throw e;
} finally {
	c.close();
}
```

UserService의 코드를 수정해야 한다는 점이 문제

**트랜잭션 API의 의존관계 문제와 해결책**

---

![Untitled](imgs/Untitled%206.png)

JDBC에 종속적인 Connection을 이용한 트랜잭션 코드가 UserService에 등장하면서부터 UserService는 UserDaoJdbc에 간접적으로 의존하는 코드가 돼버렸다는 점이 문제

특정 기술에 의존적인 Connection, UserTransaction, Session/Transaction API 등에 종속되지 않게 할 수 있는 방법 존재

공통적인 특징을 모아서 추상화된 트랜잭션 관리 계층 생성

**스프링의 트랜잭션 서비스 추상화**

---

![Untitled](imgs/Untitled%207.png)

[ 스프링의 트랜잭션 추상화 API를 적용한 upgradeLevels() ]

```java
public void upgradeLevels() {
	PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);

	Transactionstatus status =
		transactionManager.getTransaction(new DefaultTransactionDefinition());
	try {
		List<User> users = userDao.getAll();
		for (User user : users) {
			if (canUpgradeLevel(user)) {
				upgradeLevel(user);
			}
		}
		transactionManager.commit(status);
	} catch (RuntimeException e) {
		transactionManager.rollback(status);
		throw e;
	}
}
```

PlatformTransactionManager

- 스프링이 제공하는 트랜잭션 경계설정을 위한 추상 인터페이스
- DataSourceTransactionManager
  - JDBC 로컬 트랜잭션

DefaultTransactionDefinition

- 트랜잭션 정보 저장

Transactionstatus

- 시작된 트랜잭션 저장

**트랜잭션 기술 설정의 분리**

---

글로벌 트랜잭션으로 변경

- DataSourceTransactionManager → JTATransactionManager

UserService가 어떤 트랜잭션 매니저 구현 클래스를 알지 못하도록 (DI 원칙에 맞도록) 변경

[ 트랜잭션 매니저를 빈으로 분리시킨 UserService ]

```java
public class UserService(){

	private PlatformTransactionManager transactionManager;

	public void setTransactionManager(PlatformTransactionManager transactionManager){
		this.transactionManager = transactionManager;
	}

	public void upgradeLevels() {
		Transactionstatus status =
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
}
```

[ 트랜잭션 매니저 빈을 등록한 설정파일 ]

```xml
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
	<property name="transactionManager" ref="transactionManager" />
</bean>
<bean id="transactionManager"
	class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource" />
</bean>
```

[ 트랜잭션 매니저를 수동 DI 하도록 수정한 테스트 ]

```java
public class UserServiceTest {
	@Autowired
	PlatformTransactionManager transactionManager;

	@Test
	public void upgradeAllOrNothing() throws Exception {
		UserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(userDao);
		testUserService.setTransactionManager(transactionManager);
	...
```
