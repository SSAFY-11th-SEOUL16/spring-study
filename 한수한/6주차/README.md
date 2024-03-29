# 6주차 예외 02

## 4.2 예외 전환

**예외 전환 목적**

1. 런타임 예외로 포장해 굳이 필요하지 않은 catch / throws 줄임
2. 로우레벨의 예외를 의미 있고 추상화된 예외로 바꿔 던짐

### 4.2.1 JDBC의 한계

JDBC는 자바를 이용해 DB에 접근하는 방법을 추상화된 API 형태로 정의해놓고, 각 DB 업체가 JDBC 표준을 따라 만들어진 드라이버를 제공

DB를 자유롭게 변경해서 사용할 수 있는 유연한 코드를 보장해주지 못함

그 이유 2가지가 아래 내용

**비표준 SQL**

---

대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공

특별한 기능을 제공하는 함수를 SQL에 사용하려면 대부분 비표준 SQL 문장이 만들어짐

해결법

- DAO를 DB별로 만들어 사용하거나 SQL을 외부에서 독립시켜서 바꿔 쓸 수 있게 하기

**호환성 없는 SQLException의 DB 에러정보**

---

DB 예외 원인은 다양하나 전부 SQLException으로 모두 담아버림

getErrorCode()로 가져올 수 있는 DB 에러 코드는 DB별로 모두 다름

즉, DB가 바뀌면 getErrorCode()를 통해 어떤 예외가 발생했는지 확인하는 코드도 변경되어야 함

### 4.2.2 DB 에러 코드 매핑을 통한 전환

DB 종류가 바뀌더라도 DAO를 수정하지 않으려면 이 두 가지 문제를 해결해야 함

SQLException에 담긴 SQL 상태 코드는 신뢰할 만한 게 아님

DB 전용 에러 코드가 더 정확한 정보

해결법

- DB별 에러 코드를 참고해서 발생한 예외의 원인이 무엇인지 해석해
  주는 기능을 만들기

키 값이 중복돼서 중복 오류가 발생하는 경우에 MySQL이라면 1062, 오라클이라면 1, DB2라면 -803이라는 에러 코드를 받게 됨

에러 코드 확인을 통해 DuplicateKeyException이라는 의미가 분명히 드러나는 예외로 전환 가능

스프링은 DataAccessException의 서브클래스로 세분화된 예외 클래
스들을 정의

문제는 DB마다 에러 코드가 제각각이라는 점

스프링

- DB별 에러 코드를 분류
- 스프링이 정의한 예외 클래스와 매핑해놓은 에러 코드 매핑 정보 테이블을 만들어두고 이를 이용

```xml
<bean id="Oracle" class="org.springframework.jdbc.support.SQLErrorCodes">
	<property name="badSqlGrammarCodes"> // 예외 클래스종류
		<value>900,903,904,917,936,942,17006</value>
	</property>
	<property name="invalidResultSetAccessCodes">// 매핑되는 db 에러 코드. 에러 코드가 세분화된 경우에는 여러 개가 들어가기도 한다.
		<value>17003</value>
	</property>
	<property name="duplicateKeyCodes">
		<value>1</value>
	</property>
	<property name="dataIntegrityViolationCodes">
		<value>1400,1722,2291,2292</value>
	</property>
	<property name="dataAccessResourceFailureCodes">
		<value>17002,17447</value>
	</property>
	...
```

전환되는 JdbcTemplate에서 던지는 예외는 모두 DataAccessException의 서브클래스 타입

[ JdbcTemplate이 제공하는 예외 전환 기능을 이용하는 add() 메소드 ]

```java
public void add() throws DuplicateKeyException {
	// JdbcTemplate을 이용해 User를 add 하는 코드
}
```

[ 중복 키 예외의 전환 ]

```java
public void add() throws DuplicateUserIdException {
	try{
		// JdbcTemplate을 이용해 User를 add 하는 코드
	}
	catch(DuplicateKeyException e){
		throw new DuplicateUserIdException(e);
	}
}
```

JDBC 4.0부터는 SQLException를 세분화해서 정의

아직은 스프링의 에러 코드 매핑을 통한 DataAccessException 방식을 사
용하는 것이 이상적

### 4.2.3 DAO 인터페이스와 DataAccessException 계층구조

DataAccessException은 의미가 같은 예외라면 데이터 액세스 기술의 종류와 상관없이 일관된 예외가 발생하도록 만들어준다

