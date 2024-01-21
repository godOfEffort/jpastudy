## QueryDSL

### 시작

```java
 public void queryDSL() {
     EntityManager em = emf.createEntityManager();
 
     JPAQuery query = new JPAQuery(em);
     QMember qMember = new QMember("m")// JPQL의 별칭
 
     List<Member> members = query.from(qMember)
     .where(qMember.name.eq("회원1"))
     .orderBy(qMember.name.desc())
     .list(qMember);
 }
```

QueryDSL 을 사용하려면 JPAQuery 객체를 생성해야 하는데 이 때 엔티티 매니저를 생성자에 넘겨준다.

사용할 쿼리 타입(Q)을 생성하는데 생성자에는 별칭을 주면 된다.


### 기본 Q 생성

쿼리 타입은 사용하기 편하도록 기본 인스턴스를 보관하고 있다.

하지만 같은 엔티티를 조인하거나 같은 엔티티를 서브쿼리에 사용하면 같은 별칭이 사용되므로 이때는 별칭을 직접 지정해야 한다.

```java
QMember qMember = new QMember("m"); // 직접 지정
QMember qMember = QMember.member; // 기본 인스턴스 사용
```

### 결과 조회

쿼리 작성이 끝나고 결과 조회 메서드를 호출하면 실제 데이터베이스를 조회한다.

1) **uniqueResult()**: 조회 결과가 한 건일 때 사용, 조회 결과가 없으면 null, 하나 이상이면 예외가 발생

2) **singleResult()**: 결과가 하나 이상이면 처음 데이터를 반환한다.

3) **list()**: 결과가 없으면 빈 컬렉션을 반환한다.

### 페이징과 정렬

```java
QItem item = QItem.item;

query.from(item)
    .where(item.price.gt(20000))
    .orderBy(item.price.desc(), item.stockQuantity.asc())
    .offset(10).limit(20)
    .list(item);
```

실제 페이징 처리를 하려면 검색된 전체 데이터 수를 알아야 한다.

```java
    SearchResults<Item> result = 
        query.from(item)
        .where(item.price.gt(10000))
        .offset(10).limit(20)
        .listResults(item);

long total = result.getTotal();
long limit = result.getLimit();
long offset = result.getOffSet();
List<Item> results = result.getResults();
```

### 조인

조인은 innerJoin, leftJoin, rightJoin, fullJoin을 사용할 수 있다.

기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭으로 사용할 쿼리 타입을 지정하면 된다.
       
**join(조인 대상, 별칭으로 사용할 쿼리 타입)**

```java
    Qorder order = QOrder.order;
    QMember member = QMember.member;
    QOrderItem orderItem = QOrderItem.orderItem;
    
    query.from(order)
    .join(order.member, member)
    .leftJoin(order.orderItems, orderItem)
    .list(order);
```

on 과 fetch 조인을 사용할 수 있다.
```java
query.from(order)
    .leftJoin(order.orderItems, orderItem)
    .on(orderItem.count.gt(2))
    .list(order);
```


```java
query.from(order)
    .innerJoin(order.member, member).fetch()
    .leftJoin(order.orderItem, orderItem).fetch()
    .list(order);
```

아래 코드은 FROM 절에 여러 조인을 사용하는 세타 조인 방법이다.
```java
query.from(order, member)
    .where(order.member.eq(member))
    .list(order);
```

### 서브 쿼리

서브 쿼리는 **com.mysema.query.JPASubQuery**를 생성해 사용한다.

```java

    QItem item = QItem.item;
    QItem itemSub = new QItem("itemSub");

    query.from(item)
    .where(item.price.eq(
        new JPASubQuery().from(itemSub).unique(itemSub.price.max())
    ))
    .list(item);
```

서브 쿼리의 결과가 하나면 **unique()**, 여러 건이면 **list()**를 사용하면 된다.

```java

    QItem item = QItem.item;
    QItem itemSub = new QItem("itemSub");

    query.from(item)
    .where(item.in(
        new JPASubQuery().from(itemSub)
        .where(item.name.eq(itemSub.name))
        .list(itemSub)
    ))
    .list(item);
```

### 프로젝션과 결과 반환

