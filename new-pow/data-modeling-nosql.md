# 서론

- 채팅 로그를 Redis에 저장하는 슬기로운 방법을 찾자

```java
@Getter
@ToString
@RedisHash("chat-bubble")
public class ChatBubble implements Serializable {

    @Id @JsonIgnore
    private final String id;
    @Indexed
    private final String roomId;
		private final String from;
    private final String contents;
    private final String createdAt;

		// getter and others....
}

// repository
@Repository
public interface ChatBubbleRepository extends PagingAndSortingRepository<ChatBubble, String> {
}
```

- 이렇게 해서 `save()` 를 날리면 벌어지는 일
    - 자료구조를 엄청 엄청 만들어집니다 ㅜㅜ

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/c7a48ff0-54ae-4b5a-97cb-bf4b35d59f38)

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/0894fb4e-7430-42ce-9ad4-baff1b499d6a)

# 데이터 모델링

- 개념적 모델링 conceptual modeling
    - 중요 데이터를 추출하여 개념 세계로 옮기는 작업
- 논리적 모델링 logical modeling
    - 개념 세계의 데이터를 데이터베이스에 저장할 구조를 결정하고 이 구조로 표현하는 작업
- 위 2개를 합쳐 데이터 모델링이라고 한다.

## 데이터 모델

- 데이터 모델링의 결과물을 표현하는 도구
- 일반적으로 데이터 구조 data structure, 연산 operation, 제약조건 constraint로 구성된다.

# RDB 의 데이터 모델링
## 요구사항 파악
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/7ef6181f-9c07-453c-829c-bbfe889a43c2)

## 개념적 데이터 모델 설계

- E-R 다이어그램 작성
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/f1decfed-7338-437a-8ca1-f692b5aaf9be)

## 논리적 데이터 모델 설계

- 비즈니스 정보의 논리적 구조와 규칙을 명확하게 표현하는 과정
- 개념적으로 구체화한 모델을 RDB에 맞게 구축한다.

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/1a5a1bd6-d324-451a-90c3-988ca1456eb1)

## 물리적 데이터 모델 설계

1. 물리적으로 컴퓨터에 어떻게 저장도리 것인가에 대해 고려하여 데이터베이스를 설계하고 구현
2. 특히 RDB는 인덱스 조회, 테이블 조인에 의존하여 서로다른 엔티티를 연결한다.

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/daa6dae1-ccbe-41d6-837f-23c105b7a881)

# NoSQL 데이터 모델링

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/43b4cb35-8140-4785-b58c-a80ef4b09d7e)

- 먼저 근본적인 사상이 다르다.
- 객체 모델 지향에서 쿼리 결과 지향 모델링
    - RDBMS가 `도메인 모델 → 개체간의 관계 식별 -> 테이블 → 쿼리 구현` 순으로 진행한다면
    - NoSQL은 **`도메인 모델 → 쿼리 결과 → 테이블`** 순으로 디자인해야한다.
    - **NoSQL의 경우 복잡한 쿼리 기능이 없기 때문에, 도메인 모델에서 어떤 쿼리 결과가 필요한지를 정의한 후 결과를 얻기 위해 저장 모델을 역순으로 디자인한다.**
- 정규화 와 비정규화
    - RDBMS 모델링에서는 데이터 일관성과 도메인 모델과 일치를 위해 데이터 모델을 정규화한다.
        - 즉, 같은 데이터가 두개 이상의 테이블에 중복되게 저장되는 것을 지양한다.
    - NoSQL은 반대 접근 방법이 필요하다.
        - 쿼리 효율을 위해 데이터를 정규화하지 않고, 의도적으로 중복된 데이터를 저장한다.
- 애플리케이션의 필요한 쿼리와 성능을 정의하고, 요구사항에 맞게 데이터 모델링을 한다.

## 주의할 점

- 관계형 데이터베이스 모델링 보다 더 깊은 데이터 구조 및 접근 알고리즘에 대한 이해가 필요
    - NoSQL 쿼리가 실제 몇개의 물리 노드에 걸쳐서 수행되는지에 대한 이해가 있어야 제대로된 쿼리 디자인이 가능
    - NoSQL 디자인은 DB 와 어플리케이션 뿐만 아니라 인프라(네트워크,디스크)에 대한 디자인을 함께 해야 함