스프링이 왜 이렇게 DataAccessException 계층구조를 이용해 기술에 독립적인 예외를 정의하고 사용하게 하는지 생각

**DAO 인터페이스와 구현의 분리**

---

DAO를 굳이 따로 만들어서 사용하는 이유

- 데이터 액세스 로직을 담은 코드를 성격이 다른 코드에서 분리해놓기 위해

DAO의 사용 기술과 구현 코드는 전략 패턴과 이를 통해서 DAO를 사용하
는 클라이언트에게 감출 수 있지만, 메소드 선언에 나타나는 예외정보가 문제가 될 수 있다

[ 기술에 독립적인 이상적인 DAO 인터페이스 ]

```java
public interface UserDao {
	public void add(User user);
	...
```

add()가 JDBC API를 사용한다면 인터페이스 메소드도 SQLException을 throw 해줘야함

이렇게 하면 JDBC가 아닌 데이터 액세스 기술로 DAO 구현을 전환하면 사용 불가

- 데이터 액세스 기술 API는 독자적인 예외를 던지기 때문

인터페이스로 추상화했지만 던지는 예외가 달라 메소드 선언이 달라진다는 문제 발생

가장 단순한 해결법 : throws Exception

하지만 대부분의 API는 런타임 예외 사용해서 throws 선언 안해줘도 됨

그래서 위와 같이 적어줘도 됨

하지만, 모든 예외를 무시해야 하는 건 아님

DAO를 사용하는 클라이언트 입장에서는 DAO의 사용 기술에 따라서 예외 처리 방법이 달라져야 한다

- 클라이언트가 DAO의 기술에 의존적

**데이터 액세스 예외 추상화와 DataAccessException 계층구조**

---

스프링의 DataAccessException은 일부 기술에서만 공통적으로 나타나는 예외를 포함해서 데이터 액세스 기술에서 발생 가능한 대부분의 예외를 계층구조로 분류

InvalidDataAccessResourceUsageException

- 데이터 액세스 기술을 부정확하게 사용한 경우
- 대부분 프로그램을 잘못 작성해서 발생하는 오류

스프링이 기술의 종류에 상관없이 이런 성격의 예외를 InvalidDataAccessResourceUsageException으로 던져주므로 개발자에게 빠르게 통보 가능

오브젝트/엔티티 단위로 정보를 업데이트하는 경우에는 낙관적인 락킹(optimistic locking) 발생 가능

낙관적인 락킹

- 같은 정보를 두 명 이상의 사용자가 동시에 조회하고 순차적으로 업데이트를 할 때, 뒤늦게 업데이트한 것이 먼저 업데이트한 것을 덮어쓰지 않도록 막아주는 데 쓸 수 있는 편리한 기능

JDO, JPA, 하이버네이트마다 다른 종류의 낙관적인 락킹 예외를 발생

스프링은 이를 ObjectOptimisticLockingFailureException로 통일

![Untitled](imgs/Untitled.png)

![Untitled](imgs/Untitled%201.png)

JdbcTempate과 같이 스프링의 데이터 액세스 지원 기술을 이용해 DAO를 만들면 사용 기술에 독립적인 일관성 있는 예외를 던질 수 있음

데이터 액세스 기술과 구현 방법에 독립적 인 이상적인 DAO 생성법

- 인터페이스 사용
- 런타임예외 전환
- DataAccessException 예외 추상화를 적용

### 4.2.4 기술에 독립적인 UserDao 만들기

**인터페이스 적용**

---

[ UserDao 인터페이스 ]

```java
public interface UserDao {
	void add(User user);
	User get(String id);
	List<User> getAll();
	void deleteAll();
	int getCount();
}
```

이제 기존의 UserDao 클래스는 다음과 같이 이름을 UserDaoJdbc로 변경하고 UserDao 인터페이스를 구현하도록 implements로 선언

```java
public class UserDaoJdbc implements UserDao {
	...
}
```

스프링 설정파일의 UserDao 빈 클래스 이름 변경

```xml
<bean id="UserDao" class="springbook.dao.UserDaoJdbc">
	<property name="dataSource" ref="dataSource" />
</bean>
```

보통 빈의 이름은 클래스의 구현 인터페이스 이름을 따르는 경우가 일반
적

**테스트 보완**

---

