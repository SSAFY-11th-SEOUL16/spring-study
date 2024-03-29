# 예외
## 사라진 SQLEXCEPTION
```java
/*JdbcTemplate 적용 전*/
public void deleteAll() throws SQLException {
	this.jdbcContext.executeSql("delete from users");
}
/*JdbcTemplate 적용 후*/
public void deleteAll() {
	this.jdbcTemplate.update("delete from users");
}
```

`JdbcTemplate` 적용 이전에는 있었던 `throws SQLException` 선언이 적용 후에는 사라졌다.
### 초난감 예외처리
```java
/*초난감 예외처리 코드 1*/
try {
	...
}						// 예외를 잡고는 아무것도 하지 않는다. 예외 발생을 무시해버리고 정상적인 상황인것처
catch(SQLException e) {	// 럼 다음 라인으로 넘어가겠다는 분명한 의도가 있는 게 아니라면 연습 중에도 절대 만
}						// 들어서는 안 되는 코드다.
```
catch 블록을 에서 아무것도 하지 않고 별문제 없는 것처럼 넘어가 버리는 건 위험한 일이다.

프로그램 실행 중에 어디선가 오류가 있어서 예외가 발생했는데 그것을 무시하고 계속 진행하게 된다.
```java
/*초난감 예외처리 코드 2*/
} catch (SQLException e) {
	System.out.println(e);
}
```
```java
/*초난감 예외처리 코드 3*/
} catch (SQLException e) {
	e.printStackTrace();
}
```
운영서버에 올라가게 되면 콘솔 로그를 누군가가 계속 모니터링하지 않는 한 예외 발생을 알아치릴 수 없다.

모든 예외는 적절하게 복구되거나 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.
```java
/*그나마 나은 예외처리*/
} catch (SQLException e) {
	e.printStackTrace();
    System.exit(1);
}
```
#### 예외 블랙홀
#### 무의미하고 무책임한 throws
```java
/*초난감 예외처리 4*/
public void method1() throws Exception { // ⤴︎ 예외
	method2(); // ⤵︎ 호출
	...
}

public void method2() throws Exception { // ⤴︎ 예외
	method3(); // ⤵︎ 호출
	...
}

public void method3() throws Exception { // ⤴︎ 예외
	...
}
```
`throws Exception`을 기계적으로 붙이면 적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 있는 기회를 박탈당한다.
### 예외의 종류와 특징
Error:
- `java.lang.Error` 클래스의 서브클래스
- 시스템에 비정상적인 상황이 발생했을 경우에 사용된다.
- 시스템 레벨에서 특별한 작업을 하는 게 아니라면 애플리케이션에서는 신경 쓰지 않아도 된다.

Exception과 체크 예외:
- `java.lang.Exception` 클래스와 그 서브클래스
- 개발자들이 만든 애플리케이션 코드의 작업 중에 예외상황이 발생했을 경우에 사용된다.
- Exception 클래스는 다시 체크 예외와 언체크 예외unchecked exception로 구분된다.
- Exception 클래스의 서브클래스 중에서 RuntimeException을 상속하지 않은 것이 체크 예외
- 체크 예외가 발생할 수 있는 메소드를 사용할 경우 반드시 예외를 처 리하는 코드를 함께 작성해야 한다.

RuntimeException과 언체크/런타임 예외:
- `java.lang.RuntimeException` 클래스를 상속한 예외
- `catch` 문으로 잡거나 `throws`로 선언하지 않아도 된다.
- 피할 수 있지만 개발자가 부주의해서 발생할 수 있는 경우에 발생하도록 만든 것이 런타임 예외
### 예외처리 방법
#### 예외 복구
통제 불가능한 외부 요인으로 인해 예외가 발생하면 `MAX_RETRY`만큼 재시도 한다.

최대 횟수만큼 반복적으로 시도함으로써 예외상황에서 복구되게 할 수 있다.
```java
/*재시도를 통해 예외를 복구하는 코드*/
int maxretry = MAX_RETRY;
while(maxretry --> 0) {
	try { 
        ...		// 예외가 발생할 가능성이 있는 시도
	    return; // 작업 성공
	}
	catch(SomeException e) {
		// 로그 출력. 정해진 시간만큼 대기
	}
	finally {
		// 리소스 반납. 정리 작업
	}
}
throw new RetryFailedException(); // 최대 재시도 횟수를 넘기면 직접 예외 발생
```
#### 예외처리 회피
예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져버린다.

`throws` 문으로 선언해서 예외가 발생하면 알아서 던져지게 하거나 `catch` 문으로 일단 예외를 잡은 후에 로그를 남기고 다시 예외를 던진다.(rethrow)
```java
/*예외처리 회피 1*/
public void add() throws SQLException {
	// JDBC API
}
```
```java
/*예외처리 회피 2*/
public void add() throws SQLException {
	try {
    	// JDBC API
    }
	catch(SQLException e) {
    	// 로그 출력
    	throw e;
	}
}
```
예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다.
#### 예외 전환
예외 회피와 비슷하게 예외를 복구해서 정상적인 상태로는 만들 수 없기 때문에 예외를 메소드
밖으로 던지는 것이다.

예외 회피와 달리, 발생한 예외를 그대로 넘기는 게 아니라 적절한 예외로 전환해서 던진다.

