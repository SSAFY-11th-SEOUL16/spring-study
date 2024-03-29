# [토비 스프링] 5장 서비스 추상화 1

# **5.1** 사용자 레벨 관리 기능 추가

지금까지 만들었던 UserDao를 다수의 회원이 가입할 수 있는 인터넷 서비스의 사용자 관리 모듈에 적용한다고 생각해보자.

**구현해야 할 비즈니스 로직(정해진 조건에 따라 사용자의 레벨을 주기적으로 변경)**

- 사용자의 레벨은 BASIC, SILVER, GOLD 세 가지 중 하나다.
- 사용자가 처음 가입하면 BASIC 레벨이 되며, 이후 활동에 따라서 한 단계씩 업그레이드될 수 있다.
- 가입 후 50회 이상 로그인을 하면 BASIC에서 SILVER 레벨이 된다.
- SILVER 레벨이면서 30번 이상 추천을 받으면 GOLD 레벨이 된다.
- 사용자 레벨의 변경 작업은 일정한 주기를 가지고 일괄적으로 진행된다. 
변경 작업 전에는 조건을 충족하더라도 레벨의 변경이 일어나지 않는다.

## **5.1.1** 필드추가

### Level Enum

각 레벨을 코드화해서 숫자로 넣어보자!

**정수형 상수 값으로 정의한 사용자 레벨**

```java
class User {
	private static final int BASIC = 1;
	private static final int SILVER = 2;
	private static final int GOLD = 3;

	int level;

	public void setLevel(int level) {
		this.level = level;
	}
	...
}
```

**문제점**

- 타입이 int형이기 때문에 **다른 종류의 값을 넣는 실수**를 할 수 있는데 이를 **컴파일러가 잡아주지 못한다.**

**Enum을 사용해 해결**

```java
package springbook.user.domain;

public enum Level {
	BASIC(1), SILVER(2), G0LD(3);
	private final int value;
		
	Level(int value) {
		this.value = value;
	}

	public int intValue() {
		return value;
	}

	public static Level valueOf(int value) {
		switch(value) {
			case 1： return BASIC;
			case 2： return SILVER;
			case 3： return GOLD;
			default： throw new AssertionError("Unknown value： " + value);
		}
	}
}
```

- Level 이늄은 **내부에는 DB에 저장할 int 타입의 값**을 갖고 있지만, **겉으로는 Level 타입의 오브젝트**이기 때문에 안전하게 사용할 수 있다!

### User 필드 추가

Level 타입 변수를 User 클래스에 추가하자!

**User에 Level 타입 변수를 추가**

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

### UserDaoTest 테스트 수정

UserDaoJdbc와 테스트에도 필드를 추가하자.

**테스트 픽스처 수정**

```java
public class UserDaoTest {
	...
	@Before
	public void setUp() { 
		this.userl = new User("gyumee", "박성철", "springno1", Level.BASIC, 1, 0);
		this.user2 = new User("leegw700", "이길원", "springno2", Level.SILVER, 55, 10);
		this.user3 = new User("bumjin", "박범진", "springno3", Level.GOLD, 100, 40);
	}
```

- user1, user2, user3에 추가된 필드의 값을 넣는다.

**User 클래스 생성자 파라미터 수정**

```java
class User{
	...
	public User(String id, String name , String password, Level level, int login, int recommend){
		this.id = id;
		this.name = name;
		this.password = passwrod;
		this.level = level;
		this.login = login;
		this.recomment = recomment;
	}
}
```

**UserDaoJdbc 수정**

```java
public class UserDaoJdbc implement UserDao {
	...
	private RowMapper<User> userMapper = 
		new RowMapper<User>(){
			public User mapRow(ResultSer re,int rowNum) throws SQLException{
				User user = new User();
				user.setId(rs.getString("id"));
				user.setPassword(rs.getString("password"));
				user.setLevel(Level.valueOf(rs.getInt("level"))); //Enum타입 정수형으로 변환
				user.setLogin(rs.getInt("login"));
				user.setRecommend(rs.getInt("recommend"));
				return user;
			}
		};
		
		public void add(User user){
			this.jdbcTemplate.update(
				"insert into users(id,name,password,level,login,recommend)" +
				"values(?,?,?,?,?,?)",user.getId(), user.getName(),
				user.getPassword(), user.getLevel().intValue(),
				user.getLogin(),user.getRecommend());			
		}
}
```

