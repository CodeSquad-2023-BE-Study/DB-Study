
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/88c5abfd-d1ca-4b3b-8f16-3c9b6fb3e250)

# TCL(****Transaction Control Language****)의 명령어

> DCL(Data Control Language)에서 트랜젝션을 제어하는 명령인 commit, rollback을 분리하여 TCL이라고 합니다.

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/22ffdd37-bec3-45be-ab0b-7874040ce884)

## commit

- 변경된 사항을 물리적인 디스크에 저장합니다.
- Commit 명령 이후에는 Commit 이전 상태로 복구할 수 없습니다.
- 모든 사용자가 변경된 결과를 볼 수 있습니다.
- commit 이후, 변경된 튜플은 lock이 해제됩니다..
    - cf. Mysql의 경우 index를 단위로 lock됩니다.

### savepoint

- 데이터베이스 이용하는 응용 프로그램에서 명령어의 규모가 크거나 복잡할 경우 주로 사용합니다.
- commit 전 중간 중간 save point를 둡니다. 문제가 발생할 경우 중간 특정 위치부터 시작할 수 있습니다.
- 실제로 disk(보조 기억 장치)에 저장되기 때문에, 복구할 수 있습니다.
    - 트랜잭션의 처리 결과는 메모리에 있습니다.
- 직접 지정도 가능하다. (물론 보통 그렇게 하지는 않는다;)
    - 여러개를 사용할 수 있습니다.

## rollback
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/20539b77-316e-4eae-a89f-6f3f3c3150ba)

- 연산 작업 중 문제가 발생하면 이미 수행되었던 모든 작업을 취소하고 원래 상태로 복귀할 수 있습니다.
- rollback 명령 이후 메모리상에 Buffer에만 영향을 미치기 때문에 복구가 가능합니다.
- rollback 이후 해당 튜플은 Lock이 해제됩니다.

```sql
create table ();
commit;

drop table ();
rollback;  // drop 되기 전으로 돌아갑니다.

drop table ();
commit;
rollback; // drop 된 이후로 돌아갑니다. commit 된 연산이 DB에 반영되었습니다.

create table (생략);
insert into values(a,b);
savepoint a;
insert into values(c, d);
rollback to a; // c, d레코드는 출력되지 않습니다.
```

# Redo와 Undo는 회복 Recovery를 위한 것입니다.

> 둘 다 복구를 위한 개념입니다.
> 

rollback 은 트랜잭션 내 질의를 수행하면서 문제가 발생했을 경우 수행합니다.

그러나 시스템의 오류, 물리적인 문제의 경우는 시스템상의 문제 failure 이므로 트랜잭션이 다시 수행되어야 합니다. 이를 시스템 회복recovery라고 합니다.

회복으로 데이터의 신뢰성을 보장하며, 트랜잭션의 원자성과 영속성을 보장합니다.

- local failure : 하나의 트랜잭션이 실패했습니다.
- global failure : 모든 트랜잭션이 실패합니다.

실패가 발생하면 시스템 회복을 위해 undo 혹은 redo를 수행합니다.

## log

- 회복을 위해서는 작업 내용을 계속 기록해야 하는데, 이를 **log**라고 합니다.
- redo는 정상적으로 실행 되기까지의 과정을 기록
- undo는 작업 이전의 상태를 기록합니다.

## Redo : 다시 시작한다.

- 갱신이 완료된 데이터를 로그를 이용해 데이터베이스에 적용합니다.
- 주로 데이터베이스 내용이 손상되었을 때, Backup 으로 회복한 다음 Backup에 있는 데이터 이후는 로그에 갱신되어 있는 데이터를 데이터베이스에 적용하는데 사용합니다.

## Undo : 다시 되돌린다.

- 변경된 데이터를 취소하여 원래의 내용으로 복원시키는 연산입니다.
- 주로 트랜젝션 실행 중 실행이 실패하였을 경우, 원래의 내용으로 복원하는 경우에 사용됩니다.
- 사용자가 했던 작업을 반대로 진행합니다.

## 헷갈리니까 예시를 들자면!

- 이런 쿼리를 실행했다고 합시다.

```sql
UPDATE membere SET name='홍길동' WHERE member_id = '1';
```

- REDO log
    - 어떤 작업이 실행되었는가
    - e.g. 1번 member_id의 name을 ‘홍길동’으로 바꿈
- UNDO
    - 작업 전에 어떤 데이터였는가
    - e.b. 이 쿼리를 실행 전 원래 이름은 ‘김길동’이었음.

## 이게 왜 필요한건지?

