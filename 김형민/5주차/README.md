# 4장 예외 1


# **4.1** 사라진 SQLException

```java
//jdbcTemplate 적용 전
public void deleteAll() throws SQLException {
	this.jdbcContext.executeSql("delete from users")；
}

//jdbcTemplate 적용 후
public void deleteAll() {
	this.jdbcTemplate.update("delete from users")；
}
```

JdbcTemplate을 적용하고 나서 이전에 있었던 throws SQLException 선언이 적용 후에는 사라졌다.

## **4.1.1** 초난감 예외처리

### 예외 블랙홀

**난감한 예외처리 코드1**

```java
try {
	...
} catch(SQLException e) {
}
```

- 예외를 잡고 아무것도 하지 않는다. 그렇기에 예외 발생을 무시해버리고 계속해서 로직이 진행된다.

**난감한 예외처리 코드2, 3**

```java
try {
	...
} catch (SQLException e) {
	System.out.println(e);
}

try {
	...
} catch (SQLException e) {
	e.printStackTrace();
}
```

- 콘솔 로그를 누군가 계속 모니터링하지 않는 이상 이 예외 코드는 의미가 없다.

<aside>
💡 모든 예외는 적절하게 복구되든지 아니면 작업을 중단시키고 운영자 또는 개발자에게 분명하게 통보돼야 한다.

</aside>

**그나마 나은 예외처리**

```java
try {
	...
} catch (SQLException e) {
	e.printStackTraceO;
	System.exit(1);
}
```

- 실행을 종료하여 확실히 알아차릴 수 있으므로 그나마 났다.

### 무의미하고 무책임한 throws

**난감한 예외처리 코드4**

```java
public void method1() throws Exception {
	method2()；
}
public void method2() throws Exception {
	method3()；
} 
public void method3() throws Exception {
	...
}
```

- 모든 메소드에 기계적으로 throws Exception을 선언한다.
- 코드를 보고 실행중에 예외적인 상황이 발생할 수 있다는 것인지 판단할 수 없다.

## **4.1.2** 예외의 종류와 특징

### Error

- 시스템에 뭔가 비정상적인 상황이 발생했을 경우에 사용된다.
- 그래서 주로 JVM에서 발생시키는 것이고 애플리케이션 코드에서 잡을 수 없다. ex)OutOfMemoryError

### **Exception과 체크 예외**

- Error와 달리 개발자들이 만든 애플리케이션 코드의 작업 중에 예외상황이 발생했을 경우 사용된다.

**Exception의 두 가지 종류**

<img width="490" alt="Untitled" src="https://github.com/fkgnssla/spring-study/assets/92067099/61f48f96-1f33-474d-835a-63edb4b3ec17">

### 체크 예외

- RuntimeException을 상속하지 않는다.
- 예외 처리를 하지 않으면 컴파일 시점에 오류가 발생한다.
- ex) IOException, SQLException, …

### 언체크 예외

- RuntimeException을 상속한다.
- 예외 처리를 하지 않으면 실행 도중에 오류가 발생할 수 있다.
- 주로 프로그램의 오류가 있을 때 발생하도록 의도된 것들이다.
- 명시적인 예외처리를 강제하지 않는다.
- ex) NullPointerException, IllegalArgumentException

## **4.1.3** 예외처리 방법

### 예외 복구

예외가 발생했다면 예외상황을 파악하고 문제를 해결해서 정상 상태로 돌려놓아야 한다.

**재시도를 통해 예외를 복구하는 코드**

```java
int maxretry = MAX_RETRY；
while(maxretry != 0) {
	try {
		... //예외가 발생할 가능성이 있는 시도
		return； //작업 성공
	}
	catch(SomeException e) {
		//로그 출력
		//정해진 시간만큼 대기
	}
	finally {
	//리소스 반납, 정리 작업
	}
}

throw new RetryFailedException()； //최대 재시도 횟수를 넘기면 직접 예외 발생
```

### 예외처리 회피

예외처리를 자신이 담당하지 않고 자신을 호출한 쪽으로 던져 버린다.

**예외처리를 회피하는 코드**

