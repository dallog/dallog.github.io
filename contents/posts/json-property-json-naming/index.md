---
title: "@JsonProperty, @JsonNaming"
date: 2022-09-10 00:00:00
update: 2022-09-10 00:00:00
tags:
- 매트
- 역직렬화
- BE
- JsonProperty
- JsonNaming
---

> 이 글은 우테코 달록팀 크루 '[매트](https://github.com/hyeonic)'가 작성했습니다.

## @JsonProperty, @JsonNaming

구글 측에서 [OpenID connect Sever flow](https://developers.google.com/identity/protocols/oauth2/openid-connect#server-flow)를 살펴보면 사용자 인증을 진행할 경우 아래와 같은 흐름으로 진행되는 것을 확인할 수 있다.

1. Create an anti-forgery state token: 위조 방지 상태 토큰 만들기
2. Send an authentication request to Google: Google에서 인증 요청 보내기
3. Confirm anti-forgery state token: 위조 방지 상태 토큰 확인
4. `Exchange code for access token and ID token: code를 액세스 토큰 및 ID 토큰으로 교환`
5. Obtain user information from the ID token: ID 토큰에서 사용자 정보 가져오기
6. Authenticate the user: 사용자 인증

`4`를 살펴보면 서버가 액세스 토큰 및 ID 토큰을 교환받기 위해 일회성 승인 코드인 `code`를 활용하여 아래와 같이 요청을 보내야 한다.

```
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7&
client_id=your-client-id&
client_secret=your-client-secret&
redirect_uri=https%3A//oauth2.example.com/code&
grant_type=authorization_code
```

위 요청을 보낸 뒤 성공적으로 처리되면 아래와 같은 응답을 확인할 수 있다.

```json
{
    "access_token": ...,
    "expires_in": ...,
    "id_token": ...,
    "token_type": ...,
    "refresh_token": ...
} 
```

주목해야 할 것은 google 측에서 token 정보를 `snake_case` 형식으로 제공하고 있다는 것을 기억하자.

## snake_case to CamelCase

google 측에서 제공된 token 정보의 경우 `snake_case`로 되어 있기 때문에 해당 데이터를 Java 객체로 변환하기 위해서는 아래와 같이 필드명을 작성해야 정상적으로 변환이 가능하다.

```java
public class GoogleTokenResponse {

    private String access_token;
    private String refresh_token;
    private String id_token;
    private String expires_in;
    private String token_type;
    private String scope;

    private GoogleTokenResponse() {
    }
    ...
    
    public String getAccess_token() {
        return access_token;
    }
    ...
}
```

그러나 Java에서는 기본적으로 `CamelCase`를 사용하고 있다. 코드 내부에 `CamelCase`와 `snake_case`가 함께 뒹굴고 있으면 일관성이 떨어진 코드를 마주하게 될 것이다. 우리는 `snake_case`로 제공 받은 데이터를 `역직렬화`할 때 `CamelCase`로 변환하는 방법에 대해 고민하게 되었다.

> 직렬화: Java 객체 → 전송 가능한 데이터 <br> 
> 역직렬화: 전송 가능한 데이터 → Java 객체

### @JsonProperty

`@JsonProperty`는 필드나 메서드 위에 선언되어 `직렬화/역직렬화`될 때 매핑 하기 위한 `property 명을 지정`한다. 아래와 같이 적용이 가능하다.

```java
public class GoogleTokenResponse {

    @JsonProperty("access_token")
    private String accessToken;

    @JsonProperty("refresh_token")
    private String refreshToken;

    @JsonProperty("id_token")
    private String idToken;

    @JsonProperty("expires_in")
    private String expiresIn;

    @JsonProperty("token_type")
    private String tokenType;
    
    private String scope;

    private GoogleTokenResponse() {
    }
    ...
}
```

다만 변환해야 하는 필드 값이 늘어날 경우 필드마다 해당 애노테이션을 명시해야 한다. 또한 문자열로 작성하기 때문에 오타로 인해 정상적으로 변환하지 못할 가능성 또한 존재한다.

### @JsonNaming

google 측에서 제공되는 token 정보의 경우 `snake_case`의 일정한 전략을 활용하여 응답하고 있다. 이때 변환하고자 하는 클래스 위에 `@JsonNaming`을 통해 일괄적으로 적용이 가능하다.

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
public class GoogleTokenResponse {

    private String accessToken;
    private String refreshToken;
    private String idToken;
    private String expiresIn;
    private String tokenType;
    private String scope;

    private GoogleTokenResponse() {
    }
    ...
}
```

## TODO

간단히 역직렬화 시점에 `snake_case`로 명시된 JSON 데이터를 `CamelCase` 필드로 변환하는 방식에 대해 알 수 있었다. 하지만 이 밖에도 직렬화/역직렬화를 진행할 때 Jackson에서 제공되는 애노테이션의 종류는 다양하다. 추후 어떠한 시점에 해당 애노테이션이 작용되며 직렬화와 역직렬화 시점에 필수적으로 필요한 것들에 대해 알아보려 한다. 

## References.

[OpenID Connect](https://developers.google.com/identity/protocols/oauth2/openid-connect)<br>
[Jackson Annotation Examples](https://www.baeldung.com/jackson-annotations)
