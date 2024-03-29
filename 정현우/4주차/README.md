# 템플릿
## 다시 보는 초난감 DAO
### 예외처리 기능을 갖춘 DAO
#### JDBC 수정 기능의 예외처리 코드
```java
/*JDBC API를 이용한 DAO 코드인 deleteAll()*/
public void deleteAll() throws SQLException {
	Connection c = dataSource.getConnection();

	PreparedStatement ps = c.prepareStatement("delete from users");
	ps.executeUpdate(); // 여기서 예외가 발생하면 바로 메소드 실행이 중단된다.

	ps.close();
	c.close();
}
```
`PreparedStatement`를 처리하는 중에 예외가 발생하면
`Connection`과 `PreparedStatement`의 `close()` 메소드가 실행되지 않아
제대로 리소스가 반환되지 않을 수 있다.

리소스 반환과 `close()`:
- `Connection`과 `PreparedStatement`는 풀(pool) 방식으로 운영된다.
- 미리 정해진 풀 안에 제한된 수의 리소스를 만들어 두고 필요할 때 이를 할당, 반환 시 다시 풀에 넣는다.
- `close()`로 리소스를 반환한다.
- 반환하지 않으면 풀의 리소스가 고갈되고 문제가 발생할 수 있다.

`try/catch/finally`를 적용한다.
- DB 서버 문제나 네트워크, 그 밖의 예외상황으로 예외가 발생하면 `ps`, `c`가 모두 `null` 상태
- 반드시 `c`와 `ps`가 `null`이 아닌지 먼저 확인 후 `close()` 호출
- `close()`도 `SQLException`이 발생할 수 있는 메소드이므로 `try/catch` 처리
```java
/*예외 발생 시에도 리소스를 반환하도록 수정한 deleteAll()*/
public void deleteAll() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;

	try {
		c = dataSource.getConnection();					// 예외가 발생할 가능성이 있는 코드를
		ps = c.prepareStatement("delete from users");	// 모두 try 블록으로 묶어준다.
		ps.executeUpdate();
	} catch (SQLException e) {	// 예외가 발생했을 때 부가적인 작업을 해줄 수 있도록 catch 블록을 둔다.
		throw e;				// 아직은 예외를 다시 메소드 밖으로 던지는 것밖에 없다.
	} finally { // finally이므로 try 블록에서 예외가 발생했을 때나 안 했을 때나 모두 실행된다.
		if (ps != null) {
			try {
				ps.close();
			} catch (SQLException e) {	// ps.close() 메소드에서도 SQLException이 발생할 수 있기 때문에 이를 잡아줘야 한다. 
		}								// 그렇지 않으면 Connection을 close() 하지 못하고 메소드를 빠져나갈 있다.
	}
	if (c != null) {
		try {
			c.close(); // Connection 반환
		} catch (SQLException e) {
        }
	}
}
```
#### JDBC 조회 기능의 예외처리
`getCount()` 메소드에 예외처리 블록 적용
```java
/*JDBC 예외처리를 적용한 getCount() 메소드*/
public int getCount() throws SQLException {
	Connection c = null;
	PreparedStatement ps = null;
	ResultSet rs = null;
    
	try {
		c = dataSource.getConnection();
        
		ps = c.prepareStatement("select count(*) from users");

		rs = ps.executeQuery();	// ResultSet도 다양한 SQLException이 발생할 수 있는
		rs.next();				// 코드이므로 try 블록 안에 둬야 한다.
		return rs.getInt(1);
	} catch (SQLException e) {
		throw e;
	} finally {
		if (rs != null) {
			try {
				rs.close();				// 만들어진 ResultSet을 닫아주는 기능. close()는
			} catch (SQLException e) {	// 만들어진 순서의 반대로 하는 것이 원칙이다.
			}
		}
		if (ps != null) {
			try {
				ps.close();
			} catch (SQLException e) {
			}
		}
		if (c != null) {
			try {
				c.close();
			} catch (SQLException e) {
			}
        }
    }
}
```
## 변하는 것과 변하지 않는 것
### JDBC try/catch/finally 코드의 문제점
복잡한 `try/catch/finally` 블록이 2중으로 중첩되고, 모든 메소드마다 반복된다.

로직 수정시 복잡한 `try/catch/finally` 블록 안에서 필요한 부분을 찾아서 수정하게 된다.
### 분리와 재사용을 위한 디자인 패턴 적용
```java
/*개선할 deleteAll() 메소드*/
Connection c = null;
PreparedStatement ps = null;

try {
	c = dataSource.getConnection();
	// 변하지 않는 부분

	ps = c.prepareStatement("delete from users"); // 변하는부분

	// 변하지 않는 부분
	ps.executeUpdate();
} catch (SQLException e) {
	throw e;
} finally {
	if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
	if (c != null) { try { c.close(); } catch (SQLException e) {} }
}
```
`deleteAll()` 메소드와 구조가 거의 비슷한 메소드들이 존재한다.

