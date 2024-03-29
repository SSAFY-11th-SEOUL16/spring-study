# 예외
## 예외 전환
### JDBC의 한계
DB 종류에 상관없이 사용할 수 있는 데이터 액세스 코드를 작성하는 일은 쉽지 않다.
#### 비표준 SQL
대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공한다.

특별한 기능을 제공하는 함수를 SQL에 사용하려면 대부분 비표준 SQL 문장이 만들어진다.

이렇게 작성된 비표준 SQL은 결국 DAO 코드에 들어가고, 해당 DAO는 특정 DB에 대해 종속적인 코드가 된다.

사용할 수 있는 방법은 DAO를 DB별로 만들어 사용하거나 SQL을 외부에서 독립시켜서 바꿔 쓸 수 있게 하는 것이다.
#### 호환성 없는 SQLException의 DB 에러정보
DB마다 SQL만 다른 것이 아니라 에러의 종류와 원인도 제각각이다.

따라서 JDBC는 데이터 처리 중에 발생하는 다양한 예외를 그냥 `SQLException` 하나에 모두 담아버린다.
```java
if (e.getErrorCode() == MySqlErrorNumbers.ER_DUP_ENTRY) { ...
```
여기서 사용한 에러 코드는 MySQL 전용 코드이다.

DB가 MySQL에서 오라클이나 SQLServer로 바뀐다면 에러 코드도 달라지므로 기대한 대로 동작하지 못한다.
### DB 에러 코드 매핑을 통한 전환
DB별 에러 코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해주는 기능을 만들어 해결한다.

DB마다 에러 코드가 제각각이다. - 스프링은 DB별 에러 코드를 분류해서 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑 정보 테이블을 만들어두고 이를 이용한다.
```xml
<!-- 오라클 에러 코드 매핑 파일 -->
<bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
    <property name="badSqlGrammarCodes"> <!-- 예외 클래스 종류 -->
        <value>900,903,904,917,936,942,17006</value>	<!-- 매핑되는 DB 에러 코드. 에러 코드가 세분화된	-->
    </property>											<!-- 경우에는 여러 개가 들어가기도 한다.		-->
    <property name="invalidResultSetAccessCodes">
        <value>17003</value>
    </property>
    <property name="duplicateKeyCodes">
        <value>1</value>
    </property>
    <property name="dataIntegrityViolationCodes">
        <value>1400,1722,2291,2292</value>
    </property>
    <property name="dataAccessResourceFailureCodes">
        <value>17002, 17447</value>
    </property>
    ...
```
`add()` 메소드를 스프링의 `JdbcTemplate`을 사용하도록 바꾼다.

DB의 종류와 상관없이 중복 키로 인해 발생하는 에러는 `DataAccessException`의 서브클래스인 `DuplicateKeyException으`로 매핑돼서 던져진다.
```java
/*JdbcTemplate이 제공하는 예외 전환 기능을 이용하는 add() 메소드*/
public void add() throws DuplicateKeyException {
	// JdbcTemplate을 이용해 user를 add 하는 코드
}
```
애플리케이션 레벨의 체크 예외인 `DuplicateUserldException`을 던지게 하고 싶다면 스프링의 `DuplicateKeyException` 예외를 전환해주는 코드를 DAO 안에 넣는다.
```java
/*중복 키 예외의 전환*/
public void add() throws DuplicateUserIdException { // 애플리케이션 레벨의 체크 예외
	try {
		// jdbcTemplate을 이용해 User를 add 하는 코드
	}
	catch (DuplicateKeyException e) {
		// 로그를 남기는 등의 필요한 작업
		throw new DuplicateUserIdException(e);	// 예외를 전환할 때는 원인이 되는 예외를
	}											// 중첩하는 것이 좋다.
}
```
### DAO 인터페이스와 DataAccessException 계층구조
#### DAO 인터페이스와 구현의 분리
```java
/*기술에 독립적인 이상적인 DAO 인터페이스*/
public interface UserDao {
	public void add(User user); // 이렇게 선언하는 것이 과연 가능할까?
	...
```
DAO에서 사용하는 데이터 액세스 기술의 API가 예외를 던지기 때문에 위 메소드 선언은 사용할 수 없다.

인터페이스의 메소드 선언에는 없는 예외를 구현 클래스 메소드의 `throws`에 넣을 수는 없다.

따라서 인터페이스 메소드도 다음과 같이 선언돼야 한다.
```java
public void add(User user) throws SOLException;
```
이렇게 정의한 인터페이스는 JDBC가 아닌 데이터 액세스 기술로 DAO 구현을 전환하면 사용할 수 없다.
```java
public void add(User user) throws PersistentException;	// JPA
public void add(User user) throws HibernateException;	// Hibernate
public void add(User user) throws JdoException;			// JDO
...
```
JDO, Hibernate, JPA 등의 기술은 `SQLException` 같은 체크 예외 대신 런타임 예외를 사용한다.

