## 연관관계 매핑시 고려사항 3가지
1. 다중성
2. 단방향, 양방향
3. 연관관계의 주인

### 다중성
1. 다대일: @ManyToOne
2. 일대다: @OneToMany
3. 일대일: @OneToOne
4. 다대다: @ManyToMany


### 단방향, 양방향
#### 테이블
1. 외래 키 하나로 양쪽 조인 가능
2. 사실 방향이라는 개념이 없음
#### 객체
1. 참조용 필드가 있는 쪽으로만 참조 가능
2. 한쪽만 참조하면 단방향
3. 양쪽이 서로 참조하면 양방향


##### 연관관계의 주인
1. 테이블은 외래 키 하나로 두 테이블이 연관관계를 맺음
2. 객체 양방향 관계는 A->B, B->A 처럼 참조가 2군데
3. 객체 양방향 관계는 참조가 2군데 있음. 둘중 테이블의 외래 키를 관리할 곳을 지정해야함
4. 연관관계의 주인: 외래 키를 관리하는 참조
5. 주인의 반대편: 외래 키에 영향을 주지 않음, 단순 조회만 가능


####다대일
데이터베이스 테이블의 일, 다 관계에서 외래 키는 **항상 다쪽에 있다**
따라서 객체 양방향 관계에서 연관관계의 주인은 항상 다쪽이다.

#### 1. 다대일 단방향 정리

```java
@Entity
public class Member {
    @Id @GeneratedValue
    @Column(name = "MEMBER_ID")
    private Long id;

    private String username;

    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;

}
```

```java
@Entity
public class Team {
    
    @Id @GeneratedValue
    @Column(name = "TEAM_ID")
    private Long id;
 
    private String name;
}
```

회원은 Member.team으로 팀 엔티티를 참조할 수 있지만 반대로 팀에는 회원을 참조하는 필드가 없다.

따라서 회원과 팀은 다대일 단방향 연관관계이다.

@JoinColumn(name = "TEAM_ID")를 사용해서 Member.team 필드를 TEAM_ID 외래키와 매핑했다.

따라서 Member.team 필드로 회원 테이블의 TEAM_ID 외래키를 관리한다.

#### 2. 다대일 양방향
```java
    @Entity
    public class Member {
        @Id @GeneratedValue
        @Column(name = "MEMBER_ID")
        private Long id;
    
        private String username;
    
        @ManyToOne
        @JoinColumn(name = "TEAM_ID")
        private Team team;
     
        public void setTeam(Team team) {
            this.team = team;
          
            if(!team.getMembers().contains(this)) {
                team.getMembers().add(this);
            }
        }
    }
```

```java
    @Entity
    public class Team {
        @Id @GeneratedValue
        @Column(name = "TEAM_ID")
        private Long id;
    
        private String username;

        @OneToMany(mappedBy = "team")
        private List<Member> members = new ArrayList<Member>();
    
        public void addMember (Member member) {
            this.members.add(member);
            if(member.getTeam() != this) {
               member.setTeam(this);
            }
        }
    }
```

**양방향은 외래 키가 있는 쪽이 연관관계의 주인이다**
다쪽인 Member 테이블이 외래 키를 가지고 있으므로 Member.team이 연관관계의 주인이다. 

JPA는 외래 키를 관리할 때 연관관계의 주인만 사용한다.

**양방향 관계는 항상 서로를 참조해야 한다**
양방향 관계는 항상 서로를 참조해야 한다. 어느 한 쪽만 참조하면 양방향 연관관계가 성립하지 않는다.

그래서 편의 메소드 setTeam(), addMember() 등의 메소드가 필요하다.

이 떄 무한루프에 빠지지 않도록 체크하는 로직이 필요하다. 

####일대다
일대다 관계는 엔티티를 하나 이상 참조할 수 있으므로 자바 컬렉션인 Collection, List, Set, Map 중에 하나를

사용해야 한다.

##### 3.일대다 단방향
하나의 팀은 여러 회원을 참조할 수 있는데 이런 관계를 일대다 관계라 한다. 그리고 팀은

회원들을 참조하지만 반대로 회원은 팀을 참조하지 않으면 둘의 관계는 단방향이다.


```java
    @Entity
    public class Member {
        @Id @GeneratedValue
        @Column(name = "MEMBER_ID")
        private Long id;
    
        private String username;    
    }
```

```java
    @Entity
    public class Team {
        @Id @GeneratedValue
        @Column(name = "TEAM_ID")
        private Long id;
    
        private String name;

        @OneToMany
        @JoinColumn(name = "TEAM_ID") // MEMBER 테이블의 TEAM_ID(FK)
        private List<Member> members = new ArrayList<Member>();    
    }
```

Team.members로 회원 테이블의 TEAM_ID 외래키를 관리한다. 보통 자신이 매핑한 테이블의 외래 키를

관리하는데, 이 매핑은 반대쪽 테이블에 있는 외래 키를 관리한다.

그럴 수 밖에 없는 것이 일대다 관계에서 외래 키는 항상 다쪽 테이블에 있다.

하지만 다 쪽인 Member 엔티티에는 외래 키를 매핑할 수 있는 참조 필드가 없다. 

대신에 반대쪽인 Team 엔티티에만 참조 필드인 members가 있다. 따라서 반대편 테이블의

외래키를 관리하는 특이한 모습이 나타난다.

**일대다 단방향 관계를 매핑할 때는 @JoinColumn을 명시해야 한다. 그렇지 않으면 JPA는 연결 테이블을
중간에 두고 연관관계를 관리하는 조인 테이블 전략을 기본으로 사용해서 매핑한다.**

