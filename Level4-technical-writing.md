# 소프트 딜리트를 적용하는 이유와 사례

## 서론 - 삭제는 진짜 삭제일까?

우리는 댓글을 지우거나, 계정을 탈퇴하여 정보를 제거합니다.

```sql
DELETE
FROM users
WHERE id = 123;
```

위 SQL을 실행하면 데이터는 물리적으로 제거됩니다.
<br>
하지만 만약 탈퇴한 계정을 복구하고싶다면 어떻게 해야 할까요?
<br>
다음과 같은 문제가 발생합니다.

- 실수로 삭제를 했다면 복구할 수 없다.
- 누가, 언제, 왜 삭제했는지 알 수 없다.
- 삭제된 사용자의 주문 내역이 외래키 제약으로 함께 삭제된다면

실제로 페이스북은 계정 삭제 후 30일간 데이터를 보관하며, AWS는 삭제된 리소스의 이력을 CludeTrail로 추적합니다.
<br>
이외에도 수 많은 서비스는 데이터 복구에 대한 방법을 준비해두고 있습니다.
<br>
이들은 데이터를 어떻게 삭제하고 있을까요? 그리고 우리는 어떻게 데이터를 안전하게 관리할 수 있을까요?

## 대상 독자

- 백엔드 개발자(Java, Spring, JPA 환경)
- 데이터베이스 개발자
- 소프트 딜리트를 처음 접하는 주니어 개발자

## 사전 지식

- Java 기본 문법
- Spring Boot 기본 구조
- JPA 기본 개념 (Entity, Hibernate)
- SQL의 DELETE, UPDATE 쿼리 문
- 외래키(Foreign Key) 제약조건

# 목차

1. 하드 딜리트의 한계
2. 소프트 딜리트란 무엇인가?

---

## 1. 하드 딜리트의 한계

```java
userRepeository.deleteById(userId);
```

이 메서드를 실행하면 회원은 데이터베이스에서 제거됩니다.
<br>
여기서 발생할 수 있는 문제가 무엇일까요?

- 실수로 삭제한 데이터의 복구 요청 -> 불가능
- 회원 테이블과 관계를 맺고있던 주문 테이블의 외래키 제약으로 인한 회원 삭제 실패

삭제가 되어도 문제고 삭제가 안되어도 문제입니다.

하드 딜리트는 데이터를 데이터베이스에서 물리적으로 제거하여, 데이터베이스의 용량을 차지하지 않을 수 있다는 장점도 있지만, 데이터 복구요청에 대응하지 못한다는 단점이 있습니다.
<br>
실제 서비스를 운영중이라면 이러한 상황은 피해야할 것 입니다. 그러면 어떻게 하면 좋을까요?
<br>
데이터를 삭제한 것 처럼 위장하면 될 것입니다.

## 2. 소프트 딜리트란 무엇인가?

소프트 딜리트란 데이터를 물리적으로 삭제하지 않고, 삭제 여부를 표시하는 플래그 또는 타임스탬프를 사용하여 삭제처리 하는 방식입니다.

### 2.1 왜 등장하였는가?

데이터를 물리적으로 삭제하는 것에는 한계가 있었습니다.

- GDPR, 개인정보보호법 등 법적 요구사항
- 비즈니스 요구: 고객 문의 대응, 데이터 복구, 감사 추적
- 기술적 요구: 외래키 무결성 유지

위와 같은 이유로 데이터를 논리적으로 삭제하는 소프트 딜리트가 등장하였습니다.

## Spring Boot + JPA에서 소프트 딜리트 구현

소프트 딜리트 구현방법에는 크게 3가지 방법이 있습니다. 하나씩 알아보겠습니다.

### 3.1 수동 구현

레포지토리와 서비스 레이어에서 직접 구현 하는 방식입니다.

```java

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    public void softDelete() {
        this.deletedAt = LocalDateTime.now();
    }

    public boolean isDeleted() {
        return this.deletedAt != null;
    }
}
```

User 엔티티에 삭제 플래그인 deletedAt 컬럼을 추가합니다.

```java

@Service
public class UserService {

    @Transactional
    public void deleteUser(Long userId) {
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));

        user.softDelete();
        userRepository.save(user);
    }
}
```