따라서 throws에 선언을 해주지 않아도 된다.
```java
public void add(User user);
```
인터페이스로 추상화하고, 일부 기술에서 발생하는 체크 예외를 런타임 예외로 전환하는 것만으론 클라이언트가 DAO의 기술에 의존적이 될 수밖에 없다.
#### 데이터 액세스 예외 추상화와 DataAccessException 계층구조
`DataAccessException`은 자바의 주요 데이터 액세스 기술에서 발생할 수 있는 대부분의 예외를 추상화하고 있다.

스프링의 `DataAccessException`은 일부 기술에서만 공통적으로 나타나는 예외를 포함해서 데이터 액세스 기술에서 발생 가능한 대부분의 예외를 계층구조로 분류해놓았다.

![](https://velog.velcdn.com/images/torch/post/b27c8bbb-b0e6-48a2-86bf-52e09b9db186/image.png)

`DataAccessException` 계층구조에는 템플릿 메소드나 DAO 메소드에서 직접 활용할 수 있는 예외도 정의되어 있다.

`JdbcTempate`과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들면 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있다.
### 기술에 독립적인 UserDao 만들기
#### 인터페이스 적용
`UserDao` 인터페이스에는 기존 `UserDao` 클래스에서 DAO의 기능을 사용하려는 클라이언트들이 필요한 것만 추출해낸다.
```java
/*UserDao 인터페이스*/
public interface UserDao {
	void add(User user);
	User get(String id);
	List<User> getAll();
	void deleteAll();
	int getCount();
}
```
기존의 `UserDao` 클래스는 다음과 같이 이름을 `UserDaoJdbc`로 변경하고 `UserDao` 인터페이스를 구현하도록 `implements`로 선언해줘야 한다.
```java
public class UserDaoJdbc implements UserDao {
```
`userDao` 빈 클래스를 변경한다.
```xml
/*빈 클래스 변경*/
<bean id="userDao" class="springbook.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
<bean>
```
#### 테스트 보완
```java
public class UserDaoTest {
	@Autowired
	private userDao dao; // UserDaoJdbc로 변경해야 하나?
```
`UserDao` 인스턴스 변수 선언은 변경할 필요가 없다.

`@Autowired`가 스프링의 컨텍스트 내에서 정의된 빈 중에서 인스턴스 변수에 주입 가능한 타입의 빈을 찾아준다.

테스트 메소드를 `UserDaoTest`에 추가한다.
```java
/*DataAccessException에 대한 테스트*/
@Test(expected=DataAccessException.class)
public void duplicateKey() {
	dao.deleteAll();

	dao.add(user1); // 강제로 같은 사용자를 두 번 등록한다.
	dao.add(user1); // 여기서 예외가 발생해야 한다.
}
```
`expected` 부분을 빼고 돌려보면 다음과 같은 예외가 발생한다.
```
org.springframework.dao.DuplicateKeyException: PreparedStatementCallback; SQL
[insert into users (id, name, password) values (?,?,?)]; Duplicate entry 'gyumee' for
key 1; nested exception is com.mysql.jdbc.exceptions.jdbc4.MySOLIntegrityConstrain
tViolationException: Duplicate entry 'gyumee' for key 1
```
#### DataAccessException 활용 시 주의사항
`DuplicatedKeyException`은 JDBC를 이용하는 경우에만 발생한다.

`DataAccessException`을 잡아서 처리하는 코드를 만들려고 한다면 미리 학습 테스트를 만들어서 실제로 전환되는 예외의 종류를 확인해두어야 한다.

`UserDaoTest`에 `DataSource` 변수를 추가해서 `DataSource` 타입의 빈을 받아둔다.
```java
/*DataSource 빈을 주입받도록 만든 UserDaoTest*/
public class UserDaoTest {
	@Autowired UserDao dao;
	@Autowired DataSource dataSource;
```
`DataSource`를 사용해 `SQLException`에서 직접 `DuplicateKeyException`으로 전환하는 기능을 확인해본다.
```java
/*SQLException 전환 기능의 학습 테스트*/
@Test
public void sqlExceptionTranslate() {
	dao.deleteAll();

	try {
		dao.add(user1);
		dao.add(user1);
	}
	catch (DuplicateKeyException ex) {
		SQLException sqlEx = (SQLException)ex.getRootCause();
		SQLExceptionTranslator set = // 코드를 이용한 SQLException의 전환
			new SQLErrorCodeSQLExceptionTranslator(this.dataSource);
		// 에러 메시지 만들 때 사용하는 정보이므로 null로 넣어도 된다.
		assertThat(set.translate(null, null, sqlEx),
			is(DuplicateKeyException.class));
	}
}
```
JDBC 외의 기술을 사용할 때도 `DuplicatedKeyException`을 발생시키려면 SQLException을 가져와서 직접 예외 전환을 하는 방법도 있다.

JDBC를 이용하지만 `JdbcTemplate`과 같이 자동으로 예외를 전환해주는 스프링의 기능을 사용할 수 없는 경우라도 `SQLException`을 그대로 두거나 의미 없는 `RuntimeException`으로 뭉뚱그려서 던지는 대신 스프링의 `DataAccessException` 계층의 예외로 전환하게 할 수도 있다.
