---
title: "통합 테스트와 슬라이스 테스트"
date: 2022-08-01 09:00:00
update: 2022-08-01 09:00:00
tags:
  - 통합 테스트
  - 슬라이스 테스트
---

> 이 글은 우테코 달록팀 크루 '[매트](https://github.com/hyeonic)'가 작성했습니다.

# 통합 테스트와 슬라이스 테스트

## @SpringBootTest

`@SpringBootTest` 애노테이션을 사용하면 우리 애플리케이션에서 사용하고 있는 모든 빈을 등록한 뒤 간편하게 테스트를 진행한다. 하지만 모든 빈을 등록하기 때문에 아래와 같은 단점을 가질 수 있다.

- 모든 빈들을 등록하기 때문에 비교적 오랜 시간이 걸린다.
- 모든 빈들을 등록하기 때문에 의존성을 고려하지 않고 테스트를 진행할 수 있다. 즉 테스트 하고자 하는 객체의 의존성을 무시한채 테스트하게 된다.
- 도메인 혹은 영속 계층의 경우 application 계층, presentation 계층에 대해 의존하지 않기 때문에 필요 없는 리소스에 대한 소모가 늘어난다.

이러한 `@SpringBootTest`는 모든 빈을 등록한 채 테스트를 진행하는 `통합 테스트`에 적합한 애노테이션이다.

아래는 실제 프로젝트에 작성한 테스트 중 일부를 가져온 것이다. 

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Import(TestConfig.class)
class AcceptanceTest {

    @LocalServerPort
    private int port;

    @Autowired
    private DatabaseCleaner databaseCleaner;

    @BeforeEach
    void setUp() {
        RestAssured.port = port;
        databaseCleaner.execute();
    }
}
```

```java
@DisplayName("구독 관련 기능")
@Import(TestConfig.class)
public class SubscriptionAcceptanceTest extends AcceptanceTest {

    @DisplayName("인증된 회원이 카테고리를 구독하면 201을 반환한다.")
    @Test
    void 인증된_회원이_카테고리를_구독하면_201을_반환한다() {
        // given
        String accessToken = 자체_토큰을_생성하고_토큰을_반환한다(GOOGLE_PROVIDER, 인증_코드);
        CategoryResponse 공통_일정 = 새로운_카테고리를_등록한다(accessToken, 공통_일정_생성_요청);

        // when
        ExtractableResponse<Response> response = RestAssured.given().log().all()
                .auth().oauth2(accessToken)
                .contentType(MediaType.APPLICATION_JSON_VALUE)
                .body(빨간색_구독_생성_요청)
                .when().post("/api/members/me/categories/{categoryId}/subscriptions", 공통_일정.getId())
                .then().log().all()
                .statusCode(HttpStatus.CREATED.value())
                .extract();

        // then
        상태코드_201이_반환된다(response);
    }
    ...
}
```

인수 테스트의 경우 사용자의 시나리오에 맞춰 수행하는 테스트이기 때문에 실제 운영 환경과 유사하게 테스트를 진행해야 한다. 그렇기 때문에 슬라이스 테스트를 진행하는 것 보다 모든 빈들을 등록하여 시나리오를 적절히 수행하는지 집중해야 하기 때문에 `@SpringBootTest`를 활용한 통합 테스트를 진행해야 한다.

## 슬라이스 테스트

앞서 언급한 것 처럼 특정 계층은 다른 계층에 의존하지 않기 때문에 필요한 빈들만 주입받아 독립적으로 테스트할 수 있다. 이렇게 계층별로 필요한 빈들만 주입받기 위해서 Spring은 슬라이스 테스트관련 애노테이션을 제공한다.

슬라이스 테스트란? 레이어를 독립적으로 테스트하기 위해 필요한 빈들만 주입 받아 테스트를 진행하는 것을 의미한다.. 슬라이스 테스트를 적절히 활용하면 모든 빈들을 ApplicationContext에 등록하지 않기 때문에 보다 더 호율적으로 테스트가 가능하다.

### @WebMvcTest

`@WebMvcTest`는 웹 계층 테스트를 위해 필요한 빈들이 주입된다. 주입 되는 빈의 항목은 아래와 같다.

- `@Controller`
- `@ControllerAdvice`
- `@JsonComponent`
- `Converter`
- `Filter`
- `WebMvcConfigurer`
- `HandlerMethodArgumentResolver`
- `MockMvc`
- 등

아래는 실제 프로젝트에 작성한 테스트 중 일부를 가져온 것이다.

```java
@AutoConfigureRestDocs
@WebMvcTest(SubscriptionController.class)
class SubscriptionControllerTest {

