

#  오브젝트와 의존 관계

>일단 사용자 정보를 jdbc api를 통해 db에 저장하고 조회할수 있는 dao를 만들어보자.

#### 예시코드

```java
public class UserDao{
	public void add(User user) throws ClassNotFoundExcetion,SQLException {
        // 사용자 정보를 생성하는 메소드 
		Class.forName("com.mysql.jdbc.Driver");
        Connciton c = DriverManger.getConnection(
            	"jdbc:mysq://localhost/springbook","spring","book";
        	);
        PreparedStatement ps = c.prepareStatement(
            	"insert into users(id, name , password) values(?,?,?)";
        );
        
        ps.setString(1,user.getId());
        ps.setString(1,user.getName());
        ps.setString(1,user.getPassword());
        
        ps.excuteUpdate();
        
        ps.close();
        c.close();
	}
    
    public User get(String id) throws ClassNotFoundExcetion,SQLException {
        // 사용자 정보를 가져오는 메소드 
		Class.forName("com.mysql.jdbc.Driver");
        Connciton c = DriverManger.getConnection(
            	"jdbc:mysq://localhost/springbook","spring","book";
        	);
        PreparedStatement ps = c.prepareStatement(
            	"insert into users(id, name , password) values(?,?,?)";
        );
        
       ps.setStirng(1,id);
       
       ResultSet re = ps.executeQuery();
       rs.next();
       User user = new User();
       user.setId(rs.getString("id"));
       user.setName(rs.getString("name"));
       user.setPassword(rs.getString("password"));
        
       rs.close();
       ps.close();
       c.close();
        
       return user;
	}
}
```

>기능에는 전혀 문제가 없지만 유지, 보수, 확장하기 어려운 **초난감 Dao가 완성 되었다 .**
>
>이제 초난감한 Dao를 개선해보자.

### **관심사 분리**

> 현재 초난감 dao에서 기능 변경이 있을경우 **어떻게 필요한 최소한 작업을 최소화 하고, 변경이 문제를 일으키지 않게** 할수 있을까?
>
> 프로그래밍 기초 개념중에 **관심사의 분리**라는게 있다 . 이를 객체 지향에 적용해 보면, 관심이 같은 것끼리는 하나의 객체안으로 또는 친한 객체로 모이게 하고, 관심이 다른것 끼리는 가능한 따로 떨어져서로 영향을 주지 않도록 분리하는것이라 생각 해볼수 있다 .

***

> **UserDao의 관심사항**

- db연결을 위한 커넥션을 어떻게 가져올지
- 사용자 등록을 위해 db에 보낼 sql 문장을 담을 Statement를 만들고 실행하는것
- 작업이 끝나면 리소스인 Statemane 와  Connection 오브젝트를 닫아주는것.

> **가장 문제가 되는건 db 커넥션을 가져오는 부분이다**. 앞으로 여러개 , 수백개의 dao 클래스를 만들면 db커넥션을 가져오는 코드도 수백개가 되어 버릴 것이기 때문이다.

***

#### 중복 코드의 메소드 추출

```java
public void add(User user) throws ClassNotFoundExctption,SPLExcetion{
	Connection c = getConnection();
	... //생략
}
public User get(String id) throws ClassNotFoundExctption,SPLExcetion{
	Connection c = getConnection();
	... //생략
}
private Connection getConnection() throws ClassNotFoundExctption,SPLExcetion {
   Class.forName("com.mysql.jdbc.Driver");
        Connciton c = DriverManger.getConnection(
            	"jdbc:mysq://localhost/springbook","spring","book";
        	);
    return c;
}
```

> 나중에 db연결 부분에 변경이 일어 났을 경우 , getConnection 메소드만 수정해 주면 된다 .

***

### **클래스의 분리**

> 슈퍼 울트라 캡숑  dao가 되어 버려서 구매하고 싶다는 회사에 납품을 하려한다.
>
> 하지만 n사와 d사와 각기 다른 종류의 db를 사용하고 db 커넥션을 가져오는데 독자적인 방법으로 적용 하고 싶어 한다는 문제가 생겼다. 더 큰문제는 db커넥션을 가져오는 방법이 종종 변경 될 가능성이 있다는점 !
>
> 슈퍼 울트라 캡숑 UserDao 소스코드를 공개하고 알아서 수정하라 할 수 있지만,
>
> 어떻게 하면 **울트라 캡숑 기술을 공개하고 싶지않으면서 고객 스스로 원하는 db 커네션 생성 방식을 사용하게 할 수 있을까**?

***

#### 상속을 통한 확장
> 기존 UserDao 코드를 한 단계 더 분리한다 .
>
> UserDao에서 구현 코드를 제거하고 getConnection()을 추상 메소드로 만든다.
>
> -> 따라서 add(), get() 메소드에서 getConnection() 을 호출하는 코드는 그대로 유지할 수 있다.

