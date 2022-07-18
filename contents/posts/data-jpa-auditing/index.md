---
title: "Spring Data JPA의 Auditing"
date: 2022-07-18 23:00:00
update: 2022-07-18 23:00:00
tags:
- Spring
- Data JPA
- Auditing
---

> 이 글은 우테코 달록팀 크루 '[파랑](https://github.com/summerlunaa)'이 작성했습니다.

> auditing이란 엔티티와 관련된 이벤트(insert, update, delete)를 추적하고 기록하는 것을 의미한다.

모든 엔티티에 생성일시, 수정일시, 생성한 사람을 추가하고 싶은 경우를 생각해보자. 모든 엔티티에 생성일시, 수정일시, 생성한 사람에 대한 필드를 일일이 구현해주어야 한다. 이렇게 되면 모든 엔티티에 중복이 생기고 유지보수가 어려워진다. Spring Data JPA가 제공하는 `Auditing` 기능을 사용하면 이런 기능을 쉽고 빠르게 구현할 수 있다.

## Spring Data JPA Auditing 적용하기

`Auditing`을 적용하기 위해서는 우선 어노테이션을 적용해야 한다. `@Configuration` 어노테이션이 적용된 Config 클래스에 아래와 같이 `@EnableJpaAuditing`을 추가한다.

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {
}
```

⚠️ 주의 : 원활한 Slice 테스트를 위해 @SpringBootApplication 과 @EnableJpaAuditing 어노테이션 분리하기

[https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.user-configuration-and-slicing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing.spring-boot-applications.user-configuration-and-slicing)

### Spring Entity Callback Listener 적용하기

Auditing entity listener class로 지정하기 위해 `@EntityListeners` 어노테이션을 Entity 클래스에 추가한다. 인자로는 `AuditingEntityListener.class`를 넘긴다. 이 설정을 통해 엔티티에 이벤트가 발생했을 때 정보를 캡처할 수 있다.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class BaseEntity {
    // ...
}
```

### 생성일시, 수정일시 추적하기

생성일시는 `@CreatedDate`, 수정일시는 `@LastModifiedDate` 어노테이션을 통해 추적할 수 있다. 생성일시의 경우 한 번 생성되면 변경되어선 안 되며, 항상 존재해야 하므로 `nullable = false`, `updatable = false`로 지정한다.

추가적으로 BaseEntity를 다른 Entity들이 상속받아 사용할 수 있도록 `@MappedSuperclass` 어노테이션을 통해 해당 클래스를 Entity가 아닌 SuperClass로 지정했다.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    // ...
}
```

### 생성한 사람, 수정한 사람 추적하기

생성한 사람은 `@CreatedBy`, 수정한 사람은 `@LastModifiedBy` 어노테이션을 통해 추적할 수 있다. 해당 필드는 생성자, 수정자의 이름으로 채워진다.

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {

    @CreatedBy
    @Column(name = "created_by")
    private String createdBy;

    @LastModifiedBy
    @Column(name = "modified_by")
    private String modifiedBy;

    // ...
}
```

유저에 대한 정보는 SecurityContext's Authentication 인스턴스로부터 가져온다. 이 값을 커스텀하고 싶다면 `AuditorAware<T>` 인터페이스를 구현해야 한다.

```java
public class AuditorAwareImpl implements AuditorAware<String> {
    
    @Override
    public String getCurrentAuditor() {
        // your custom logic
    }
}
```

이렇게 만든 `AuditorAwareImpl`를 사용하려면 Config 클래스에 `AuditorAwareImpl` 인스턴스로 초기화되는 `AuditorAware` 타입의 빈을 설정해주어야 한다. 그리고 `@EnableJpaAuditing` 어노테이션에 `auditorAwareRef="auditorProvider"` 설정을 추가한다.

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef="auditorProvider")
public class JpaConfig {
    //...
    
    @Bean
    AuditorAware<String> auditorProvider() {
        return new AuditorAwareImpl();
    }
    
    //...
}
```

---

### References

[https://www.baeldung.com/database-auditing-jpa](https://www.baeldung.com/database-auditing-jpa)
