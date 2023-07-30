# 들어가기
DBMS에서 동시성 제어(concurrency control)를 학습하기 위해 필요한 내용을 학습합니다. transaction schedules에 대해 학습합니다.

**concurrency control을 제대로 이해하기 위해서는 먼저 트랜잭션(Transaction)에 대한 이해가 사전에 있어야합니다.**

동시성 제어가 필요한 이유는?
다중 사용자 환경을 지원하는 DBMS의 경우, 반드시 지원해야 하는 기능이고,
다중 사용자 환경에서 둘 이상의 트랜잭션이 동시에 수행될 때, 일관성을 해치지 않도록 트랜잭션의 데이터를 접근 제어 해야합니다.


# schedules과 serializability

DB에서 concurrency control이 트랜잭션의 isolation을 보장하기 위해 serializability 개념을 사용을 합니다.


한 트랜잭션에서 다른 트랜잭션으로 이어지는 일련의 작업을  말합니다. 트랜잭션을 정렬하고 하나씩 실행하는 프로세스입니다.
쉽게 말해 read나, write와 같은 오퍼레이션들의 실행 순서를 의미합니다.

![](https://velog.velcdn.com/images/leekhy02/post/52991c4c-24a4-40bc-a3ab-7470418b0c94/image.png)

## 직렬 스케줄(Serial Schedule)
![](https://velog.velcdn.com/images/leekhy02/post/6ccfeb18-e5a5-4251-b94a-942019145ca1/image.png)

트랜잭션의 연산을 모두 순차적으로 실행하는 유형을 의미합니다. 즉, **하나의 트랜잭션이 실행되면 해당 트랜잭션이 완료**되어야 다른 트랜잭션이 실행될 수 있습니다.

**인터리빙 방식을 이용하지 않고트랜잭션이 직렬 스케줄에 따라 수행되면, 모든 트랜잭션이 완료될 때
까지 다른 트랜잭션의 방해를 받지 않고 독립적으로 수행된다. 그래서 직렬 스케줄에 따라 트랜잭션이 수행되고 나면 항상 모순이 없는 정확한 결과를 얻는다.**

**하지만** 직렬 스케쥴은 성능이 떨어집니다. 한 트렌젝션이 진행하는동안 실제 디스크에있는 i/o작업을 하는동안 cpu가 놀고있기 때문에(T1트렌젝션이 끝나야 T2트렌젝션이 진행하므로) 성능이 떨어질수 밖에 없습니다.

이상한 결과가 나올 가능성은 없지만 동시성이 없기때문에 좋은 성능을 낼수는 없고 현실적으로 사용할 수 없는 방식입니다.

## 비직렬 스케줄(Nonserial Schedule)
![](https://velog.velcdn.com/images/leekhy02/post/5eb1a046-1c49-456d-8813-6559cf43a1ec/image.png)

은 인터리빙 방식을 이용하여 트랜잭션을 병행해서 수행시키는 방법입니다.
 트랜잭션이 돌아가면서 연산들을 실행하기 때문에 하나의 트랜잭션
이 완료되기 전에 다른 트랜잭션의 연산이 실행될 수 있습니다.

인터리빙 방식을 사용하기 때문에 **동시성이** 높아져서 같은 시간에 더 많은 트렌젝션들을 처리할 수 있습니다. 하지만 동시에 **동시성 문제**가 생길수 있습니다.

트렌젝션들의 형태에 따라 이상한 결과가 발생할 수 습니다.(갱신 분실, 모순성, 연쇄 복귀 등의 문제가 발생할 수 있습니다.)


**결국 안정성과 성능을 위해 Nonserial Schedule을 이용해 이상한 결과가 나오지않는 Serial Schedule과 동일한 (equivalent) Nonserial Schedule을 실행 하자 라는 결론에 도달합니다.** -> conflict serializable한 nonserail schedule을 허용하자!

## 직렬 가능 스케줄(Serializable Schedule)

직렬 스케줄에 따라 수행한 것과 같이 정확한 결과를 생성하는 비직렬 스케줄을 말합니다. 모든 비직렬 스케줄이 직렬 가능 스케줄은 아닙니다.

### conflict operations
1. 서로 다른 트렌젝션 소속
2. 같은 데이터에 접근
3. 최소 하나는 write operation

하나의 schedule에 존재하는 두 operations에 대해서 사용할 수 있는 개념입니다.

한 schedule 내에 있는 두 operations가 아래 세 가지 조건을 만족하면 두 operations는 conflict하다 라고 말할 수 있습니다.

데이터 x1과 x2를 다루는 아래와 같은  스케쥴이 있다 가정하겠습니다.
숫자 1,2 는 각각 트렌젝션을 나타냅니다.

> (1)   r1(x1) w1(x1) r2(x2) w2(x2) c2 r1(x2) w1(x2) c1 

여기서 r2(x2) 오퍼레이션과 w1(x2) 이 오퍼레이션이 conflict 하다 말합니다.
즉 다른트렌젝션에 소속되어있고. x2라는 같은 데이터에 접근. 최소하나는 write 오퍼레이션 조건을 만족 시킵니다.

또한 w2(x2) 와 w1(x2) 또한 conflict하다 말할수 있습니다.(write-write conflict)

- **여기서 Conflict가 operation이 중요한 이유는 순서가 바뀌면 결과도 바뀝니다.**

### conflict equivalent
1. 두 schedule은 같은 transaction들을 가진다
2. 어떤(any) conflict operations의 순서도 양쪽 schedule이 모두 동일하다

위 두개의 조건을 만족 한다면 두 개의 schedule은 은 conflict equivalent하다 라고 말할 수 있습니다.

> (1)   r1(x1) w1(x1) r2(x2) w2(x2) c2 r1(x2) w1(x2) c1
(2)   r2(x2) w2(x2) c2 r1(x1) w1(x1) r1(x1) w1(x1) c1


1번 스케줄의 conflict operations들이  2번 스케줄(직렬 스케줄)의 conflict operations과 순서가 동일한 이런 경우를 conflict equivalent라고 말합니다.

### conflict serializable

schedule이 직렬 스케줄과 conflict equivalent하다면, 이 schedule은 conflict serializable하다'라고 말할 수 있습니다.

두 개의 schedule이 conflict equivalent 할 때, 한 schedule이 serial schedule이라면, 다른 하나의 schedule은 conflict serializable하다 라고 말할 수 있습니다.

# concurrency control
그 어떤 스케쥴도 serializable하게 만들어 주는 역할을 합니다. 이 때 뗄레야 뗄수없는 트렌젝션의 속성이 **Isolation** 입니다. 하지만 엄격한 isolation(완벽한 serializable을 추구하면) 은 성능을 떨 어뜨리기 때문에 개발자가 조절 할수 있는것이 **isolation level**입니다

> 참고자료 : https://www.youtube.com/watch?v=DwRN24nWbEc&list=PLcXyemr8ZeoREWGhhZi5FZs6cvymjIBVe&index=16