- level 필드는 Enum 타입으로 DB에 저장할 수 있는 타입이 아니다.
- 고로 **DB에 저장 가능한 정수형 값으로 변환**해야하는데 이때 **intValue()**를 사용한다.
- 반대로 DB에서 조회할때는 **정수형 값을 Enum 타입으로 변환**하기 위해 **valueOf()**를 사용한다.

## 5.1.2 사용자 수정 기능추가

수정할 정보가 담긴 User 오브젝트로 해당 사용자를 찾아 필드 정보를 **UPDATE 문**을 이용해 수정하는 메소드를 만들자!

### 수정 기능 테스트 추가

**사용자 정보 수정 테스트 코드**

```java
@Test
public void update(){
	dao.deleteAll();
	
	dao.add(user1); //픽스처 등록
	
	user1.setName("오민규");
	user1.setPassword("springno6");
	user1.setLevel(Level.GOLD);
	user1.setLogin(1000);
	user1.setRecommend(999);
	
	dao.update(user1); //픽스처 수정
	
	User user1update = dao.get(user1.getId());
	checkSameUser(user1,user1update); //검증
}
```

- 픽스처를 생성 후 등록한다.
- 이후 픽스처에 들어있는 정보를 변경해서 제대로 수정이 되었는 지 검증하는 코드이다.

### UserDao와 UserDaoJdbc 수정

dao 변수의 타입인 UserDao 인터페이스에 update() 메소드가 없어 컴파일 오류가 발생하니 update() 메소드를 추가하자!

**UserDao에 update() 메소드 추가**

```java
public interface UserDao {
	...
	public void update(User user1);
}
```

- 이를 추가하면 UserDao를 구현하는 UserDaoJdbc에서 이를 구현하지 않았다고 오류가 발생할 것이다.

**UserDaoJdbc에 update() 메소드 구현**

```java
public void update(User user) {
	this.jdbcTemplate.update(
		"update users set name = ?, password = ?, level = ?, login = ?, " +
		"recommend = ? where id = ? ", user.getName(), user.getPassword(),
		user.getLevel().intValue(), user.getLogin(), user.getRecommend(),	user.getId());
}
```

### 수정 테스트 보완

update에서 where절을 빼먹으면 수정하지 않아야 할 로우가 변경될 위험이 있다!

**해결 방법**

1. JdbcTemplate의 update()가 돌려주는 **리턴 값을 확인**하자!
- update()는 테이블의 내용에 영향을 주는 SQL을 실행하면 **영향받은 로우의 개수**를 돌려주기에 이를 통해 확인할 수 있다.
1. 테스트를 보강해서 원**하는 사용자 외의 정보는 변경되지 않았음을 직접 확인**하자!
- 테스트 코드를 보완해서 **직접 조회**해보면 된다.
- 여기선 이 방법을 사용한다.

**테스트 코드 보완**

```java
@Test
public void update() {
	dao.deleteAll();
	dao.add(user1); //수정할 사용자
	dao.add(user2); //수정하지 않을 사용자
	
	userl .setName("오민규");
	userl.setPassword("springno6");
	userl.setLevel(Level.GOLD);
	userl.setLogin(1000);
	user1.setRecommend(999);
	
	dao.update(user1);
	
	User user1update = dao.get(user1.getld());
	checkSameUser(user1, user1update);
	User user2same = dao.get(user2.getld());
	checkSameUser(user2, user2same);
}
```

## **5.1.3 UserService.upgradeLevels(**)

레벨 관리 기능을 구현하자!

레벨 관리 기능처럼 사용자 관리 로직을 DAO에 두는 것은 옳지 않다.
DAO는 데이터를 어떻게 가져오고 조작할지를 다루는 곳이기 때문이다.

고로, 사용자 관리 비즈니스 로직을 담을 클래스인 **UserService를 추가한다.**