`add()` 메소드의 경우 `deleteAll()`에서 변하는 부분의 코드만 바꾸면 된다.
```java
/*add() 메소드에서 수정할 부분*/
...
ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
ps.setString(1, user.getId());
ps.setString(2, user.getName());
ps.setString(3, user.getPassword());
...
```
#### 메소드 추출
변하지 않는 부분이 변하고 있는 부분을 감싸고 있다.

변하지 않는 부분을 추출하기 어려워 보이므로 변하는 부분을 메소드로 빼 본다.
```java
/*변하는 부분을 메소드로 추출한 후의 deleteAll()*/
public void deleteAll() throws SQLException {
	...
	try {
		c = dataSource.getConnection();

		ps = makeStatement(c);	// 변하는 부분을 메소드로 추출하고 변하지 않는
								// 부분에서 호출하도록 만들었다.
		ps.executeUpdate();
	} catch (SQLException e)
	...
}

private PreparedStatement makeStatement(Connection c) throws SQLException {
	PreparedStatement ps;
	ps = c.prepareStatement("delete from users");
	return ps;
}
```
분리시키고 남은 메소드가 재사용이 필요한 부분으로, 반대로 됐다.
#### 템플릿 메소드 패턴의 적용
변하지 않는 부분은 슈퍼클래스에 두고 변하는 부분은 추상 메소드로 정의해둬서
서브클래스에서 오버라이드하여 새롭게 정의해 쓰도록 만든다.

`makeStatement()` 메소드를 추상 메소드 선언으로 변경한다.
```java
abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
```
이를 상속하는 서브클래스를 만들어서 이 메소드를 구현한다.
```java
/*makeStatement()를 구현한 서브클래스*/
public class UserDaoDeleteAll extends UserDao {

	protected PreparedStatement makeStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```
DAO 로직마다 상속을 통해 새로운 클래스를 만들어야 한다.
- 각각의 메소드마다 서브클래스를 만들어서 사용해야 한다.
확장 구조가 클래스를 설계하는 시점에서 고정되어 버린다.
- 변하지 않는 코드를 가진 `try/catch/finally` 블록과 변하는 `PreparedStatement`를 담고 있는 서브클래스들이 이미 컴파일 시점에 관계가 결정되어있다. -> 관계에 대한 유연성이 떨어진다.
#### 전략 패턴의 적용
템플릿 메소드 패턴보다 유연하고 확장성이 뛰어난 전략 패턴을 사용해 본다.

오브젝트를 둘로 분리하고 인터페이스를 통해서만 의존하도록 만든다.

`deleteAll()`의 컨텍스트:
- DB 커넥션 가져오기
- `PreparedStatement`를 만들어줄 외부 기능 호출하기
- 전달받은 PreparedStatement 실행하기
- 예외가 발생하면 이를 다시 메소드 밖으로 던지기
- 모든 경우에 만들어진 `PreparedStatement`와 `Connection`을 적절히 닫아주기

전략 패턴의 구조를 따라 `PreparedStatement`를 만들어주는 외부 기능을 인터페이스로 만든다.

인터페이스의 메소드를 통해 `PreparedStatement`를 생성하는 전략을 호출한다.
```java
/*StatementStrategy 인터페이스*/
package springbook.user.dao;
...
public interface StatementStrategy {
	PreparedStatement makePreparedStatement(Connection c) throws SQLException;
}
```
이 인터페이스를 상속해서 `PreparedStatement`를 생성하는 클래스를 만든다.
```java
/*deleteAll() 메소드의 기능을 구현한 StatementStrategy 전략 클래스*/
package springbook.user.dao;
...
public class DeleteAllStatement implements StatementStrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("delete from users");
		return ps;
	}
}
```
만들어진 `DeleteAllStatement`를 `UserDao`의 `deleteAll()` 메소드에서 사용한다.
```java
/*전략 패턴을 따라 DeleteAllStatement가 적용된 deleteAll() 메소드*/
public void deleteAll() throws SQLException {
	...
	try {
		c = dataSource.getConnection();
        
		StatementStrategy strategy = new DeleteAllStatement();
		ps = strategy.makePreparedStatement(c);
        
		ps.executeUpdate();
	} catch (SQLException e) {
    ...
}
```
전략패턴은 필요에 따라 컨텍스트는 유지되면서 전략을 바꿔쓸 수 있어야 한다.

컨텍스트 안에서 이미 구체적인 전략 클래스 `DeleteAllStatement`를 사용하도록 고정되어 있으므로 전략 패턴과 OCP(개방 폐쇄 원칙)에 잘 들어맞는다고 볼 수 없다.
#### DI 적용을 위한 클라이언트/컨텍스트 분리
컨텍스트에 해당하는 JDBC `try/catch/finally` 코드를 클라이언트 코드인 `StatementStrategy`를 만드는 부분에서 독립시킨다.