**수정코드**

```java
public avstract class UserDao{
    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = getConnection();
        ... //생략
    }
     public User get(String id) throws ClassNotFoundException, SQLException{
        Connection c = getConnection();
        ... //생략
    }
    pubilc abstract Connection getConnection() throws ClassNotFoundException, SQLException;
    //구현코드는 제거 되고 추상메서드로 바뀌었다 , 메소드의 구현은 서브클래스가 담당 
}
```

```java
public class NuserDao extends UserDao{
	public Connection getConnection()  throws ClassNotFoundException, SQLException{
		//n사 db 생성코드
	}
}

public class DuserDao extends UserDao{
	public Connection getConnection()  throws ClassNotFoundException, SQLException{
		//d사 db 생성코드
	}
}
```

> **상속을 통해 관심을 분리하고 변신이 가능하도록 확장성도 줬지만 문제점이 있다.**

- 자바는 다중 상속을 허용하지 않기때문에 나중에 다른 이유로 UserDao에 상속을 하기 어렵다.
- 서브클래스는 슈퍼클래스의 기능을 직접 사용할 수 있기 때문에, 슈퍼 클래스 내부의 변경이 있을 경우 모든 서브클래스를 함께 수정하거나 다시 개발해야할 수도 있음
- 확장 기능인 db 커넥션을 생성하는 코드를 다른 Dao 클래스에 적용할 수 없다. (다른 Dao 클래스들이 계속 만들어 진다면 상속을 통해 만들어진 getConnection() 의 구현 코드가 매 Dao 클래스마다 중복됨)

***

#### 독립적인 클래스 분리
>
> 새로운 SimpleConnectionMaker 이라는 클래스를 만들고 db 생성기능을 그 안에 넣는다 .
>

**수정 코드 ( db생성 기능을 독립시킨 SimpleConnectionMaker 클래스는 생략)**

```java
public class UserDao{
	private SimpleConnectionMaker  simpleConnectionMaker;
	
	public UserDao(){
		simpleConnectionMaker = new SimpleConnectionMaker ();
	}
    
    public void add(User user) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
        ... //생략
    }
    
    public User add(String id) throws ClassNotFoundException, SQLException{
        Connection c = simpleConnectionMaker.makeNewConnection();
        ... //생략
    }
}
```

> **독립적인 클래스를 통한 수정을 했지만 상속과는 다른 문제가 발생했다**

- 상속을 통해 db 커넥션 기능을 확장해서 사용하게 했던게 다시 불가능해 졌다.
- 코드의 수정없이 db 커넥션 생성 기능을 변경할 방법이 없다.

***

#### **인터페이스의 도입**

>클래스를 분리하면서도 이런 문제를 해결하기 위해 두개의 클래스가 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리를 만들어 주는것이다 .
>
>인터페이스를 사용해 초기 코드에서 클래스를 분리 ,
>
>자신을 구현한 클래스에 대한 구체적인 정보는 감춰버림
>
>-> 오브젝트를 만들면 구체적인 클래스 하나를 선택해야 겠지만 ,
>
>인터페이스로 추상화 해놓은 최소한의 통로를 통해 접근하는 쪽에서는 **사용할 클래스가 무엇인지 몰라도된다**.
>
>**인터페이스를 통해 접근하게 하면 실제 구현 클래스를 바꾸어도 신경쓸 일이없다.**

**수정 코드**

```java
public class DConnectionMaker implements ConnectionMaker{
	public Conneciton makeConnection() throws ClassNotFoundException , SQLException{
			// Connetction 을 생성하는 코드 
	}
}
```

```java
public class UserDao{
	private ConnectionMaker connecitonMaker;
    //인터페이스를 통해 오브젝트에 접근 하므로 구체적인 프로젝트 정보를 알 필요가 없다 .
	
	public UserDao(){
		connectionMaker = new DConnectionMaker();
        // 근데 여기는 클래스 이름이 나오네 ;
	}
	
	public void add(User user) throws lassNotFoundException, SQLException{
		Connection c = connectionMaker.makeConnection();
		...
        //인터에이스에 정의 된 메소드를 사용하므로 클래스가 바뀐다고 해도 이름이 변경될 걱정 x 
	}
}
```

>db 커넥션을 제공하는 클래스에 대한 구체적인 정보는 모두 제거가 가능했지만
>
>**여전히 초기에 한번 어떤 클래스의 오브젝트를 사용할지 결정하는 생성자의 코드는 제거되지 않고 남아있다 !**

