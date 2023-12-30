## JPQL

JPQL의 데이터 타입

1) JPQL은 객체지향 쿼리 언어다.따라서 테이블을 대상으로 쿼리
하는 것이 아니라 엔티티 객체를 대상으로 쿼리한다.
2) JPQL은 SQL을 추상화해서 특정데이터베이스 SQL에 의존하
지 않는다.
3) JPQL은 결국 SQL로 변환된다.

---

### JPQL 문법

select_문 :: =
    select_절
    from_절
    [where_절]
    [groupby_절]
    [having_절]
    [orderby_절]
update_문 :: = update_절 [where_절]
delete_문 :: = delete_절 [where_절]


1) select m from Member as m where m.age > 18

2) 엔티티와 속성은 대소문자 구분O (Member, age)

3) JPQL 키워드는 대소문자 구분X (SELECT, FROM, where)

4) 엔티티 이름 사용, 테이블 이름이 아님(Member)

5) 별칭은 필수(m) (as는 생략가능)


---

### 집합과 정렬

```sql
select
    COUNT(m), //회원수
    SUM(m.age), //나이 합
    AVG(m.age), //평균 나이
    MAX(m.age), //최대 나이
    MIN(m.age) //최소 나이
from Member m
```

### TypeQuery, Query

1) TypeQuery: 반환 타입이 명확할 때 사용

```sql
TypedQuery<Member> query =
em.createQuery("SELECT m FROM Member m", Member.class);
```

2) Query: 반환 타입이 명확하지 않을 때 사용

```sql
Query query =
em.createQuery("SELECT m.username, m.age from Member m");
```

### 결과 조회 API

1) query.getResultList(): 결과가 하나 이상일 때, 리스트 반환

결과가 없으면 빈 리스트 반환

2) query.getSingleResult(): 결과가 정확히 하나, 단일 객체 반환

결과가 없으면: javax.persistence.NoResultException

둘 이상이면: javax.persistence.NonUniqueResultException


### 파라미터 바인딩 - 이름 기준, 위치 기준

SELECT m FROM Member m where m.username=**:username**
query.setParameter("**username**", usernameParam);

SELECT m FROM Member m where m.username=**?1**
query.setParameter(**1**, usernameParam);


### 프로젝션

SELECT 절에 조회할 대상을 지정하는 것
• 프로젝션 대상: 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타
입)
• SELECT m FROM Member m -> 엔티티 프로젝션
• SELECT m.team FROM Member m -> 엔티티 프로젝션
• SELECT m.address FROM Member m -> 임베디드 타입 프로젝션
• SELECT m.username, m.age FROM Member m -> 스칼라 타입 프로젝션
• DISTINCT로 중복 제거


### 프로젝션 - 여러 값 조회
SELECT m.username, m.age FROM Member m
• 1. Query 타입으로 조회
• 2. Object[] 타입으로 조회
• 3. new 명령어로 조회
• 단순 값을 DTO로 바로 조회
SELECT new jpabook.jpql.UserDTO(m.username, m.age) FROM
Member m
• 패키지 명을 포함한 전체 클래스 명 입력
• 순서와 타입이 일치하는 생성자 필요

### 페이징 API

JPA는 페이징을 다음 두 API로 추상화
• setFirstResult(int startPosition) : 조회 시작 위치
(0부터 시작)
• setMaxResults(int maxResult) : 조회할 데이터 수

### 페이징 API 예시
```sql
//페이징 쿼리
String jpql = "select m from Member m order by m.name desc";
List<Member> resultList = em.createQuery(jpql, Member.class)
    .setFirstResult(10)
    .setMaxResults(20)
    .getResultList();
```

### 조인

내부 조인:
SELECT m FROM Member m [INNER] JOIN m.team t
• 외부 조인:
SELECT m FROM Member m LEFT [OUTER] JOIN m.team t
• 세타 조인:
select count(m) from Member m, Team t where m.username
= t.name

### 조인 ON 절

ON절을 활용한 조인(JPA 2.1부터 지원)
• 1. 조인 대상 필터링
• 2. 연관관계 없는 엔티티 외부 조인(하이버네이트 5.1부터)

### 조인 대상 필터링
예) 회원과 팀을 조인하면서, 팀 이름이 A인 팀만 조인
JPQL:
SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'
SQL:
SELECT m.*, t.* FROM
Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='A'

### 서브쿼리

나이가 평균보다 많은 회원
select m from Member m
where m.age > (select avg(m2.age) from Member m2)
• 한 건이라도 주문한 고객
select m from Member m
where (select count(o) from Order o where m = o.member) > 0


### 서브 쿼리 지원 함수

• [NOT] EXISTS (subquery): 서브쿼리에 결과가 존재하면 참
• {ALL | ANY | SOME} (subquery)
• ALL 모두 만족하면 참
• ANY, SOME: 같은 의미, 조건을 하나라도 만족하면 참
• [NOT] IN (subquery): 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참


### 서브 쿼리 - 예제
팀A 소속인 회원
select m from Member m
where exists (select t from m.team t where t.name = ‘팀A')
• 전체 상품 각각의 재고보다 주문량이 많은 주문들
select o from Order o
where o.orderAmount > ALL (select p.stockAmount from Product p)
• 어떤 팀이든 팀에 소속된 회원
select m from Member m
where m.team = ANY (select t from Team t)

### JPA 써브 쿼리 한계

JPA는 WHERE, HAVING 절에서만 서브 쿼리 사용 가능
• SELECT 절도 가능(하이버네이트에서 지원)
• FROM 절의 서브 쿼리는 현재 JPQL에서 불가능
• 조인으로 풀 수 있으면 풀어서 해결