삭제 요청을 받으면 서비스 레이어에서 User 엔티티에 구현되어있던 softDelete() 메서드를 호출하여
deletedAt 필드의 값을 삭제한 시간으로 변경합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {

    @Query("SELECT u FROM User u WHERE u.deletedAt IS NULL")
    List<User> findAllActive();

    @Query("SELECT u FROM User u WHERE u.id = :id AND u.deletedAt IS NULL")
    Optional<User> findActiveById(@Param("id") Long id);
}
```

그리고 레포지토리의 쿼리메서드에 deletedAt IS NULL 조건을 명시해줍니다.
<br>
이렇게 직접 명시해주는 이유는 데이터를 조회할 때 삭제 처리된 데이터는 조회하지 않기 위함입니다.

장점

- 명시적이고 이해하기 쉽습니다.
- 프레임 워크와 Hibernate 버전에 독립적입니다. (Spring Data JPA만 있으면 동작)
- 개발자가 삭제 조건을 세밀하게 조정할 수 있습니다. (특정 쿼리에서만 삭제 조건 제외 등)

단점

- 모든 레포지토리 메서드에 deletedAt IS NULL 조건을 수동으로 추가해야 합니다.
- 실수로 조건 누락 가능성이 있습니다.
- 삭제조건 추가로 인한 보일러 플레이트 코드가 증가할 수 있습니다.

위 방식이 적합한 상황

- 소프트 딜리트 적용 범위가 제한적일 때 (일부 엔티티에 한정)
- 프레임워크 의존도를 최소화 하고 싶을 때
- 삭제 조건을 쿼리마다 다르게 적용해야 할 때

### 3.2 Hibernate @SoftDelete 어노테이션을 사용한 구현

@SoftDelete는 Hibernate 6.4 부터 제공하는 네이티브 어노테이션 입니다.
DELETE 쿼리를 UPDATE로 치환하고, 조회 시 자동 필터링을 할 수 있습니다.

```java

@Entity
@SoftDelete(columnName = "deleted")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    // deleted 컬럼은 자동 관리됨 (명시적 선언 불필요)
}
```

@SoftDelete는 디폴트로 boolean 타입의 deleted 필드가 추가되며, 엔티티 내부에 자바코드로 동일한 이름의 필드를 지정할 수 없습니다.
<br>
deleted 외에도 필드명은 자유롭게 커스텀이 가능하고, boolean 타입이 싫을 경우 converter를 사용할 수 있습니다.

```java

@Service
public class UserService {

    private final UserRepository userRepository;

    @Transactional
    public void delete(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("User not found"));

        userRepository.delete(user); // 실제론 UPDATE is_deleted = true 로 변환됨
    }
}
```

서비스 레이어에서 delete() 메서드가 호출되면 JPA 내부적으로 DELETE 쿼리 대신 UPDATE 쿼리를 발생시켜 자동으로 소프트 딜리트를 수행합니다.

```java

public interface UserRepository extends JpaRepository<User, Long> {

    List<User> findAll();

    Optional<User> findById(Long id);
}
```

또한, 데이터를 조회할 때 명시적인 조건 없이도 삭제된 데이터를 자동으로 제외하여 조회합니다.

장점

- 최소한의 코드로 구현 가능
- 모든 쿼리에 자동으로 삭제 조건 적용
- delete() 호출 시 자동으로 소프트 딜리트 수행
- Hibernate 공식 지원으로 안정성 확보

단점

- Hibernate 6.4 이상 필요 (Spring Boot 3.2 이상 필요)
- 삭제된 데이터 조회가 어려움 (별도의 네이티브 쿼리 필요)
- 커스터마이징 제한적 (컬럼명, 타입 등)
- 플래그 타입으로 LocalDateTime 타입 사용이 불가능

적합한 상황

- Hibernate 6.4+ 사용 가능한 환경
- 대부분의 엔티티에 소프트 딜리트 적용
- 삭제된 데이터 조회 필요성이 낮을 때

### 3.3 @SQLDelete + @SQLRestriction (Hibernate 5.x 이상)

@SoftDelete와 동일하게 DELETE 쿼리를 UPDATE로 치환하고, 조회 시 자동 필터링을 할 수 있습니다.

```java

@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE user SET is_deleted = true, deleted_at = now() WHERE id = ?")
@SQLRestriction("is_deleted = false")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt;

    @Column(name = "is_deleted")
    private boolean isDeleted;

}
```

@SQLDelete 어노테이션의 속성으로 쿼리문을 지정할 수 있습니다.
<br>
@SQLDelete 는 플래그로 LocalDateTime 사용이 불가능 하다는 단점과 하나의 컬럼만 사용가능한 단점이 있는 @SoftDelete와 달리, 실행될 쿼리문을 직접 정의해줌으로써 2개 이상의 플래그 사용이
가능합니다.
<br>
@SQLRestriction 어노테이션의 속성으로 조회시 조건을 지정할 수 있습니다.

```java

@Service
public class UserService {

    private final UserRepository userRepository;

    @Transactional
    public void delete(Long id) {
        User user = userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("User not found"));

        userRepository.delete(user); // 실제론 UPDATE is_deleted = true 로 변환됨
    }
}
```

서비스 레이어에서 delete() 메서드가 호출되면 JPA 내부적으로 DELETE 쿼리 대신 UPDATE 쿼리를 발생시켜 자동으로 소프트 딜리트를 수행합니다.

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // @SQLRestriction으로 인해 자동으로 is_deleted = false 조건 추가됨
    List<User> findAll();

    Optional<User> findById(Long id);
}
```