#### 일대다 단방향 매핑의 단점
일대다 단방향 매핑의 단점은 매핑한 객체가 관리하는 외래 키가 다른 테이블에 있다는 점이다.

**일대다 단방향 매핑보다는 다대일 양방향 매핑을 사용하자**


##### 4.일대다 양방향

일대다 양방향 매핑은 존재하지 않는다. 대신 다대일 양방향 매핑을 사용해야 한다.

더 정확히 말하면 양방향 매핑에서 @OneToMany는 연관관계의 주인이 될 수 없다. 

왜냐하면 관계형 데이터베이스의 특성상 일대다, 다대일 관계는 항상 다 쪽에 외래 키가 있다.

따라서 @OneToMany @ManyToOne 둘 중에 연관관계의 주인은 항상 다 쪽인 @ManyToOne을 사용한 곳이다.

이런 이유로 @ManyToOne 에는 mappedBy 속성이 없다.

그렇다고 양방향 매핑이 완전히 불가능한 것은 아니다 역시 일대다 단방향 매핑이 가지는 단점을 그대로 가진다.

##### 5. 일대일 
일대일 관계는 양쪽이 서로 하나의 관계만 가진다. 

일대일 관계는 다음과 같은 특징이 있다.

**일대일 관계는 그 반대도 일대일 관계다**

**일대일 관계는 주 테이블이나 대상 테이블 둘 중 어느 곳이나 외래 키를 가질 수 있다.**

테이블은 주 테이블이든 대상 테이블이든 외래 키 하나만 있으면 양쪽으로 조회할 수 있다.

따라서 일대일 관계는 주 테이블이나 대상 테이블 중에 누가 외래 키를 가질지 선택해야 한다.

1) 주 테이블에 외래 키

 ```java
 //일대일 주 테이블에 외래 키, 단방향 예제 코드
    @Entity
    public class Member {
        @Id @GeneratedValue
        @Column(name = "MEMBER_ID")
        private Long id;
    
        private String username;
        
        @OneToOne
        @JoinColumn(name ="LOCKER_ID")
        private Locker locker;
    
    }
 ```

```java
    @Entity
    public class Locker {
        @Id @GeneratedValue
        @Column(name = "LOCKER_ID")
        private Long id;
    
        private String name;
    
    }
```

```java
 //일대일 주 테이블에 외래 키, 양방향 예제 코드
    @Entity
    public class Member {
        @Id @GeneratedValue
        @Column(name = "MEMBER_ID")
        private Long id;
    
        private String username;
        
        @OneToOne
        @JoinColumn(name ="LOCKER_ID")
        private Locker locker;
    
    }
 ```

```java
    @Entity
    public class Locker {
        @Id @GeneratedValue
        @Column(name = "LOCKER_ID")
        private Long id;
    
        private String name;
        
        @OneToOne(mappedBy = "locker")
        private Membmer member;
    }
```
양방향이므로 연관관계의 주인을 정해야 한다. MEMBER 테이블이 외래 키를 가지고 있으므로 

Member 엔티티에 있는 Member.locker가 연관관계의 주인이다. 따라서 반대 매핑인 사물함의 

Locker.member는 mappedBy를 선언해서 연관관계의 주인이 아니라고 설정했다. 
 
2) 대상 테이블에 외래 키

```java
 //일대일 대상 테이블에 외래 키, 양방향 예제 코드
    @Entity
    public class Member {
        @Id @GeneratedValue
        @Column(name = "MEMBER_ID")
        private Long id;
    
        private String username;
        
        @OneToOne(mappedBy="member")
        private Locker locker;
    
    }
 ```

```java
    @Entity
    public class Locker {
        @Id @GeneratedValue
        @Column(name = "LOCKER_ID")
        private Long id;
    
        private String name;
        
        @OneToOne
        @JoinColumn(name = "MEMBER_ID")
        private Membmer member;
    }
```
주 엔티티인 Member 엔티티 대신에 대상 엔티티인 Locker를 연관관계의 주인으로 만들어서

LOCKER 테이블의 외래 키를 관리하도록 했다.


##### 6. 다대다
관계형 데이터베이스는 정규화된 테이블 2개로 다대다 관계를 표현할 수 없어 중간에 연결 테이블을 추가해야 한다.

**그런데 객체는 테이블과 다르게 객체 2개로 다대다 관계를 만들 수 있다**
1) 다대다 단방향

```java
    // 다대다 단방향 회원
    @Entity
    public class Member {
        @Id @Column(name = "MEMBER_ID")
        private Long id;
    
        private String username;
        
        @ManyToMany
        @JoinTable(name = "MEMBER_PRODUCT",
                   joinColumns = @JoinClumn(name= "MEMBER_ID"),
                   inverseJoinColumns = @JoinColumn(name ="PRODUCT_ID"))
        private List<Product> products = new ArrayList<Product>();
    
    }
``` 

````java
    @Entity
    public class Product {
        
        @Id @Column(name = "PRODUCT_ID")
        private String id;
        
        private String name;
    }   
````

**@JoinTable 속성**

@JoinTable.joinColumns : 현재 방향인 회원과 매핑할 조인 컬럼 정보를 지정한다. 

@JoinTable.inverseJoinColumns : 반대 방향인 상품과 매핑할 조인 컬럼 정보를 지정한다.


2) 다대다 양방향
다대다 매핑이므로 역방향도 @ManyToMany를 사용한다. 그리고 양쪽 중 원하는 곳에 mappedBy로 연관관계의

주인을 지정한다 (물론 mappedBy가 없는 곳이 연관관계의 주인이다)