- 대부분 NoSQL DB는 인증이나 인가 체계가 없어서 보안에 매우 취약
    - 방화벽이나 Reverse Proxy 등 별도의 보안 체계가 필요하다.
 
## NoSQL의 데이터 모델 종류

> 모델링 전에 어떻게 저장되는지부터 파악해보자.
> 

### 1. Key/Value Store

- 대부분의 NoSQL은 Key/Value 개념을 지원한다.
- Unique Key에 하나의 Value를 가지는 형태.
- `put(Key, Value)`, `value := get(key)` 형태 API를 사용한다.

```sql
-- reids DB 에서 데이터를 조회하는 쿼리
get [key]
```

- Key-Value DB 의 예시
    - Redis
    - Riak
    - Oracle Berkely
    - AWS DynamoDB
- 주로 사용되는 서비스 애플리케이션
    - 성능 향상을 위해 관계형 데이터베이스에서 데이터 캐싱
    - 장바구니 같은 웹 애플리케이션에서 일시적인 속성 추적
    - 모바일 애플리케이션용 사용자 데이터 정보와 구성 정보 저장
    - 이미지나 오디오 파일 같은 대용량 객체 저장

### 2. Column Family

- Key 안에 (column:value) 조합으로 된 여러개의 필드를 가진다.
- 테이블간 조인을 지원하지 않고, 매우 넓은 단일 Row 에 넣어서 보관한다.

```json
UserProfile = {
    Cassandra = {emailAddress:"cassandra@apache.org", age:20},
    TerryCho = {emailAddress:"terry.cho@apache.org", gender:"male"},
    Cath = {emailAddress:"cath@apache.org", age:20, gender:"female", address:"Seoul"},
}

-- 출처 : https://en.wikipedia.org/wiki/Standard_column_family
```

- super column family
    - column family 가 모이면 super가 됨

```json
UserList={ 
   Cath:{
     username:{firstname:”Cath”,lastname:”Yoon”}
     address:{city:”Seoul”,postcode:”1234”}
   }
   Terry:{
     username:{firstname:”Terry”,lastname:”Cho”}
     account:{bank:”hana”,accounted:”1234”}
   }
 }

-- 출처 : https://en.wikipedia.org/wiki/Super_column_family
```

- 예시
    - Hbase
    - Cassandra
    - GCP BigTable
    - Microsoft Azure Cosmos DB
- 다음 애플리케이션에서 활용을 고려하면 좋다.
    - 데이터베이스에 쓰기 작업이 많은 애플리케이션
    - 지리적으로 여러 데이터 센터에 분산되어 있는 애플리케이션
    - 복제본 데이터가 단기적으로 불일치하더라도 큰 문제가 없는 애플리케이션
    - 동적 필드를 처리하는 애플리케이션
    - 수백만 테라바이트 정도의 대용량 데이터를 처리할 수 있는 애플리케이션

### 3. Ordered Key/Value Store, OKVS

- 데이터가 내부적으로 Key 순서로 Sorting 되어 저장된다.
    - 탐색시 전체 스캔을 수행할 필요가 없다.
- 대표적으로 Hbase, Cassandra

### 4. Document Key/Value Store

- Key/Value Store의 확장된 형태이다.
- 저장되는 Value의 데이터 타입으로 `Document` 라는 구조화된 데이터타입을 사용한다.
    - 구조화된 데이터타입 : JSON, XML, YAML 등…
- 복잡한 계층 구조를 표현할 수 있다.
- 제품에 따라 Soring Join, grouping을 지원하기도 한다.
- Documnet DB 예시
    - MongoDB
    - CouchDB
    - Couchbase
- 다음 어플리케이션에서 활용을 고려하면 좋다
    - 대용량 데이터를 읽고 쓰는 웹 사이트용 백엔드 지원
    - 제품처럼 다양한 속성이 있는 데이터 관리
    - 다양한 유형의 메타데이터 추적
    - JSON 데이터 구조를 사용하는 애플리케이션
    - 비정규화된 중첩 구조의 데이터를 사용하는 애플리케이션

