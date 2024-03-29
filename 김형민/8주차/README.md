# [토비 스프링] 5장 서비스 추상화 2
## **5.2.4** 트랜잭션 서비스 추상화

### 기술과 환경에 종속되는 트랜잭션 경계설정 코드

### 트랜잭션 API의 의존관계 문제와 해결책

JDBC 트랜잭션 api와 JdbcTemplate과 동기화하는 api로 인해 JDBC DAO에 의존하게 된다. (다시 특정 데이터 액세스 기술에 종속되는 구조가 되어 버리고 말았다.)

### 스프링의 트랜잭션 서비스 추상화

스프링은 트랜잭션 기술의 공통점을 담은 **트랜잭션 추상화 기술을 제공**하는데 이를 사용하면 각 기술의 트랜잭션 API를 이용하지 않고도. **일관된 방식으로 트랜잭션을 제어하는 트랜잭션 경계설정 작업이 가능**해진다!

**스프링의 트랜잭션 추상화 계층**

![Untitled (3)](https://github.com/fkgnssla/spring-study/assets/92067099/df530c08-69da-4eb9-ba2a-9600fdec7c5e)


**스프링의 트랜잭션 추상화 API를 적용한 upgradeLevels()**

```java
pulic void upgradeLevels(){
	PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
	
	//트랜잭션 시작
	TransactionStatus status = transactionManager.getTransaction(new DefaultTransactonDefinition());
    
		List<User> users = userDao.getAll();
		for(User user:users){
			if(canUparadeLevel(user)){
				upgradeLevel(user);
			}
		}
		transactionManger.commit(status);
	}catch(RuntimeException e){
		transactionManager.rollback(status);
		throw e;
	}
}
```

- PlatformTransactionManager: 스프링이 제공하는 트랜잭션 경계설정을 위한 추상 인터페이스


# 5.3 서비스 추상화와 단일 책임 원칙

### 수직, 수평 계층구조와 의존관계

수평적인 구분이든,  수직적인 구분이든 **모두 결합도가 낮으며, 서로 영향을 주지 않고 자유롭게 확장될 수있는 구조**를 만들수 있다. 이때 **스프링의 DI가 중요한 역할**을 하고있다. **DI의 가치**는 이렇게 **관심, 책임, 성격이 다른 코드를 깔끔하게 분리**하는데 있다.

### 단일 책임 원칙

하나의 모듈은 한 가지 책임을 가져야 한다!

**장점**

- 어떤 변경이 필요할 때 수정 대상이 명확해진다.

<br>
<aside>
💡 스프링이 지원하는 DI 덕분에 쉽게 이를 지킬 수 있다!
</aside>
