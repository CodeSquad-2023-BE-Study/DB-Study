# 전문 검색 FullText Search

- 게시물의 내용이나 제목과 같은 문장이나 문서의 내용에서 키워드를 검색하는 기능입니다.
- 원래는 Myisam에서만 가능했는데, 5.6 버전 이후 innoDB 에서 가능하게 되었습니다.
- varchar, text, Binary char에 대하여 검색할 수 있으며, 한글 데이터를 검색하기 위해서는 테이블을 utf8로 인코딩해주어야 합니다.

---

# 왜 이것을 필요하게 되었나?

- secondhand 미션을 구현하는 도중, 지역을 검색해야하는 API 요구사항을 만나게 되었다.
- 가장 간단하게 구현할 수 있는 방법은 무엇일까?

## 1. 발단 : 초기 구현 방법

- where ~ like 를 사용

```sql
select * from region where concat(city,county,district) like '%string%';
```

- 그런데 여기서는 많은 제약 조건이 있었는데..

### 초기 구현방법의 단점

이 경우 검색 성능이 매우 낮다.

- concat 도 사용하고 Like %~% 도 사용하여 인덱스를 못탄다 🥲
- `서울시` 로 검색할 경우, `서울특별시` 로 검색할 경우, `서울강남구` 로 검색할 경우 모두 해결할 수 없다.

### 성능
![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/327fd85b-6697-48a9-8632-8e0992d0b097)

## 2. 전개 : 검색 성능을 위해 복합 index를 추가

- 다음과 같이 index를 추가해주었다.

```java
CREATE INDEX **idx_address** ON region (city, county, district);
```

- 성능테스트 (위와 같은 쿼리)
    - explain 시
    ![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/95274416-8811-4754-81d8-da81a1cd6cc4)

    
    - 실행시 성능
    
    ```java
    430 rows retrieved starting from 1 in 68 ms (execution: 13 ms, fetching: 55 ms)
    ```

    - 신기하게도 index를 타고 있다는 점을 발견했다! 인덱스를 타니 성능이 2배이상 좋아졌다.

## 3. 위기 : 이대로는 안된다!

- 내가 무엇을 구현하고 싶은지 정리해보자
    - `목적1` 시, 구, 동을 한꺼번에 검색할 수 있다.
    - `목적2` `서울특별시 강남구 역삼1동` 이렇게 정확하게 검색하는 것이아니라 `서울강남` 으로 검색해도 역삼1동이 추천되어야 한다.
    - `목적3` 검색 성능을 향상시킨다.
- 어떻게 해결할 수 있을까?
    1. Mapper를 만들어 볼까? → 그 많은 지명을 어떻게 커버할 수 있나.. 노노
    2. `Elastic Search` 로 구현? → 지금은 시간이 없어서 학습과 도입에 부담스럽다.
 
### 전문검색에 대한 학습내용
- [참조링크](https://gngsn.tistory.com/162)

## 4. 결말 : Fulltext index를 추가하고 Fulltext search

- 목적2를 달성하면서 성능을 향상시킬 수 있었다.

```java
SELECT * FROM region r WHERE MATCH(city, county, district) AGAINST('서울' IN BOOLEAN MODE);
```

- 성능테스트
    - explain 시
    ![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/7f4a3e5d-5aab-4bf1-9558-9698a8c72c9c)
    
    
    - 실행시 성능
    
    ```java
    430 rows retrieved starting from 1 in 51 ms (execution: 16 ms, fetching: 35 ms)
    ```

- 성능이 조금 떨어지지만 `natural language` 검색 모드를 통해 `서울강남역삼` 으로 검색해도 원하는 결과가 나올 수 있게 되었다.
    
    ```java
    SELECT * FROM region r WHERE MATCH(city, county, district) AGAINST('서울강남역삼' IN natural language MODE);
    ```
    ![image](https://github.com/CodeSquad-2023-BE-Study/DB-Study/assets/103120173/a9a61ff1-bcd7-4533-8ac0-21e2f8e4f600)
    
    
    - 스코어 확인

# Elastic search vs Mysql Fulltext search
- 한 번 비교를 해보면?
- [참고링크1](https://stackoverflow.com/questions/41892179/elastic-search-full-text-vs-mysql-full-text)
- [참고링크2](https://medium.com/@mazraara/full-text-search-with-mysql-and-elasticsearch-b48e79a8ad4a)

---
# Refs.
- [참고링크1](https://hoing.io/archives/16853)
- [참고링크2](https://velog.io/@ddh963963/%EA%B2%80%EC%83%89-%EA%B8%B0%EB%8A%A5-%EA%B3%A0%EB%8F%84%ED%99%94-%ED%95%98%EA%B8%B0-3-%EC%84%B1%EB%8A%A5-%ED%85%8C%EC%8A%A4%ED%8A%B8)
- 
