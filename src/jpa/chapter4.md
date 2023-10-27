## 엔티티 매핑

---

### @Entity
JPA를 사용해서 테이블과 매핑할 클래스는 @Entity 어노테이션을 필수로 붙여야 한다.
name은 같은 엔티티 클래스가 있다면 이름을 지정해서 충돌하지 않도록 해야 한다.

**주의사항**
- 기본 생성자는 필수다
- final 클래스, enum, interface, inner 클래스에는 사용할 수 없다
- 저장할 필드에 final을 사용하면 안 된다.

JPA가 엔티티 객체를 생성할 때 기본 생성자를 사용하므로 이 생성자는 반드시 있어야 한다.

### @Table
엔티티와 매핑할 테이블을 지정한다. 
**hibernate.hbm2ddl.auto 속성**
- create : 기존 테이블을 삭제하고 새로 생성한다 (DROP + CREATE)
- create-drop : create 속성에 추가로 애플리케이션을 종료할 때 생성한 DDL을 제거한다. (DROP + CREATE + DROP)
- update : 데이터베이스 테이블과 엔티티 매핑정보를 비교해서 변경사항만 수정한다
- validate : 데이터베이스 테이블과 엔티티 매핑정보를 비교해서 차이가 있으면 경고를 남기고 애플리케이션을 실행하지 않는다. 이 설정은 DDL을 수정하지 않는다.
- none : 자동 생성 기능을 사용하지 않으려면 hibernate.hbm2ddl.auto 속성 자체를 삭제하거나 유효하지 않은 옵션 값을 주면 된다.

**hibernate.hbm2ddl.auto 주의사항**
운영서버에서 create, create-drop, update처럼 DDL을 수정하는 옵션은 절대 사용하면 안된다. 개발서버나 개발 단계에서만 사용해야 한다.
`개발초기` : create 또는 update
`테스트서버` : update 또는 validate
`스테이징과 운영서버` : validate 또는 none

### 기본키 매핑
JPA가 제공하는 데이터베이스 기본 키 생성 전략
- 직접할당 : 기본 키를 애플리케이션에서 직접 할당
- 자동생성 : 대리 키 사용 방식
    IDENTITY : 기본 키 생성을 데이터베이스에 위임한다.
    SEQUENCE : 데이터베이스 시퀀스를 사용해서 기본 키를 할당한다.
    TABLE : 키 생성 테이블을 사용한다. 이 전략은 키 생성용 테이블을 하나 만들어두고 마치 시퀀스처럼 사용하는 방식이다. 이 전략은 모든 데이터베이스에서 사용가능
- 기본 키를 직접할당하려면 @Id만 사용하면 되고, 자동생성 전략을 사용하려면 @Id에 @GeneratedValue를 추가하고 원하는 키 생성 전략을 선택하면 된다.

**주의사항**
키 생성 전략을 사용하려면 persistence.xml에 hibernate.id.new_generator_mappings=true 속성을 반드시 추가해야 한다.

### 기본 키 직접 할당 전략
@Id 적용이 가능한 자바 타입은 다음과 같다
- 자바 기본형
- 자바 래퍼형
- String
- java.util.Date
- java.sql.Date
- java.math.BigDecimal
- java.math.BigInteger

기본 키 직접 할당 전략은 em.persist()로 엔티티를 저장하기 전에 애플리케이션에서 기본 키를 직접
할당하는 방법
```java
Board board = new Board();
board.setId("id1");
em.persist(board);
```

### IDENTITY 전략
기본 키 생성을 데이터베이스에 위임하는 전략. 이 전략은 AUTO_INCREMENT를 사용한 예제처럼 데이터베이스에
값을 저장하고 나서야 기본 키 값을 구할 수 있을때 사용한다. 이 전략을 사용하면 JPA는 기본 키 값을
얻어오기 위해 데이터베이스를 추가로 조회한다.

**주의**
엔티티가 영속상태가 되려면 식별자가 반드시 필요하다. 그런데 IDENTITY 식별자 생성 전략을 엔티티를 데이터베이스에 저장해야 
식별자를 구할 수 있으므로 em.persist()를 호출하는 즉시 INSERT SQL이 데이터베이스에 전달된다. 따라서 이 전략은
트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다.

### SEQUENCE 전략
데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트이다.
SEQUENCE 전략은 이 시퀀스를 사용해서 기본 키를 생성한다.
```
CREATE TABLE BOARD(
 ID BIGINT NOT NULL PRIMARY KEY,
 DATA VARCHAR(255)
)

//시퀀스 생성
CREATE SEQUENCE BOARD_SEQ START WITH 1 INCREMENT BY 1;
```