```java
//1.
public void add() throws SQLException {
// JDBC API
}

//2.
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

**정당성**

- 콜백과 템플릿처럼 긴밀하게 역할을 분담하고 있는 관계가 아니라면 자신의 코드에서 발생하는 예외를 그냥 던져버리는 건 무책임한 책임회피이다.
- 예외를 회피하는 것은 예외를 복구하는 것처럼 의도가 분명해야 한다!

### 예외 전환

예외 회피와 달리, 발생한 예외를 그대로 넘기는 게 아니라 적절한 예외로 전환해서 던진다.

**목적**

1. 예외를 그대로 던지는 것이 그 예외상황에 대한 적절한 의미를 부여해주지 못하는 경우 **의미를 분명하게 해줄 수 있는 예외**로 바꿔 던지기 위해 사용한다.

```java
public void add(User user) throws  DuplicateUserIdException,SQLException{
	try {
	  //Jdbc를 이용해 user 정보를 db에 추가하는 코드 
    //또는 그런기능을 가진 다른 SQLException을 던지는 메소드를 호출하는 코드
	}catch (SQLException e){
    //ErrorCode가 MySQL의 "Duplicate Entry(1062)"면 예외 전환
    if(e.getErrorCode() == MySqlErrorNumbers.ER_DUP_ENTRY)
	    throw DuplicateUserIdException();
    else 
        throw e;
        
  }
}
```

- 중복된 아이디로 인한 SQLException 발생! 이를 DuplicateKeyException으로 전환하여 의미있는 예외를 던진다.

1. 예외를 처리하기 쉽고 단순하게 만들기 위해 **포장**하는 것이다. 주로 예외처리를 강제하는 **체크 예외를 언체크 예외인 런타임 예외로 바꾸는 경우**에 사용한다.

```java
try{
	OrderHome orderHome = EJBHomeFactiory.getInstance().getOrderHome();
	Order order = orderHome.findByPrimaryKey(Integer id);
}catch(NamingException ne){
	throw new EJBException(ne);
}catch(NamingException ne){
	throw new EJBException(ne);
}catch(NamingException ne){
	throw new EJBException(ne);
}
```

- EJBException은 RuntimeException 클래스를 상속한 런타임 예외다.
- 잡아도 복구할 만한 방법이 없는 예외에 쓰인다.(?)

## **4.1.4** 예외처리 전략

### 런타임 예외의 보편화

자바의 환경이 서버로 이동하면서 체크 예외의 활용도와 가치는 점점 떨어지고 언체크 예외(런타임 예외)는 증가하고 있다. 대응이 불가능한 체크 예외라면 빨리 런타임 예외로 전환해서 던지는 게 낫다.

### add() 메소드의 예외처리

4-9에 나온 add() 메소드는 **DuplicatedUserldException**과 **SQLException** 두 가지의 체크 예외를 던진다.

**SQLException**은 **대부분 복구 불가능한 예외**이므로 그냥 런타임 예외로 포장해 던져버려서 그 밖의 메소드들이 신경 쓰지 않게하자!

**DuplicatedUserldException**은 add()메소드를 바로 호출한 오브젝트 보다 더 앞단의 오브젝트에서 다룰수도 있다. 이렇게 **어디에서든 예외처리가 가능**하다면 체크 예외가 아니라 **언체크 예외(런타임 예외)**로 만드는게 났다. 고로 언체크 예외로 만들자!

**아이디 중복 시 사용하는 예외 클래스**

```java
public class DuplicateUserldException extends RuntimeException {
	public DuplicateUserldException(Throwable cause) {
		super(cause);
	}
}
```

**예외처리 전략을 적용한 add()**

```java
public void add(User user) throws DuplicateUserIdException{
	try {
        //Jdbc를 이용해 user 정보를 db에 추가하는 코드 
        //또는 그런기능을 가진 다른 SQLException을 던지는 메소드를 호출하는 코드
	}catch (SQLException e){
        if(e.getErrorCode() == MySqlErrorNumbers.ER_DUP_ENTRY)
            throw DuplicateUserIdException(e); //예외전환
        else 
            throw new new RuntimeException(e); //예외포장
        }
    }
}
```

- DuplicatedUserldException 외에 시스템 예외에 해당하는 SQLException도 언체크 예외가 됐다.
- 따라서 메소드 선언의 throws에 포함시킬 필요가 없지만, 아이디 중복 예외를 처리하고 싶은 경우 활용할 수 있음을 알려주도록 DuplicatedUserldException을 메소드의 throws 선언에 포함시킨다.

### 애플리케이션 예외

시스템 또는 외부의 예외상황이 원인이 아니라 **애플리케이션 자체의 로직에 의해 의도적으로 발생**시키고, **반드시 catch 해서 무엇인가 조치를 취하도록 요구하는 예외**를 의미한다.

## **4.1.5 SQLException**은 어떻게 됐나?

JdbcTemplate을 적용하는 중에 throws SQLException 선언이 왜 사라졌을까?

1. 99%의 SQLException은 코드 레벨에서는 복구할 방법이 없다.
2. 복구할 방법이 없는 SQLException을 필요도 없는 기계적인 throws 선언이 등장하도록 방치하지 말고 가능한 빨리 언체크/런타임 예외로 전환해줘야 한다.
3. 스프링의 JdbcTemplate은 바로 이 예외처리 전략을 따른다.
4. JdbcTemplate은 템플릿과 콜백 안에서 발생하는 모든 SQLException을 런타임 예외인 DataAccessException으로 포장해서 던져준다.
5. 그 밖에도 스프링의 API 메소드에 정의되어 있는 대부분의 예외는 런타임 예외다.