>new DConnectionMaker() 라는 코드는 짧고 간단하지만 그자체로 충분히 독립적인 관심사를 담고 있다 .
>
>바로 UserDao가 어떤 ConnectionMaker 구현클래스의 오브젝트를 이용하게 할지 결정하는 것이다 .
>
>간단히 말하자면 **UserDao와 UserDao가 사용할 ConnectionMaker의 특정구현 클래스 사이의 관계를 설정**해주는것에 대한 관심이다.
>
>**이 관심사를 담은 코드를 UserDao에서 분리하지 않으면 UserDao는 결코 독립적으로 확장 가능한 클래스가 될수 없다.**

***

#### 그럼 어떻게 분리하죠 ?

> UserDao를 사용하는 클라이언트가 적어도 하나는 존재할 것이다 .
>
> 왜 뜬금없이 클라이언트 오브젝트를( 서비스를 제공하는 쪽 클라이언트 )  를 끄집어 내냐면
>
> **UserDao의 클라이언트 오브젝트가 바로 제3의 관심사항인 UserDao와 ConnectionMaker의 구현 클래스의 관계를 결정해주는 기능을 분리해서 두기 적절한곳이기 때문이다 !**

***

> UserDao의 모든 코드는 ConnectionMaker 인터페이스 외에는 어떤 클래스와도 관계를 가져서는 안되게 해야한다 .



> UserDao 오브젝트가 DConnectionManager 오브젝트를 사용하게 하려면 두 클래스의 오브젝트 사이에 의존 관계라고 불리는 관계를 맺어주면된다 .

***

**의존 관계를 어떻게 맺어 줄까?**

> 클라이언트는 자기가 UserDao를 사용해야 할 입장이기 때문에 UserDao의 세부 전략이라고 볼 수 있는 ConnectionMaker의 구현 클래스를 선택하고, 선택한 클래스의 오브젝트를 생성해서 UserDao와 연결해줄 수 있다.

***

### **클라이언트에게 떠넘기자!**

> UserDaoTest라는 클래스를 하나 만들고 UserDao의 생성자를 수정해서 클라이언트가 미리 만들어둔 ConnectionMaker 의 오브젝트를 전달 받을수 있도록 파라미터를 하나 추가한다 .
>
> 클라이언트와 같은 제 3의 오브젝트가 UserDao 오브젝트가 사용할 ConnectionMaker 오브젝트를 전달해주도록 만든것이다 .

```java
public UserDao (ConnectionMaker connectionMaker){
	this.connectionMaker = connectionMaker;
}
```

> **DConnectionMaker을 생성하는 코드는 어디에 ?**
>
> DConnectionMaker를 생성하는 코드는 UserDao 와 특정 ConnectionMaker 구현 클래스의 오브젝트간 관계를 맺는 책임을 담당 하는 코드 였는데 그것을 UserDao의 클라이언트에 넘겨 버렸기 때문이다 .

**클라이언트 코드**

```java
public class UserDaoTser {
	public static void main(String [] args) throws ClassNotFoundException, SQLException{
		ConnectionMaker connectionMeker = new DConnectionMaker();
		//UserDao가 사용할 ConnectionMaker 구현 클래스를 결정하고 오브젝트를 생성한다 .
		UserDao dao = new UserDao(connectionMaker);
        ... // 생략
	}
}
```

> 서로 영향을 주지 않으면서도 필요에따라 자유롭게 확장 할수 있는 구조가 되었다 .

***

### 원칙과 패턴

- 개방 폐쇄 원칙 (클래스나 모듈은 확장에는 열려 있어야하고 변경에는 닫혀있어야한다.)
- 높은 응집도 (변화가 일어날 때 해당 모듈에서 변하는 부분이 크다.)

- 낮은 결합도 (결합도 - 하나의 오브젝트가 변경이 일어날 때에 관계를 맺고 있는 다른 오브젝트에게 변화를 요구하는 정도)

#### 전략패턴
>
> 개선한 구조를 디자인 패턴의 시작으로 보면 전략패턴에 해당한 다고  볼수 있다 .
>
> 전략 패턴은 **필요에 따라 변경이 필요한 알고리즘은 인터페이스를 통해 외부로 분리시키고 , 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용 할수 있게하는 디자인 패턴이다**

***

## 제어의 역전 (IOC)

> **지금 까지는 초난감 Dao를 깔끔한 구조로 리팩토링 하는 작업을 수행했다 .**
>
> 얼렁 뚱땅 넘어갔지만 현재 UserDaoTest가 어떤 ConnectionMaker 구현 클래스를 사용할지 떠맡고 있다 .
>
> UserDao 와 ConnectionMaker구현 클래스의 오브젝트를 만드는 것과 ,
>
> 그렇게 만들어진 두개의 오브젝트를 연결되어 사용 될수 있도록 관계를 맺어 주는것을 분리하자 !

### 팩토리

