Spring Transaction
=================

# 1. Transaction의 이해

> 더 이상 쪼갤 수 없는 최소 작업 단위
> commit으로 성공하거나 rollback으로 실패 이후 취소되어야 하는 최소 작업 단위 
> 이 부분에서는 프로그래밍적 트랜잭션보다 spring AOP를 활용하는 선언적 트랜잭션에서 다룰 예정이다.


# 2. Spring이 제공하는 Transaction 핵심기술

1. 트랜잭션 동기화
2. 트랜잭션 추상화
3. AOP를 이용한 트랜잭션 분리

## 2-1. 트랜잭션 동기화

> JDBC를 이용하는 개발자가 직접 여러 개의 작업을 하나의 트랜잭션으로 관리하려면 Connection 객체를 공유하는 등 상당히 불필요한 작업들이 많이 생길 것이다.

> Spring은 이러한 문제를 해결하고자 트랜잭션 동기화(Transaction Synchronization) 기술을 제공하고 있다. 트랜잭션 동기화는 트랜잭션을 시작하기 위한 Connection 객체를 특별한 저장소에 보관해두고 필요할 때 꺼내쓸 수 있도록 하는 기술이다.

> 트랜잭션 동기화 저장소는 작업 쓰레드마다 Connection 객체를 독립적으로 관리하기 때문에, 멀티쓰레드 환경에서도 충돌이 발생할 여지가 없다. 그래서 다음과 같이 트랜잭션 동기화를 적용하게 된다.


## 2-2. 트랜잭션 추상화

> Spring은 트랜잭션 기술의 공통점을 담은 트랜잭션 추상화 기술을 제공한다. 이를 이용함으로써 JDPC, JPA 등 각 기술에 종속적인 코드를 사용하지 않고도 일관적인 트랜잭션 처리가 가능하다.

## 2-3. AOP를 이용한 트랜잭션 분리

> AOP를 이용한 트랜잭션 분리는 선언적 트랜잭션으로 @Transactional 어노테이션을 선언하여 편리하게 트랜잭션을 적용할 수 있다.

> 이와 반대로 트랜잭션 매니저, 트랜잭션 템플릿 등을 사용해서 트랜잭션 관련 코드를 직접 작성하는 것을 프로그래밍 방식의 트랜잭션 관리라 한다. 

> 프로그래밍 방식의 트랜잭션 관리를 사용하게 되면 애플리케이션 코드와 트랜잭션 관리 코드가 강하게 결합하기 때문에 선언적 트랜잭션이 훨씬 간편하고 실용적이다.


# 3. 트랜잭션의 적용위치

> 스프링에서 우선순위는 항상 더 구체적이고 자세한 것이 높은 우선순위를 가진다 

* 우선순위

1. 클래스의 메서드
2. 클래스의 타입
3. 인터페이스의 메서드
4. 인터페이스의 타입

> 가급적 구체 클래스에 @Transactional을 사용하자

# 4. 트랜잭션 AOP 주의사항 - 프록시 내부 호출

> @Transactional을 사용하면 spring의 트랜잭션 AOP가 적용된다.

> 이 때 프록시 객체가 요청을 먼저 받아서 트랜잭션을 처리하고 실제 객체를 호출한다. 

> 만약 프록시를 거치지 않고 대상 객체를 직접 호출할 경우 AOP가 적용되지 않고 트랜잭션 역시 적용되지 않는다.


<img width="842" alt="image" src="https://github.com/all-day-and-night/spring-data-access/assets/94096054/a9ca11c4-e4cd-4f97-91b0-44936cb1514f">

```
public class CallService {
    
    public void external() {
        log.info("call external");
        printTxInfo();
        internal();
    }
    
    @Transactional
    public void internal() {
        log.info("call internal");
        printTxInfo();
    }
}
```

> external 메서드는 트랜잭션 적용을 하지 않았고 internal 메서드는 @Transactional을 적용하였다. 

> 이 때 external 메서드에서 internal 메서드를 호출하게 될 경우 트랜잭션은 적용되지 않는다.

> 이유는 external은 트랜잭션이 적용되지 않아 프록시 객체를 통해 호출되지 않았고 이 때, internal을 호출할 경우 프록시를 거치지 않고 호출했기 때문에 Transaction이 적용되지 않는다.