<img width="600" alt="Untitled (1)" src="https://github.com/fkgnssla/spring-study/assets/92067099/6173587c-4aa8-4c2e-bb88-382a536b4223">

### UserService 클래스와 빈 등록

**UserService 클래스 생성**

```java
package springbook.user.service;

public class UserService {
	UserDao userDao;
	
	public void setUserDao(UserDao userDao) {
		this.userDao = userDao;
	}
}
```

- UserDao를 DI 받는다.
- 그러므로 UserService도 빈으로 등록되어야 한다.

**userService 빈 설정**

```xml
<bean id="userService" class="springbook.user.service.UserService">
	<property name="userDao" ref="userDao" />
</bean>

<bean id="userDao" class="springbook.dao.UserDaoJdbc")
	<property name="dataSource" ref="dataSource" />
</bean>
```

### UserServiceTest 테스트 클래스

```java
@RunWith(SpringJunit4ClassRunner.class)
@ContextConfiguration(locations="/tset-applicationContext.xml")
public class UserServiceTset{
	@Autowired
	UserService userService;
    
  @Test
  public void bean(){ //메소드가 없으면 에러가 나오기때문에 빈이 생성되서 변수에 주입되는지만 확인하는 메소드 추가
       assertThat(this.userService, is(notNullValue())); 
  }
}
```

### upgradeLevels() 메소드

**사용자 레벨 업그레이드 메소드**

```java
public void upgradeLevels(){
	List<User> users = userDao.getAll();
	
	for(User:users){
		Boolean changed = null; //레벨의 변화가 있는지를 확인하는 플래그
		if(user.getLevel() == Level.BASIC && user.getLogin() >= 50){
			user.setLevel(Level.SILVER); //BASIC 레벨 업그레이드
			changed = true;
		}
    else if(user.getLevel() == level.SILVER && user.getRecommend() >= 30){
      user.setLevel(Level.GOLD); //SILVER 레벨 업그레이드
      changed = true;
    }
    else if (user.getLevel() == Level.GOLD){ changed = false; } //GOLD레벨은 변경 x
    else { changed = false; } //조건 없으면 변경 없음
    
    if(changed){ userDao.update(user); } //변경이 있는 경우에만 update 호출
	}
}
```

- changed 플래그를 확인해서 레벨 변경이 있는 경우에만 update를 호출한다.

### **upgradeLevels() 테스트**

```java
class UserServiceTest{
	...
	List<User> users; //테스트 픽스처 
	
	@Before
	public void setUp(){
		users = Arrays.asList( 
				new User("bumjin","박범진","p1",Level.BASIC,49,0),
				new User("joytouch","강명성","p2",Level.BASIC,50,0),
				new User("erwins","신승한","p3",Level.SILVER,60,29),
				new User("madnite1","이상호","p4",Level.SILVER,60,30),
				new User("green","오민규","p5",Level.GOLD,100,100),
		);
	}
	
	@Test
	public void upgradeLevels(){
		userDao.deleteAll();
		for(User user : users) userDao.add(user);
		
		userService.upgradeLevels();
		
		checkLevel(users.get(0),Level.BASIC);
		checkLevel(users.get(1),Level.SILVER);
		checkLevel(users.get(2),Level.SILVER);
		checkLevel(users.get(3),Level.GOLD);
		checkLevel(users.get(4),Level.GOLD);
	}
	
	//업그레이드 후의 예상레벨 검증 메소드
	private void checkLevel(User user, Level expectedLevel){
		User userUpdate = userDao.get(user.getId());
		assertThat(UserUpdate.getLevel(),is(expectedLevel));
	}
}
```

- BASIC, SILVER, GOLD, BASIC(유지), SILVER(유지) 총 다섯 종류의 사용자가 있을 수 있다. 이를 **@Before**을 선언한 메소드의 **초기 설정값으로 등록**한다.
- 레벨 업그레이드 작업이 끝나면 사용자 정보를 하나씩 가져와 **레벨의 변경 여부를 검증**한다.

## **5.1.4 UserService.add(**)

