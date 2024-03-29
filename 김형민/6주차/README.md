# **4.2** 예외 전환

스프링의 JdbcTemplate이 던지는 DataAccessException은 일단 **런타임 예외로 SQLException을 포장해주어 애플리케이션 레벨에서는 신경 쓰지 않도록** 해주는 것이다. 또한 **일관성 있는 예외로 전환해서 추상화**해주려는 용도로 쓰이기도 한다.

## **4.2.1 JDBC의** 한계

DB 종류에 상관없이 사용할 수 있는 데이터 액세스 코드를 작성하는 일은 쉽지 않다. DB를 자유롭게 바꾸어 사용할 수 있는 DB 프로그램을 작성하는 데는 **두 가지 걸림돌**이 있기 때문이다.

### 비표준 SQL

**문제점**

1. 대부분의 DB는 표준을 따르지 않는 비표준 문법과 기능도 제공한다. 해당 DB의 특별한 기능을 사용하거나 최적화된 SQL을 만들 때 유용하기 때문이다.
2. 이렇게 작성된 비표준 SQL은 결국 DAO 코드에 들어가고, 해당 DAO는 특정 DB에 대해 종속적인 코드가 되고 만다.

### 호환성 없는 SQLException의 DB 에러정보

**문제점**

1. DB를 사용하다가 발생할 수 있는 예외의 원인은 다양하다.
2. 문제는 DB마다 SQL만 다른 것이 아니라 에러의 종류와 원인도 제각각이다!
3. 그래서 SQLException은 예외가 발생했을 때의 DB 상태를 담은 SQL 상태정보를 부가적으로 제공한다. Open Group의 XOPEN SQL 스펙에 정의된 SQL 상태 코드를 따른다.
4. DB의 JDBC 드라이버에서 SQLException을 담을 상태 코드를 정확하게 만들어주지 않는다. 그러므로 이 상태 코드를 믿고 코드를 작성하는 것은 위험하다.

## **4.2.2 DB** 에러 코드 매핑을 통한 전환

DB 에러 코드는 DB에서 직접 제공해주는 것이니 버전이 올라가더라도 어느 정도 일관성이 있다.

예를 들어 키 값이 중복돼서 중복 오류가 발생하는 경우 MySQL: 1062, Oracle: 1, DB2: -803이라는 에러 코드를 받게된다.

스프링은 DB별 에러 코드를 분류해서 스프링이 정의한 **예외 클래스**와 **매핑해놓은 에러 코드 매핑정보 테이블**을 만들어두고 이를 이용한다. 이후 **드라이버나 DB메타정보를 참고해서 DB종류를 확인**하고 **DB별로 미리 준비된 매핑정보를 참고**해서 적절한 예외 클래스를 선택하기 때문에 **DB가 달라져도 같은 종류의 에러라면 동일한 예외를 받을수 있다!**

**JdbcTemplate이 제공하는 예외 전환 기능을 이용하는 add() 메소드**

```java
public void add() throws DuplicateKeyException {
	// JdbcTemplate을 이용해 User를 add 하는 코드
}
```

- 이해가 안 되는게 DuplicateUserldException을 언체크 예외로 위에 만들었는데 왜 갑자기 체크예외라고 하는거지

## **4.2.3 DAO 인터페이스와 DataAccessException 계층구조**

DataAccessException은 JDBC의 SQLException을 전환하는 용도로만 만들어진게 아니라 **JDBC 외의 자바 데이터 액세스 기술에서 발생하는 예외에도 적용**된다.

스프링이 왜 이렇게 DataAccessException 계층구조를 이용해 기술에 독립적인 예외를 정의하고 사용하게 하는지 생각해보자!

### DAO 인터페이스와 구현의 분리

DAO를 굳이 따로 만들어서 사용하는 이유는?

- 데이터 액세스 로직을 담은 코드를 분리
- 분리된 DAO를 전략 패턴을 적용해 구현 방법을 변경해서 사용할 수 있게 만들기 위함

**이상적인 DAO 인터페이스**

```java
public interface UserDao{
	public void add(User user); 
}
```

- 위의 메소드 선언을 사용할 수 없다.
- 인터페이스의 메소드 선언에는 없는 예외를 구현 클래스 메소드에 throws에 넣을 수는 없기 때문이다.

```java
public void add(User user) throws SQLException;
public void add(User user) throws PersistentException; // JPA 
public void add(User user) throws HibernateException; // Hibernate
```

- 결국 인터페이스로 메소드의 구현은 추상화 했지만 구현 기술마다 던지는 예외가 다르기 때문에 **메소드의 선언이 달라진다는 문제**가 발생한다.
- Dao 인터페이스를 기술에 완전히 독립적으로 만들려면 예외가 일치하지 않는 문제도 해결해야한다!

<aside>
💡 다행히도 JDBC보다는 늦게 등장한 JDO, Hibernate, JPA 등의 기술은 SQLException 같은 체크 예외 대신 런타임 예외를 사용하기에 throws에 선언을 해주지 않아도 된다.

</aside>

그 말의 즉슨, JDBC를 이용한 DAO에서 모든 SQLException을 런타임 예외로 포장하면 방금전의 **이상적인 DAO 인터페이스**를 사용할 수 있다.

**결국 DAO를 사용하는 클라이언트 입장에서는 DAO의 사용 기술에 따라서 예외 처리 방법이 달라져야한다 . 클라이언트가 DAO의 기술에 의존적이 될 수 밖에없다!**

### 데이터 액세스 예외 추상화와 DataAccessException 계층구조

