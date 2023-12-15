## 프록시와 연관관계 관리


### 프록시

프록시를 사용하면 연관된 객체를 처음부터 데이터베이스에서 조회하는 것이 아니라,

실제 사용하는 시점에 데이터베이스에서 조회할 수 있다. 하지만 자주 함께

사용하는 객체들은 조인을 사용해서 함께 조회하는 것이 효과적이다.

JPA는 **즉시로딩**과 **지연로딩**이라는 방법으로 둘을 모두 지원한다.

```java
 Member member = em.find(Member.class, memberId);
 Team team  = member.getTeam();
 System.out.println("회원이름 :" + member.getUsername());
 System.out.println("소속팀 :" + team.getName());
```

JPA는 엔티티가 실제 사용될 때까지 데이터베이스 조회를 지연하는 방법을

제공하는데 이것을 지연 로딩이라 한다. team.getName()처럼 실제 사용하는

시점에 데이터베이스에서 팀 엔티티에 필요한 데이터를 조회하는 것이다.

그런데 지연 로딩 기능을 사용하려면 실제 엔티티 객체 대신에 데이터베이스

조회를 지연할 수 있는 가짜 객체가 필요한데 이것을 프록시 객체라 한다.

### 프록시 기초

```java
    Member mebmer = em.find(Member.class, "member1");
```

이렇게 엔티티를 직접 조회하면 조회한 엔티리를 실제 사용하든 사용하지 않든 데이터베이스를 조회하게 된다.

엔티티를 실제 사용하는 시점까지 데이터베이스 조회를 미루고 싶으면 EntityManager.getReference() 메소드를

사용하면 된다.


```java
    Member mebmer = em.getReference(Member.class, "member1");
```

이 메서드를 호출할 때 JPA는 데이터베이스를 조회하지 않고 실제 엔티티 객체도 생성하지 않는다.

**대신에 데이터베이스 접근을 위한 프록시 객체를 반환한다.**

1) 프록시의 특징

프록시 객체는 실제 객체에 대한 참조를 보관한다. 

프록시 객체의 메소드를 호출하면 프록시 객체는 실제 객체의 메소드를 호출한다.

2) 프록시 객체의 초기화

프록시 객체는 member.getName()처럼 실제 사용될 때 데이터베이스를 조회해서 

실제 엔티티 객체를 생성하는데 이것을 프록시 객체의 초기화라 한다.

```java
Member member = em.getReference(Member.class, "id1");
member.getName(); 
```

#### 프록시 초기화 과정

1. 프록시 객체에 member.getName()을 호출해서 실제 데이터를 조회한다.

2. 프록시 객체는 실제 엔티티가 생성되어 있지 않으면 영속성 컨텍스트에 실제 엔티티 생성을 요청하는데

이것을 초기화라고 한다.

3. 영속성 컨텍스트는 데이터베이스를 조회해서 실제 엔티티 객체를 생성한다

4. 프록시 객체는 생성된 실제 엔티티 객체의 참조를 Member target 멤버변수에 보관한다

5. 프록시 객체는 실제 엔티티 객체의 getName()을 호출해서 결과를 반환한다.

#### 프록시의 특징

1. 프록시 객체는 처음 사용할 때 한번만 초기화 된다.

2. 프록시 객체를 초기화 한다고 프록시 객체가 실제 엔티티로 바뀌지 않는다. 

프록시 객체가 초기화되면 프록시 객체를 통해서 실제 엔티티에 접근할 수 있다.

3. 프록시 객체는 원본 엔티티를 상속받은 객체이므로 타입 체크 시에 주의해서 사용해야 한다.

4. 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 데이터베이스를 조회할 필요가 없으므로

em.getReference()를 호출해도 프록시가 아닌 실제 엔티티를 반환한다.

5. 초기화는 영속성 컨텍스트의 도움을 받아야 가능하다. 따라서 준영속 상태의 프록시를 초기화하면

문제가 발생한다.


### 즉시 로딩과 지연 로딩
프록시 객체는 주로 연관된 엔티티를 지연 로딩할 때 사용한다.

1) 즉시 로딩 : 엔티티를 조회할때 연관된 엔티티도 함께 조회한다

2) 지연 로딩 : 연관된 엔티티를 실제 사용할 때 조회한다


#### 즉시 로딩
즉시 로딩은 @ManyToOne(fetch=FetchType.EAGER) 를 사용하면 된다.

```java
Member member = em.find(Member.class, "member1");
Team team = member.getTeam();
``` 

위 쿼리는 즉시로딩이므로 em.find(Member.class, "member1")로 회원을 조회하는 순간 

팀도 함께 조회한다. 이 때 즉시로딩을 최적화하기 위해 가능하면 조인쿼리를 사용한다.

#### 지연 로딩
지연 로딩은 @ManyToOne(fetch=FetchType.LAZY)를 사용하면 된다.
```java
Member member = em.find(Member.class, "member1");
Team team = member.getTeam(); // 객체 그래프 탐색
team.getName() // 팀 객체 실제 사용
```
위 쿼리는 지연 로딩이므로 em.find(Member.class, "member1")를 호출하면 회원만 조회하고

팀은 조회하지 않는다. 대신에 team 멤버 변수에 프록시 객체를 넣어둔다.

```java
Team team = member.getTeam(); //프록시 객체
```

반환된 팀 객체는 프록시 객체다. 이 프록시 객체는 실제 사용될 때까지 데이터 로딩을 미룬다.

그래서 지연 로딩이라 한다.

**조회 대상이 영속성 컨텍스트에 이미 있으면 프록시 객체를 사용할 이유가 없다. 따라서 프록시가 아닌**

**실제 객체를 사용한다**

### 즉시 로딩, 지연 로딩 정리

1) 지연 로딩 : 연관된 엔티티를 프록시로 조회한다. 프록시를 실제 사용할 때 초기화화면서 데이터베이스를 조회한다.

2) 즉시 로딩 : 연관된 엔티티를 즉시 조회한다. 하이버네이트는 가능하면 SQL 조인을 사용해서 한 번에 조회한다.

---

### 영속성 전이 : CASCADE

특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들고 싶으면 영속성 전이

기능을 사용하면 된다.

**JPA는 엔티티를 저장할 때 연관된 모든 엔티티는 영속상태야 한다**

#### 영속성 전이 : 저장

영속성 전이를 활성화화는 CASCADE옵션을 적용해보자.

```java
@Entity
public class Parent{
    @OneToMany(mappedBy = "parent", cascade =CascadeType.PERSIST)
    private List<Child> children = new ArrayList<Child>();
}
```

이 옵션을 적용하면 간편하게 부모와 자식 엔티티를 한 번에 영속화할 수 있다.

```java
  Child child1 = new Child();
  Child child2 = new Child();     

  Parent parent = new Parent();
  child1.setParent(parent); //연관관계 추가
  child2.setParent(parent); //연관관계 추가
  parent.getChildren().add(child1);
  parent.getChildren().add(child2);

  em.persist(parent);

```