# NoSQL 모델링 기본개념

## 비정규화 Denormalization

- 데이터 중복 허용
- 쿼리 프로세싱을 간단하게 하거나 최적화하기 위해서 또는, 사용자의 데이터를 특정한 데이터 모델에 맞추기 위해 같은 데이터를 여러 document나 테이블에 복제하여 중복을 허용한다.
- 이로인한 트레이드 오프가 발생한다.
    - Query data volume 혹은 I/O per query ↔ Total data volume
        - 쿼리 수행을 위한 I/O가 줄어 전체 성능을 향상시킬 수 있지만 중복이 허용되므로 전체 데이터 사이즈는 증가한다.
    - Processing complexity ↔ total data volume
        - 시스템과 데이터 사이즈가 큰 분산 환경에서 데이터 정규화 했을 때, 쿼리 수행 복잡도는 더 높아짐
        - 데이터 모델링 시점에서 데이터 정규화(중복 제거)를 하거나 쿼리 실행 시점에서 데이터 간의 연속적인 JOIN은 쿼리 실행 프로세스의 복잡도를 증가
        - 데이터들을 한 곳에 쿼리친화적인 구조로 쌓을 수 있기 때문에 전체적인 쿼리 프로세싱을 단순화하고 쿼리 실행 시간도 단축
        - 다른 Document나 Table에 중복되어 저장되기 때문에 전체 데이터 사이즈는 필연적으로 증가
- NoSQL에서는 처음부터 Join 되어있는 테이블 추가하여 조회

## Aggregates

- 유연한 스키마
    - 복잡하고 다양한 구조의 내부 요소를 가지고 있는 데이터 클래스를 가능하게 한다.
- File 테이블로 다음의 데이터 모델을 하나로 합칠 수 있다.

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/cd05069a-622f-4129-8067-cf658116f618)

```sql
File {
	type : "music",
	filedId: ,
	size: ,
	name: ,
	directory: ,
	meta {
		musicId: "",
		title: "",
		artist: ""
	}
}
```

- 데이터 자체간의 관계와 속성에 따라 구성하기 보다, 해당 데이터를 이용하는 **어플리케이션이 그 데이터를 어떻게 이용(질의)**하는가에 따라 구조화

- Key
    - 조합키 (composite key를 활용하여 다수의 레코드를 하나의 항목 entity로 구성 할 수 있다.
    
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/1e49b944-d68d-46f5-b4f9-5a3bedfd2d14)

- Value에 저장되는 Row 데이터들은 한 줄의 Row 구조를 가지기는 하지만, 전체 테이블에 대해서 그 Row 구조가 일치할 필요는 없다
    - 데이터가 구조는 가지고 있지만 구조가 각각 다른 것을 허용한다고 해서 이를 Soft-Scheme 또는 Schemeless라고 한다. 이 특성을 이용하면 위의 데이터 모델을 하나의 테이블로 합칠 수 있다

## Application Side Joins

- 어쩔 수 없이 Join을 수행해야 할 경우가 있다. 이때는 NoSQL의 쿼리로는 불가능. NoSQL을 사용하는 클라이언트 Application 단에서 로직으로 처리
- Request/Response I/O가 발생해서 비용이 들긴 하지만 스토리지 사용량을 비교적 절약할 수 있다.
- 만약 대상 데이터가 수시로 변경되는 경우
    - 비정규화와 어그리게이션을 통해서 많은 entity에 해당 데이터의 중복을 허용했기 때문에 해당 데이터 업데이트 시에 모두 찾아서업데이트해야 하기 때문에 많은 비용 발생한다.
    - 변경이 잦은 데이터만을 추려내어 쿼리 타임 조인을 수행하는 것이 대안
        - *동시에 변경되는 데이터를 모아 한꺼번에 업데이터를 하는 방법*
     
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/29ca0909-6a61-4084-abad-0ba80343b811)

- 반대로 ServerSide Join 이 있다.
    - MongoDB의 경우, Stored procedure와 비슷한 기능인 MapReduce를 지원

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/36c4d083-0b24-4217-89d2-f51640212578)

