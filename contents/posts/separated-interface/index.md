---
title: "외부와 의존성 분리하기"
date: 2022-07-24 15:00:00
update: 2022-07-24 15:00:00
tags:
  - 분리된 인터페이스
  - 의존성 분리
---

> 이 글은 우테코 달록팀 크루 '[매트](https://github.com/hyeonic)'가 작성했습니다.

## 외부와 의존성 분리하기

도메인 로직은 우리가 지켜야할 매우 소중한 비즈니스 로직들이 담겨있다. 이러한 도메인 로직들은 변경이 최소화되어야 한다. 그렇기 때문에 외부와의 의존성을 최소화 해야 한다. 

### 인터페이스 활용하기

우선 우리가 지금까지 학습한 것 중 객체 간의 의존성을 약하게 만들어 줄 수 있는 수단으로 인터페이스를 활용할 수 있다. 간단한 예시로 `JpaRepository`를 살펴보자.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {

    Optional<Member> findByEmail(final String email);

    boolean existsByEmail(final String email);
}
```

이러한 인터페이스 덕분에 우리는 실제 DB에 접근하는 내부 구현에 의존하지 않고 데이터를 조작할 수 있다. 핵심은 `실제 DB에 접근하는 행위`이다.

아래는 `Spring Data`가 만든 `JpaRepository의 구현체` `SimpleJpaRepository`의 일부를 가져온 것이다.

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {

	private static final String ID_MUST_NOT_BE_NULL = "The given id must not be null!";

	private final JpaEntityInformation<T, ?> entityInformation;
	private final EntityManager em;
	private final PersistenceProvider provider;

	private @Nullable CrudMethodMetadata metadata;
	private EscapeCharacter escapeCharacter = EscapeCharacter.DEFAULT;

	public SimpleJpaRepository(JpaEntityInformation<T, ?> entityInformation, EntityManager entityManager) {

		Assert.notNull(entityInformation, "JpaEntityInformation must not be null!");
		Assert.notNull(entityManager, "EntityManager must not be null!");

		this.entityInformation = entityInformation;
		this.em = entityManager;
		this.provider = PersistenceProvider.fromEntityManager(entityManager);
	}
  ...
}
```

해당 구현체는 `entityManger`를 통해 객체를 영속 시키는 행위를 진행하고 있기 때문에 `영속 계층`에 가깝다고 판단했다. 즉 도메인의 입장에서 `MemberRepository`를 바라볼 때 단순히 `JpaRepository`를 상속한 인터페이스를 가지고 있기 때문에 `영속 계층`에 대한 직접적인 의존성은 없다고 봐도 무방하다. 정리하면 우리는 인터페이스를 통해 실제 구현체에 의존하지 않고 로직을 수행할 수 있게 된다. 

### 관점 변경하기

이러한 사례를 외부 서버와 통신을 담당하는 우리가 직접 만든 인터페이스인 `OAuthClient`에 대입해본다. `OAuthClient`의 가장 큰 역할은 n의 소셜에서 `OAuth 2.0`을 활용한 인증의 행위를 정의한 인터페이스이다. google, github 등 각자에 맞는 요청을 처리하기 위해 `OAuthClient`를 구현한 뒤 로직을 처리할 수 있다. 아래는 실제 google의 인가 코드를 기반으로 토큰 정보에서 회원 정보를 조회하는 로직을 담고 있다.

```java
public interface OAuthClient {

    OAuthMember getOAuthMember(final String code);
}
```