전략 인터페이스인 `StatementStrategy`를 컨텍스트 메소드 파라미터로 지정한다.
```java
/*메소드로 분리한 try/catch/finally 컨텍스트 코드*/
public void jdbcContextWithStatementStrategy(StatementSrategy stmt) throws 
		SQLException { // 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터
	Connection c = null;
	PreparedStatement ps = null;

	try {
		c = dataSource.getConnection();

		ps = stmt.makePreparedStatement(c);

		ps.executeUpdate()
	} catch (SQLException e) {
		throw e;
	} finally {
		if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
		if (c != null) { try { c.close(); } catch (SQLException e) {} }
	}
}
```
사용할 전략 클래스는 `DeleteAllStatement`이므로 이 클래스의 오브젝트를 생성하고, 컨텍스트로 분리한 `jdbcContextWithStatementStrategy()` 메소드를 호출해준다.
```java
/*클라이언트 책임을 담당할 deleteAll() 메소드*/
public void deleteAll() throws SQLException {
	StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트 생성
	jdbcContextWithStatementStrategy(st); // 컨텍스트 호출. 전략 오브젝트 전달
}
```
## JDBC 전략 패턴의 최적화
### 전략 클래스의 추가 정보
`add()` 메소드에서도 변하는 부분인 `PreparedStatement`를 만드는 코드를 `AddStatement` 클래스로 옮겨 담는다.
```java
/*add() 메소드의 PreparedStatement 생성 로직을 분리한 클래스*/
public class AddStatement implements Statementstrategy {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword()); // 그런데 user는 어디서 가져올까?

return ps;
```
클라이언트로부터 `User` 타입 오브젝트를 받을 수 있도록 `AddStatement`의 생성자를 통해 제공받게 만든다.
```java
/*User 정보를 생성자로부터 제공받도록 만든 AddStatement*/
package springbook.user.dao;
...
public class AddStatement implements Statementstrategy {
	User user;

	public AddStatement(User user) {
		this.user = user;
	}

	public PreparedStatement makePreparedStatement(Connection c) {
		...
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());
		...
	}
}
```
클라이언트인 `UserDao`의 `add()` 메소드에 `User` 정보를 생성자를 통해 전달해주도록 한다.
```java
/*user 정보를 AddStatement에 전달해주는 add() 메소드*/
public void add(User user) throws SQLException {
	StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```
`deleteAll()`과 `add()` 두 군데에서 모두 `PreparedStatement`를 실행하는 JDBC `try/catch/finally` 컨텍스트를 공유해서 사용할 수 있게 됐다.
### 전략과 클라이언트의 동거
DAO 메소드마다 새로운 `StatementStrategy` 구현 클래스를 만들어야 하므로 기존 `UserDao` 때보다 클래스 파일의 개수가 많이 늘어난다.

오브젝트를 전달받는 생성자와 이를 저장해둘 인스턴스 변수를 번거롭게 만들어야 한다.
#### 로컬 클래스
`DeleteAllStatement`나 `AddStatement`는 `UserDao` 밖에서는 사용되지 않으므로 로컬 클래스로 만들 수 있다. 
```java
/*add() 메소드 내의 로컬 클래스로 이전한 AddStatement*/
public void add(User user) throws SQLException {
	class AddStatement implements Statementstrategy { // add() 메소드 내부에 선언된 로컬 클래스다.
		User user;

		public AddStatement(User user) {
			this.user = user;
		}
        
		public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) value(?,?,?)");
			ps.setString(1, user.getId());
			ps.setString(2, user.getName());
			ps.setstring(3, user.getPassword());

			return ps;
        }
    }
    
    StatementStrategy st = new AddStatement(user);
	jdbcContextWithStatementStrategy(st);
}
```
`add()` 메소드 내에 `AddStatement` 클래스를 정의했으므로 생성자를 통해 `User` 오브젝트를 전달해줄 필요가 없다.
```java
/*add() 메소드의 로컬 변수를 직접 사용하도록 수정한 AddStatement*/
public void add(final User user) throws SQLException {
	class AddStatement implements StatementStrategy {
		public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
			PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
			ps.setString(1, user.getId());			// 로컬(내부) 클래스의 코드에서 외부의 메소드
			ps.setString(2, user.getName());		// 로컬 변수에 직접 접근할 수 있다.
			ps.setString(3, user.getPassword());
            
			return ps;
		}
    }
    
	Statementstrategy st = new AddStatement(); // 생성자 파라미터로 user를 전달하지 않아도 된다.
	jdbcContextWithStatementStrategy(st);
}
```
메소드마다 추가해야 했던 클래스 파일을 줄일 수 있다.

