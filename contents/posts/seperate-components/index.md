---
title: "컴포넌트 분리 기준"
date: 2022-08-01 11:00:00
tags:
  - react
---

> 이 글은 우테코 달록팀 크루 [나인](https://github.com/jhy979)이 작성했습니다.

## UI가 비슷하면 재사용

> 초반에 개발을 진행할 때 `재사용의 기준, 컴포넌트의 분리 기준`은 대부분 UI였습니다.

"UI가 비슷하면 분명 재사용될 것이니 이 부분들을 묶어서 컴포넌트화하자!"

다음은 달록의 카테고리 목록을 조회하는 페이지입니다.

![](https://velog.velcdn.com/images/jhy979/post/efeb82b3-fc46-4acc-b9d3-2df795c0fe51/image.png)

- BE 공식일정 카테고리
- 알록달록 팀 회의 카테고리

두 카테고리가 매우 흡사하게 생겼죠? 비슷하게 생겼으니 하나의 컴포넌트로 취급하여 사용했습니다. (CategoryItem)

```tsx
function CategoryItem({ category, subscriptionId }: CategoryItemProps) {
  // ... 생략
  
  // ⚠️구독을 위한 react query
  const { mutate: postSubscription } = useMutation<
    AxiosResponse<Pick<SubscriptionType, 'color'>>,
    AxiosError,
    Pick<SubscriptionType, 'color'>,
    unknown
  >(() => subscriptionApi.post(accessToken, category.id, body), {
    onSuccess: () => {
      queryClient.invalidateQueries(CACHE_KEY.SUBSCRIPTIONS);
    },
  });

  // ⚠️구독 해제를 위한 react query
  const { mutate: deleteSubscription } = useMutation(
    () => subscriptionApi.delete(accessToken, subscriptionId),
    {
      onSuccess: () => {
        queryClient.invalidateQueries(CACHE_KEY.SUBSCRIPTIONS);
      },
    }
  );

  // ⚠️구독 해제 로직 
  const unsubscribe = () => {
    if (window.confirm(CONFIRM_MESSAGE.UNSUBSCRIBE)) {
      deleteSubscription();
    }
  };

  // ⚠️구독 버튼을 눌렀을 때 구독 여부에 따라 수행해야할 로직 변경
  const handleClickSubscribeButton = () => {
    subscriptionId > 0 ? unsubscribe() : postSubscription(body);
  };

  return (
    <div css={categoryItem(theme)}>
      <span css={item}>{category.createdAt.split('T')[0]}</span>
      <span css={item}>{category.name}</span>
      <div css={item}>
        // ⚠️구독 여부에 따른 버튼 스타일링 변경
        <SubscribeButton
          isSubscribing={subscriptionId > 0}
          handleClickSubscribeButton={handleClickSubscribeButton}
        ></SubscribeButton>
      </div>
    </div>
  );
}

export default CategoryItem;

```
구독 여부에 따라 버튼 스타일링만 다르게 하였고 실제로 큰 문제없이 의도한대로 렌더링되었습니다.

---

## 하지만.. 문제 발생

> 문제는 API를 연동했을 때 발생했습니다. (부끄럽게도 컴포넌트에 많은 고민과 설계 없이 구현한 대가를 치룬 것이죠.)

![](https://velog.velcdn.com/images/jhy979/post/ebd07538-e6d0-48cf-a43c-e04562ba3b65/image.png)

BE 공식일정은 이미 구독중인 상태여서 버튼을 누르면 `구독해제` api가 실행되어야합니다.

![](https://velog.velcdn.com/images/jhy979/post/236b2064-2dba-4e75-8130-f4074fe20732/image.png)
알록달록 팀 회의는 버튼을 누르면 `구독` api가 실행되어야합니다.

😢 문제는 바로 두 카테고리의 `데이터 스키마가 다르다는 점`이였습니다.


- `구독 api`는 구독을 위한 `카테고리 id`가 필요합니다. (애초에 구독을 안했기 때문에 구독 id가 존재할 수 없습니다.)

- `구독 해제 api`에서는 카테고리 id가 아닌 구독으로 새롭게 발급된 `구독 id`가 필요합니다.

구독 여부와 관계없이 하나의 컴포넌트로(CategoryItem) 묶었을 때에는 위 2가지 케이스로 인해 `구독 id`에서 문제가 발생합니다. 

하나의 컴포넌트로 묶었기 때문에 구독id가 존재하든 존재하지 않든 subscriptionId라는 필드가 반드시 존재해야합니다.

😱 요리조리 머리를 굴려 `구독하지 않았을 때에는 구독 id를 -1로 할당했습니다.`
(아마 위의 코드를 읽으시면서 subscriptionId > 0인 경우 구독 중이라고 판단하는 로직을 어색하게 느끼셨을 겁니다.)

물론 실제 DB에서 구독 id에 -1이 할당되는 경우는 없기 때문에 큰 문제는 아닐 수 있습니다.

---
## 새로운 컴포넌트의 분리기준: 데이터 스키마, 모델

하지만 태초에 "이런 컴포넌트 설계가 맞을까?" 라는 생각이 들었고 팀 회의를 통해 컴포넌트의 분리 기준을 `✨데이터 스키마와 모델✨`에 초점을 맞추는 방향으로 바꾸었습니다.

> 즉, 구독 중인 카테고리와 구독하지 않은 카테고리는 구독id의 존재 여부가 다르기 때문에 데이터 스키마가 다르다고 말할 수 있겠습니다.

따라서, 구독 중인 카테고리 컴포넌트와 구독하지 않은 카테고리 컴포넌트 2가지 컴포넌트로 분리하였습니다.

---

실제 코드를 볼까요?

### 👇 구독하지 않은 컴포넌트 (UnsubscribedCategoryItem)

![](https://velog.velcdn.com/images/jhy979/post/236b2064-2dba-4e75-8130-f4074fe20732/image.png)

- 구독 id가 없습니다.
- 구독 react query가 있습니다.

```tsx
// ⚠️subscriptionId를 아예 받지 않음
function UnsubscribedCategoryItem({ category }: UnsubscribedCategoryItemProps) {
  // ... 생략

  // ⚠️구독을 위한 react query
  const { mutate } = useMutation<
    AxiosResponse<Pick<SubscriptionType, 'color'>>,
    AxiosError,
    Pick<SubscriptionType, 'color'>,
    unknown
  >(() => subscriptionApi.post(accessToken, category.id, body), {
    onSuccess: () => {
      queryClient.invalidateQueries(CACHE_KEY.SUBSCRIPTIONS);
    },
  });

  // ⚠️오직 한 가지 일만 담당하는 핸들러 함수
  const handleClickSubscribeButton = () => {
    mutate(body);
  };

  return (
    <div css={categoryItem}>
      <span css={item}>{category.createdAt.split('T')[0]}</span>
      <span css={item}>{category.name}</span>
      <div css={item}>
        <Button cssProp={subscribeButton(theme)} onClick={handleClickSubscribeButton}>
          구독
        </Button>
      </div>
    </div>
  );
}
```

---
### 👇 구독중인 컴포넌트 (SubscribedCategoryItem)

![](https://velog.velcdn.com/images/jhy979/post/ebd07538-e6d0-48cf-a43c-e04562ba3b65/image.png)

- 구독 id가 있습니다.
- 구독 해제 react query가 있습니다.

```tsx
function SubscribedCategoryItem({ category, subscriptionId }: SubscribedCategoryItemProps) {
  // ... 생략
  
  // ⚠️구독 해제를 위한 react query
  const { mutate } = useMutation(() => subscriptionApi.delete(accessToken, subscriptionId), {
    onSuccess: () => {
      queryClient.invalidateQueries(CACHE_KEY.SUBSCRIPTIONS);
    },
  });

  // ⚠️오직 한 가지 일만하는 핸들러 함수
  const handleClickUnsubscribeButton = () => {
    if (window.confirm(CONFIRM_MESSAGE.UNSUBSCRIBE)) {
      mutate();
    }
  };

  return (
    <div css={categoryItem}>
      <span css={item}>{category.createdAt.split('T')[0]}</span>
      <span css={item}>{category.name}</span>
      <div css={item}>
        <Button cssProp={unsubscribeButton(theme)} onClick={handleClickUnsubscribeButton}>
          구독중
        </Button>
      </div>
    </div>
  );
}
```

---
### 마무리

> 어떤가요?

💪 subscriptionId가 필요하지 않은 경우에는 애초에 props로 들어가지 않기 때문에 `subscriotionId가 -1이 되는 경우가 없습니다.`

또한 react query와 관련된 로직들은 어떠한가요? 

💪 관심사 분리가 적절히 이루어져 각 컴포넌트에서 알맞게 호출되고 있습니다.


> UI를 기준으로 컴포넌트를 나누는 것도 좋지만 먼저 데이터 스키마에 따라 컴포넌트를 분리하는 것을 먼저 고려해보면 더 좋을 것이라고 이번 리팩토링을 통해 느끼게 되었습니다.