# NoSQL 모델링의 과정

## 1. 도메인 모델 파악

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/94e18548-b406-4702-a8e7-2aee0c9c2933)
- NoSQL에서는 이렇게 하지 않고 바로 애플리케이션 관점으로 접근하는 경우도 많다.
- 애플리케이션의 특징적인 데이터 접근 패턴에 따라 모델링한다.
- ERD를 그려 도식화합니다.
    - NoSQL에서 저장할 데이터를 명확하게 이해하기 위해 필수적입니다.

## 쿼리 결과 디자인

- 블로그의 예시를 보자. API가 무슨 결과를 가져야 할까?
- 도메인 모델을 기반으로 어플리케이션에서 쿼리가 수행되는 결과값을 미리 정합니다.

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/3996a197-18a7-4527-8285-b4590487c7c4)

- 쿼리 예시

```sql
select categoryID,name from Category where userID=“사용자ID” -- 1

select po.postID,po.Contents,po.date,ca.name
from Category ca,Post po
where userID=“사용자ID”
order by date desc -- 2

select fileID,filename,date,fileLocation
from Attachment
where userID=“사용자ID” and postID=“현재 포스팅 ID”
order by date desc -- 3

select userID,email,Contents,date
from Comment
where userID=“사용자ID” and postID=“현재 포스팅 ID”
order by date desc --4
```

![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/52d3e8a7-3c35-4b78-aa93-4188761997e2)

## 패턴을 이용한 데이터 모델링

- NoSQL은 Sorting, Grouping, Join 등의 RDBMS 기능을 제공하지 않는다. 그러므로 Put/Get으로만 데이터를 가지고 올 수 있는 형태로 테이블을 재정의 해야 한다.
- 데이터를 가급적 중복으로 저장해 한번에 데이터를 읽어오는 횟수를 줄이도록 한다.
    - I/O 자체가 엄청난 부담이기 때문
- Join을 없애기 위해 아예 userID나 postID를 “:”로 구분되는 deliminator로 해서 Key에 포함시킬 수 있습니다.
    - 여기에는 다른 이유가 있는데, Ordered Key/Value 저장 구조에서는 Key를 기반으로 소팅하기 때문에 포스트 ID를 증가시키면 자연스럽게 하위까지 정렬이 된다.
 
## 기능 최적화

- 각 기능별 요구에 따라 저장 형식을 최적화한다.
- 위의 예시에서
    - 첨부파일의 경우 : value를 data 타입으로 하나의 키밸류에 다 포함할 수 있지 않을까?
        
        ```java
        Attachment: {
        	fileId1: {
        		name: "",
        		date: "",
        		filelocation: ""
        	}, ...
        }
        ```
        
    - 카테고리의 경우
        - 카테고리로 분류해야할 수 있으므로 secondary index를 생성하자
    - 등등

## NoSQL 선정 및 테스트

- 데이터 구조를 효과적으로 실행할 수 있는 NoSQL을 선정
- 분석과 부하테스트, 안정성, 확장성 등을 고려한다.
- 여러개의 NoSQL을 복합해서 사용할 수 있다. → application side join

## 선정된 NoSQL에 최적화

- 데이터 모델을 더 적합하게 최적화
- 어플리케이션 인터페이스 설계

# 결론

- NoSQL 시스템 개발시 데이터 모델링이 매우 중요하다.
- 반드시 데이터 모델과 내부 아키텍처를 제대로 파악하고 NoSQL을 선정, 데이터 타입을 선정할 것
- RDBMS와 혼용하거나 다른 NoSQL과 연동하여 최적의 시스템을 설계할 수 있는 가능성을 열어둘 것

---

# Refs.
- [데이터 모델링이란? (관계형 DB 편)](https://bitnine.tistory.com/446)
- [성공적인 NoSQL 도입을 위한 키포인트 : NoSQL 모델링](https://dataonair.or.kr/db-tech-reference/d-lounge/technical-data/?mod=document&uid=236123)
- [NoSQL 데이타 모델링 #1-데이타모델과, 모델링 절차](https://bcho.tistory.com/665)