1. 예외를 그대로 던지는 것이 그 예외상황에 대한 적절한 의미를 부여해주지 못하는 경우에. 의미를 분명하게 해줄 수 있는 예외로 바꿔주기 위해 사용
```java
/*예외 전환 기능을 가진 DAO 메소드*/
public void add(User user) throws DuplicateUserIdException, SQLException {
	try {
		// JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
		// 그런 기능을 가진 다른 SQLException을 던지는 메소드를 호출하는 코드
	}
    catch(SQLException e) {
		// ErrorCode가 MySQL의 "Duplicate Entry(1062)"이면 예외 전환
		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
        	throw DuplicateUserIdException();
		else
			throw e; // 그 외의 경우는 SQLException 그대로
    }
}
```
전환하는 예외에 원래 발생한 예외를 담아서 중첩 예외(nested exception)로 만드는 것이 좋다.
```java
/*중첩 예외 1*/
catch(SOLException e) {
	...
	throw DuplicateUserIdException(e);
```
```java
/*중첩 예외 2*/
catch(SQLException e) {
	...
	throw DuplicateUserIdException().initCause(e);
```
2.  예외를 처리하기 쉽고 단순하게 만들기 위해 포장(wrap)할 때 사용

주로 예외처리를 강제하는 체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우에 사용한다.
```java
/*예외 포장*/
try {
	OrderHome orderHome = EJBHomeFactory.getInstance().getOrderHome();
	Order order = orderHome.findByPrimaryKey(Integer id);
} catch (NamingException ne) {
	throw new EJBException(ne);
} catch (SOLException se) {
	throw new EJBException(se);
} catch (RemoteException re) {
	throw new EJBException(re);
}
```
### 예외처리 전략
#### 런타임 예외의 보편화
체크 예외는 복구할 가능성이 조금이라도 있는 예외적인 상황이기 때문에 자바는 이를 처리하는 `catch` 블록이나 `throws` 선언을 강제하고 있다.

하지만 자바 엔터프라이즈 서버환경에서는 수많은 사용자가 동시에 요청을 보내고 각 요청이 독립적인 작업으로 취급된다.

하나의 요청을 처리하는 중에 예외가 발생하면 해당 작업만 중단시키면 된다.

따라서 애플리케이션 차원에서 예외상황을 미리 파악하고, 예외가 발생하지 않도록 차단하는 게 좋다.

대응이 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 낫다.
#### add() 메소드의 예외처리
먼저 사용자 아이디가 중복됐을 때 사용하는 `DuplicateUserIdException`을 만든다.
```java
/*아이디 중복 시 사용하는 예외*/
public class DuplicateUserIdException extends RuntimeException {
	public DuplicateUserIdException(Throwable cause) {
		super (cause);
	}
}
```
`add()` 메소드를 사용하는 쪽에서 아이디 중복 예외를 처리하고 싶은 경우 활용할 수 있음을 알려주도록 `DuplicatedUserldException`을 메소드의 `throws` 선언에 포함시킨다
```java
/*예외처리 전략을 적용한 add()*/
public void add() throws DuplicateUserIdException {
	try {
		// JDBC를 이용해 user 정보를 DB에 추가하는 코드 또는
		// 그런 기능이 있는 다른 SQLException을 던지는 메소드를 호출하는 코드
	}
	catch (SQLException e) {
		if (e.getErrorCode() == MysqlErrorNumbers.ER_DUP_ENTRY)
			throw new DuplicateUserIdException(e); // 예외 전환
		else
			throw new RuntimeException(e); // 예외 포장
	}
}
```
런타임 예외를 사용하는 경우에는 API 문서나 레퍼런스 문서 등을 통해, 메소드를 사용할 때 발생할 수 있는 예외의 종류와 원인, 활용 방법을 자세히 설명해둔다.
#### 애플리케이션 예외
런타임 예외 중심의 전략은 일단 복구할 수 있는 예외는 없다고 가정하고 예외가 생겨도 어차피 런타임 예외이므로 시스템 레벨에서 알아서 처리해줄 것이고, 꼭 필요한 경우는 런타임 예외라도 잡아서 복구하거나 대응해줄 수 있으니 문제 될 것이 없다는 낙관적인 태도를 기반으로 하고 있다.

반면에 시스템 또는 외부의 예외상황이 원인이 아니라 애플리케이션 자체의 로직에 의해 의도적으로 발생시키고, 반드시 `catch` 해서 무엇인가 조치를 취하도록 요구하는 예외들을 애플리케이션 예외라고 한다.

예금을 인출해서 처리하는 코드를 정상 흐름으로 만들어두고, 잔고 부족을 애플리케이션 예외로 만들어 처리하도록 만든 예시 코드: `InsufficientBalanceException` 사용
```java
/*애플리케이션 예외를 사용한 코드*/
try {
	BigDecimal balance = account.withdraw(amount);
	...
	// 정상적인 처리 결과를 출력하도록 진행
catch(InsufficientBalanceException e) { // 체크 예외
	// InsufficientBalanceExcention에 담긴 인출 가능한 잔고금액 정보를 가져옴
	BigDecimal availfunds = e.getAvailFunds();
	...
	// 잔고 부족 안내 메시지를 준비하고 이를 출력하도록 진행
}
```
### SQLException은 어떻게 됐나?
99%의 `SQLException`은 코드 레벨에서는 복구할 방법이 없다.

필요 없는 기계적인 `throws` 선언이 등장하도록 방치하지 말고 가능한 한 빨리 언체크/런타임 예외로 전환해줘야 한다.

스프링의 `JdbcTemplate`은 이 예외처리 전략을 따르고 있다.

`JdbcTemplate` 템플릿과 콜백 안에서 발생하는 모든 `SQLException`을 런타임 예외인 `DataAccessException`으로 포장해서 던져준다.

`JdbcTemplate`의 `update()`, `queryForInt()`, `query()` 메소드 선언에는 다음과 같이 모두 `throws DataAccessException`이라고 되어 있다.
```java
public int update(final String sql) throws DataAccessException { ... }
```
`throws`로 선언되어 있긴 하지만 `DataAccessException`이 런타임 예외이므로 `update()`를 사용하는 메소드에서 이를 잡거나 다시 던질 의무는 없다.