```java
@Entity
@SequenceGenerator(
    name = "BOARD_SEQ_GENERATOR",
    sequenceName = "BOARD_SEQ",
    initialValue = 1, allocationSize = 1)
public class Board(

 @Id
 @GeneratedValue(strategy = GenerationType.SEQUENCE,
                 generator = "BOARD_SEQ_GENERATOR")
 private Long id;
```
)

사용할 시퀀스를 매핑해야 한다.

SEQUENCE 전략은 em.persist()를 호출할 때 먼저 데이터베이스 시퀀스를 사용해서 식별자를 조회한다. 그리고 조회한 식별자를 엔티티에 할당한 후에
엔티티를 영속성 컨텍스트에 저장한다. 이 후 트랜잭션을 커밋해서 플러시가 일어나면 엔티티를 데이터베이스에 저장한다.

반대로 이전에 설명했던 IDENTITY 전략은 먼저 엔티티를 데이터베이스에 저장한 후에 식별자를 조회해서 엔티티의 식별자에 할당한다.

SequenceGenerator 속성 중 allocationSize의 기본값은 50인데 이것은 최적화 때문이다.

### SEQUENCE 전략과 최적화
SEQUENCE 전략과 데이터베이스 시퀀스를 통해 식별자를 조회하는 추가 작업이 필요. 따라서 데이터베이스와 2번 통신한다.

1) 식별자를 구하려고 데이터베이스 시퀀스를 조회
    SELECT BORAD_SEQ.NEXTVAL FROM DUAL
2) 조회한 시퀀스를 기본 키 값으로 사용해 데이터베이스에 저장한다
    INSERT INTO BOARD...

JPA는 시퀀스에 접근하는 횟수를 줄이기 위해 @SequenceGenerator.allocationSize를 사용한다. 여기 설정한 값만큼
한 번에 시퀀스 값을 증가시키고 나서 그만큼 메모리에 시퀀스 값을 할당한다. 이 최적화 방법은 시퀀스 값을 선점하므로
여러 JVM이 동시에 동작해도 기본키 값이 충돌하지 않는 장점이 있다. 반면에 데이터베이스에 직접 접근해서 데이터를
등록할 때 시퀀스 값이 한번에 많이 증가한다는 점을 염두에 둬야 한다.

### TABLE 전략
키 생성 전용 테이블을 하나 만들고 여기에 이름과 값으로 사용할 컬럼을 만들어 데이터베이스 
시퀀스를 흉내내는 전략이다. 이 전략은 테이블을 사용하므로 모든 데이트베이스에 적용할 수 있다.

```
CREATE TABLE MY_SEQUENCES (
    sequence_name varchar(255) not null,
    next_val bigint,
    primary key (  sequence_name )
)
```

```java
@Entity
@TableGenerator(
  name = "BOARD_SEQ_GENERATOR",
  table = "MY_SEQUENCES",
  pkColumnValue = "BOARD_SEQ", allocationSize = 1)
public class Board (
 @Id
 @GeneratedValue(strategy = GenerationType.TABLE,
           generator = "BOARD_SEQ_GENERATOR")
  private Long id;
)
```

### 기본 키 매핑 정리
영속성 컨텍스트는 엔티티를 식별자 값으로 구분하므로 엔티티를 영속 상태로 만들려면 식별자 값이 반드시 있어야 한다.
em.persist()를 호출한 직후에 발생하는 일을 식별자 할당 전략별로 정리하면 아래와 같다.

1) 직접 할당 : em.persist()를 호출하기 전에 애플리케이션에서 직접 식별자 값을 할당. 만약 식별자 값이 없으면 예외가 발생
2) SEQUENCE : 데이터베이스 시퀀스에서 식별자 값을 획득한 후 영속성 컨텍스트에 저장한다.
3) TABLE : 데이터베이스 시퀀스 생성용 테이블에서 식별자 값을 획득한 후 영속성 컨텍스트에 저장한다
4) IDENTITY : 데이터베이스에 엔티티를 저장해서 식별자 값을 획득한 후 영속성 컨텍스트에 저장한다 (IDENTITY 전략은 테이블에 데이터를 저장해야 식별자 값을 획득할 수 있다)

** 권장 식별자 선택 전략 **
DB 기본키는 다음 3가지 조건을 모두 만족해야 함
1) NOT NULL
2) 유일해야 한다
3) 변해선 안된다

기본 키 선택 전략 2가지
1) 자연 키
- 비즈니스에 의미가 있는키 예) 주민등록번호, 이메일, 전화번호
2) 대리 키
- 비즈니스와 관련 없는 임의로 만들어진 키. 대체 키로도 불린다.
- 오라클 시퀀스, AUTO_INCREMENT, 키 생성 테이블

**자연키 보다는 대리키를 권장한다.**