![Untitled](imgs/Untitled%202.png)

[ DataAccessException에 대한 테스트 ]

```java
@Test(expected=DataAccessException.class)
public void duplicateKey() {
	dao.deleteAll();
	dao.add(user1);
	dao.add(user1);
}
```

**DataAccessException 활용 시 주의사항**

---

DuplicateKeyException은 아직까지는 JDBC를 이용하는 경우에만 발생

하이버네이트는 중복 키가 발생 하는 경 우에 하이버네이트의 ConstraintViolationException을 발생

스프링은 이를 해석해서 좀 더 포괄적인 예외인 DatalntegrityViolationException으로 변환할 수밖에 없다.

물론 DatalntegrityViolationException로 잡아줘도 되지만 중복 키 예외가 아닌 다른 예외일 때도 이 예외가 발생하므로 이용가치가 떨어짐

DataAccessException을 잡아서 처리하는 코드를 만들려고 한다면 미리
학습 테스트를 만들어서 실제로 전환되는 예외의 종류를 확인해둘 필요가 있다.

DAO에서 사용하는 기술의 종류와 상관없이 동일한 예외를 얻고 싶다면
DuplicatedUserldException처럼 직접 예외를 정의해두고, 각 DAO의 add() 메소드에서 좀 더 상세한 예외 전환을 해줄 필요가 있다

학습 테스트를 하나 더 만들어서 SQLException을 직접 해석해 DataAccessException으로 변환하는 코드의 사용법을 살펴보자

가장 보편적이고 효과적인 방법

- DB 에러 코드를 이용하는 것

SQLException을 코드에서 직접 전환

- SQLErrorCodeSQLExceptionTranslator를 사용
- SQLErrorCodeSQLExceptionTranslator는 DataSource 필요

[ DataSource 빈을 주입받도록 만든 UserDaoTest ]

```java
public class UserDaoTest {
	@Autowired UserDao dao;
	@Autowired DataSource dataSource;

	@Test
	public void sqlExceptionTranslate() {
	dao.deleteAll();
	try {
		dao.add(user1);
		dao.add(user1);
	}
	catch(DuplicateKeyException ex) {
		SQLException sqlEx = (SQLException)ex.getRootCause();
		SQLExceptionTranslator set =
			new SQLErrorCodeSQLExceptionTranslator(this.dataSource);
		assertThat(set.translate(null, null, sqlEx),
			is(DuplicateKeyException.class));
	}
}
```

1. JdbcTemplate을 사용하는 UserDao를 이용해 강제로 DuplicateKeyException을발생
2. getRootCause() 메소드를 이용하여 중첩되어 있는 SQLException 가져옴
3. SQLErrorCodeSQLExceptionTranslator로 SQLException 번역
4. 번역한 Exception이 DuplicateKeyException인지 확인

## 4.3 정리

**배운것**

---

- 엔터프라이즈 애플리케이션에서 사용할 수 있는 바람직한 예외처리 방법
- JDBC 예외의 단점
- 스프링이 제공하는 효과적인 데이터 액세스 기술의 예외처리 전략과 기능

**주요 내용**

---

- 예외를 잡아서 아무런 조취를 취하지 않거나 의미 없는 throws 선언을 남발하는 것은 위험하다
- 예외는 복구하거나 예외처리 오브젝트로 의도적으로 전달하거나 적절한 예외로 전환해야 한다.
- 좀 더 의미 있는 예외로 변경하거나. 불필요한 catch/throws를 피하기 위해 런타임 예외로 포장하는 두 가지 방법의 예외 전환이 있다
- 복구할 수 없는 예외는 가능한 한 빨리 런타임 예외로 전환하는 것이 바람직하다.
- 애플리케이션의 로직을 담기 위한 예외는 체크 예외로 만든다
- JDBC의 SQLException은 대부분 복구할 수 없는 예외이므로 런타임 예외로 포장해야 한다.
- SQLException의 에러 코드는 DB에 종속되기 때문에 DB에 독립적인 예외로 전환될 필요가있다
- 스프링은 DataAccessException을 통해 DB에 독립적으로 적용 가능한 추상화된 런타임 예외 계층을 제공한다.
- DAO를 데이터 액세스 기술에서 독립시키려면 인터페이스 도입과 런타임 예외 전환. 기술에 독립적인 추상화된 예외로 전환이 필요하다.