처음 가입하는 사용자는 기본적으로 BASIC 레벨이어야 한다.

**add() 메소드 테스트 코드**

```java
@Test
public void add(){
	userDao.deleteAll();
	
	User userWithLevel = users.get(4); //이미 레벨이 gold일 경우 레벨을 초기화 하지 않는다.
	User userWithoutLevel = users.get(0); //레벨이 비어있는 사용자 
	userWithoutLevel.setLevel(null);
	
	userService.add(userWithLevel);
	userService.add(userWithoutLevel);
	
	User userWithLevelRead = userDao.get(userWithLevel.getId());
	User userWihtoutLevelRead = userDao.get(userWithoutLevel.getId());
  
  //DB에 저장된 결과를 가져와 확인
	assertThat(userWithLevelRead.getLevel(),is(userWithLevel.getLevel()));
	assertThat(userWithoutLevelRead.getLevel(), is(Level.BASIC));
}
```

- 레벨이 미리 정해진 경우와 레벨이 비어 있는 두 가지 경우에 각각 add() 메소드를 호출하고 결과를 확인한다.

**UserService에 add() 추가**

```java
public void add(User user){
	if(user.getLevel() == null) user.setLevel(Level.BASIC);
	userDao.add(user);
} 
```

- 레벨이 null이면, BASIC으로 설정한다.

## **5.1.5** 코드 개선

### upgradeLevels() 메소드 코드의 문제점

for 루프 속에 들어 있는 **if / elseif / else 블록**들이 읽기 불편하다. 여러가지 로직이 한데 섞여있기 때문이다.

### upgradeLevels() 리팩토링

**기본 작업 흐름만 남겨둔 upgradeLevels()**

```java
public void upgradeLevels() {
	List<User> users = userDao.getAll();
	for(User user: users) {
		if(canUpgrandeLevel(user)) {
			upgradeLevel(user);
		}
	}
}
```

- 모든 사용자 정보를 가져와 한 명씩 업그레이드가 가능한지 확인하고, 가능하면 업그레이드를 한다.

**업그레이드 가능 확인 메소드**

```java
private boolean canUpgradeLevel(User user){
	Level currentLevel = user.getLevel();
	switch(currentLevel){
		//레벨별로 구분해서 조건을 판단
		case BASIC: return (user.getLogin()>=50);
		case SILVER: return (user.getRecommend()>=30);
		case GOLD: return false;
		default: throw new IllegalArgumentException("Unknown Level:" + currentLevel); //다룰수 없는 레벨이 주어지면 예외 발생
	}
}
```

- 로직에서 처리할 수 없는 레벨인 경우 예외를 던진다.

**레벨 업그레이드 작업 메소드**

```java
private void upgradeLevel(User user){
	if(user.getLevel() == Level.BASIC) user.setLevel(Level.SILVER);
	else if(user.getLevel() == Level.SILVER) user.setLevel(Level.GOLD);
	userDao.update(user);
}
```

- 사용자의 레벨을 다음 단계로 변경하고 변경사항을 DB에 업데이트한다.

**upgradeLevel()의 문제점**

1. **예외상황에 대한 처리**가 안 되어있다.
2. 레벨이 많아지면 if문이 길어진다. 또한 레벨 변경 시 level 필드 이외 값도 변경해야한다면 **코드는 더욱 길어질 것**이다.

**업그레이드 순서를 담고 있도록 수정한 Level** 

```java
public enum Level{
	GOLD(3, null), SILVER(2, GOLD),BASIC(1, SILVER);
	//Enum 선언에 DB에 저장할 값과 함께 다음 단계의 레벨정보도 추가한다.
	private final int value;
	private final Level next; //다음 단계의 레벨 정보를 스스로 갖고 있도록 추가
	
	Level(int value, Level next){
		this.value = value;
		this.next = next;
	}
	
	public int intValue(){
		return value;
	}
	
	public Level nextLevel(){
		return this.next;
	}
	
	public static Level value(int value){
		switch(value){
			case 1: return BASIC;
			case 2: return SLCVER;
			case 3: retrun GOLD;
			default: throws new AssertionError("Unknown value: "+value);
		}
	}
}
```

