# Spring의 트랜잭션 - @Tranctional의 동작 원리

트랜잭션을 유지하려면 트랜잭션의 시작부터 끝까지 같은 데이터베이스 커넥션을 유지해아한다. 

그러면 같은 커넥션을 동기화하기 위해서 어떻게 해야할까?
제일 간단한 방법은 파라미터로 커넥션을 전달하는 방법이 있겠다.

```java
public void bizLogic() throws SQLException {
      Connection con = dataSource.getConnection();
      try {
          con.setAutoCommit(false); //트랜잭션 시작 
          //비즈니스 로직 시작
          bizLogic1(con); //비즈니스 로직1
          bizLogic2(con); //비즈니스 로직2
          // ...
          con.commit(); // 성공시 커밋
      } catch (Exception e) { 
          con.rollback(); //실패시 롤백
          throw new IllegalStateException(e);
      } finally {
          release(con);
      }
}
```

하지만 파라미터로 커넥션을 전달하는 방법은 코드가 지저분해지는 것은 물론이고, 
커넥션을 넘기는 메서드와 넘기지 않는 메서드를 중복해서 만들어야 하는 등 여러가지 단점들이 많다.

Spring은 트랜잭션을 어떻게 처리를 하고 있고 각기 다른 Data Access 기술을 사용했을 때 어떻게 처리하고 있을까?

## 1. 트랜잭션 동기화

### 트랜잭션 동기화 매니저

트랜잭션을 유지하려면 트랜잭션의 시작부터 끝까지 같은 데이터베이스 커넥션을 유지해아한다고 했다.

그래서 Spring은 **트랜잭션 동기화 매니저**를 제공하고 있다.
트랜잭션 동기화 매니저는 쓰레드 로컬( ThreadLocal )을 사용해서 커넥션을 동기화해준다. 

트랜잭션 매니저는 내부에서 이 트랜잭션 동기화 매니저를 사용한다.

> 💡 참고:
>
> ThreadLocal 을 사용하면 각각의 쓰레드마다 별도의 저장소가 부여된다. 
> 따라서 해당 쓰레드만 해당 데이터에 접근할 수 있다.

트랜잭션 동기화 매니저는 쓰레드 로컬을 사용하기 때문에 멀티쓰레드 상황에 안전하게 커넥션을 동기화 할 수 있다.
따라서 커넥션이 필요하면 트랜잭션 동기화 매니저를 통해 커넥션을 획득하면 된다.

**TransactionSynchronizationManager**

```java
public abstract class TransactionSynchronizationManager {

	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");

	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");

	private static final ThreadLocal<String> currentTransactionName =
			new NamedThreadLocal<>("Current transaction name");

	// ...
}
```

### 트랜잭션 동기화 매니저의 동작 방식

