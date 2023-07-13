# 락의 레벨

- MySQL에서 사용되는 락은 크게 **스토리지 엔진 레벨**과 **MySQL 엔진 레벨**로 나눌 수 있다.
    - MySQL 엔진은 MySQL 서버에서 스토리지 엔진을 제외한 나머지 부분
    - MySQL 레벨의 락은 모든 스토리지에 엔진에 영향을 미침
    - 스토리지 엔진의 락은 스토리지 엔진 간 상호 영향을 미치지 않는다.

# MySQL 엔진의 락

## 글로벌 락 (Global Lock)

- `FLUSH TABLES WITH READ LOCK`
- MySQL에서 제공하느 잠금 가운데 가장 범위가 크다.
    - 범위: MySQL 서버 전체에 존재하는 모든 테이블을 닫고 잠금을 건다. (다른 DB 포함)
    - 한 세션에서 글로벌 락을 획득하면 다른 세션에서 SELECT롤 제외한 대부분의 DDL, DML 문이 락에 걸린다.
- 여러 DB가 존재하는 MyISAM, MEMORY 테이블에 대해 주로 일관된 백업을 받아야 할 때 사용
(mysqldump)

### InnoDB에서 글로벌 락 → 백업 락

- `LOCK INSTANCE FOR BACKUP`(백업 락 시작), `UNLOCK INSTANCE`(락 해제)
- InnoDB 스토리지 엔진에서는 트랙잭션을 지원하기 때문에 모든 데이터 변경 작업을 멈출 필요는 없다.
    - InnoDB가 채택되면서 조금 더 가벼운 락의 필요성이 생겼다. 그래서 MySQL 8.0 버전부터는 BACKUP LOCK이 도입되었다.
- 백업 락에서 불가능 한 것
    - DB 및 테이블 등 모든 **객체 생성, 변경, 삭제 불가**
    - 사용자 관리 및 비밀번호 변경 불가
    - REPAIR TABLE, OPTIMIZE TABLE 명령 불가
- 백업 락에서 가능한 것
    - 일반적인 테이블의 **데이터 변경은 허용**됨
    - ??? 백업 도중 이미 백업된 부분에서 데이터가 추가되면 어떻게 ???
    - (Jinny의 의견) 백업 도중 DBA가 다른 작업을 하기 위해 사용하지 않을까?

## 테이블 락 (Table Lock)

- 테이블 단위로 설정되는 잠금
- 명시적으로 잠그는 방법
    - `LOCK TABLES table_name [READ | WRITE]` (테이블 락 적용)
    - `UNLOCK TABLES` (테이블 락 해제)
- 묵시적으로 잠그는 방법
    - 데이터를 변경하는 Query를 실행하면 자동으로 발생 (InnoDB 이전)
    - MySQL 서버가 데이터가 변경되는 테이블에 잠금을 설정하고 데이터를 변경한 후, 즉시 잠금을 해제하는 형태
- 하지만, InnoDB 테이블의 경우 스토리지 엔진 차원에서 레코드 기반의 잠금을 제공하기 때문에 단순 데이터 변경 Query로 인해 묵시적 테이블 락이 설정되지는 않는다.
    - 대부분의 데이터 변경(DML) Query에서는 무시되고, 스키마를 변경하는 쿼리(DDL)의 경우에만 영향을 미친다.
    - MySQL에서 InnoDB는 언제부터 적용 되었는지?
        
        *한편, 핀란드의 헤이키 튜리(Heikki Tuuri)는 1995년 이노베이스(Innobase Oy)를 설립해 트랜잭션 ACID를 지원하는 이노DB(InnoDB)를 발표하게 된다. 이노DB는 초기에 오픈소스가 아니었는데, 투자자를 찾다 실패한 후 2000년 MySQL과 협력을 하게 된다. 그리고 **2001년 MySQL 버전 4.0에서 처음으로 MySQL/이노DB 콤비가 출시**됐다.*
        
        **
        
        *2005년 이노DB는 오라클에 인수됐고, 2008년에는 MySQL이 썬(Sun)에 인수됐다가 2010년 썬이 오라클에 인수됐다. 오라클은 이노DB와 MySQL 모두를 인수하게 되면서 **2010년에 MySQL 5.5에서 이노DB를 MySQL의 기본 스토리지 엔진으로 채택**했다.*
        