그래서 스프링은 자바의 다양한 데이터 액세스 기술을 사용할 때 발생하는 예외들을 추상화해서 DataAccessException 계층구조안에 정리해놓았다. 그 결과로 스프링의 JdbcTemplate은 SQLException의 에러 코드를 DB별로 매핑해서 그에 해당하는 의미 있는 DataAccessException의 서브클래스 *중* 하나로 전환해서 던져준다.

DataAccessException은 자바의 주요 데이터 액세스 기술에서 발생할 수 있는 대부분의 예외를 추상화 하고있다 . 데이터 액세스 기술에 상관없이 공통적인 예외도 있지만 일부 기술에서만 발생하는 예외도 있다 . orm에서는 발생하지만 jdbc에는 없는 예외가 있다 . **DataAccessException 은 이런 일부 기술에서만 공통적으로 나타나는 예외를 포함해서 데이터 액세스 기술에서 발생 가능한 대부분의 예외를 계층구조로 분류해놓았다.**

**결론**

**인터페이스 사용, 런타임 예외 전환**과 함께 **DataAccessException 예외 추상화**를 적용하면 데이터 액세스 기술과 구현 방법에 독립적인 DAO를 만들 수가 있다!

## **4.2.4** 기술에 독립적인 **UserDao** 만들기

 **UserDao 인터페이스**

```java
public interface UserDao {
	void add(User user);
	User get(String id);
	List<User> getAll();
	void deleteAll();
	int getCount();
}
```

- 이를 사용할 데이터 액세스 기술 별로 구현하여 사용하면 된다. (UserDaoJdbc, UserDaoJpa, …)

**기존의 UserDao 클래스 변경**

```java
public class UserDaoJdbc implements UserDao{
}
```

- UserDao 인터페이스를 구현하도록 implements로 선언한다.
- 데이터 액세스 기술 관련 클래스로 이름을 변경한다.

**스프링 설정파일의 userDao 빈 클래스 변경**

```xml
<bean id="userDao" class="springbook.dao.UserDaoJdbc">
    	<property name="dataSource" ref="dataSource" />
</bean>
```

- 보통 **빈의 이름**은 클래스 이름이 아니라 **인터페이스 이름**을 따른다!
- 그래야 나중에 구현 클래스를 바꿔도 혼란이 없다.

### 테스트 보완

**중복된 키 테스트**

```java
@Test(expected=DataAccessExcetion.class)
public void duplciateKey(){
	dao.deleteAll();
	
	dao.add(user1);
	dao.add(user1); //예외발생 
}
```

- 테스트는 성공하므로 **DataAccessExcetion 타입**의 예외가 던져진 것을 확인할 수 있다.
- **구체적인 예외 타입**을 알기위해 expected=DataAccessExcetion.class 부분을 빼고 테스트 해보자!
- 테스트는 실패하지만 DataAccessExcetion의 서브 클래스인 **DuplicteKeyException**이 발생한 것을 확인할 수 있다.

### DataAccessException 활용 시 주의사항

안타깝게 **DuplicteKeyException**은 아직 까지는 JDBC를 이용하는 경우에만 발생한다. 데이터 액세스 기술을 하이버네이트나 JPA를 사용했을때도 동일한 예외가 발생 할것으로 기대하지만 실제로 다른 예외가 던져진다.

그 이유는 SQLException에 담긴 DB 에러 코드를 바로 해석하는 JDBC의 경우와달리 JPA나 하이버네이트등에서는 각 기술이 재정의한 예외를 가져와 **스프링이 최종적으로 DataAccessExcetion 으로 변환하는데 , DB의 에러 코드와 달리 이런 예외들은 세분화되어 있지 않기 때문이다.**

기술에 상관없이 어느정도 추상화된 공통 예외로 변환해주긴하지만 근본적인 한계 때문에 완벽하다고 기대할 수는 없다. 따라서 사용에 주의를 기울여야한다.

SQLException을 직접 해석해 DataAccessException으로 변환하는 코드의 사용법을 살펴보자!

**dataSource 주입**

```java
public class UserDaoTest{
	@Autowried UserDao dao;
	@Autowride DataSource dataSource;
}
```

- **SQLErrorCodeSQLExceptionTranslator**는 **에러 코드 변환에 필요한 DB의 종류를
알아내기 위해** 현재 연결된 **DataSource**를 필요로 한다

**DataSource를 사용해 SQLException 에서 직접 DuplicateKeyException으로 전환하는 기능을 확인하는 학습 테스트**

```java
@Test
public void sqlExceptionTranslate(){
	dao.deleteAll();
	try{
		dao.add(user1);
		dao.add(user1);
	}catch(DuplicateKeyException ex){
		SQLException sqlEx = (SQLException)ex.getRootCause();
		SQLExceptionTranslator set = //코드를 이용한 SQLException의 전환
			new SQLErrorCodeSQLExceptionTranslator(this.dataSource);
		assertThat(set.thanslate(null,null,sqlEx)),
			is(DuplicateKeyException.class));
	}
}
```

- 강제로 DuplicateKeyException을 발생시킨다 . DuplicateKeyException은 중첩된 예외 이므로 JBDC API에서 처음 발생한 SQLException을 갖고 있다 .getRootCause을 이용하면 중첩되어있는 SQLException을 가져올수 있다.
- 이제 검증해 볼 사항은 스프링의 예외 전환 API를 직접 적용해서 DuplicateKeyException 이 만들어지는가이다.
- 주입받은 dataSource를 이용해서 SQLErrorCodeSQLExceptionTranslator의 오브젝트를 만든다. 그리고 SQLException을 파라미터로 넣어 translate()메소드를 호출해주면 DataAccessException 타입의 예외로 변환해준다 . **변환된 DataAccessException타입의 예외가 정확히 DuplicateKeyException타입인지를 확인하면된다.**