내부 클래스의 특징을 이용해 로컬 변수를 바로 가져다 사용할 수 있다.
#### 익명 내부 클래스
`AddStatement` 클래스는 `add()` 메소드에서만 사용할 용도로 만들어졌다.

따라서 클래스 이름도 제거할 수 있다.
```java
/*AddStatement를 익명 내부 클래스로 전환*/
// 익명 내부 클래스는 구현하는 인터페이스를 생성자처럼 이용해서 오브젝트로 만든다.
Statementstrategy st = new StatementStrategy() {
	public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
		PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());
        
		return ps;
	}
};
```
익명 내부 클래스의 오브젝트는 딱 한 번만 사용하므로 굳이 변수에 담아두지 말고 `jdbcContextWithStatementStrategy()` 메소드의 파라미터에서 바로 생성하게 해준다.
```java
/*메소드 파라미터로 이전한 익명 내부 클래스*/
public void add(final User user) throws SQLException {
	jdbcContextWithStatementStrategy(
			new Statementstrategy() {
				public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
					PreparedStatement ps = c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
					ps.setString(1, user.getId());
					ps.setString(2, user.getName());
					ps.setString(3, user.getPassword());
                    
					return ps;
				}
			}
	);
}
```
`DeleteAllStatement`도 `deleteAll()` 메소드의 익명 내부 클래스로 처리한다.
```java
/*익명 내부 클래스를 적용한 deleteAll() 메소드*/
public void deleteAll() throws SQLException {
	jdbcContextWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement("delete from users");
			}
		}
	);
}
```
## 컨텍스트와 DI
### JdbcContext의 분리
`jdbcContextWithStatementStrategy()`를 `UserDao` 클래스 밖으로 독립시켜서 모
든 DAO가 사용할 수 있게 만든다.
#### 클래스 분리
`JdbcContext` 클래스로 분리해준다.

`JdbcContext`가 `DataSource`에 의존하고 있으므로 `DataSource` 타입 빈을 DI 받을 수 있게 해준다.
```java
/*JDBC 작업 흐름을 분리해서 만든 JdbcContext 클래스*/
package springbook.user.dao;
...
public class JdbcContext {
	private DataSource dataSource;
    // DataSource 타입 빈을 DI 받을 수 있게 준비해둔다.
	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}
    // jdbcContext 클래스 안으로 옮겼으므로 이름도 그에 맞게 수정했다.
	public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException { 
		Connection c = null;
		PreparedStatement ps = null;
        
		try {
			c = this.dataSource.getConnection();
            
			ps = stmt.makePreparedStatement(c);
            
			ps.executeUpdate();
		} catch (SQLException e) {
			throw e;
		} finally {
			if (ps != null) { try { ps.close(); } catch (SQLException e) {} }
			if (c != null) { try { c.close(); } catch (SQLException e) {} }
		}
	}
}
```
`UserDao`가 분리된 `JdbcContext`를 DI 받아서 사용할 수 있게 만든다.
```java
/*JdbcContext를 DI 받아서 사용하도록 만든 UserDao*/
public class UserDao {
	...
	private JdbcContext jdbcContext;
    
	public void setJdbcContext(JdbcContext jdbcContext) {
		this.jdbcContext = jdbcContext; // JdbcContext를 DI 받도록 만든다.
	}

	public void add(final User user) throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() { ... } // DI 받은 JdbcContext의 컨텍스트 메소드를 사용하도록 변경한다.
		);
	}

	public void deleteAll() throws SQLException {
		this.jdbcContext.workWithStatementStrategy(
			new StatementStrategy() { ... }
		);
	}
}
```
#### 빈 의존관계 변경
`UserDao`와 `JdbcContext`는 인터페이스를 사이에 두지 않고 의존 관계가 형성된다.

`userDao` 빈과 `dataSource` 빈 사이에 `jdbcContext` 빈이 들어가게 된 의존관계에 따라 XML 설정파일을 수정한다.
```xml
<!-- JdbcContext 빈을 추가하도록 수정한 설정파일 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2O01/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
		http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="userDao" class="springbook.user.dao.UserDao">
      	<!-- UserDao 내에 아직 JdbcContext를 적용하지 않은 메소드가 있어서 제거하지 않았다. -->
		<property name="dataSource" ref="dataSource" />
		<property name="jdbcContext" ref="jdbcContext" />
	</bean>
  
  	<!-- 추가된 JdbcContext 타입 빈 -->
	<bean id="jdbcContext" class="springbook.user.dao.JdbcContext">
		<property name="dataSource" ref="dataSource" />
	</bean>
  
	<bean id="dataSource"
		class="org.springframework.jdbc.datasource.SimpleDriverDataSource" >
      	...
	</bean>
</beans>
```
### JdbcContext의 특별한 DI
#### 스프링 빈으로 DI
#### 코드를 이용하는 수동 DI
`JdbcContext`를 스프링의 빈으로 등록해서 `UserDao`에 DI 하는 대신 `UserDao` 내부에서 직접 DI를 적용하는 방법이다.