select 절에 조회 대상을 지정하는 것을 프로젝션이라 한다.

1) 프로젝션 대상이 하나

프로젝션 대상이 하나이면 해당 타입으로 반환된다.

```java
QItem item = QItem.item;

// item의 name만 select, name이 String이니까 해당 타입으로 반환
List<String> result = query.from(item).list(item.name);


for(String name : result){
System.out.println("name : " + name);
}
```

2) 여러 컬럼 반환과 튜플

프로젝션 대상이 여러 필드면 Tuple이라는 Map과 비슷한 내부 타입을 사용하게 된다.

```java
QItem item = QItem.item;

List<Tuple> result = query.from(item).list(item.name, item.price); 
//List<Tuple> result = query.from(item).list(new QTuple(item.name, item.price));

for (Tuple tuple: result) {
  System.out.println("name = " + tuple.get(item.name));
  System.out.println("price = " + tuple.get(item.price));
}
```

### 빈 생성

쿼리 결과가 엔티티가 아닌 특정 객체로 받고 싶으면 빈 생성 기능을 사용한다.

QueryDSL은 객체를 생성하는 다양한 방법을 제공한다.

- 프로퍼티 접근
- 필드 직접 접근
- 생성자 사용

```java
package com.compare;

public class ItemDTO {
    
    private String username;
    private int price;

    public ItemDTO() {
    }

    public ItemDTO(String username, int price) {
        this.username = username;
        this.price = price;
    }


    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public int getPrice() {
        return price;
    }

    public void setPrice(int price) {
        this.price = price;
    }
}

```

```java
// 프로퍼티 접근
QItem item = QItem.item;

// 프로퍼티 접근(Setter)
// Setter 사용해서 값 채워줌
List<ItemDTO> result = query
  .from(item)
  .list(
    Projections.bean(ItemDTO.class, item.name.as("username"), item.price) // 쿼리 결과와 매핑할 프로퍼티 이름 다르면 as 사용
  );

// field 직접 접근
// Projections.fields() 메소드 사용시 필드에 직접 접근해서 값 채워줌
// 필드를 private으로 설정해도 동작함
List<ItemDTO> result = query
  .from(item)
  .list(
    Projections.fields(ItemDTO.class, item.name.as("username"), item.price)
  );

// constructor 사용
// 생성자 이용해서 값 채움
// 지정한 프로젝션과 파라미터 순서 동일한 생성자가 필요
List<ItemDTO> result = query
  .from(item)
  .list(
    Projections.constructor(ItemDTO.class, item.name, item.price)
  );

```

### 수정, 삭제 배치 쿼리

QueryDSL에서도 수정, 삭제 같은 배치 쿼리를 지원한다.

JPQL 배치 쿼리와 같이 영속성 컨텍스트를 무시하고 데이터베이스를 직접 쿼리하게 된다는 점에 유의해야 한다.

1) 동적 쿼리

BooleanBuilder를 사용하면 특정 조건에 따른 동적 쿼리를 편리하게 생성할 수 있다.

```java
SearchParam param = new SearchParam();
param.setName("itemA")
param.setPrice(10000);

QItem item = QItem.item;

BooleanBuilder builder = new BooleanBuilder();
if (StringUtils.hasText(param.getName())) {
  builder.and(item.name.contains(param.getName()));
}
ig (param.getPrice() != null) {
  builder.and(item.price.gt(param.getPrice()));
}

List<Item> result = query
  .from(item)
  .where(builder)
  .list(item);
```


### 네이티브 SQL

JPQL은 표준 SQL이 지원하는 대부분의 문법과 SQL 함수들을 지원하지만 특정 데이터베이스에 종속적인

기능은 지원하지 않는다.

때로는 특정 데이터베이스에 종속적인 기능이 필요한데 JPA는 특정 데이터베이스에 종속적인 기능을 사용할 수 있는 방법을 제공한다.

1) 특정 데이터베이스만 사용하는 함수
    - JPQL에서 네이티브 SQL 함수를 호출할 수 있다.
    - 하이버네이트는 데이터베이스 방언에 각 데이터베이스에 종속적인 함수들을 정의해두었다.
        - 직접 호출할 함수를 정의할 수도 있다.
        