- 레벨의 순서와 다음 단계 레벨이 무엇인지를 결정하는 일은 Level이 관리하도록 next 필드와 관련된 메소드를 추가한다.

**User 클래스에 레벨 업그레이드 작업 메소드 추가**

```java
public void upgradeLevel(){
	Level nextLevel = this.level.nextLevel();
	if(nextLevel == null){
		throws new IllegalStateException(this.level + "은 업그레이드가 불가능합니다.")
	}
	else{
		this.level = nextLevel;
	}
}
```

- **User의 내부 정보가 변경**되는 것은 UserService보다는 **User가 스스로** 다루는 게 적절하다.
- UserService가 User에게 레벨 업그레이드를 해야 하니 정보를 변경하라고 요청하자!

**간결해진 UserSerivce의 upgradeLevel() 메소드**

```java
private void upgradeLevel(User user){
	//UserService가 User에게 정보를 변경하라고 요청
	user.upgradeLevel();
	userDao.update(user);
}
```

<aside>
💡 개선한 코드를 보면 각 오브젝트와 메소드가 **각각 자기 몫의 책임을 맡아 일을 하는 구조**로 바뀌었다.

각자 자기 책임에 충실한 작업만 하고 있으니 **코드를 이해하기도 쉽고** 변경이 필요할 때 **어디를 수정해야 할지도** 쉽게 알 수 있다.

</aside>

### User 테스트 추가

**UserTest**

```java
public class UserTest{
	User user;
	
	@Before
	public void setUp(){
		user = new User();
	}
	
	//정상 테스트
	@Test()
	public void upgradeLevel(){
		Level[] levels = Level.values();
		for(Level level : levels){
			if(level.nextLevel() == null) continue;
			user.setLevel(level);
			user.upgradeLevel();
			assertThat(user.getLevel(),is(level.nextLevel()));
		}
	}
	
	//예외 테스트
	@Test(expected = IllegalStateException.class)
	public void cannotUpgradeLevel(){
		Level[] levels = Level.values();
		for(Level level:levels){
			if(level.nextLevel()!=null) continue;
			user.setLevel(level);
			user.upgradeLevel();
		}
	}
}
```

- **upgradeLevel()**: 정상적인 레벨 업그레이드 테스트
- **cannotUpgradeLevel()**: 예외 테스트
    - nextLevel()이 null인 경우 강제로 업그레이드 메소드 실행

### 리팩토링

**UserSerice 상수 도입**

```java
public static final int MIN_LOGCOUNT_FOR_SILVER = 50;
public static final int MIN_RECCOMEND_FOR_GOLD = 30;

private boolean canUpgradeLevel(User user) {
	Level currentLevel = user.getLevel();
	switch(currentLevel) {
		case BASIC： return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
		case SILVER： return (user.getRecommend() >= MIN_RECCOMEND_FOR_GOLD);
		case GOLD： return false;
		default： throw new IllegalArgumentException("Unknown Level： " + currentLevel);
	}
}
```

- 상수를 도입하여 숫자의 중복을 해결했다!

# 5.2 트랜잭션 서비스 추상화

시용자 레벨 관리 작업을 수행하는 도중에 네트워크가 끊기거나 서버에 장애가 생겨서 일부 작업만 수행된 경우에는 어떻게 해야할까?

## **5.2.1** 모 아니면 도

현재 사용자 레벨 업그레이드 코드는 위 경우에 어떻게 동작하는 지 테스트해보자!

### 테스트용 UserService 대역

**UserService의 테스트용 대역 클래스**

```java
static class TestUserService extends UserService{
	private String id;
	
	private TestUserService(String id){ //예외를 발생시킬 user 오브젝트의 ID를 지정할 수 있게함
		this.id = id;
	}
	
	protected void upgradeLevel(User user){ //UserService 메소드를 오버라이드
		//지정된 id에 해당하는 User 객체이면 예외를 발생시킨다.
		if(user.getId().equals(this.id)) throws new TestServiceEcxeption(); 
		super.upgradeLevel(user);
	}
} 
```