- inno DB는 쿼리를 실행한 후 바로바로 디스크에 데이터를 변경하지 않습니다. 데이터 변경 작업은 순차적으로 데이터를 한꺼번에 변경하는 것이 아니고, 랜덤으로 디스크에 기록해야해서 굉장히 부하가 많이 드는 작업입니다.
- 그래서 이것을 줄이기 위해 대부분 DBMS는 변경된 데이터를 저장해두는 innoDB 버퍼 풀 과 같은 장치가 있습니다.
- 더불어 ACID 를 보장하기 위해 변경된 내용을 순차적으로 디스크에 기록하는 redo log를 가집니다.

# inno DB Undo table space

- inno DB에는 undo 테이블 스페이스가 있습니다.
- 클러스터형 인덱스 레코드에 대한 트랜잭션의 최신 변경사항 정보를 포함하여 undo 로그가 저장됩니다.
    - 지니한테 넘깁니다
- 실제 undo table space 이름과 경로를 볼 수 있습니다.
    
    ```sql
    SELECT TABLESPACE_NAME, FILE_NAME FROM INFORMATION_SCHEMA.FILES
      WHERE FILE_TYPE LIKE 'UNDO LOG';
    ```
    

innoDB는 이 undo table space를 통해 MVCC를 구현합니다.

# MVCC

- multiversion concurrency control

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/5c91318e-2492-4f8a-8877-0c13b186ce67)

- 어떤 데이터의 수정이 가해지는 시점마다 개별 버전을 모두 저장하고 이런 개별 버전을 연속체로서 정의함으로써 읽기와 쓰기간 경합(충돌)을 최소화합니다.
- 데이터를 읽을 때 특정 시점을 기준으로 가장 최근에 commit된 데이터를 읽습니다. (mysql 에서는 `Consistent read` 라고 표현합니다.)
- 데이터 변화 이력을 관리합니다.

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/5d880cbe-4479-4ee6-8f62-6037eeaac655)

- 경합하는 2개의 트랜젝션이 write-write 가 아니라면 잠기지 않습니다.
    - commit된 데이터만 읽기 때문입니다.

- 오라클의 롤백 세그먼트 동작원리와 비슷합니다.

> 데이터 블록에 있는 현재 버전인 A를 B로 업데이트하게 되면 이전 버전인 A를 별도의 롤백 블록에 저장하고, 데이터 블록에는 새로운 버전인 B로 변경하게 된다. 첫 번째 아키텍처인 MGA 아키텍처와는 달리 이전 버전을 별도의 저장 공간인 롤백 세그먼트에 저장하고 데이터 블록에는 오직 현재 버전만 존재하는 것이다.
> 
> 
> *출처 : 데이터넷(http://www.datanet.co.kr)*
> 

### 그럼 다른 트랜잭션이 쓰는 동안 어떤 값을 읽을까요?

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/3432f439-3d56-46b3-b352-85b2395c3a29)

- 트랜젝션 격리수준에 따라서 다릅니다.

> **복습삼아 격리 수준 정리**
> 
> - `Read uncommitted` 각 트랜잭션에서의 변경 내용이 commit 혹은 rollback 여부에 상관없이 값을 읽을 수 있음
> - `Read comitted` 트랜잭션에 의해 실행되는 각 쿼리는 쿼리가 시작되는 시점에 커밋된 데이터만 읽을 수 있음 (default Oracle, PostgreSQL)
> - `Repeatable read` 각 트랜잭션마다 부여된 트랜잭션 ID보다 작은 트랜잭션 번호에서 변경한 것만 읽게 된다. (default MySQL)
> - `Serializable` 트랜잭션들이 동시에 일어나지 않고 순차적으로 실행되는 것처럼 동작. 처리성능이 가장 낮음.

- 격리수준에 따라서 MVCC 사용 여부가 다릅니다.
    - MySQL
        - Repatable read, Read comitted 수준에서 MVCC 사용.
        - serializable은 shared lock
    - postgreSQL
        - Repatable read, Read comitted 수준에서 MVCC 방식을 사용
        - MVCC인 [SSI(serializable snapshot isolation)](https://medium.com/@fineroot1253/postgresql-mvcc-ssl-2f3ba4ce12da)으로 구현

## 단점

- 계속해서 변화 이력을 저장해야 하기 때문에, 추가 저장공간을 필요로 합니다.
- MySQL에서는 lost update가 발생할 수 있습니다. (자세한건 쉬운코드 영상 참고)
    - Locking read 를 수동으로 걸어주어야 합니다.
    
    ```sql
    select balance from account where id = 1 **for update**;
    ```
    
    - [JPA는 이렇게 비관적 락을 걸 수 있습니다.](https://www.baeldung.com/jpa-pessimistic-locking)
- 특정 조건에서 다르게 동작할 수 있기 때문에 각 RDBMS마다 파악이 필요합니다.
