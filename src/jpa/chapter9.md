## 값 타입

JPA의 데이터 타입

1. 엔티티 타입
    - @Entity로 정의하는 객체

2. 값 타입
    - int, Integer, String처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체를 의미
    
    값 타입은 다시 아래 3개로 나뉜다
    
    1) 기본값 타입
        - 자바 기본 타입(int, double)
        - 래퍼 클래스(Integer)
        - String
        
    2) 임베디드 타입(복합 값 타입)
    
    3) 컬렉션 값 타입

---

### 기본값 타입

1. 자바 기본 타입(int, double)

2. 래퍼 클래스(Integer)

3. String

---

### 임베디드 타입(복합 값 타입)

새로운 값 타입을 직업 정의해서 사용

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Embedded Period workPeriod;
    @Embedded Address homeAddress;

}
```

```java
@Entity
public class Period {
    
    @Temporal(TemporalType.DATE)
    Date startDate;

    @Temporal(TemporalType.DATE)
    Date endDate;
    
    public boolean isWork(Date date) {
        //값 타입을 위한 메소드 정의 가능
    }
}
```

```java
@Entity
public class Address {
    
    @Column(name="city")
    private String city;
    private String street;
    private String zipcode;
}
```

**임베디드 타입은 기본 생성자가 필수다**

### 임베디드 타입과 테이블 매핑

임베디드 타입은 엔티티의 값일 뿐이다. 따라서 값이 속한 엔티이의 테이블에 매핑한다.

예제에서 임베디드 타입을 사용하기 전과 후에 매핑하는 테이블은 같다.


### 임베디드 타입과 연관관계

임베디드 타입은 값 타입을 포함하거나 엔티티를 참조할 수 있다.


### @AttributeOverride 속성 재정의

임베디드 타입에 정의한 매핑정보를 재정의하려면 엔티티에 @AttributeOverride를 사용하면 된다.

```java
@Entity
public class Address {
    
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @Embedded
    Address homeAddress;

    @Embedded
    Address companyAddress;  //테이블에 매핑하는 컬럼명이 중복
}
```

위 예제는 테이블에 매핑하는 컬럼명이 중복되므로 @AttributeOverride를 사용해서 매핑정보를 재정의해야 한다.

```java
@Entity
public class Address {
    
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @Embedded
    Address homeAddress;

    @Embedded
    @AttributeOverrides({
          @AttributeOverride(name="city", column=@Column(name ="COMPANY_CITY")),
          @AttributeOverride(name="street", column=@Column(name ="COMPANY_STREET")),
          @AttributeOverride(name="zipcode", column=@Column(name ="COMPANY_ZIPCODE")),     
    })
    Address companyAddress;
}
```

### 임베디드 타입과 null

임베디드 타입이 null이면 매핑한 컬럼 값은 모두 null이 된다.

---

## 값 타입과 불변객체

### 값 타입 공유 참조

```java
    member1.setHomeAddress(new Address("oldCity"));
    Address address = member1.getHomeAddress();

    address.setCity("NewCity"); //회원1의 address 값을 공유해서 사용
    member2.setHomeAddress(address);
```

회원 2의 주소만 "NewCity"로 변경되길 기대했지만 회원1의 주소도 "NewCity"로 변경되어 버린다.

---

### 값 타입 복사

값 타입의 실제 인스턴스 값을 공유하는 것은 위험하다. 대신에 값(인스턴스)을 복사해서 사용해야 한다.

```java
    member1.setHomeAddress(new Address("OldCity"));
    Address address = member1.getHomeAddress();

    //회원1의 address 값을 복사해서 새로운 newAddress 값을 생성
    Address newAddress = address.clone();

    newAddress.setCity("NewCity"); //회원1의 address 값을 공유해서 사용
    member2.setHomeAddress(newAddress);
```

객체를 대입할 때마다 인스턴스를 복사해서 대입하면 공유 참조를 피할 수 있다. 

**문제는 복사하지 않고 원본의 참조 값을 직접 넘기는 것을 막을 방법이 없다** 

**즉, 객체의 공유 참조는 피할 수없다. 따라서 근본적인 해결책은 객체의 값을 수정하지 못하게 막는것이다.**

---

### 불변 객체

객체를 불변하게 만들면 값을 수정할 수 없으므로 부작용을 원천 차단할 수 있다. 따라서 값 타입은

될 수 있으면 불변 객체로 설계해야 한다.

한 번 만들면 절대 변경할 수 없는 객체를 불변 객체라 한다. 불변 객체의 값은 조회는 할 수 있지만 수정할 수 없다.

불변 객체도 객체기 때문에 인스턴스의 참조 값 공유는 피할 수 없지만 참조 값을 공유해도 인스턴스의 값을 수정할 수 없으므로

부작용이 발생하지 않는다.

```java
@Embeddable
public class Address {   
    private String city;
   