2) 특정 데이터베이스만 지원하는 SQL힌트
    - 하이버네이트를 포함한 몇몇 JPA 구현체들이 지원한다.
    
3) 인라인 뷰, UNION, INTERSECT
    - 하이버네이트는 지원하지 않지만 일부 JPA 구현체들이 지원한다.
    
4) 스토어 프로시저
    - JPQL에서 스토어 프로시저를 호출할 수 있다.
    
5) 특정 데이터베이스만 지원하는 문법
    - 오라클의 CONNECTY BY 처럼 특정 데이터베이스에 너무 종속된 SQL 문법은 지원하지 않는다.
     
    이때는 네이티브 SQL을 사용해야 한다.
    
다양한 이유로 JPQL을 사용할 수 없을 떄 JPA는 SQL을 직접 사용할 수 있는 기능을 제공하는데

이것을 네이티브 SQL이라 한다. 네이티브 SQL은 이 SQL을 개발자가 직접 정의한다.

네이티브 SQL은 수동모드라고 생각하면 된다. 

네이티브 SQL을 사용하면 엔티티를 조회할 수 있고 JPA가 지원하는 영속성컨텍스트의 기능을 그대로 사용할 수 있다.


### 네이티브 SQL 사용

1) 엔티티 조회

네이티브 SQL은 **em.createNativeQuery(SQL, 결과 클래스)** 를 사용한다.
          
첫 번째 파라미터는 네이티브 SQL을 입력하고 두 번째 파라미터에는 조회할 엔티티의 클래스 타입을 입력한다.

```java
String sql = 
  "SELECT id, age, name, team_id " +
  "FROM member " +
  "WHERE age > ?";

Query nativeQuery = em.createNativeQuery(sql, Member.class) // native SQL은 type 정보 줘도 TypeQuery 아니고 Query 임
  .setParameter(1, 20);

List<Member> resultList = nativeQuery.getResultList();
```

**여기서 가장 중요한 점은 네이티브 SQL로 직접 SQL을 사용할 뿐이지 나머지는 JPQL을 사용할 때와 같다.**

**조회한 엔티티도 영속성 컨텍스트에서 관리된다.**

### 결과 매핑 사용

매핑이 복잡해지면 @SqlResultSetMapping을 정의해 결과 매핑을 사용해야 한다.

```java
String sql =
  "SELECT m.id age, name, team_id, i.order_count " +
  "FROM member m " +
  " LEFT JOIN " +
  "   (SELECT im.id, COUNT(*) AS order_count " +
  "   FROM orders o, member im " +
  "   WHERE o.member_id = im.id) i " +
  " ON m.id = i.id";

Query nativeQuery = em.createNativeQuery(sql, "memberWithOrderCount");
```

```java
@Entity
@SqlResultSetMapping(name = "memberWithOrderCount",
  entities = {@EntityResult(entityClass = Member.class)}, 
  columns = {@ColumnResult(name = "order_count")} 
)
public class Member { ... }
```

### Named 네이티브 SQL
JPQL처럼 네이티브 SQL도 Named 네이티브 SQL을 사용해서 정적 SQL을 작성할 수 있다.

```java
@Entity
@NamedNativeQuery(
  name = "Member.memberSQL",
  query = 
    "SELECT id, age, name, team_id " +
    "FROM member " +
    "WHERE age > ?", 
  resultClass = Member.class
)
public class Member { ... }
```

@NamedNativeQuery로 Named 네이티브 SQL을 등록했다. 다음으로 사용하는 예제이다.

```java
TypedQuery<Member> nativeQuery =
     em.createNamedQuery("Member.memberSQL", Member.class)
      .setParamter(1, 20);
```

### 네이티브 SQL 정리

네이티브 SQL도 JPQL을 사용할 떄와 마찬가지로 Query, TypeQuery를 반환한다. 따라서 JPQL API를 그대로

사용할 수 있다. 네이티브 SQL은 JPQL이 자동 생성하는 SQL을 수동으로 직접 정의하는 것이다. 

따라서 JPA가 제공하는 기능 대부분을 그대로 사용할 수 있다.