    private static final String AUTHORIZATION_HEADER_NAME = "Authorization";
    private static final String AUTHORIZATION_HEADER_VALUE = "Bearer aaaaa.bbbbb.ccccc";

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private AuthService authService;

    @MockBean
    private SubscriptionService subscriptionService;

    @DisplayName("회원과 카테고리 정보를 기반으로 구독한다.")
    @Test
    void 회원과_카테고리_정보를_기반으로_구독한다() throws Exception {
        // given
        CategoryResponse 공통_일정_응답 = 공통_일정_응답(관리자_응답);
        SubscriptionResponse 빨간색_구독_응답 = 빨간색_구독_응답(공통_일정_응답);

        given(authService.extractMemberId(any())).willReturn(매트_응답.getId());
        given(subscriptionService.save(any(), any(), any())).willReturn(빨간색_구독_응답);

        // when & then
        mockMvc.perform(post("/api/members/me/categories/{categoryId}/subscriptions", 빨간색_구독_응답.getId())
                        .header(AUTHORIZATION_HEADER_NAME, AUTHORIZATION_HEADER_VALUE)
                        .accept(MediaType.APPLICATION_JSON).contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(빨간색_구독_생성_요청)))
                .andDo(print())
                .andDo(document("subscription/save",
                        preprocessRequest(prettyPrint()),
                        preprocessResponse(prettyPrint()),
                        pathParameters(
                                parameterWithName("categoryId").description("카테고리 id")
                        ),
                        requestHeaders(
                                headerWithName("Authorization").description("JWT 토큰")
                        ),
                        requestFields(
                                fieldWithPath("color").type(JsonFieldType.STRING).description("구독 색 정보")
                        )))
                .andExpect(status().isCreated());
    }
    ...
]
```

컨트롤러 테스트의 경우 실제 동작하는 로직을 활용하는 것 보다 API의 문서화에 집중했기 때문에 웹 계층에 대한 의존성만 추가한 뒤 의존하는 객체는 Mocking을 통해 진행하였다.

### @DataJpaTest

Spring Data JPA를 사용하고 있다면 테스트하기 위해 간단히 `@DataJpaTest`를 활용할 수 있다. `@Entity` 객체, `JpaRepository` 등 JPA 사용에 필요한 빈들을 등록하여 테스트할 때 사용한다. 아래는 실제 `@DataJpaTest`의 코드를 가져온 것이다.

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@BootstrapWith(DataJpaTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@TypeExcludeFilters(DataJpaTypeExcludeFilter.class)
@Transactional
@AutoConfigureCache
@AutoConfigureDataJpa
@AutoConfigureTestDatabase
@AutoConfigureTestEntityManager
@ImportAutoConfiguration
public @interface DataJpaTest {
	...
}
```

주의 깊게 봐야할 애노테이션들이 많다. 몇 가지 예시로 `@AutoConfigureTestDatabase`, `@Transactional`에 대해 간단히 살펴보자.

#### @AutoConfigureTestDatabase

```java
...
@AutoConfigureTestDatabase
...
public @interface DataJpaTest {
	...
}
```

