---
title: "refreshToken으로 accessToken 재발급하기"
date: 2022-10-31 17:00:00
tags:
  - trouble-shooting
---

> 이 글은 우테코 달록팀 크루 [나인](https://github.com/jhy979)이 작성했습니다.

## 😂로그인이 너무 짧아요

[달록 서비스](https://dallog.me)를 배포하고 사용자들로부터 많은 피드백과 QA를 받았어요. 덕분에 빠른 시간 내에 많은 문제점들을 해결할 수 있었어요.

> 그 중에서 로그인이 너무 짧다는 의견이 있었어요.

아무래도 달력은 매일 확인하는 도메인이다보니, 매일 다시 로그인해야하는 번거로움이 사용자들에게는 좋지 않은 경험을 제공했던 것이죠.

물론 구글 아이디로 로그인을 해서 터치 한 번이면 로그인이 되긴 하지만 `이조차도 굉장히 귀찮은 일이죠.`

---
## 어떻게 할까
> 로그인 연장 태스크를 맡은 백엔드 팀원과 이야기를 나누어봤어요.

### 🤯 문제 상황 정의

> 현재 accessToken의 유효기간이 1일이예요.

1일이 지나면 accessToken은 쓸모가 없어지고 다시 로그인을 해야하는 상황이 발생해요. 이 기간이 너무 짧다는 것이 문제임을 인지했습니다.

---
## 🔨 해결 방법

### 1. accessToken의 유효기간을 길게?

> 가장 먼저 떠오른 "방법은 accessToken의 유효기간을 늘린다" 였습니다.

기간을 더 늘리면 당연히 해결되는 문제죠. `하지만 문제가 있습니다.`

😂 accessToken은 api 요청 시 header에 굉장히 많이 사용되는 정보예요. 그러다보니 탈취될 가능성도 더 높다고 할 수 있겠죠.

- accessToken의 기간을 만약 한 달로 늘린다고 가정해봅시다. 
- 악성 이용자가 한 달짜리 accessToken을 탈취했다면 한 달간 해당 accessToken으로 무슨 짓이든 할 수 있겠죠?

🤦‍♂️ 결론적으로 accessToken의 유효기간을 늘리는 것은 위험한 일이라고 할 수 있어요.

**이 방식은 아니다~**

### 2. 매번 재검증 로직
> accessToken이 필요한 api 요청들을 처리하기 전에 "accessToken의 유효성을 검사하는 재검증 요청을 먼저 진행하자" 라는 의견이 나왔어요.

재검증 로직에서 만약 유효하지 않은 accessToken이라면 refreshToken을 기반으로 새로운 accessToken을 발급한다는 해결 방안입니다.

하지만 이게 정말 괜찮은 방식일까요?

아래는 달록의 rest docs입니다. 
<img src="https://velog.velcdn.com/images/jhy979/post/643a0d38-8131-422d-ad8f-64b2780c6277/image.png" width="700" />

accessToken을 필요로 하는 api가 20개나 되는데요, 이 요청들의 안전성을 보장하기 위해 매번 재검증 요청을 보낸다...?

😅 이 방식은 `배보다 배꼽이 더 크다고 생각했어요.`
  - 하루에 한 번 발생하는 재로그인 때문에 매번 네트워크 요청을 한 번씩 더하는 것이 어불성설이라고 느꼈거든요.

**너도 나가라~**

### 3. 에러 응답 코드로 판별하자

> 매번 요청을 보내지 말고 특정 에러 응답코드(invalid accessToken)가 올 때만 accessToken을 재발급 받으면 되지 않을까?

저희 달록은 accessToken이 유용하지 않을때 401 응답코드를 보내주고 있어요.
<img src="https://velog.velcdn.com/images/jhy979/post/5ed347d5-bd05-4a75-a156-fc6e35303dc0/image.png" width="300" />

그렇다면 401 status code를 받았을때 후속 처리로 refreshToken으로 accessToken을 받으면 되지 않을까요?

그렇다면 2번째 해결방법처럼 불필요한 네트워크 요청도 필요없을 것 같아요.

```ts
// accessToken 재발급을 위한 react-query문
const { mutate } = useLoginAgain();

// react-query queryClient의 전역 onError
const onError = (error: unknown) => {
  	// 401 에러가 발생했을 경우
    if (error instanceof AxiosError && error.response?.status === RESPONSE.STATUS.UNAUTHORIZED) {
      // refreshToken으로 accessToken을 재발급해주세요!
      mutate();

      return;
    }
  };
```

이런 식으로 전역 queryClient에 onError 로직을 달아주었어요. 그렇다면 달록의 모든 요청 중에서 `401 에러가 발생하는 경우에 대해 재발급 요청이 가능하겠죠?`

🥵 하지만 이 방식도 우아하지 않았어요...

아래는 달록의 달력 페이지 접속 시 요청하는 api들이예요.
<img src="https://velog.velcdn.com/images/jhy979/post/7410a969-d29f-4457-baa2-8f897df2475a/image.png" width="500" />
- 총 5개의 네트워크 요청이 발생하는데요, 이 중에서 `4개가 accessToken이 필요한 요청들입니다.`

- 만약 accessToken이 유효하지 않게 된다면 401 에러가 4개 발생하게 될 거예요.

🤔 전역으로 onError를 달아두었기 때문에 `accessToken 재발급 요청도 반복적으로 4번 발생하게 됩니다.`

재발급 요청을 한 번만 요청하면 되는데 무려 4번이나 보내고 있네요. 접근 방식은 좋았으나 이 트러블도 해결해야겠어요.

### 4. 에러 응답 코드 + 반복 발생하는 재발급 쿼리 무시 
> 재발급 쿼리가 진행중이라면 이를 무시해야겠어요.

여태까지 Mutation 작업에 키값을 적용하지 않았었는데요, 이제야 MutateKey값을 달아야할 필요성이 생겼네요.


```ts
// accessToken 재발급을 위한 react-query문
const { mutate } = useLoginAgain();

// 🙌 accessToken 재발급 mutate가 진행중인가요?
const isMutatingLoginAgain = useIsMutating(CACHE_KEY.LOGIN_AGAIN);

// react-query queryClient의 전역 onError
const onError = (error: unknown) => {
  if (error instanceof AxiosError && error.response?.status === RESPONSE.STATUS.UNAUTHORIZED) {
      // ✋ 재발급 중이 아닐때에만 재발급 요청 보내주세요. (재발급 중이면 mutate 무시)
      !isMutatingLoginAgain && mutate();

  return;
}
// ...
};
```
😉 실제로 재발급 요청이 한 번만 가는지 확인해볼까요?

<img src="https://velog.velcdn.com/images/jhy979/post/4d764f96-c622-4903-a2d3-8708a879cf67/image.png" width="500" />

`access`가 accessToken을 재발급하는 요청이예요. 볼 수 있듯이 딱 한번만 요청이 가고 있음을 확인할 수 있네요. 아주 굿~

---
## 😛 더 나아가기

### 실패했던 요청들까지 처리
> 더 나아가 401로 실패했던 요청들을 다시 실행시켜줄 수 있을 것 같아요.

1. mutation 키 값을 기준으로 401 `에러가 발생했던 요청들에 대해서 저장`한 다음 

2. accessToken을 재발급 받아 `유효성을 확신시`하고

3. 저장했던 (에러가 났던) 요청들을 다시 (유효한 accessToken으로) 실행시켜줄 수 있을 것 같네요.

### 만약 react-query를 쓰지 않았다면?
> 🤯 현재는 react-query를 쓰고 있어서 이런 mutating 과정을 제가 체크할 수 있었는데요 그렇지 않은 경우라면 어떻게 해결할 수 있었을까요?

라이브러리를 사용하지 않는다고 해도 `비슷한 방식으로 커스텀훅을 만들 것 같아요.`

예를 들면 `useFetch` 라는 훅을 만들어서 `onError` 공통 로직을 주입하여 에러 핸들링을 진행할 것 같습니다.

401 에러가 발생하는지 status code를 확인하여 accessToken을 재발급하는 과정을 진행해야겠죠?