### 스토어드 프로시저

JPA는 2.1부터 스토어드 프로시저를 지원한다.

#### 스토어드 프로시저 사용

```sql
DELIMITER //

CREATE PROCEDURE proc_multiply (INOUT inParam INT, INOUT outParam INT)
BEGIN
  SET outParam = inParam * 2;
END //

```


```sql
/ proc_multiply 프로시저 호출 (순서 기반 파라미터)
StoredProcedureQuery spq = em.createStoredProcedureQuery("proc_multiply");
spq.registerStoredProcedureParamter(1, Integer.class, ParamterMode.IN);
spq.registerStoredProcedureParamter(2, Integer.class, ParamterMode.OUT);

spq.setParamter(1, 100);
spq.execute();

Integer out = (Integer) spq.getOutputParamterValue(2);
System.out.println("out = " + out); // out = 200

...

// proc_multiply 프로시저 호출 (이름 기반 파라미터)
StoredProcedureQuery spq = em.createStoredProcedureQuery("proc_multiply");
spq.registerStoredProcedureParamter("inParam", Integer.class, ParamterMode.IN);
spq.registerStoredProcedureParamter("outParam", Integer.class, ParamterMode.OUT);

spq.setParamter("inParam", 100);
spq.execute();

Integer out = (Integer) spq.getOutputParamterValue(2);
System.out.println("out = " + out); 
```

스토어드 프로시저를 사용하려면 em.createStoredProcedureQuery()메서드에 사용할 스토어드 프로시저 이름을 입력하면 된다.

그 후 registerStoredProcedureParameter()메서드를 사용해 프로시저에서 사용할 파라미터를 순서, 타입, 파리미터 모드 순으로 정의하면 된다.


### Named 스토어드 프로시저 사용

스토어드 프로시저 쿼리에 이름을 부여해서 사용하는 것을 **Named 스토어드 프로시저**라고 한다.

```java
@Entity
@NamedStoredProcedureQuery(
  name = "multiply",
  procedureName = "proc_multiply",
  paramter = {
    @StoredProcedurePatameter(name = "inParam", mode = ParameterMode.IN, type = Intger.class),
    @StoredProcedureParamter(name = "outParam", mode = ParameterMode.OUT, type = Integer.class)
  }
)
public class Member { ... }
```

@NamedStoredProcedureQuery로 정의하고 name 속성을 부여하면 된다.

procedureName 속성에 실제 호출할 프로시저 이름을 적어주고 @StoredProcedurePatameter를 

사용해 파라미터 정보를 정의하면 된다. 

둘 이상을 정의하려면 @NamedStoredProcedureQueries를 사용하면 된다.

스토어드 프로시저는 아래와 같이 사용한다

```java
StoredProcedureQuery spq = em.createNamedStoredProcedureQuery("multiply");

spq.setParamter("inParam", 100);
spq.execute();

Integer out = (Integer) spq.getOutputParamterValue("outParam");
System.out.println("out = " + out);
```

### 객체지향 쿼리 심화

#### 벌크 연산

엔티티를 수정하려면 영속성 컨텍스트의 변경 감지 기능이나 병합을 사용하고, 삭제하려면

EntityManager.remove() 메소드를 사용해야 하지만 수백 개 이상의 엔티티를 처리하기에는 시간이 너무 오래걸린다

이럴때 벌크 연산을 사용하면 된다.

```java
//UPDATE 벌크 연산
String qlString =
  "UPDATE Product p " +
  "SET s.price = p.price * 1.1 " +
  "WHERE p.stockAmount < :stockAmount";

int resultCounht = em.createQuery(qlString)
  .setParamter("stockAmount", 10)
  .executeUpdate();
```

벌크 연산은 excuteUpdate()를 사용한다.

이 메서드는 벌크 연산으로 영향을 받은 엔티티 건수를 반환한다.

삭제에서도 같은 메서드를 사용한다.

```java
//DELETE 벌크 연산
String qlString = "delete from Product p " +
                  "where p.price < :price";

int resultCounht = em.createQuery(qlString)
  .setParamter("price", 100)
  .executeUpdate();
```