`JdbcContext`를 싱글톤으로 만드는 것은 포기해야 하지만 DAO마다 하나의 `JdbcContext` 오브젝트를 갖고 있게 할 수 있다.

먼저 설정파일에 등록했던 `JdbcContext` 빈과 `UserDao`의 `jdbcContext` 프로퍼티를 제거하고 UserDao는 DataSource 타입 프로퍼티만 갖도록 한다. 
```xml
<!-- jdbcContext 빈을 제거한 설정파일 -->
<beans>
	<bean id="userDao" class="springbook.user.dao.UserDao">
		<property name="dataSource" ref="dataSource" />
	</bean>
	<bean id="dataSource" class="org.springframework.jdbc.datasource.SimpleDriverDataSource" >
	...
  	</bean>
</beans>
```
`UserDao`가 `JdbcContext`를 외부에서 주입받을 필요가 없으니 `setJdbcContext()`를 제거한다.

`setDataSource()` 메소드를 수정한다.
```java
/*JdbcContext 생성과 DI 작업을 수행하는 setDataSource() 메소드*/
public class UserDao {
	...
	private JdbcContext jdbcContext;
    // 수정자 메소드이면서 JdbcContext에 대한 생성, DI 작업을 동시에 수행한다.
	public void setDataSource(DataSource dataSource) {
		this.jdbcContext = new JdbcContext(); // JdbcContext생성(IoC)

        
		this.jdbcContext.setDataSource(dataSource); // 의존 오브젝트 주입(DI)

        
		this.dataSource = dataSource; // 아직 JdbcContext를 적용하지 않은 메소드를 위해 저장해둔다.
	}
}
```
## 템플릿과 콜백
### 템플릿/콜백의 동작원리
템플릿(template): 고정된 틀 안에 바꿀 수 있는 부분을 넣어서 사용하는 경우
콜백(callback): 템플릿 안에서 호출되는 것을 목적으로 만들어진 오브젝트
#### 템플릿/콜백의 특징
콜백 인터페이스의 파라미터는 템플릿의 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달받을 때 사용된다.