- 근데 UserService가 지금 그냥 클래슨데 상속이 가능한가?
    - 가능하지. 그리고 upgradeLevel()의 접근지정자를 잠시 protected로 선언해서 오버라이딩이 가능한 거고

### 강제 예외 발생을 통한 테스트

**예외 발생 시 작업 취소 여부 테스트**

```java
@Test
public void upgradeAllOrNothing(){
	UserService testUserService = new TestService(users.get(3).getId());
  
  //예외를 발생시킬 네번째 사용자 id를 넣어서 테스트용 userService 오브젝트 생성 
	testUserService.setUserDao(this.userDao); //수동 DI
	userDao.deleteAll();
	for(User user : users) userDao.add(user);
	
	try { //예외가 발생하지 않는다면 실패 
		testUserService.upgradeLevels();
		fail("TestUserServiceException expected");
	} catch(TestUserServiceException e){	 //예외를 잡아서 계속 진행되도록 한다. 
	}
	
	//예외가 발생하기 전에 레벨 변경이 있었던 사용자의 레벨이 롤백됐는지 확인.
	checkLevelUpgraded(user.get(1),false);
}
```

- 4번째 User를 업데이트할 때 TestUserServiceException 가 발생하고 try/catch문을 빠져나온다.
- 그 이후 예외가 발생하기 전에 업데이트가 일어난 2번째 User의 상태가 기존과 같은지 확인한다.

### 테스트 실패의 원인

upgradeLevels() 메소드가 **하나의 트랜잭션** 안에서 동작하지 않았기 때문이다.

**트랜잭션**

- 중간에 예외가 발생해서 작업을 완료할 수 없다면 아예 작업이 시작되지 않은 것처럼 초기 상태로 돌려놔야 한다. (원자성)

## **5.2.2** 트랜잭션 경계설정

**하나의 SQL 명령**을 처리하는 경우는 **DB가 트랜잭션을 보장**해준다.

하지만 **여러 개의 SQL**이 사용되는 작업을 **하나의 트랜잭션으로 취급해야 하는 경우**도 있다.

**트랜잭션 롤백**

- DB에서 수행되기 전에 문제가 발생할 경우에는 **앞에서 처리한 SQL 작업도 취소**시킨다.

**트랜잭션 커밋**

- 모든 SQL 수행 작업이 다 성공적으로 마무리됐다고 **DB에 알려 작업을 확정**시킨다.

### JDBC 트랜잭션의 트랜잭션 경계설정

애플리케이션 내에서 **트랜잭션이 시작되고 끝나는 위치**를 **트랜잭션의 경계**라고 부른다.

**트랜잭션을 사용한 JDBC 코드**

```java
Connection c = dataSource.getConnection();

c.setAutoCommit(false); //트랜잭션 시작

try{
	PrepardeStatement st1 =	c.prepareStatement("update users ...");
	st1.executeUpdata();
	
	PrepardeStatement st2 =	c.prepateStatement("delete users...");
	st2.executeUpdata();
   
	c.commit(); //트랜잭션 커밋 
    
}catch(Exception e){
	c.rollback() //트랜잭션 롤백
}
c.close();
```

- JDBC의 트랜잭션은 하나의 Connection을 가져와 사용하다가 닫는 사이에서 일어난다.
- **오토 커밋 기능을 false**로 설정해주면 **새로운 트랜잭션이 시작**되게 할 수 있다.
- 트랜잭션이 한 번 시작되면 commit() 또는 rollback() 메소드가 호출될 때까지의 작업이 하나의 트랜잭션으로 묶인다.
    - 트랜잭션 시작: **setAutoCommit(false)**
    - 트랜잭션 종료: **commit()** or **rollback()**

### UserService와 UserDao의 트랜잭션 문제

**JdbcTemplate**의 메소드를 사용하는 UserDao는 **각 메소드마다 하나씩의 독립적인 트랜잭션으로 실행**된다.

그러므로 update() 메소드가 정상적으로 실행되면 자동으로 UPDATE 작업의 트랜잭션을 종료시킬 것이고, 수정 결과는 영구적으로 DB에 반영된다. 그 후 예외가 발생하든, 서버가 다운되든 상관없이 수정 결과는 DB에 남게되는 것이다!