## 네임드 락 (Named Lock)

- `GET_LOCK` 함수를 사용해 임의의 문자열에 대해 락을 설정함.
- 
- 테이블, 레코드 등의 DB 객체에 락을 거는 것이 아닌, 단순히 사용자가 지정한 문자열(String)에 대해 획득하고 반납하는 잠금.

```sql
-- "mylock"이라는 문자열에 락을 건다. 이미 잠금 중이면 2초간 대
-- 락 성공 시 1 반환
SELECT GET_LOCK('mylock', 2);

-- "mylock"이라는 문자열에 대해 락이 설정되어 있는지 확인
-- 걸려있으면 0, 걸려있지 않으면 1
SELECT IS_FREE_LOCK('mylock');

-- "mylock"이라는 문자열에 대해 획득했던 락을 반납
-- 성공 시 1, 실패 시 NULL
SELECT RELEASE_LOCK('mylock');

-- 모든 네임드 락 반납
SELECT RELEASE_ALL_LOCKS();
```

[NamedLock - [재고 시스템으로 알아보는 동시성 이슈]](https://sudal.site/namedLock/)

## 메타 데이터 락 (Metadata Lock)

- DB 객체(테이블, VIew …)의 이름이나 구조를 변경하는 경우 획득하는 잠금
- 묵시적 잠금만 가능
    - `RENAME TABLE table_name-a TO table_name_b` 같이 테이블의 이름을 변경하는 경우 자동으로 획득 된다.

### 적용 사례: 실시간으로 테이블을 교체해야 하는 상황

- 상황: 기존에 사용하던 rank 테이블을 rank_backup으로 백업하고, 새로 만들어진 rank_new 테이블을 서비스용으로 대체하고자 하는 경우

```sql
RENAME TABLE rank TO rank_backup, rank_new TO rank;
```

- 상황: Log를 기록하는 access_log 테이블이 있다. 어느 날 이 테이블의 구조를 변경해야 할 상황이 생겼다. DDL을 사용하기엔 시간이 너무 오래 걸릴 것 같다.(MySQL의 DDL은 단일 스레드로 동작하기 때문)
    - 이때는 새로운 구조의 테이블을 생성하고 먼저 최근의 데이터까지는 primary key인 id 값을 범위 별로 나눠서 여러 개의 스레드로 빠르게 복사한다.

```sql
-- 새로운 테이블 생성
CREATE TABLE access_log_new ( 
	id BIGINT NOT NULL AUTO_INCREMENT,
	client_ip INT NOT NULL,
	access_dttm TIMESTAMP,
	...
	PRIMARY KEY(id)
) ROW_FORMAT=COMPRESSED
KEY_BLOCK_SIZE=4;

-- 4개의 스레드를 이용해 id 범위별로 기존 테이블의 레코드를 신규 테이블로 복사
INSERT INTO access_log_new SELECT * FROM access_log WHERE id >= 0 AND id < 10000;
INSERT INTO access_log_new SELECT * FROM access_log WHERE id >= 10000 AND id < 20000;
INSERT INTO access_log_new SELECT * FROM access_log WHERE id >= 20000 AND id < 30000;
INSERT INTO access_log_new SELECT * FROM access_log WHERE id >= 30000 AND id < 40000;

-- 트랜잭션 실행
SET autocommit = 0;

-- 작업 대상인 2 개의 테이블에 테이블 쓰기락 획득
LOCK TABLES access_log WRITE, access_log_new WRITE;

-- 남은 데이터 복사
SELECT MAX(id) as @MAX_ID FROM acess_log;
INSERT INTO access_log_new SELECT * FROM access_log WHERE pk > @MAX_ID;
commit;

-- 새로 만든 테이블로 데이터 복사가 완료되면 RENAME 명령으로 새로운 테이블을 서비스에 적용한다
RENAME TABLE access_log TO access_log_old, access_log_new TO access_log;
UNLOCK TABLES;

-- 불필요한 테이블 삭제
DROP TABLE access_log_old;
```

- ??? 뭐야 이게????
    
    ### ◎ 4개의 스레드를 사용한다???
    
    ### ◎ `ROW_FORMAT=COMPRESSED` : 테이블에 저장되는 ROW의 형식을 지정
    
    - `COMPRESSED` 로 설정하면 데이터를 압축해서 저장할 수 있다.
    
    [MySQL :: MySQL 5.7 Reference Manual :: 14.11 InnoDB Row Formats](https://dev.mysql.com/doc/refman/5.7/en/innodb-row-format.html)
    
    ### ◎ `KEY_BLOCK_SIZE` ??? :  테이블 데이터 압축할 때 압축될 페이지의 목표 크기를 명시함
    
    - 2 이상의 수 2n으로만 설정할 수 있다.
    - 목표 크기 8KB로 가정:
        - 16KB의 데이터 페이지를 압축
        - 압축된 결과가 8KB 이하이면 그대로 디스크에 저장(압축 완료)
        - 압축된 결과가 8KB를 초과하면 원본 페이지를 스플릿 해서 2개의 페이지에 8KB씩 저장
    - 주의사항: 목표 크기 보다 작거나 같을 때까지 반복해서 페이지를 스필릿하므로, 목표 크기가 잘못되면 성능 ↓
    - INSERT만 되고 변경과 조회가 거의 없는 테이블(예:로그)의 경우 압축을 하는 것이 효율적
    - 데이터가 빈번히 조회 및 변경되는 테이블은 압축을 안 하는 것이 좋다.
    - 테이블 압축은 zlib을 이용해 압축을 실행하고, 압축 알고리즘은 많은 CPU 자원을 소모한다.
    
    [MySQL :: MySQL 5.7 Reference Manual :: 14.9.1.2 Creating Compressed Tables](https://dev.mysql.com/doc/refman/5.7/en/innodb-compression-usage.html)
    
    ### ◎ `@MAX_ID` ??? : 사용자 정의 변수
    
    ```sql
    -- SET으로 변수 설정
    SET @변수이름 = 대입값;   혹은  SET @변수이름 := 대입값;
    
    -- SELECT 문에서 바로 변수 설정
    SELECT @변수이름 := 대입값;
    
    -- SET으로 변수 설정 예시
    SELECT @start := 15, @finish := 20;
    
    -- SELECT 문에서 바로 변수 설정 예
    SELECT * FROM employee WHERE id BETWEEN @start AND @finish;
    ```
    

# InnoBD 스토리지 엔진 레벨의 락

- InnoBD에서는 레코드 기반의 락 방식을 제공한다. (MyISAM은 안됨 ㅋ)

[MySQL :: MySQL 8.0 Reference Manual :: 15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

## 레코드 락 (Record Lock, Record Only Lock)

- 레코드 자체만 잠그는 것.
- InnoDB 스토리지 엔진의 레코드 락은 **레코드 자체가 아니라 인덱스의 레코드를 잠근다**. ???
    - 인덱스가 없는 테이블이면 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.
    - 하…. 클러스터 인덱스는 뭔데?
        - 실제 물리적으로 정렬된 인덱스
            - 검색은 빠르지만, 수정/삭제 시 느림
        - 인덱스용 페이지를 따로 생성하지 않는다.
        - 테이블당 하나만 생성 가능(PK를 걸면 자동으로 PK 컬럼에 클러스터 인덱스가 만들어 진다)

## 갭 락 (Gap Lock)

- 레코드와 바로 인접한 레코드 사이의 간격만 잠그는 것
- 레코드와 레코드 사이의 간격에 새로운 레코드가 생성(INSERT)되는 것을 막는다.
- 넥스트 키 락의 일부로 자주 사용됨

## 넥스트 키 락 ( Next Key Lock)

- 레코드 락과 갭 락을 합쳐 놓은 형태의 락

## 자동 증가 락 (Auto Increment Lock)

- `AUTO_INCREMENT`를 위해 InnoDB 스토리지 엔진에서 내부적으로 Auto Increment Lock이라고 하는 테이블 수준의 락을 사용한다.
- INSERT, REPLACE 같은 새로운 레코드를 저장하는 Query에서만 필요
    - UPDATE, DELETE 등의 Query에서는 사용하지 않는다.
- 트랜잭션과 관계 없이 `AUTO_INCREMENT`값을 가져오는 순간만 락이 걸렸다가 즉시 해제된다.
- `AUTO_INCREMENT`을 가져오고 나면 INSERT 문이 실패해도 값은 다시 줄어들지 않는다.

# InnoDB 스토리지 엔진의 레코드 락은 **레코드 자체가 아니라 인덱스의 레코드를 잠근다**. ???

- MySQL에서 레코드를 변경할 때 조건에 맞는 레코드들을 인덱스로 검색하게 된다.
    - 락도 인덱스에 맞춰서 걸린다는 뜻인 듯
    - 테이블에 인덱스가 하나도 없으면 테이블 풀 스캔이 필요하다 → 락도 레코드 단위가 아닌 테이블 전체에 걸리게 된다

### 예시

- 상황:
    - `직원` 테이블에는 사원의 `성` 컬럼만 담긴 `ix_성`이라는 index가 있다.
    - `직원` 테이블에서 `성='김'`인 사원은 253명이 있다.
    - `직원` 테이블에서 `성='김'`이며 `이름='일수'`인 사원은 딱 1명만 있다.
    - `직원` 테이블에서 `성='김'`이며 `이름='일수'`인 사원의 `입사일`을 변경하려고 한다.

```sql
UPDATE 사원 
SET 입사일 = NOW() 
WHERE 성='김' AND 이름='일수'
```

- WHERE 조건절에 걸린 컬럼이 2개이면 컬럼 2개를 한번에 INDEX로 걸어줘야 한다. 현재 `성` 컬럼에만 Index가 걸려 있으므로, 이 1건의 UPDATE를 위해 `성='김'`인 253개의 레코드가 모두 잠기게 된다.
- index를 잘 걸어야 하는 이유 [+]됨
- index가 하나도 없다면 테이블의 모든 레코드에 락이 걸린다.

# 레코드 잠금 확인 및 해제

## 레코드 잠금 확인 방법 1 - `information_schema`, `performance_schema`

- MySQL 5.1 버전 이후 `information_schema` DB의 `INNODB_LOCK`, `INNODB_LOCK_WAIT` 라는 테이블을 통해 확인 가능.
- MySQL 8.0 버전 부터 `information_schema` DB의 정보들은 조금씩 제거되고 있다
    - 대신 `performance_schema`의 `data_locks`, `data_lock_waits` 테이블로 대체되고 있다.

[MySQL :: MySQL 5.7 Reference Manual :: 14.16.3 InnoDB INFORMATION_SCHEMA System Tables](https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-system-tables.html)

[MySQL :: MySQL 8.0 Reference Manual :: 27 MySQL Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema.html)

[MySQL :: MySQL 8.0 Reference Manual :: 27.12.13.1 The data_locks Table](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-locks-table.html)

[MySQL :: MySQL 8.0 Reference Manual :: 27.12.13.2 The data_lock_waits Table](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-lock-waits-table.html)

## 레코드 잠금 확인 방법 2 - `SHOW processlist`

- `SHOW processlist;` 명령어를 통해 현재 작업이 밀려있는지 확인 가능하다

```sql
SHOW processlist;
+----+-----------------+-----------------+-------+---------+--------+------------------------+------------------+
| Id | User            | Host            | db    | Command | Time   | State                  | Info             |
+----+-----------------+-----------------+-------+---------+--------+------------------------+------------------+
|  5 | event_scheduler | localhost       | NULL  | Daemon  | 263975 | Waiting on empty queue | NULL             |
|  9 | root            | localhost:14398 | kiosk | Sleep   |     20 |                        | NULL             |
| 10 | root            | localhost:14399 | kiosk | Sleep   |     20 |                        | NULL             |
| 11 | root            | localhost:2491  | NULL  | Query   |      0 | init                   | SHOW processlist |
+----+-----------------+-----------------+-------+---------+--------+------------------------+------------------+
4 rows in set (0.00 sec)
```

```sql
SHOW processlist;
+----+--------+----------+-----------------------+
| Id | Time   | State    | Info                  |
+----+--------+----------+-----------------------+
| 17 |    607 |          | NULL                  |
| 18 |     22 | updating | UPDATE 직원 SET ...   |
| 19 |     21 | updating | UPDATE 직원 SET ...   |
+----+--------+----------+-----------------------+
```

- ID 18, 19번 프로세스가 대기 상태이다. → 락이 걸린 것으로 유추할 수 있다.

## 레코드 잠금 확인 방법 3

- `performance_schema`의 `data_locks`, `data_lock_waits` 테이블을 조인해서 잠금 대기 순서를 볼 수 있다.

```sql
SELECT 
	r.trx_id AS waiting_trx_id,
	r.trx_mysql_thread_id AS waiting_thread,
	r.trx_query AS waiting_query,
	b.trx_id AS blocking_trx_id,
	b.trx_mysql_thread_id AS blocking_thread,
	b.trx_query AS blocking_query
FROM 
	performance_schema.data_lock_waits w
INNER JOIN
	information_schema.innodb_trx b
ON 
	b.trx_id = w.blocking_engine_transaction_id
INNER JOIN
	information_schema.innodb_trx r
ON
	r.trx_id = w.requesting_engine_transaction_id;

+---------+---------+---------------+----------+----------+----------------+
| waiting | waiting | waiting_query | blocking | blocking | blocking_query |
| _trx_id | _thread | waiting_query | _trx_id  | _thread  | blocking_query |
+---------+---------+---------------+----------+----------+----------------+
|   11990 |      19 | UPDATE emp .. |    11989 |       18 | UPDATE emp ..  |
|   11990 |      19 | UPDATE emp .. |    11984 |       17 | NULL           |
|   11990 |      18 | UPDATE emp .. |    11984 |       17 | NULL           |
+---------+---------+---------------+----------+----------+----------------+
```

- 2번째 컬럼 `waiting_thread`,를 보면 대기중인 스레드는 18, 19번 임을 알 수 있다.
- 5번째 컬럼 `blocking_thread` 를 보면 18번 스레드는 17번 스레드를 기다리고 있고 19번 스레드는 17, 18번 스레드를 기다리고 있음을 알 수 있다.

## 레코드 잠금 확인 방법 4

- 스레드가 어떤 잠금을 가지고 있는지 확인하려면 `performance_schema`의 `data_locks` 테이블이 가진 컬럼을 살펴보면 된다.
    - `LOCK_TYPE` 컬럼에선 Lock이 table에 걸렸는지, record에 걸렸는지 확인 가능.
    - `LOCK_MODE` 컬럼에서 Lock 방식 확인 가능

```sql
SELECT * FROM performance_schema.data_locks\G
```

- `\G` 이거 뭐임?
    - 쿼리 결과를 가로로 출력해 줌
    - CMD에서 만 되는 듯 Workbench에서는 안됨
    
    ```sql
    mysql> SHOW processlist\G
    *************************** 1. row ***************************
         Id: 5
       User: event_scheduler
       Host: localhost
         db: NULL
    Command: Daemon
       Time: 266105
      State: Waiting on empty queue
       Info: NULL
    *************************** 2. row ***************************
         Id: 9
       User: root
       Host: localhost:14398
         db: kiosk
    Command: Sleep
       Time: 280
      State:
       Info: NULL
    *************************** 3. row ***************************
         Id: 10
       User: root
       Host: localhost:14399
         db: kiosk
    Command: Sleep
       Time: 280
      State:
       Info: NULL
    *************************** 4. row ***************************
         Id: 11
       User: root
       Host: localhost:2491
         db: NULL
    Command: Query
       Time: 0
      State: init
       Info: SHOW processlist
    4 rows in set (0.00 sec)
    ```
    

## 레코드 잠금 해제하려면? KILL processlist_id

- `KILL` 명령어로 락을 걸고 있는 프로세스를 강제 중지 시키면 락이 해제된다.

```sql
KILL 17;
```

[MySQL :: MySQL 8.0 Reference Manual :: 13.7.8.4 KILL Statement](https://dev.mysql.com/doc/refman/8.0/en/kill.html)
