## 상속관계 매핑


### 상속관계 매핑
관계형 데이터베이스는 상속 관계가 없다.
슈퍼타입 서브타입 관계라는 모델링 기법이 객체 상속과 유사하다.
**상속관계 매핑** : 객체의 상속구조와 DB의 슈퍼타입 서브타입 관계를 매핑

### 슈퍼타입 서브타입 논리 모델을 실제 물리 모델로 구현하는 방법
1. 각각 테이블로 변환 -> 조인 전략
2. 통합 테이블로 변환 -> 단일 테이블 전략
3. 서브타입 테이블로 변환 -> 구현 클래스마다 테이블 전략

### 주요 어노테이션
1. @Inheritance(strategy=InheritanceType.XXX)
    - JOINED : 조인전략
    - SINGLE_TABLE : 단일 테이블 전략
    - TABLE_PER_CLASS : 구현 클래스마다 테이블 전략
2. @DiscriminatorColumn(name=“DTYPE”)
3. @DiscriminatorValue(“XXX”)

### 조인전략
1. 장점 
    - 테이블 정규화
    - 외래 키 참조 무결성 제약조건 활용가능
    - 저장공간 효율화
    
2. 단점
    - 조회시 조인을 많이 사용, 성능 저하
    - 조회 쿼리가 복잡함
    - 데이터 저장시 INSERT SQL 2번 호출
---

### 단일 테이블 전략
1. 장점 
    - 조인이 필요 없으므로 일반적으로 조회 성능이 빠름
    - 조회 쿼리가 단순함
    
2. 단점
    - 자식 엔티티가 매핑한 컬럼은 모두 null 허용
    - 단일 테이블에 모든 것을 저장하므로 테이블이 커질 수 있다. 상황에 따라서 조회 성능이 오히려 느려질 수 있다.   

---    
### 구현 클래스마다 테이블 전략 
**이 전략은 사용하지 않는 것이 좋다**
1. 장점 
    - 서브 타입을 명확하게 구분해서 처리할 때 효과적
    - not null 제약 조건 사용 가능
    
2. 단점
    - 여러 자식 테이블을 함꼐 조회할 때 성능이 느림(UNION SQL 필요)
    - 자식 테이블을 통합해서 쿼리하기 어려움     

---
### @MappedSuperclass
공통 매핑 정보가 필요할 떄 사용하는 기능이다

1. 상속관계 매핑이 아니다.
2. 엔티티도 아니며 테이블과 매핑되는 것도 아니다
3. 부모 클래스를 상속받는 **자식 클래스에 매핑 정보만 제공**하는 것이다.
4. 조회, 검색이 불가능 하다
5. 직접 생성해서 사용할 일이 없으므로 추상클래스로 생성하는 것이 좋다.

@MappedSuperclass는 테이블과 관계 없고, 단순히 엔티티가 공통으로 사용하는 매핑 정보를 모으는 역할이다.

주로 등록일, 수정일, 등록자, 수정자 같은 전체 엔티티에서 공통으로 적용하는 정보를 모을 때 사용한다

@Entity 클래스는 엔티티나 @MappedSuperclass로 지정한 클래스만 상속 가능하다.

---
### 복합 키와 식별 관계 매핑

1. 식별관계

부모 테이블의 기본 키를 내려 받아서 **자식 테이블의 기본 키 + 외래 키**로 사용하는 관계

2. 비식별관계

부모 테이블의 기본 키를 내려 받아서 **자식 테이블의 외래 키**로만 사용하는 관계

   - 1) 필수적 비식별 관계 : 외래 키에 NULL을 허용하지 않는다. 연관관계를 필수적으로 맺어야 한다.
   - 2) 선택적 비식별 관계 : 외래 키에 NULL을 허용한다. 연관관계를 맺을지 말지 선택할 수 있다.
   
   
3. 복합 키 : 비식별 관계 매핑
기본 키를 구성하는 컬럼이 하나면 다음처럼 단순하게 매핑한다.

```java
@Entity
public class Hello {
    @Id
    private String id;
}
```

JPA에서 식별자 둘 이상 사용하려면 별도의 식별자 클래스를 만들어야 한다.

```java
@Entity
public class Hello {
    @Id
    private String id1;
    
    @Id
    private String id2; //실행 시점에 매핑 예외 발생

}
```

JPA는 복합 키를 지원하기 위해 @IdClass와 @EmbeddedId 2가지 방법을 제공하는데

@IdClass는 관계형 데이터베이스에 가까운 방법이고 @EmbeddedId는 좀 더 객체지향에 가까운 방법이다.

#### @IdClass
```java
@Entity
@IdClass(ParentId.class)
public class Parent {
    @Id
    @Column(name = "PARENT_ID1")
    private String id1;  // ParentId.id1과 연결
    
    @Id
    @Column(name = "PARENT_ID2")
    private String id2; // ParentId.id2와 연결

}
```

```java
import java.io.Serializable;
public class ParentId implements Serializable {
    private String id1; //Parent.id1 매핑
    private String id2; //Parent.id2 매핑
    
    public ParentId() {
    
    }
    
    public ParentId(String id1, String id2) {
        this.id1 = id1;
        this.id2 = id2;
    }
    
    @Override
    public boolean equals(Object o) {}

    @Override
    public int hashCode() {}

}
```

@IdClass를 사용할 떄 식별자 클래스는 다음 조건을 만족해야 한다

1) **식별자 클래스의 속성명과 엔티티에서 사용하는 식별자의 속성명이 같아야 한다.**
2) Serializable 인터페이스를 구현해야 한다
3) equals, hashCode를 구현해야 한다
4) 기본 생성자가 있어야 한다
5) 식별자 클래스는 public이어야 한다.


```java
@Entity
public class Child {
    @Id
    private String id;
    
    @ManyToOne
    @JoinColumns( {
        @JoinColumn(name ="PARENT_ID1", 
            referencedColumnName = "PARENT_ID1"),
        @JoinColumn(name ="PARENT_ID2", 
            referencedColumnName = "PARENT_ID2")       
    })
    private Parent parent;
}
```


#### @EmbeddedId

```java
@Entity
public class Parent {
    @EmbeddedId
    private Parent id;
    
    private String name;
}
```

```java
import java.io.Serializable;
public class ParentId implements Serializable {
    @Column(name = "PARENT_ID1")
    private String id1;
    @Column(name = "PARENT_ID2")
    private String id2;

    //equals and hashCode 구현
}
```

@IdClass와는 다르게 @EmbeddedId를 적용한 식별자 클래스는 **식별자 클래스에 기본 키를 직접 매핑한다**

@EmbeddedId를 적용한 식별자 클래스는 다음 조건을 만족해야 한다.

1) @EmbeddedId 어노테이션을 붙여주어야 한다
2) Serializable 인터페이스를 구현해야 한다
3) equals, hashCode를 구현해야 한다
4) 기본 생성자가 있어야 한다
5) 식별자 클래스는 public 이어야 한다