```java
@Component
public class GoogleOAuthClient implements OAuthClient {

    private static final String JWT_DELIMITER = "\\.";

    private final String googleRedirectUri;
    private final String googleClientId;
    private final String googleClientSecret;
    private final String googleTokenUri;
    private final RestTemplate restTemplate;
    private final ObjectMapper objectMapper;

    public GoogleOAuthClient(@Value("${oauth.google.redirect_uri}") final String googleRedirectUri,
                             @Value("${oauth.google.client_id}") final String googleClientId,
                             @Value("${oauth.google.client_secret}") final String googleClientSecret,
                             @Value("${oauth.google.token_uri}") final String googleTokenUri,
                             final RestTemplate restTemplate, final ObjectMapper objectMapper) {
        this.googleRedirectUri = googleRedirectUri;
        this.googleClientId = googleClientId;
        this.googleClientSecret = googleClientSecret;
        this.googleTokenUri = googleTokenUri;
        this.restTemplate = restTemplate;
        this.objectMapper = objectMapper;
    }

    @Override
    public OAuthMember getOAuthMember(final String code) {
        GoogleTokenResponse googleTokenResponse = requestGoogleToken(code);
        String payload = getPayloadFrom(googleTokenResponse.getIdToken());
        String decodedPayload = decodeJwtPayload(payload);

        try {
            return generateOAuthMemberBy(decodedPayload);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException();
        }
    }

    private GoogleTokenResponse requestGoogleToken(final String code) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        MultiValueMap<String, String> params = generateRequestParams(code);

        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(params, headers);
        return restTemplate.postForEntity(googleTokenUri, request, GoogleTokenResponse.class).getBody();
    }

    private MultiValueMap<String, String> generateRequestParams(final String code) {
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("client_id", googleClientId);
        params.add("client_secret", googleClientSecret);
        params.add("code", code);
        params.add("grant_type", "authorization_code");
        params.add("redirect_uri", googleRedirectUri);
        return params;
    }

    private String getPayloadFrom(final String jwt) {
        return jwt.split(JWT_DELIMITER)[1];
    }

    private String decodeJwtPayload(final String payload) {
        return new String(Base64.getUrlDecoder().decode(payload), StandardCharsets.UTF_8);
    }

    private OAuthMember generateOAuthMemberBy(final String decodedIdToken) throws JsonProcessingException {
        Map<String, String> userInfo = objectMapper.readValue(decodedIdToken, HashMap.class);
        String email = userInfo.get("email");
        String displayName = userInfo.get("name");
        String profileImageUrl = userInfo.get("picture");

        return new OAuthMember(email, displayName, profileImageUrl);
    }
}
```

보통의 생각은 인터페이스인 `OAuthClient`와 구현체인 `GoogleOAuthClient`를 같은 패키지에 두려고 할 것이다. `GoogleOAuthClient`는 외부 의존성을 강하게 가지고 있기 때문에 `domain` 패키지와 별도로 관리하기 위한 `infrastructure` 패키지가 적합할 것이다. 결국 인터페이스인 `OAuthClient` 또한 `infrastructure`에 위치하게 될 것이다. 우리는 이러한 생각에서 벗어나 새로운 관점에서 살펴봐야 한다.

앞서 언급한 의존성에 대해 생각해보자. 위 `OAuthClient`를 사용하는 주체는 누구일까? 우리는 이러한 주체를 `domain` 내에 인증을 담당하는 `auth` 패키지 내부의 `Authservice`로 결정 했다. 아래는 실제 `OAuthClient`를 사용하고 있는 주체인 `AuthService`이다.

```java
@Transactional(readOnly = true)
@Service
public class AuthService {

    private final OAuthEndpoint oAuthEndpoint;
    private final OAuthClient oAuthClient;
    private final MemberService memberService;
    private final JwtTokenProvider jwtTokenProvider;

    public AuthService(final OAuthEndpoint oAuthEndpoint, final OAuthClient oAuthClient,
                       final MemberService memberService, final JwtTokenProvider jwtTokenProvider) {
        this.oAuthEndpoint = oAuthEndpoint;
        this.oAuthClient = oAuthClient;
        this.memberService = memberService;
        this.jwtTokenProvider = jwtTokenProvider;
    }

    public String generateGoogleLink() {
        return oAuthEndpoint.generate();
    }

    @Transactional
    public TokenResponse generateTokenWithCode(final String code) {
        OAuthMember oAuthMember = oAuthClient.getOAuthMember(code);
        String email = oAuthMember.getEmail();

        if (!memberService.existsByEmail(email)) {
            memberService.save(generateMemberBy(oAuthMember));
        }

        Member foundMember = memberService.findByEmail(email);
        String accessToken = jwtTokenProvider.createToken(String.valueOf(foundMember.getId()));

        return new TokenResponse(accessToken);
    }

    private Member generateMemberBy(final OAuthMember oAuthMember) {
        return new Member(oAuthMember.getEmail(), oAuthMember.getProfileImageUrl(), oAuthMember.getDisplayName(), SocialType.GOOGLE);
    }
}
```

