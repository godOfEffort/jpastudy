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

---

### 결과 조회 API

1) query.getResultList(): 결과가 하나 이상일 때, 리스트 반환

     - 결과가 없으면 빈 리스트 반환

2) query.getSingleResult(): 결과가 정확히 하나, 단일 객체 반환

     - 결과가 없으면: javax.persistence.NoResultException

     - 둘 이상이면: javax.persistence.NonUniqueResultException


### 파라미터 바인딩 - 이름 기준, 위치 기준

1) SELECT m FROM Member m where m.username=**:username**

    query.setParameter("**username**", usernameParam);

2) SELECT m FROM Member m where m.username=**?1**

    query.setParameter(**1**, usernameParam);


---

### 프로젝션

SELECT 절에 조회할 대상을 지정하는 것

- 프로젝션 대상 : **엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타입)**

     - SELECT m FROM Member m -> 엔티티 프로젝션

     - SELECT m.team FROM Member m -> 엔티티 프로젝션

     - SELECT m.address FROM Member m -> 임베디드 타입 프로젝션

     - SELECT m.username, m.age FROM Member m -> 스칼라 타입 프로젝션

     - DISTINCT로 중복 제거

---

### 프로젝션 - 여러 값 조회
**SELECT m.username, m.age FROM Member m**

- Query 타입으로 조회

- Object[] 타입으로 조회

- new 명령어로 조회

- 단순 값을 DTO로 바로 조회

**SELECT new jpabook.jpql.UserDTO(m.username, m.age) FROM Member m**

- 패키지 명을 포함한 전체 클래스 명 입력

- 순서와 타입이 일치하는 생성자 필요


---


### 페이징 API

JPA는 페이징을 다음 두 API로 추상화

- setFirstResult(int startPosition) : 조회 시작 위치 (0부터 시작)

- setMaxResults(int maxResult) : 조회할 데이터 수

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

- 내부 조인: **SELECT m FROM Member m [INNER] JOIN m.team t**

- 외부 조인: **SELECT m FROM Member m LEFT [OUTER] JOIN m.team t**

- 세타 조인: **select count(m) from Member m, Team t where m.username = t.name**
  

### 조인 ON 절

ON절을 활용한 조인(JPA 2.1부터 지원)

- 1. 조인 대상 필터링

- 2. 연관관계 없는 엔티티 외부 조인(하이버네이트 5.1부터)
     

### 조인 대상 필터링
예) 회원과 팀을 조인하면서, 팀 이름이 A인 팀만 조인

- JPQL: **SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'A'**

- SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='A'

---

### 페치 조인

1) SQL 조인 종류가 아니다

2) JPQL 에서 성능 최적화를 위해 제공하는 기능이다.

3) 연관된 엔티티나 컬렉션을 SQL 한 번에 함께 조회하는 기능이다.

4) join fetch 명령어를 사용한다

5) 페치 조인 ::= [ LEFT [OUTER] | INNER ] JOIN FETCH 조인경로
   

### 엔티티 페치 조인

회원을 조회하면서 연관된 팀도 함께 조회(SQL 한 번에) SQL을 보면 회원 뿐만 아니라 팀(T.*)도 함께 SELECT

```sql
//JPQL
select m from Member m join fetch m.team
```

```sql
//SQL
SELECT M.*, T.* FROM MEMBER M
INNER JOIN TEAM T ON M.TEAM_ID=T.ID
```

### 페치 조인과 일반 조인의 차이
JPQL은 결과를 반환할 때 연관관계까지 고려하지 않는다. 단지 SELECT 절에 지정한 엔티티만 조회할 뿐이다.
따라서 팀 엔티티만 조회하고 연관된 회원 컬렉션은 조회하지 않는다.

페치 조인은 사용할 때만 연관된 엔티티도 함께 조회(즉시로딩)한다.
페치 조인은 객체 그래프를 SQL 한 번에 조회하는 개념이다.


### 페치 조인의 특징과 한계
1) 페치 조인 대상에는 별칭을 줄 수 없다.
   - 하이버네이트는 가능하지만 데이터 무결성이 깨질 수 있으므로 조심해서 사용해야 한다.
  
2) 둘 이상의 컬렉션은 페치 조인 할 수 없다.

3) 컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다.
   - 컬렉션이 아닌 단일 값 연관 필드(일대일, 다대일) 들은 페치 조인을 사용해도 페이징 API를 사용할 수 있다.
   - 하이버네이트에서 컬렉션을 페치 조인하고 페이징 API를 사용하면 경고 로그를 남기면서 메모리에서 페이징 처리를 한다.
     데이터가 적으면 상관없겠지만 데이터가 많으면 성능 이슈와 메모리 초과 예외가 발생할 수 있어서 위험하다.

### 경로 표현식