    protected Address () {}
  
    public Address(String city) {
       this.city = city;
    }
    
    public String getCity() {
        return city;
    }
    
    //setter를 만들지 않는다
}
```


```java
   Address address = member1.getHomeAddress();
   Address newAddress = new Address(address.getCity());
   member2.setHomeAddress(newAddress);
```

이제 Address는 불변객체다. 불변이라는 작은 제약으로 부작용을 막을 수 있다.


### 값 타입의 비교

1. 동일성 비교 : 인스턴스의 참조 값을 비교, == 사용

2. 동등성 비교 : 인스턴스의 값을 비교, equals() 사용

Address 값 타입을 a == b 로 동일성 비교하면 둘은 서로 다른 인스턴스이므로 결과는 거짓이다.

하지만 이것은 기대하는 결과가 아니다. 값 타입은 비록 인스턴스가 달라도 그 안에 값이 같으면 같은 것으로 봐야 한다.

값 타입은 비록 인스턴스가 달라도 그 안에 값이 같으면 같은 것으로 봐야 한다. 

따라서 값 타입을 비교할 떄는 a.equals(b)를 사용해서 동등성 비교를 해야 한다.

물론 Address의 equals() 메소드를 재정의해야 한다.

**자바에서 equals()를 재정의하면 hashCode()도 재정의하는 것이 안전하다. 그렇지 않으면 해시를 사용하는 컬렉션이 정상동작하지 않는다.**

### 값 타입 컬렉션

값 타입을 하나 이상 저장하려면 컬렉션에 보관하여 @ElementCollection, @CollectionTable 어노테이션을 사용하면 된다.

```java
@Entity
public class Member {
    @Id @GeneratedValue
    private Long id;    

    @Embedded Address homeAddress;
    
    @ElementCollection
    @CollectionTable(name ="FAVORITE_FOODS",
        joinColumns = @JoinColumn(name ="MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private Set<String> favoriteFoods = new HashSet<String>();   

    @ElementCollection
    @CollectionTable(name ="ADDRESS",
        joinColumns = @JoinColumn(name ="MEMBER_ID"))
    @Column(name = "FOOD_NAME")
    private List<String> addressHistory = new ArrayList<Address>();   
}

@Embeddable
public class Address {
    @Column
    private String city;
    private String street;
    private String zipcode;
}
``` 

favoriteFoods는 기본값 타입인 String을 컬렉션으로 가진다. 

데이터베이스 테이블로 매핑해야 되는데 관계형 데이터베이스의 테이블은 컬럼 안에 컬렉션을 포함할 수 없다.

따라서 별도의 테이블을 추가하고 @CollectionTable를 사용해서 추가한 테이블을 매핑해야 한다.


### 값 타입 컬렉션 사용

```java
Member member = new Member();

//임베디드 값 타입
member.setHomeAddress(new Address("1", "몽동해수욕장", "660-23"));

//기본값 타입 컬렉션
member.getFavoriteFoods().add("짬봉");
member.getFavoriteFoods().add("짜장");
member.getFavoriteFoods().add("탕수육");

//임베디드 값 타입 컬렉션
member.getAddressHistory().add(new Address("서울", "강남", "123-123"));
member.getAddressHistory().add(new Address("서울", "강북", "000-000"));

em.persist(member);
```

**값 타입 컬렉션은 영속성 전이 + 고아 객체 제거 기능을 필수로 가진다**

---

### 값 타입 컬렉션의 제약사항

1. 값 타입은 엔티티와 다르게 식별자 개념이 없다.

2. 값은 변경하면 추적이 어렵다.

3. 값 타입 컬렉션에 변경 사항이 발생하면, 주인 엔티티와 연관된 모든 데이터를 삭제하고, 

   값 타입 컬렉션에 있는 현재 값을 모두 다시 저장한다.

4. 값 타입 컬렉션을 매핑하는 테이블은 모든 컬럼을 묶어서 기본키를 구성해야 함: null 입력도 안되고, 중복 저장도 안된다.

### 값 타입 컬렉션 대안

1. 실무에서는 상황에 따라 값 타입 컬렉션 대신에 일대다 관계를 고려

2. 일대다 관계를 위한 엔티티를 만들고, 여기에서 값 타입을 사용해야 한다.

**값 타입은 정말 값 타입이라 판단될 때만 사용**

**엔티티와 값 타입을 혼동해서 엔티티를 값 타입으로 만들면 안됨**

**식별자가 필요하고, 지속해서 값을 추적, 변경해야 한다면 그것은 값 타입이 아닌 엔티티**