> 분리시킬 기능을 담당할 클래스를 하나 만들어 보자. 이 클래스의 역할은 객체의 생성 방법을 결정하고 그렇게 만들어진
>
> 오브젝트를 돌려주는 것인데 , **이런 일을 하는 오브젝트를 흔히 팩토리 라고 부른다.**
>
> **단지 오브젝트를 생성하는 쪽과 생성된 오브젝트를 사용하는 쪽의 역할과 책임을 깔끔하게 분리하려는 목적으로 사용하는 것이다.**

> 팩토리 역할을 맡을 클래스를 DaoFactory라고 하자 ,
>
> 그리고 UserDaoTest에 담겨있던 UserDao, ConnectionMaker관련 생성 작업을
>
> DaoFactory로 옮기고 ,  UserDaoTest서는 DaoFactory에 요청에서 미리 만들어진 UserDao 오브젝트를 가져와 사용하게 만든다 .

```java
public class DaoFactory{
	public UserDao userDao(){
		ConnectionMaker connectionMaker = new DConnectionMaker();
		UserDao userDao = new UserDao(connectionMaker);
        //userdao 타입의 오브젝트를 어떻게 만들고 준비 시킬지 결정한다 .
		return userDao;
	}
}
```

```java
public class UserDaoTest{
	public static void main(String []args) throws ClassNotFoundException,SQLException{
		UserDao dao = new DaoFactory().userDao();
	}
}
```

> userDao 메소드를 호출하면 DConnectionMaker를 사용해 db 커넥션을 가져오도록 이미 설정된 UserDao 오브젝트를 돌려준다.
>
> UserDaoTest 는 이제 UserDao가 어떻게 만들어지는지 초기화 되는지에 신경쓰지 않을 수 있다 .

***

### 제어권의 이전을 통한 제어관계 역전

> 이제 제어의 역전이라는 개념에 대해 알아보자. 제어의 역전이라는 건，
>
> 간단히 프로그램의 제어 흐름 구조가 뒤바뀌는 것이라고 설명할 수 있다.
>
> 일반적으로 프로그램의 흐름은 main() 메소드와 같이 프로그램이 시작되는 지점에서 다음에 시용할 오브젝트를 결정하고， 결정한 오브젝트를 생성하고，
>
> 만들어진 오브젝트에 있는 메소드를 호출하고 그 오브젝트 메소드 안에서 다음에 사용할 것을 결정하고 호출하는 식의 작업이 반복된다.
>
>  초기 UserDao를 보면 테스트용 main() 메소드는 UserDao 클래스의 오브젝트를 직접 생성하고， 만들어진 오브젝트의 메소드를 사용한다.
>
>  UserDao 또한 자신이 사용할  ConnectionMaker의 구현 클래스( 예를 들면 DConnectionMaker )를 자신이 결정하고， 그 오브젝트를 펼요한 시점에서 생성해두고 각 메소드에서 이를 사용한다.
>
> **제어의 역전이란 이런 제어 흐름의 개념을 꺼꾸로 뒤집는 것이다.**

***

> **제어의 역전에서는 오브젝트가 자신이 사용할 오브젝트를 스스로 선택하지 않는다. 당연히 생성하지도  않는다.**
>
> **또 자신도 어떻게 만들어지고 어디서 사용되는지를 알 수 없다.**
>
> **모든 제어 권한  을 자신이 아닌 다른 대상에게 위임하기 때문이다.**
>
> 프로그햄의 시작을 담당하는 main()  과 같은 엔트리 포인트를 제외하면
>
> **모든 오브젝트는 이렇게 위임받은 제어 권한을 갖는  특별한 오브젝트에 의해 결정되고 만들어진다.**

> 우리가만든 UserDao 와 DaoFactory 에도 제어의 역전이 적용되어 있다.

-  원래  ConnectionMaker 의 구현 클래스를 결정하고 오브젝트를 만드는 제어권은 UserDao에게  있었다. 그런데 지금은 DaoFactory 에게 있다.
-  UserDaoTest는 DaoFactory가 만들고 초기화해서 자신에게 사용하도록 공급해주는 ConnectionMaker를 사용할 수밖에 없다.
-  더욱이 UserDao와 ConnectionMaker  의 구현체를 생성히는 책임도 DaoFactory 가 맡고 있다.

> 자연스럽게 관심을 분리하고 책임을 나누고 유연하게 확장 가능한 구조로 만들기 위해 DaoFactory를 도입했던 과정이
>
>  바로 IoC를 적용하는 작업이었다고 볼 수있다.
>
>  우리는 대표적인 IoC 프레입워크라고 불리는 스프링 없이도 IoC 개념을 이미 적용한 셈이다.
>
> **IoC는 기본적으로 프레임워크만의 기술도 아니고 프레임워크가 꼭 필요한 개념도 아니다.**
>
> 단순하게 생각하면 디자인 패턴에서도 발견할 수 있는 것처럼 상당히 폭넓게 사용되는 **프로그래밍 모델**이다.

***