데이터를 조회할 때 명시적인 조건없이도 @SQLRestriction에 명시한 조건이 추가되어 조회 쿼리를 수행합니다.

장점

- Hibernate 5.x부터 사용 가능 (Spring Boot 2.x 환경에서도 동작)
- 모든 쿼리에 자동으로 삭제 조건 적용
- delete() 호출 시 자동으로 소프트 딜리트 수행
- deleted_at 컬럼 직접 제어 가능

단점

- 삭제된 데이터 조회가 어려움
- Native SQL 직접 작성 필요 (DB 종속적)

적합한 상황

- Hibernate 5.x 또는 6.x 사용 환경
- @SoftDelete 사용 불가능한 레거시 프로젝트
- 삭제 시각(deleted_at)을 명시적으로 관리하고 싶을 때

## 중요: @Where Deprecated 주의사항

Hibernate 6.3부터 @Where 어노테이션이 Deprecated되었습니다. 따라서
반드시 @SQLRestriction을 사용해야 합니다.

### Hibernate 버전별 사용 방법

**Hibernate 5.x ~ 6.2 (Spring Boot 2.x ~ 3.1)**

```java

@Entity
@Where(clause = "deleted_at IS NULL")  // 구버전에서만 동작
public class User {
    // ...
}
```

**Hibernate 6.3+ (Spring Boot 3.2+)**

```java

@Entity
@SQLRestriction("deleted_at IS NULL")  // 신버전 권장
public class User {
    // ...
}
```

만약, @Where에서 @SQLRestriction으로 마이그레이션을 해야 한다면
<br>
@Where와 @SQLRestriction의 문법은 동일합니다.
<br>
그래서 단순 어노테이션 이름만 변경하면 됩니다.
<br>
Hibernate 6.3+에서 @Where 사용 시 컴파일 경고가 발생하여 @SQLRestriction로 변경해야 함을 알 수 있습니다.

```html
Warning: @Where is deprecated, use @SQLRestriction instead
```

### 3가지 방법 비교 정리

| 항목                 | 수동 구현 | `@SoftDelete` | `@SQLDelete` + `@SQLRestriction` |
|--------------------|-------|---------------|----------------------------------|
| **Hibernate 버전**   | 무관    | 6.4+          | 5.x+ (`@Where` 는 6.2까지)          |
| **Spring Boot 버전** | 무관    | 3.2+          | 2.x+                             |
| **코드 복잡도**         | 높음    | 낮음            | 중간                               |
| **자동 필터링**         | 수동    | 자동            | 자동                               |
| **삭제 데이터 조회**      | 쉬움    | 어려움           | 어려움                              |
| **커스터마이징**         | 완전 자유 | 제한적           | 중간                               |
| **실수 위험**          | 높음    | 낮음            | 낮음                               |
| **Deprecated 위험**  | 없음    | 없음            | `@Where` 사용 시 있음                 |

3가지 방법 모두 소프트 딜리트를 구현할 수 있는 방법입니다.
<br>
정해진 답은 없으며, 현재 자신의 프로젝트 상황, 팀의 상황에 따라서 최적을 선택을 하면 됩니다.

## 실무 적용 사례

실무에서 소프트 딜리트를 적용한 사례를 소개합니다.

프로젝트 환경

- Spring Boot 3.5.3 (Hibernate 6.6)
- MySQL 8.0
- Spring Data JPA

선택

- @SQLDelete + @SQLRestriction 방식

선택 이유

- 두개의 삭제 플래그(isDeleted, deletedAt)를 사용해야 하는 상황
- 가장 간단한 방법인 @SoftDelete는 하나의 컬럼만 지원하기 때문에 제외
- 삭제 플래그로 LocalDateTime 타입을 사용해야 하는 상황
- @SoftDelete는 LocalDateTime 타입의 타임스탬프를 지원하지 않기 때문에 제외

적용 결과

- 소프트 딜리트 조건 누락에 대한 부담감이 해소되었습니다. (어노테이션을 이용해 자동으로 소프트 딜리트가 이뤄지기 때문)
- 코드 리뷰시 삭제 조건 확인 불필요로 리뷰에 집중할 수 있게 되었습니다.

## 트레이드 오프

어떠한 기술이든 트레이드 오프가 존재합니다.
<br>
소프트 딜리트 역시 명확한 장점과 단점을 가지고 있으며, 이를 정확히 이해해야 올바른 의사결정을 할 수 있습니다.

### 스토리지 비용 증가

- 데이터 누적으로 인한 비용이 증가할 수 있습니다.

### 보안 위험

- 삭제된 데이터를 실수로 노출하지 않도록 주의해야 합니다.

# 소프트 딜리트 적용시 주의할 점

작성 예정

# 마무리

작성 예정