JPA 표준은 아니지만 하이버네이트는 INSERT 벌크 연산도 지원한다. 다음 코드는 100원 미만의 모든 상품을 선택해서

ProductTemp에 저장한다.

```java
String qlString =
  "INSERT into ProductTemp(id, name, price, stockAmount) " +
  "SELECT p.id, p.name, p.price, p.stockAmount FROM Product p " +
  "WHERE p.price < :price";

int resultCount = em.createQuery(qlString)
  .setParameter("price", 100)
  .executeUpdate();
```


### 벌크 연산의 주의점

벌크 연산을 사용할 때는 **영속성 컨텍스트를 무시하고 데이터베이스에 직접 쿼리 한다는 점을 주의해야 한다.**

이런 문제를 해결하는 방법은 아래와 같다.

**1) em.refresh()사용**

벌크 연산을 수행한 직후에 정확한 엔티티를 사용해야 한다면 em.refresh()를 사용해 

데이터베이스에서 상품A를 다시 조회하면 된다.

em.refresh(ProductA);

**2) 벌크 연산 먼저 실행**

가장 실용적인 해결책으로 벌크 연산을 가장 먼저 실행하는 것이다.

예를 들어 벌크 연산을 먼저 실행하고 나서 상품 A를 조회하면 벌크 연산으로 이미 변경된 상품A를 조회하게 된다.

이 방법은 JPA와 JDBC를 함께 사용할 때도 유용하다.

**3) 벌크 연산 수행 후 영속성 컨텍스트 초기화**

- 벌크 연산을 수행한 직후 바로 영속성 컨텍스트를 초기화하는 방법도 있다.

- 영속성 컨텍스트를 초기화하면 이후 엔티티를 조회할 때 벌크 연산이 적용된 데이터베이스에서 엔티티를 조회한다.


벌크 연산은 영속성 컨텍스트와 2차 캐시를 무시하고 데이터베이스에 직접 실행한다.

영속성 컨텍스트와 데이터베이스 간에 데이터차이가 발생할 수 있으므로 주의해서 사용해야 한다.

가능하면 벌크 연산을 먼저 수행하는 것이 좋고 상황에 따라 영속성 컨텍스트를 초기화하는 것도 필요하다.


### 영속성 컨텍스트와 JPQL

#### 쿼리 후 영속 상태인 것과 아닌 것

JPQL로 엔티티를 조회하면 영속성 컨텍스트에서 관리되지만 엔티티가 아니면 영속성 컨텍스트에서 관리되지 않는다.

조회한 엔티티만 영속성 컨텍스트가 관리한다.

#### JPQL로 조회한 엔티티와 영속성 컨텍스트

그런데 만약 다음 예제처럼 영속성 컨텍스트에 회원1이 이미 있는데 JPQL로 회원1을 다시 조회하면 어떻게 될까?

```java
em.find(Member.class, "member1"); // 회원1 조회

List<Member> resultList = em.createQuery("select m from Member m",
                          Member.class)
                          .getResultList();
```

**JPQL로 데이터베이스에서 조회한 엔티티가 영속성 컨텍스트에 이미 있으면 JPQL로 데이터베이스에서 조회한 결과를 버리고 영속성 컨텍스트에 있던 엔티티를 반환한다.**

이때는 식별자 값을 사용해서 비교한다.

1) JPQL을 사용해서 조회를 요청

2) JPQL은 SQL로 변환되어 데이터베이스 조회

3) 조회한 결과와 영속성 컨텍스트 비교

4) 식별자 값을 기준으로 이미 영속성 컨텍스트에 있으면 JPQL로 가져온 값을 버리고 기존에 있던 엔티티를 반환

5) 식별자 값을 기준으로 영속성 컨텍스트에 엔티티가 없다면 영속성 컨텍스트에 조회한 엔티티 추가

6) 쿼리 결과 반환

이를 통해 다음 2가지를 확인할 수 있다.

1) JPQL로 조회한 엔티티는 영속 상태다.

2) 영속성 컨텍스트에 이미 존재하는 엔티티가 있으면 기존 엔티티를 반환한다.