애플리케이션에 정의되어 있거나 자동으로 설정된 DataSoruce를 대신하여 테스트용 DB를 정의할 때 사용된다. 아래 실제 코드를 살펴보자.

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@ImportAutoConfiguration
@PropertyMapping("spring.test.database")
public @interface AutoConfigureTestDatabase {

	@PropertyMapping(skip = SkipPropertyMapping.ON_DEFAULT_VALUE)
	Replace replace() default Replace.ANY;

	EmbeddedDatabaseConnection connection() default EmbeddedDatabaseConnection.NONE;

	enum Replace {
		ANY,
		AUTO_CONFIGURED,
		NONE
	}
}
```

- `replace()`:  대체할 수 있는 기존 DataSource 빈의 유형을 결정한다.
    - `Replace.ANY`: 자동 구성 또는 수동 정의의 여부에 상관 없이 DataSource를 교체한다. default 설정 이기 때문에 `@DataJpaTest`를 사용하면 기본적으로 `in-memory embedded database`를 활용한다.
    - `Replace.AUTO_CONFIGURED`: 자동 설정된 경우에만 DataSource를 교체한다.
    - `Replace.NONE`: 기본 DataSource를 교체하지 않는다. 즉 우리가 직접 빈으로 등록하거나 명시한 DataSource를 사용한다. 만약 `in-memory embedded database`가 아닌 외부 DB나 테스트 용 DB를 사용하고 싶다면 `@AutoConfigureTestDatabase(replace = Replace.NONE)`으로 설정을 덮어야 한다.

#### @Transactional

```java
...
@Transactional
...
public @interface DataJpaTest {
	...
}
```

앞서 언급한 것 처럼 `@DataJpaTest`는 기본적으로 `@Transactional` 애노테이션을 들고 있기 때문에 테스트가 완료되면 자동으로 롤백된다.

아래는 실제 프로젝트를 진행하며 작성한 테스트 코드의 일부를 가져온 것이다.

```java
@DataJpaTest
@Import(JpaConfig.class)
class ScheduleRepositoryTest {

    @Autowired
    private ScheduleRepository scheduleRepository;

    @DisplayName("시작일시와 종료일시를 전달하면 그 사이에 해당하는 일정을 조회한다.")
    @Test
    void 시작일시와_종료일시를_전달하면_그_사이에_해당하는_일정을_조회한다() {
        // given
        Schedule schedule1 = new Schedule(TITLE, LocalDateTime.of(2022, 7, 14, 14, 20),
                LocalDateTime.of(2022, 7, 15, 16, 20), MEMO);

        Schedule schedule2 = new Schedule(TITLE, LocalDateTime.of(2022, 8, 15, 14, 20),
                LocalDateTime.of(2022, 8, 15, 16, 20), MEMO);

        scheduleRepository.save(schedule1);
        scheduleRepository.save(schedule2);

        // when
        List<Schedule> schedules = scheduleRepository.findByBetween(START_DAY_OF_MONTH, END_DAY_OF_MONTH);

        // then
        assertThat(schedules).hasSize(1);
    }
    ...
}
```

## 정리

이 밖에도 `@JdbcTest`, `@DataMongoTest`, `@RestClientTest` 등 다양한 슬라이스 테스트를 위한 애노테이션이 제공된다. 어떠한 애노테이션을 사용하는 것에 집중하기 보다 `테스트의 목적`에 대해 고민해야 한다. 테스트하고자 하는 것에 집중하여 의존하거나 필요한 빈들에 대해 고민한 뒤 적절한 애노테이션을 적용하면 불필요한 리소스를 줄일 수 있으며 보다 더 빠른 테스트 피드백을 확인할 수 있을 것이라 판단한다.

## References.
[Spring Boot Test](https://meetup.toast.com/posts/124)<br>
[8. Testing](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)<br>
[Spring Boot 슬라이스 테스트](https://tecoble.techcourse.co.kr/post/2021-05-18-slice-test/)