![](https://i.imgur.com/IQUUepw.png)

1. 서비스 계층에서 트랜잭션을 시작한다. 트랜잭션을 시작하려면 커넥션이 필요하다.

2. 트랜잭션 매니저는 내부에서 데이터소스를 통해 커넥션을 만들고 트랜잭션을 시작한다.

3. 트랜잭션 매니저는 트랜잭션이 시작된 커넥션을 트랜잭션 동기화 매니저에 보관한다.

4. 리포지토리는 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내서 사용한다. 따라서 파라미터로 커넥션을 전달하지 않아도 된다.

5. 트랜잭션이 종료되면 트랜잭션 매니저는 트랜잭션 동기화 매니저에 보관된 커넥션을 통해 트랜잭션을 종료하고, 커넥션도 닫는다. 

### 중간 정리

- 트랜잭션 동기화 매니저를 통해 더이상 파라미터로 커넥션을 넘기지 않아도 된다.
- 남은 문제는?  각기 다른 Data Access 기술을 사용했을 때이다.

## 2. 트랜잭션 기능 추상화

Spring의 Data Access의 구현 기술마다 트랜잭션을 사용하는 방법이 다르다.

예를 들면 JDBC는 다음과 같이 트랜잭션을 사용하며,

```java
Connection con = dataSource.getConnection();
con.setAutoCommit(false) 
```

JPA는 다음과 같이 트랜잭션을 사용한다.

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabook");
EntityManager em = emf.createEntityManager(); 
EntityTransaction tx = em.getTransaction()
tx.begin();
```

기술을 변경하게 되면 기반 코드를 모두 바꿔야 하는 문제가 있다. 

Spring은 항상 그러하듯이 추상화를 통해 해결했다.
간단하게 생각해보자. 기술에 상관없이 필요한 기능은 사실 다음과 같다.

- 트랜잭션 시작
- 커밋
- 롤백

그러면 인터페이스를 만들 수 있다. Spring은 `PlatformTransactionManager`을 통해 트랜잭션 기능을 추상화했다.


**PlatformTransactionManager**

```java
public interface PlatformTransactionManager extends TransactionManager {

	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;
}

```

그동안 우리는 `PlatformTransactionManagerger` 덕분에 Data Access 기술에 상관없이 편리하게 트랜잭션을 사용할 수 있었던 것이다.


### 중간 정리

트랜잭션 동기화 매니저를 통해 더이상 파라미터로 커넥션을 넘기지 않아도 되고,
트랜잭션 매니저를 통해 특정 Data Access 기술에 의존하지 않아도 된다.

하지만 트랜잭션을 사용하기 위해서 서비스 계층에서 직접 트랜잭션 매니저를 호출해야 하고,
그렇게 되면 순수한 비즈니스 로직만 남길 수 없는 문제가 있다.


## 3. 트랜잭션 AOP

스프링은 AOP를 통해 해당 문제를 해결했다.
프록시를 사용해 "트랜잭션을 처리하는 객체"와 "비즈니스 로직을 처리하는 서비스 객체"를 분리한 것이다

간단히 말하면 `@Transactional` 을 사용하면 스프링이 AOP를 사용해서 트랜잭션을 편리하게 처리해준다...고 이해하면 된다.


### 프록시 도입 전

- 프록시를 도입하기 전에는 서비스의 로직에서 트랜잭션을 직접 시작했다.

![](https://i.imgur.com/J3rjvIN.png)

**서비스 계층 예시 코드**

```java

//트랜잭션 시작
TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

try {
	//비즈니스 로직
	bizLogic(); 
    transactionManager.commit(status);   // 성공시 커밋
} catch (Exception e) { 
    transactionManager.rollback(status); // 실패 시 롤백
    throw new IllegalStateException(e);
}

```


### 프록시 도입 후

- 프록시를 사용해 "트랜잭션을 처리하는 객체"와 "비즈니스 로직을 처리하는 서비스 객체"를 분리했다.


![](https://i.imgur.com/NUxbaZr.png)

**트랜잭션 프록시 예시 코드**

```java
public class TransactionProxy {
    
    private MemberService target;
    
    public void logic() { 
        //트랜잭션 시작
      	TransactionStatus status = transactionManager.getTransaction(..);
        
      	try {
			//실제 대상 호출 
            target.logic();
			transactionManager.commit(status);   //성공시 커밋 
        } catch (Exception e) {
			transactionManager.rollback(status); //실패시 롤백
            throw new IllegalStateException(e);
      	}
	} 
}
```


**서비스 코드 예시 코드**

```java
public class Service {
    public void logic() {
        //트랜잭션 관련 코드가 제거되고 순수 비즈니스 로직만 남음
     	bizLogic(fromId, toId, money);
  	}
}
```


### 최종 정리

![](https://i.imgur.com/cQwQfkz.png)



## 4. 선언적 트랜잭션

스프링의 트랜잭션 AOP를 사용하는 방법은 2가지가 있다.

- tx 네임스페이스
- 어노테이션



> 💡 참고:
>
> **선언적 트랜잭션 관리** **vs** **프로그래밍 방식 트랜잭션 관리**
>
> - 프로그래밍 방식 트랜잭션 관리:
>     트랜잭션 매니저 또는 트랜잭션 템플릿 등을 사용해서 트랜잭션 관련 코드를 직접 작성하는 것을 프로그래밍 방식의 트랜잭션 관리라 한다.
> - 선언적 트랜잭션 관리가 프로그래밍 방식에 비해서 훨씬 간편하고 실용적이기 때문에 실무에서는 대부분 선언적 트랜잭션 관리를 사용한다.
> - 프로그래밍 방식의 트랜잭션 관리는 스프링 컨테이너나 스프링 AOP 기술 없이 간단히 사용할 수 있지만 실무에서는 대부분 스프링 컨테이너와 스프링 AOP를 사용하기 때문에 거의 사용되지 않는다.
> -  프로그래밍 방식 트랜잭션 관리는 테스트 시에 가끔 사용될 때는 있다.



### tx 네임스페이스

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <!-- JPA 설정 -->
    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="packagesToScan" value="com.example.model"/>
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter"/>
        </property>
    </bean>

    <!-- 트랜잭션 관리 설정 -->
    <tx:annotation-driven/>

    <!-- 서비스 빈 -->
    <bean id="userService" class="com.example.UserService"/>

</beans>

```


### 어노테이션

- 트랜잭션 처리가 필요한 곳에 `@Transactional` 애노테이션만 붙여주면 된다.
    그러면 스프링의 트랜잭션 AOP는 이 애노테이션을 인식해서 트랜잭션 프록시를 적용해준다.

- 나머지는 스프링 트랜잭션 AOP가 자동으로 처리해준다.


> 💡 참고:
>
> 스프링 부트가 등장하기 이전에는 데이터소스와 트랜잭션 매니저를 개발자가 직접 스프링 빈으로 등록해서 사용했다. 
> 그런데 스프링 부트로 개발을 시작한 개발자라면 데이터소스나 트랜잭션 매니저를 직접 등록한 적이 없을 것이다.
>
> 스프링 부트는 데이터소스( DataSource )와 트랜잭션 매니저를 스프링 빈에 자동으로 등록한다.
> 참고로 개발자가 직접 빈으로 등록하면 스프링 부트는 자동으로 등록하지 않는다.
>
> **데이터소스 자동 등록**
> 이때 스프링 부트는 application.properties 에 있는 속성을 사용해서 DataSource 를 생성하고 스프링 빈에 등록한다.
>
> **트랜잭션 매니저 자동 등록**
> 스프링 부트는 적절한 트랜잭션 매니저( PlatformTransactionManager )를 자동으로 스프링 빈에 등록한다.
> 어떤 트랜잭션 매니저를 선택할지는 현재 등록된 라이브러리를 보고 판단하는데, JDBC를 기술을 사용하면 `DataSourceTransactionManager` 를, JPA를 사용하면 `JpaTransactionManager` 를 빈으로 등록한다. 
> 둘다 사용하는 경우 `JpaTransactionManager` 를 등록한다. 참고로 `JpaTransactionManager` 는 `DataSourceTransactionManager` 가 제공하는 기능도 대부분 지원한다.


---


# 정리 못한 것

JDBC - DataSourceUtils Connection
JPA - EntityManagerFactory EntityManager
Hibernate - SessionFactoryUtils Session

추상화를 통해 더이상 특정 데이터 접근 기술에 대해 의존적이지 않게 됨!



### Controller에 @Transactional 달면 안될까?

- Spring의 AOP 개념, 결론적으로 Controller에 달면 작동하지 않음



트랜잭션 처리 로직을 트랜잭션 프록시가 가져간다.

그리고 트랜잭션을 시작한 후에 실제 서비스를 대신 호출한다.



@Transactional 애노테이션은 메서드에 붙여도 되고, 클래스에 붙여도 된다. 클래스에 붙이면 외부에서

호출 가능한 public 메서드가 AOP 적용 대상이 된다.



## Spring 트랜잭션 속성
- propagation: 트랜잭션을 시작하거나 기존 트랜잭션에 참여하는 방법을 결정하는 속성
	- required: 기본 속성. 트랜잭션이 있으면 참여하고, 없으면 새로 시작
	- requires-new?: 항상 새로운 트랜잭션을 시작, 진행중인 트랜잭션이 있다면 트랜잭션 잠시 보류
	- supports: 이미 트랜잭션이 있으면 참여, 그렇지 않으면 트랜잭션 없이 진행
	- nested: 
		- 이미 진행중인 트랜잭션이 있으면 중첩 트랜잭션 시작
		- 부모 트랜잭션 커밋, 롤백엔 영향을 받음
		- 자신의 커밋, 롤백은 부모 트랜잭션에 영향 못줌
	- never: 트랜잭션을 사용하지 않게 한다. 트랜잭션이 존재하면 예외가 발생한다.
	- not_supported: 트랜잭션을 사용하지 않게 한다. 트랜잭션이 있다면 보류한다.

- isolation: 여러 트랜잭션이 진행될 때에 트랜잭션의 작업 결과를 타 트랜잭션에게 어떻게 노출할지 결정
	- default: 
		- 사용하는 데이터 접근 기술, DB 드라이버의 기본 설정
		- Oracle은 READ_COMMITED, MySQL은 REPEATABLE_READ가 기본 격리 수준
- read-only: 트랜잭션 내에서 데이터를 조작하려는 시도를 막음, 데이터 접근 기술, 사용 DB에 따라 적용 차이가 있음
- timeout: 트랜잭션을 수행하는 제한 시간을 설정할 수 있음, 기본 옵션에는 제한시간이 없음
- rollback-for: 기본적으로 RuntimeException 시 롤백, 체크 예외지만 롤백 대상으로 삼고 싶다면 사용
- no-rollback-for: 롤백 대상인 RuntimeException을 커밋 대상으로 지정

---

# Reference

- 인프런 강의: 김영한님의 "스프링 DB 1편 - 데이터 접근 핵심 원리: 섹션4"
- [[10분 테코톡] 후니의 스프링 트랜잭션](https://www.youtube.com/watch?v=cc4M-GS9DoY)
- [[10분 테코톡] 🐤 샐리의 트랜잭션](https://www.youtube.com/watch?v=aX9c7z9l_u8&t=79s)