`JdbcContext`에서는 템플릿인 `workWithStatementStrategy()` 메소드 내에서 생성한 `Connection` 오브젝트를 콜백의 메소드인 `makePreparedStatement()`를 실행할 때 파라미터로 넘겨준다.
![](https://velog.velcdn.com/images/torch/post/d537143f-e8b7-4248-8435-8adc7d4c1069/image.png)
#### JdbcContext에 적용된 템플릿/콜백
![](https://velog.velcdn.com/images/torch/post/cda756d1-230c-4aa2-81dd-2f34ee853a9e/image.png)
### 편리한 콜백의 재활용
DAO 메소드에서 매번 익명 내부 클래스를 사용하기 때문에 상대적으로 코드를 작성하고 읽기가 조금 불편하다.
#### 콜백의 분리와 재활용
분리를 통해 재사용이 가능한 코드를 찾아낼 수 있다면 익명 내부 클래스를 사용한 코드를 간결하게 만들 수도 있다.
```java
/*익명 내부 클래스를 사용한 클라이언트코드*/
public void deleteAll() throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
    	// 변하지 않는 콜백 클래스 정의와 오브젝트 생성
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement("delete from users"); // 변하는 SQL 문장
			}
		}
	);
}
```
SQL 문장만 파라미터로 받아서 바꿀 수 있게 하고 메소드 내용 전체를 분리해 별도의 메소드로 만든다.
```java
/*변하지 않는 부분을 분리시킨 deleteAll() 메소드*/
public void deleteAll() throws SQLException {
	executeSql("delete from users"); // 변하는 SQL 문장
}
// --------------------------------------------------------------- 분리
private void executeSql(final String query) throws SQLException {
	this.jdbcContext.workWithStatementStrategy(
    	// 변하지 않는 콜백 클래스 정의와 오브젝트 생성
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement(query);
			}
		}
	);
}
```
#### 콜백과 템플릿의 결합
`executeSql()` 메소드는 `UserDao` 이외에도 사용할 수 있게 만든다.

콜백 생성과 템플릿 호출이 담긴 `executeSql()` 메소드를 `JdbcContext` 클래스로 옮긴다.
```java
/*JdbcContext로 옮긴 executeSql() 메소드*/
public class JdbcContext {
...
public void executeSql(final String query) throws SQLException {
	workWithStatementStrategy(
		new StatementStrategy() {
			public PreparedStatement makePreparedStatement(Connection c) throws SQLException {
				return c.prepareStatement(query);
			}
		}
	);
}
```
`UserDao`의 메소드에서도 `jdbcContext`를 통해 `executeSql()` 메소드를 호출하도록 수정한다.
```java
/*JdbcContext로 옮긴 executeSql()을 사용하는 deleteAll() 메소드*/
public void deleteAll() throws SQLException {
	this.jdbcContext.executeSql("delete from users");
}
```
모든 DAO 메소드에서 `executeSql()` 메소드를 사용할 수 있게 됐다.
### 템플릿/콜백의 응용
스프링에는 다양한 자바 엔터프라이즈 기술에서 사용할 수 있도록 미리 만들어져 제공되는 수십 가지 템플릿/콜백 클래스와 API가 있다.
#### 테스트와 try/catch/finally
템플릿/콜백 예제를 만든다.

`numbers.txt` 파일을 준비한다.
```txt
1
2
3
4
```
모든 라인의 숫자를 더한 합(`10`)을 돌려주는 코드의 테스트를 만든다.
```java
/*파일의 숫자 합을 계산하는 코드의 테스트*/
package springbook.learningtest.template;
...
public class CalcSumTest {
	@Test
	public void sumOfNumbers() throws IOException {
		Calculator calculator = new Calculator();
		int sum = calculator.calcSum(getClass().getResource("numbers.txt").getPath());
		assertThat(sum, is(10));
	}
}
```
`calcSum()` 메소드를 갖는 `Calculator` 클래스를 만든다.
```java
/*처음 만든 Calculator 클래스 코드*/
package springbook.learningtest.template;
...
public class Calculator {
	public Integer calcSum(String filepath) throws IOException {
    	// 한 줄씩 읽기 편하게 BufferedReader로 파일을 가져온다.
		BufferedReader br = new BufferedReader(new FileReader(filepath));
		Integer sum = 0;
		String line = null;
		while((line = br.readLine()) != null) { // 마지막 라인까지 한 줄씩 읽어가면서 숫자를 더한다.
			sum += Integer.valueOf(line);
		}
        
		br.close(); // 한번 연 파일은 반드시 닫아준다.
		return sum;
	}
}
```
`calcSum()` 메소드에 `try/catch/finally`를 적용한다.
```java
/*try/catch/finally를 적용한 calcSum() 메소드*/
public Integer calcSum(String filepath) throws IOException {
	BufferedReader br = null;
    
	try {
		br = new BufferedReader(new FileReader(filepath));
		Integer sum = 0;
		String line = null;
		while((line = br.readLine()) != null) {
			sum += Integer.valueOf(line);
		}
		return sum;
	}
	catch(IOException e) {
		System.out.printin(e.getMessage());
		throw e;
	}
	finally {
		if (br != null) { // BufferedReader 오브젝트가 생성되기 전에 예외가 발생할 수도 있으므로 반드시 null 체크를 먼저 해야 한다.
			try { br.close(); }
			catch(IOException e) { System.out.printin(e.getMessage()); }
		}
	}
}
```
#### 중복의 제거와 템플릿/콜백 설계
파일에 있는 모든 숫자의 곱을 계산하는 기능을 추가해야 한다.

파일을 읽어서 처리하는 비슷한 기능을 재사용 할 수 있도록 템플릿/콜백 패턴을 적용한다.

템플릿이 `BufferedReader`를 만들어서 콜백에게 전달해주고, 콜백이 각 라인을 읽어서 알아서 처리한 후에 최종 결과만 템플릿에게 돌려주도록 한다.
```java
/*BufferedReader를 전달받는 콜백 인터페이스*/
package springbook.learningtest.template;
...
public interface BufferedReaderCallback {
	Integer doSomethingWithReader(BufferedReader br) throws IOException;
}
```
`BufferedReaderCallback`을 적용한 템플릿 코드를 만든다.
```java
/*BufferedReaderCallback을 사용하는 템플릿 메소드*/
public Integer fileReadTemplate(String filepath, BufferedReaderCallback callback)
		throws IOException {
	BufferedReader br = null;
	try {
		br = new BufferedReader(new FileReader(filepath));	// 콜백 오브젝트 호출. 템플릿에서 만든 컨
		int ret = callback.doSomethingWithReader(br);		// 텍스트 정보인 BufferedReader를 전달해
		return ret;											// 주고 콜백의 작업 결과를 받아둔다.
	}
	catch(IOException e) {
		System.out.printin(e.getMessage());
		throw e;
	}
	finally {
		if (br != null) {
			try { br.close(); }
			catch(lOException e) { System.out.println(e.getMessage()); }
		}
	}
}
```
`fileReadTemplate()`을 사용하도록 `calcSum()`을 수정한다.
```java
/*템플릿/콜백을 적용한 calcSum() 메소드*/
public Integer calcSum(String filepath) throws IOException {
	BufferedReaderCallback sumCallback =
		new BufferedReaderCallback() {
			public Integer doSomethingWithReader(BufferedReader br) throws IOException {
				Integer sum = 0;
				String line = null;
				while((line = br.readLine()) != null) {
					sum += Integer.valueOf(line);
				}
				return sum;
			}
	};
	return fileReadTemplate(filepath, sumCallback);
}
```
테스트 클래스에 곱을 계산하는 기능의 테스트 메소드를 추가한다.
```java
/*새로운 테스트 메소드를 추가한 CalcSumTest*/
package springbook.learningtest.template;
...
public class CalcSumTest {
	Calculator calculator;
	String numFilepath;
    
	@Before public void setUp() {
			this.calculator = new Calculator();
			this.numFilepath = getClass().getResource("numbers.txt").getPath();
	}
    
	@Test public void sumOfNumbers() throws IOException {
		assertThat(calculator.calcSum(this.numFilepath), is(10));
	}
    
	@Test public void multiplyOfNumbers() throws IOException {
		assertThat(calculator.calcMultiply(this.numFilepath), is(24));
	}
}
```
곱하는 기능을 담은 `calcMultiply()` 메소드를 만든다.
```java
/*곱을 계산하는 콜백을 가진 calcMultiply() 메소드*/
public Integer calcMultiply(String filepath) throws IOException {
	BufferedReaderCallback multiplyCallback =
		new BufferedReaderCallback() {
			public Integer doSomethingWithReader(BufferedReader br) throws IOException {
				Integer multiply = 1;
				String line = null;
				while((line = br.readLine()) != null) {
					multiply *= Integer.valueOf(line);
				}
				return multiply;
			}
	};
	return fileReadTemplate(filepath, multiplyCallback);
}
```
#### 템플릿/콜백의 재설계
`calcSum()`과 `calcMultiply()` 메소드가 거의 유사하다.

템플릿과 콜백을 분리하기 위해 라인별 작업을 콜백 인터페이스로 정의한다.
```java
/*라인별 작업을 정의한 콜백 인터페이스*/
package springbook.learningtest.template;
...
public interface LineCallback {
	Integer doSomethingWithLine(String line, Integer value);
}
```
`LineCallback` 인터페이스를 경계로 해서 새로운 템플릿을 만든다.
```java
/*LineCallback을 사용하는 템플릿*/
public Integer lineReadTemplate(String filepath, LineCallback callback, int initVal) throws IOException {
	BufferedReader br = null;
    try {
		br = new BufferedReader(new FileReader(filepath));
		Integer res = initVal;
		String line = null;
		while((line = br.readLine()) != null) { // 파일의 각 라인을 루프를 돌면서 가져오는 것도 템플릿이 담당한다.
        	// 콜백이 계산한 값을 저장해뒀다가 다음 라인 계산에 다시 사용한다.
			res = callback.doSomethingWithLine(line, res);
            // 각 라인의 내용을 가지고 계산하는 작업만 콜백에게 맡긴다.
		}
		return res;
	}
	catch(IOException e) { ... }
	finally { ... }
}
```
새로운 템플릿을 사용하도록 `calSum()`, `calcMutiply()` 코드를 수정한다.
```java
/*lineReadTemplate()을 사용하도록 수정한 calSum(), calcMutiply() 메소드*/
public Integer calcSum(String filepath) throws IOException {
	LineCallback sumCallback =
		new LineCallback() {
			public Integer doSomethingWithLine(String line, Integer value) {
				return value + Integer.valueOf(line);
			}};
	return lineReadTemplate(filepath, sumCallback, 0);
}

public Integer calcMultiply(String filepath) throws IOException {
	LineCallback multiplyCallback =
		new LineCallback() {
			public Integer doSomethingWithLine(String line, Integer value) {
				return value * Integer.valueOf(line);
			}};
	return lineReadTemplate(filepath, multiplyCallback, 1);
}
```
#### 제네릭스를 이용한 콜백 인터페이스
`Integer` 타입의 결과만 다루는 콜백과 템플릿을 스트링 타입의 값도 처리할 수 있도록 제네릭 타입을 적용해 확장한다.
```java
/*타입 파라미터를 적용한 LineCallback*/
public interface LineCallback<T> {
	T doSomethingWithLine(String line, T value);
}
```
`lineReadTemplate()` 메소드도 타입 파라미터를 사용해 제네릭 메소드로 만든다.
```java
/*타입 파라미터를 추가해서 제네릭 메소드로 만든 lineReadTemplate()*/
public <T> T lineReadTemplate(String filepath, LineCallback<T> callback, T initVal) throws IOException {
	BufferedReader br = null;
	try {
		br = new BufferedReader(new FileReader(filepath));
		T res = initVal;
		String line = null;
        while((line = br.readLine()) != null) {
			res = callback.doSomethingWithLine(line, res);
		}
		return res;
	}
	catch(IOException e) {...}
	finally {...}
}
```
모든 라인의 내용을 문자열로 길게 연결하는 기능을 가진 `concatenate()` 메소드를 만든다.
```java
/*문자열 연결 기능 콜백을 이용해 만든 concatenate() 메소드*/
public String concatenate(String filepath) throws IOException {
	LineCallback<String> concatenateCallback =
		new LineCallback<String>() {
		public String doSomethingWithLine(String line, String value) {
			return value + line;
		}};
	return lineReadTemplate(filepath, concatenateCallback, ""); // 템플릿 메소드의 T는 모두 스트링이 된다.
}
```
`concatenate()` 메소드에 대한 테스트를 만든다.
```java
/*concatenate() 메소드에 대한 테스트*/
@Test
public void concatenateStrings() throws IOException {
	assertThat(calculator.concatenate(this.numFilepath), is("1234"));
}
```
## 스프링의 JDBCTEMPLATE
`JdbcContext`를 스프링의 `JdbcTemplate`으로 교체한다.

`JdbcTemplate` 생성자의 파라미터로 `DataSource`를 주입한다.
```java
/*JdbcTemplate의 초기화를 위한 코드*/
public class UserDao {
	...
	private JdbcTemplate JdbcTemplate;
    
	public void setDataSource(DataSource dataSource) {
		this.jdbcTemplate = new JdbcTemplate(dataSource);
        
		this.dataSource = dataSource;
	}
```
### update()
`JdbcTemplate`의 콜백과 템플릿 메소드를 사용하도록 `deleteAll()` 메소드를 수정한다.

`Statementstrategy` 인터페이스의 `makePreparedStatement()` 메소드에 대응되는
`PreparedStatementCreator` 인터페이스의 `createPreparedStatement()` 메소드가 존재한다.

`PreparedStatementCreator` 타입의 콜백을 받아서 사용하는 `JdbcTemplate`의 템플릿 메소드는 `update()`이다.
```java
/*JdbcTemplate을 적용한 deleteAll() 메소드*/
public void deleteAll() {
	this.jdbcTemplate.update(
		new PreparedStatementCreator() {
			public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
				return con.prepareStatement("delete from users");
			}
		}
	);
}
```
`JdbcTemplate`에는 `executeSql()`과 기능이 비슷한 `update()` 메소드가 존재한다.

앞의 `update()`와 이름이 동일한데 SQL 문장을 파라미터로 받는다.
```java
/*내장 콜백을 사용하는 update()로 변경한 deleteAll() 메소드*/
public void deleteAll() {
	this.jdbcTemplate.update("delete from users");
}
```
`add()` 메소드의 기능도 포함하고 있다.
```java
/*add() 메소드의 콜백 내부*/
PreparedStatement ps =
	c.prepareStatement("insert into users(id, name, password) values(?,?,?)");
ps.setString(1, user.getId());
ps.setString(2, user.getName());
ps.setString(3, user.getPassword());
```
바인딩할 파라미터를 `update()`에 순서대로 넣어준다.
```java
this.jdbcTemplate.update("insert into users(id, name, password) values(?,?,?)",
	user.getId(), user.getName(), user.getPassword());
```
### queryForInt()
아직 템플릿/콜백 방식을 적용하지 않았던 메소드에 `JdbcTemplate`을 적용한다.

기존의 `getCount()`는 SQL 쿼리를 실행하고 `ResultSet`을 통해 결과 값을 가져오는 메소드이다.

`PreparedStatementCreator` 콜백과 `ResultSetExtractor` 콜백을 파라미터로 받는 `query()` 메소드를 사용한다.
```java
/*JdbcTemplate을 이용해 만든 getCount()*/
public int getCount() {
	return this.jdbcTemplate.query(new PreparedStatementCreator() { // 첫 번째 콜백. Statement 생성
		public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
			return con.prepareStatement("select count(*) from users");
        }
	}, new ResultSetExtractor<Integer>() { // 두 번째 콜백. ResultSet으로부터 값 추출
		public Integer extractData(ResultSet rs) throws SQLException,
				DataAccessException {
			rs.next();
			return rs.getInt(1);
		}
	});
}
```
`ResulSet`에서 추출할 수 있는 값의 타입이 다양하기 때문에 `ResultSetExtractor`는 제네릭 타입 파라미터를 갖는다.

SQL의 실행 결과가 하나의 정수 값이 되는 경우는 자주 볼 수 있다.

따라서 `JdbcTemplate`에서는 `queryForInt()`라는 메소드를 제공한다.
```java
/*queryForInt()를 사용하도록 수정한 getCount()*/
public int getCount() {
	return this.JdbcTemplate.queryForInt("select count(*) from users");
}
```
