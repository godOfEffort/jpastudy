## 연관관계 매핑 기초

1. 객체의 참조와 테이블의 외래 키를 매핑

### 용어 이해
1. 방향 : 단방향, 양방향
2. 다중성 : 다대일, 일대다, 일대일, 다대다 이해
3. 연관관계의 주인 : 객체 양방향 연관관계는 관리 주인이 필요

### 연관관계
**객체를 테이블에 맞추어 데이터 중심으로 모델링하면, 협력 관계를 만들 수 없다.**

**테이블은 외래 키로 조인**을 사용해서 연관된 테이블을 찾는다.

**객체는 참조**를 통해서 연관된 객체를 찾는다.

테이블과 객체 사이에는 이런 큰 간격이 있다.

### 단방향 연관관계

```java
package hellojpa;

import javax.persistence.*;

@Entity
public class Member {

    @Id
    @GeneratedValue
    @Column(name="MEMBER_ID")
    private Long id;

    @Column(name ="USERNAME")
    private String name;

//    @Column(name ="TEAM_ID")
//    private Long teamId;

    @ManyToOne  //어떤 관계인지
    @JoinColumn(name ="TEAM_ID") // 매핑하는 컬럼이 무엇인지
    private Team team;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Team getTeam() {
        return team;
    }

    public void setTeam(Team team) {
        this.team = team;
    }
}

```

```java
package hellojpa;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

@Entity
public class Team {

    @Id
    @GeneratedValue
    @Column(name ="TEAM_ID")
    private Long id;
    private String name;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
public class JpaMain {
    public static void main(String[] args) {
        final EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");

        final EntityManager em = emf.createEntityManager();

        final EntityTransaction tx = em.getTransaction();

        tx.begin();

        try {
            final Team team = new Team();
            team.setName("TeamA");
            em.persist(team);

            final Member member = new Member();
            member.setName("member1");
            member.setTeam(team);
            em.persist(member);

            em.flush();
            em.clear();

            final Member findMember = em.find(Member.class, member.getId());

            final Team findTean = findMember.getTeam();
            System.out.println("findTean = " + findTean.getName());

            tx.commit();
        } catch (Exception e) {
            tx.rollback();
        } finally {
            em.close();
        }

        emf.close();
    }
}
```


### 양방향 연관관계와 연관관계의 주인
테이블의 연관관계에서는 외래키 만으로도 양방향 연관관계를 이룰 수 있으나

객체 연관관계에서는 양쪽에 세팅을 해줘야 한다.

**객체 연관관계 = 2개**

회원 -> 팀 연관관계 1개 (단방향)
팀 -> 회원 연관관계 1개 (단방향)


**테이블 연관관계 = 1개**

회원 <-> 팀의 연관관계 1개 (양방향)