영속성 컨텍스트는 영속 상태인 엔티티의 동일성을 보장한다. em.find()로 조회하든 JPQL을 사용하든

영속성 컨텍스트가 같으면 동일한 엔티티를 반환한다.

### find() vs JPQL

em.find() 메서드는 엔티티를 영속성 컨텍스트에서 먼저 찾고 없으면 데이터베이스에서 검색하기 때문에

해당 엔티티가 영속성 컨텍스트에 있으면 메모리에서 바로 찾으므로 성능상 이점이 있다.

그래서 1차 캐시라고 부른다.

JPQL은 항상 데이터베이스에 SQL을 실행해서 결과를 조회한다.

JPQL의 특징

1) JPQL은 항상 데이터베이스를 조회한다.

2) JPQL로 조회한 엔티티는 영속 상태다.

3) 영속성 컨텍스트에 이미 존재하는 엔티티가 있으면 기존 엔티티를 반환한다.

### JPQL과 플러시 모드

플러시는 영속성 컨텍스트의 변경 내역을 데이터베이스에 동기화하는 것이다.

보통 플러시 모드에 따라 커밋하기 직전이나 쿼리 실행 직전에 자동으로 플러시가 호출된다.

플러시 모드는 FlushModeType.AUTO가 기본값이기 때문에 JPA는 트랙잭션 커밋 직전이나 쿼리 실행 직전에 자동으로 플러시를 호출한다.

FlushModeType.COMMIT이 있는데 이는 커밋 시에만 플러시를 호출한다.

이 옵션은 성능 최적화를 위해 꼭 필요할 때만 사용해야 한다.

#### 쿼리와 플러시 모드

JPQL은 영속성 컨텍스트에 있는 데이터를 고려하지 않고 데이터베이스에서 조회하기 때문에

따라서 JPQL을 실행하기 전에 영속성 컨텍스트의 내용을 데이터베이스에 반영해야 한다.

플러시 모드를 COMMIT으로 설정하면 쿼리 시에 플러시하지 않으므로 방금 수정한 데이터를 조회할 수 없다.

이때는 직접 em.flush()를 호출하거나 Query객체에 플러시 모드를 설정해주면 된다.

```java
em.setFlushMode(FlushModeType.COMMIT);

//가격을 1000-> 2000원으로 변경
product.setPrice(2000); 

//1. em.flush() 직접 호출

//가격이 2000원인 상품 조회
Product product = em.createQuery("select p from product p where p.price = 2000", Product.class)
        .setFlushMode(FlushModeType.Auto) // 2. setFlushMode() 설정
        .getSingleResult();
```


코드를 보면 첫 줄에서 플러시 모드를 COMMIT으로 설정했다. 따라서 쿼리를 실행할 때 플러시를

자동으로 호출하지 않는다.


### 플러시 모드와 최적화

**em.setFlushMode(FlushModeType.COMMIT)**

**FlushModeType.COMMIT** 모드는 트랜잭션 커밋 시에만 플러시하고 쿼리를 실행할 때는 플러시하지 않는다.

따라서 JPA 쿼리를 사용할 때 영속성 컨텍스트에 있지만 아직 데이터베이스에 반영하지 않은 데이터를 조회할 수 없다.

그럼에도 플러시가 너무 자주 일어나는 상황에 이 모드를 사용하면 쿼리시 발생하는 플러시 횟수를 줄여 성능을 최적화할 수 있다.

- FlushModeType.AUTO : 쿼리와 커밋할 떄 총 4번 플러시 한다

- FlushModeType.COMMIT : 커밋시에만 1번 플러시 한다.

JPA를 사용하지 않고 JDBC를 사용해 직접 SQL을 실행할 때도 플러시 모드를 고민해야 한다.

JPA를 통하지 않고 JDBC로 쿼리를 직접 실행하면 JPA는 JDBC가 실행한 쿼리를 인식할 방법이 없다.

별도의 JDBC 호출은 플러시 모드를 AUTO로 설정해도 플러시가 일어나지 않는다.

이때는 JDBC로 쿼리를 실행하기 직전에 em.flush()를 호출해서 영속성 컨텍스트의 내용을 데이터베이스에 동기화하는 것이 안전하다.