JdbcTemplate을 사영할 수 있도록 Connection을 없애자!

## **5.2.3** 트랜잭션 동기화

### Connection 파라미터 제거

트랜잭션 동기화를 통해 Connection 파라미터를 제거해보자!

**트랜잭션 동기화**

- 트랜잭션을 시작하기 위해 만든 **Connection 오브젝트를 특별한 저장소에 보관**해두고, 이후에 호출되는 DAO의 메소드에서는 **저장된 Connection을 가져다가 사용**하게 하는 것

**트랜잭션 동기화를 사용한 경우의 작업 흐름**

<img width="772" alt="Untitled (2)" src="https://github.com/fkgnssla/spring-study/assets/92067099/1c8b0ad0-af4f-49d9-954c-a80d00e687c4">

1. UserService 는 Connection을 생성하고
2. 이를 트랜잭션 동기화 저장소에 저장 해두고 Conneciton의 setAutoCommit(false)를 호출해 트랜잭션을 시작 시킨후에 본격적으로 DAO의 기능을 이용하기 시작한다.
3. 첫번째 update() 메소드가 호출되고 update()메소드 내부에서 이용하는 JdbcTemplate 메소드에서는 가장먼저
4. 트랜잭션 동기화 저장소에 현재 시작된 트랜잭션을 가진 Connection 오브젝트가 존재하는지 확인한다 . (2) upgradeLevels() 메소드 시작 부분에서 저장해둔 Connection을 발견하고 이를 가져온다 .
5. Connection을 이용해 PreparedStatement를 만들어 수정 SQL을 실행한다. 트랜잭션 동기화 저장소에서 DB 커넥션을 가져왔을때는 JdbcTemplate은 Connection을 닫지 않은채로 작업을 마친다.

### 트랜잭션 동기화 적용

**트랜잭션 동기화 방식을 적용한 UserService**

```java
private DataSource dataSource;

public void setDateSource(DataSource dataSource){
	this.dataSource = dataSource;
}

public void upgradeLevels() throws Exception{
		//트랜잭션 동기화 관리자를 이용해 동기화 작업 초기화
    TransactionSynchronizationManager.initSynchronization();
	  //DB 커넥션 생성과 동기화를 함께 해주는 메소드
    Connection c = DataSourceUtils.getConnection(dataSource);
    //커넥션을 생성하고 트랜잭션을 시작한다.
    c.setAutoCommit(false);
    
    try{
        List<User> users = userDao.getAll();
        for(User user:users){
            if(canUpgradeLevel(user)){
                upgradeLevel(user);
            }
        }
        c.commit();
    }catch(Exception e){ //예외 발생시 롤백
        c.rollback();
        throw e;
    }finally{
        DataSourceUtils.releaseConnection(c,dataSource);
        TransactionSynchronizationManager.unbindResource(this.dataSource);
        TransactionSynchronizationManager.clearSynchronization();
    }
}
```

- DataSourceUtils의 getConnection() 메소드는 Connection 오브젝트를 생성해줄 뿐만 아니라 트랜잭션 동기화에 사용하도록 저장소에 바인딩해준다!

### 트랜잭션 테스트 보완

```java
@Autowried DataSource dataSource;

@Test
pulic void upgradeAllOrNothing() throws Exception{
	UserService testUserService = new TestUserService(users.get(3).getId());
	testUserService.setUserDao(this.userDao);
	testUserService.setDataSource(this.dataSource);
	...
}
```

### JdbcTemplate과 트랜잭션 동기화

트랜잭션 동기화 저장소에 **등록된 DB 커넥션이나 트랜잭션이 없는 경우**

- JdbcTemplate이 **직접 DB 커넥션을 만들고 트랜잭션을 시작**해서 JDBC 작업을 진행한다.

트랜잭션 동기화를 **시작해놓은 경우**

- JdbcTemplate의 메소드에서 직접 DB 커넥션을 만드는 대신 **트랜잭션 동기화 저장소에 들어 있는 DB 커넥션을 가져와서 사용**한다.