> 해결 방법은 internal 메서드를 프록시 객체를 거쳐 호출될 수 있도록 별도의 클래스를 사용하여 분리한다. 

```
 public class CallService {
          private final InternalService internalService;
          
          public void external() {
              log.info("call external");
              printTxInfo();
              internalService.internal();
          }
          private void printTxInfo() {
              boolean txActive =
              TransactionSynchronizationManager.isActualTransactionActive();
              log.info("tx active={}", txActive);
          } 
}

public class InternalService {
          @Transactional
          public void internal() {
              log.info("call internal");
              printTxInfo();
          }
          private void printTxInfo() {
              boolean txActive =
              TransactionSynchronizationManager.isActualTransactionActive();
              log.info("tx active={}", txActive);
          } 
}
```

> 내부 프록시 호출 문제는 눈으로 봤을 때 전혀 이상이 없어 보이기 때문에 해결하기 어려운 문제다. 

> 항상 AOP를 적용하여 사용한다는 점을 기억하고 @Transaction만 붙인다고 해결된다고 생각하지 말자!


# 5. 트랜잭션 옵션

1. value, transactionManager 
- 스프링 빈에 등록된 어떤 트랜잭션 매니저를 사용할 지 지정

```
 public class TxService {
    @Transactional("memberTxManager")
    public void member() {...}
    
    @Transactional("orderTxManager")
    public void order() {...}
}
```

2. rollbackFor
> 예외 발생 시 스프링 트랜잭션의 기본 정책은 다음과 같다.
- 언체크드 예외인 RuntimeException.class를 상속하는 예외 클래스, Error
- 체크드 예외인 Exception과 그 하위 예외들은 커밋

> 하지만 rollbackFor옵션을 사용하면 기본 정책에 추가로 어떤 예외가 발생할 때 롤백할 지 지정할 수 있으며 개발자가 customizing 가능
> 체크드 예외에 속한 exception 역시 롤백하도록 지정 가능
```
public class TxService{
    @Transactional(rollbackFor = MyException.class)
    public void member() {...}
}
```

> 추가로 rollbackForClassName도 있으며 예외 이름을 넣으면 된다.

3. noRollbackFor
 
> 위의 rollbackFor와 반대

4. propagation

> 전파의 경우 더 다양한 옵션과 적용사항이 있으므로 추후에 따로 설명하는 부분을 만드려고 한다. 

5. isolation
> 트랜잭션의 격리 수준을 지정
> 기본값은 default이며 대부분 데이터베이스에서 설정한 기준을 따른다.

- DEFAULT : 데이터베이스에서 설정한 격리 수준을 따른다. 
- READ_UNCOMMITTED : 커밋되지 않은 읽기 
- READ_COMMITTED : 커밋된 읽기
- REPEATABLE_READ : 반복 가능한 읽기
- SERIALIZABLE : 직렬화 가능

> 지금은 각 격리 수준에 대해 자세하게 다루지 않을 예정이며 추후에 따로 정리하기로 예정

6. timeout

> 트랜잭션 수행시간에 대한 타임아웃을 초 단위로 지정

7. label

> 일반적으로 사용하지 않음

> 트랜잭션 어노테이션에 있는 값을 직접 읽어서 어떤 동작을 하고 싶을 때 사용할 수 있다. 

8. readOnly

> default옵션은 false

> readOnly=true 옵션을 사용하면 읽기 전용 트랜잭션 생성

> 읽기에서 성능 최적화 

> CUD의 기능이 없다면 readOnlu=true 입력 

# 6. 트랜잭션 범위(@Transactional이 적용된 AOP) 밖으로 예외 던지기 


<img width="850" alt="image" src="https://github.com/all-day-and-night/spring-data-access/assets/94096054/3fed6bbd-2774-45f5-8431-3d8c873dc15b">

> 이 때는 spring에서 관리하는 트랜잭션 AOP의 원칙에 따라 트랜잭션을 커밋하거나 롤백한다.

- unchecked Exception

>  RuntimeException, Error와 하위 예외 발생시 롤백

- checked Exception

> Exception과 하위 예외 발생시 커밋 