지금 까지 설명한 구조의 패키지 구조는 아래와 같다.

```
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── allog
    │   │           └── dallog
    │   │               ├── auth
    │   │               │   └── application
    │   │               │       └── AuthService.java
    │   │               ...
    │   │               ├── infrastructure
    │   │               │   ├── oauth
    │   │               │   │   └── client
    │   │               │   │       ├── OAuthClient.java
    │   │               │   │       └── GoogleOAuthClient.java
    │   │               │   └── dto
    │   │               │       └── OAuthMember.java     
    │   │               └── AllogDallogApplication.java
    |   |
    │   └── resources
    │       └── application.yml
```

결국 이러한 구조는 아래와 같이 `domain` 패키지에서 `infrastructure`에 의존하게 된다.
  
```java
...
import com.allog.dallog.infrastructure.dto.OAuthMember; // 의존성 발생!
import com.allog.dallog.infrastructure.oauth.client.OAuthClient; // 의존성 발생!
...

@Transactional(readOnly = true)
@Service
public class AuthService {
	...
    private final OAuthClient oAuthClient;
    ...

    @Transactional
    public TokenResponse generateTokenWithCode(final String code) {
        OAuthMember oAuthMember = oAuthClient.getOAuthMember(code);
        ...
    }
    ...
}
```

### Separated Interface Pattern

`분리된 인터페이스`를 활용하자. 즉 `인터페이스`와 `구현체`를 각각의 패키지로 분리한다. 분리된 인터페이스를 사용하여 `domain` 패키지에서 인터페이스를 정의하고 `infrastructure` 패키지에 구현체를 둔다. 이렇게 구성하면 인터페이스에 대한 종속성을 가진 주체가 구현체에 대해 인식하지 못하게 만들 수 있다.

아래와 같은 구조로 인터페이스와 구현체를 분리했다고 가정한다.

```
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── allog
    │   │           └── dallog
    │   │               ├── auth
    │   │               │   ├── application
    │   │               │   │   ├── AuthService.java
    │   │               │   │   └── OAuthClient.java
    │   │               │   └── dto
    │   │               │       └── OAuthMember.java         
    │   │               ...
    │   │               ├── infrastructure
    │   │               │   ├── oauth
    │   │               │       └── client
    │   │               │           └── GoogleOAuthClient.java
    │   │               └── AllogDallogApplication.java
    |   |
    │   └── resources
    │       └── application.yml
```

자연스럽게 `domain` 내에 있던 `infrastructure` 패키지에 대한 의존성도 제거된다. 즉 외부 서버와의 통신을 위한 의존성이 완전히 분리된 것을 확인할 수 있다.

```java
...
import com.allog.dallog.auth.dto.OAuthMember; // auth 패키지 내부를 의존
...
@Transactional(readOnly = true)
@Service
public class AuthService {
	...
    private final OAuthClient oAuthClient;
    ...

    @Transactional
    public TokenResponse generateTokenWithCode(final String code) {
        OAuthMember oAuthMember = oAuthClient.getOAuthMember(code);
        ...
    }
    ...
}
```

## References.

[Separated Interface](https://www.martinfowler.com/eaaCatalog/separatedInterface.htmlhttps://www.martinfowler.com/eaaCatalog/separatedInterface.html)