#### 용어정리
1) **상태 필드** : 단순히 값을 저장하기 위한 필드
2) **연관 필드** : 연관관계를 위한 필드, 임베디드 타입 포함
    - 단일 값 연관 필드 : @ManyToOne, @OneTooOne, 대상이 엔티티
    - 컬렉션 값 연관 필드 : @OneToMany, @ManyToMany, 대상이 컬렉션

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;

    @Column(name = "name)
    private String username; //상태필드
    private Integer age;  //상태필드

    @ManyToOne
    private Team team; //연관필드(단일 값 연관 필드)

    @OneToMnay
    private List<Order> orders; // 연관 필드(컬렉션 값 연관 필드)
}
```

#### 경로 탐색을 사용한 묵시적 조인 시 주의사항
- 항상 내부조인을 사용한다
- 컬렉션은 경로 탐색의 끝, 명시적 조인을 통해 별칭을 얻어야 한다
- 경로 탐색은 주로 SELECT, WHERE 절에서 사용하지만 묵시적 조인으로 인해 SQL의 FROM (JOIN) 절에 영향을 준다.

**묵시적 조인은 조인이 일어나는 상황을 한눈에 파악하기 어렵기 때문에 가급적 명시적 조인을 사용하는 것이 좋다**

  

### 서브쿼리

- 나이가 평균보다 많은 회원
  
**select m from Member m where m.age > (select avg(m2.age) from Member m2)**

- 한 건이라도 주문한 고객
  
**select m from Member m where (select count(o) from Order o where m = o.member) > 0**


---

### 서브 쿼리 지원 함수

- [NOT] EXISTS (subquery): 서브쿼리에 결과가 존재하면 참

- {ALL | ANY | SOME} (subquery)

- ALL 모두 만족하면 참

- ANY, SOME: 같은 의미, 조건을 하나라도 만족하면 참

- [NOT] IN (subquery): 서브쿼리의 결과 중 하나라도 같은 것이 있으면 참


---


### 서브 쿼리 - 예제
- 팀A 소속인 회원
  
**select m from Member m where exists (select t from m.team t where t.name = ‘팀A')**

- 전체 상품 각각의 재고보다 주문량이 많은 주문들
  
**select o from Order o where o.orderAmount > ALL (select p.stockAmount from Product p)**

- 어떤 팀이든 팀에 소속된 회원
  
**select m from Member m where m.team = ANY (select t from Team t)**

### JPA 서브 쿼리 한계

JPA는 WHERE, HAVING 절에서만 서브 쿼리 사용 가능

- SELECT 절도 가능(하이버네이트에서 지원)

- FROM 절의 서브 쿼리는 현재 JPQL에서 불가능

- 조인으로 풀 수 있으면 풀어서 해결
  

### JPQL 타입 표현
1) 문자: ‘HELLO’, ‘She’’s’
2) 숫자: 10L(Long), 10D(Double), 10F(Float)
3) Boolean: TRUE, FALSE
4) ENUM: jpabook.MemberType.Admin (패키지명 포함)
5) 엔티티 타입: TYPE(m) = Member (상속 관계에서 사용)

### 조건식 - CASE 식
- COALESCE: 하나씩 조회해서 null이 아니면 반환
- NULLIF: 두 값이 같으면 null 반환, 다르면 첫번째 값 반환

사용자 이름이 없으면 이름 없는 회원을 반환

```sql
select coalesce(m.username,'이름 없는 회원') from Member m
```

사용자 이름이 '관리자'면 null을 반환하고 나머지는 본인의 이름을 반환

```sql
select NULLIF(m.username, '관리자') from Member m
```


### Named 쿼리 - 정적쿼리
JPQL 쿼리는 크게 동적 쿼리와 정적 쿼리로 나눌 수 있다.

동적 쿼리: e.createQuery("select...") 처럼 JPQL을 문자로 완성해 직접 넘기는 것을 의미

정적 쿼리: 미리 정의한 쿼리에 이름을 부여해 필요할 때 사용하는 것을 Named 쿼리라 한다.

Named 쿼리는 애플리케이션 로딩 시점에 JPQL 문법을 체크하고 미리 파싱해둔다. 따라서 오류를 빨리 확인할 수 있고, 사용하는
시점에는 파싱된 결과를 재사용하므로 성능상 이점도 있다. 그리고 Named 쿼리는 변하지 않는 정적 SQL이 생성되므로
데이터베이스의 조회 성능 최적화에도 도움이 된다.

```java
@NamedQuery(
    name = "Member.findByName",
    query = "select m from Member m where m.name = :name
)
public class Member {

}
```

```java
List<Member> resultList = em.createNamedQuery("Member.findByUsername", Member.class)
        .setParameter("username", "회원1")
        .getResultList();
```

**Named 쿼리 - XML 에 정의**
```xml
<persistence-unit name="jpabook" >
<mapping-file>META-INF/ormMember.xml</mapping-file>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<entity-mappings xmlns="http://xmlns.jcp.org/xml/ns/persistence/orm" version="2.1">
    <named-query name="Member.findByUsername">
        <query><![CDATA[
            select m
            from Member m
            where m.username = :username
        ]]></query>
    </named-query>

    <named-query name="Member.count">
        <query>select count(m) from Member m</query>
    </named-query>
</entity-mappings>
```


#### Named 쿼리 환경에 따른 설정
1) XML이 항상 우선권을 가진다.

2) 애플리케이션 운영 환경에 따라 다른 XML을 배포할 수 있다.
