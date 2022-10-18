---
title: "react-query useMutation onSuccess 안 되는줄 알았던 트러블 슈팅"
date: 2022-10-18 13:30:00
tags:
  - react
  - react-query
  - trouble-shooting
---

> 이 글은 우테코 달록팀 크루 [나인](https://github.com/jhy979)이 작성했습니다.

<img src="https://velog.velcdn.com/images/jhy979/post/60cffbe1-a0bb-4f53-aa8f-e08ee092d038/image.png" />

> 달록 프로젝트를 진행하던도중 예상하지 못한 에러가 발생했어요.

달록에서는 `react-query`를 사용하고 있습니다.
`useMutation`을 활용하여 데이터의 변경을 일으키는 작업을 수행하고 있죠!
(예를들면 `Delete, Patch, Post` 같은 작업들이 있겠죠.)

🤦‍♂️ 그런데 간헐적으로 데이터가 변경(delete,patch,post)된 후 다시 데이터를 갱신(get)하는 작업에서 변경된 데이터가 들어오지 않는 이슈가 발생했습니다.

더욱 더 문제는 Mac을 사용하는 사용자들은 이런 에러가 발생하지 않았는데 Window 이용자들만 이런 에러가 발생했던 것입니다...

🤯 Wow... 이 문제를 직면하고 나서 React-Query가 OS에 따라서 동작이 다른가 의심이 들 정도로 문제의 핀트를 잡지 못했습니다

---
> 에러가 발생한 시나리오는 다음과 같아요. 한 번 볼까요?

### 1. 일정 추가
다음과 같이 일정을 추가했습니다.
이 때는 다행히도 새롭게 추가된 일정이 바로 get하는 response에 반영이 되어있네요.
<img src="https://velog.velcdn.com/images/jhy979/post/22dd80f0-fe58-4edd-99a3-7530f366d9b7/image.png" width="500" />

### 2. 일정 삭제
이제 일정을 삭제해볼까요?
<img src="https://velog.velcdn.com/images/jhy979/post/68e631e1-bfaf-4e16-b6a3-2075022eea91/image.png" width="500" />

개발자 도구의 네트워크 탭을 열어서 확인해봤어요.

<img src="https://velog.velcdn.com/images/jhy979/post/30df7bb6-115f-4227-a4cd-18ba91a7c799/image.png" />

`427`은 삭제 요청이고
`schedules?startDateTime...`은 다시 일정을 get하는 요청입니다.

보니깐 `delete` 이후에 다시 `get` 하는 작업이 이루어지고 있음을 확인할 수 있었어요.

🔨 코드를 잠깐 살펴보면

```tsx
function useDeleteSchedule({ scheduleId, onSuccess }: UseDeleteScheduleParams) {
  const { accessToken } = useRecoilValue(userState);
  const queryClient = useQueryClient();

  const { mutate } = useMutation<AxiosResponse, AxiosError>(
    () => scheduleApi.delete(accessToken, scheduleId),
    {
      onSuccess: () => {
        // ✋ 삭제가 성공하면 일정을 다시 get해주세요!!!
        queryClient.invalidateQueries(CACHE_KEY.SCHEDULES);

        onSuccess && onSuccess();
      },
    }
  );

  return { mutate };
}
```
delete useMutation에서 `onSuccess`안에 일정을 get하는 쿼리를 `invalidation`해주었기 때문에 당연히 삭제가 성공하고 나서 일정을 다시 가져오는 줄 알았습니다.

실제로도 네트워크탭에서 보시다시피 `427` 다음에 일정을 다시 get하고 있으니까요!
 
### 3. 대참사
하지만 결과는 다음처럼 삭제가 되지 않고 그대로 남아 있었습니다😂 ~~(이게 말이되냐고)~~

<img src="https://velog.velcdn.com/images/jhy979/post/22dd80f0-fe58-4edd-99a3-7530f366d9b7/image.png" width="500" />

> `새로고침`을 해서 아예 get 쿼리 요청을 다시 해야 삭제가 된채로 response가 왔어요. 하.. 이거 리렌더링을 일부러 발생시켜줘야하나? ㅋㅋㅋ큐ㅠㅠ

---
## 삽질

### 1. DB 시점 문제일까?
> 가장 먼저 든 생각은 DB에서 일정이 삭제되지 않은채로 get 요청이 진행되지 않았을까였습니다.

삭제가 성공했다고 `응답코드 204`를 주었지만 실제로 DB에는 반영이 되지 않은채로 바로 데이터를 가지고 오는건 아닐까?

백엔드 팀원들은 절대 절대 그런 일이 발생할 수는 없다고 하더라구요. 그럼에도 불구하고 직접 로그를 출력하여 확인해준 우리 팀원들 너무 감사합니다..😀😀

자 그럼 프론트엔드쪽에서의 문제겠죠?

---
### 2. useMutation의 onSuccess의 문제일까?

달록 프로젝트에서는 `react-query`를 사용하고 있어요. 깃허브 star가 `✨30.1k`가 찍힐만큼 인기 많은 라이브러리죠.

하지만 최근 프로젝트에서 상태 관리 라이브러리로 `recoil`을 사용하던 도중 특정 메서드가 실행되지 않는 에러가 있었던 적이 있었어요.

🥵 이 이후로 아무리 공신력있는 라이브러리라도 맹신하지 않겠다고 다짐했었어요.

호옥시 호옥시 `react-query useMutation의 onSuccess`가 mutate 작업이 확실히 성공된 후에 실행이 되는 것이 맞을까? 의심해봤어요.

관련해서 `react-query`의 깃허브 이슈 목록들을 전부 찾아봤지만... 없더라구요.

그렇다면 이건 문제가 아니라고 생각이 들더라구요.

---
### 3. 그럼 문제는 단 하나죠. react-query를 잘못 쓰고 있다!

> 돌고 돌아 다시 네트워크 탭에 들어가 요청이 실행되는 시점을 비교해봤어요.

이번엔 더 자세한 요청 시간을 보기 위해서 `Timing 탭까지 들어가서 확인해봤어요.`


😀 삭제 요청 큐에 들어가고 실행되는 시점입니다.

<img src="https://velog.velcdn.com/images/jhy979/post/8bf786cb-77e4-4090-81c6-83e340e07025/image.png" width="600" />

- 1.62초에 실행되고 삭제 성공까지 0.039정도 걸리네요.
- 그럼 성공시에 다음 로직은 1.66초 정도에 시작되어야할 것 같은데요! 놀라운 일이 발생합니다. 아래를 보시죠.


😀 삭제 요청 이후 Get 요청이 큐에 들어가고 실행되는 시점입니다.
<img src="https://velog.velcdn.com/images/jhy979/post/bc3bdc64-074c-4810-8cc7-503100c47350/image.png" width="600" />

- 엥? 시작 시점이 1.63초? 비상!!!!! 초비상!!!!!
- 삭제가 성공한 시점은 (1.66초)인데 get 요청이 (1.63초)에 실행된다니 말이 안되죠.

> 결론이 나왔습니다. 

1. DB문제도 아니고 
2. onSuccess의 문제도 아니다. (실행되는 Timing을 보니 onSuccess가 제대로 실행되고 있지 않다.)
3.  `저 get` 요청은 다른 이유로 실행되고 있다!!! 
(조금 더 보태어 설명하자면 `저 get` 요청이 `onSuccess의 get`요청이 실행될 시점에 이미 실행되고 있기 때문에 `onSuccess의 get` 요청이 무시되었던 거죠.)

그럼 왜 때문에 저 get요청이 실행되고 있는지 파악해야겠죠.

---
## 해결
> React Query는 데이터 Fetching, 캐싱, 동기화, 서버 쪽 데이터 업데이트 등을 쉽게 만들어 주는 React 라이브러리입니다.

- 네 맞죠, 저희는 리액트쿼리를 `캐싱`을 가장 큰 목적으로 사용하고자 했습니다.

- 하지만 프로젝트에서 queryClient의 `staleTime, cacheTime, refetchOnWindowFocus, refetchOnMount` 등을 지정해주지 않은채로 `default Option으로 그대로` 사용하고 있었어요.

`refetchOnWindowFocus` : 데이터가 stale 상태일 경우 포커스될 때 refetch를 실행하는 옵션

`refetchOnMount` : 데이터가 stale 상태일 경우 마운트 시 마다 refetch를 실행하는 옵션

> 그리고 중요한 것은.. staleTime의 default 값은 0이라는 점이죠!

🥵 staleTime이 0이므로 항상 데이터가 오염된 상태로 인식을 했고 

그러니 해당 컴포넌트가 마운트되거나 포커스되었을때 get요청을 계속계속계속 했던 것이죠. 으아... 문제 발견 발견!!!

🤯 삭제 후에 반영되지 않은채로 온 것 같은 response는 사실 삭제 후의 응답이 아니라 `마운트 혹은 포커스로 인해 발생한 get 요청`였던거죠ㅠㅠ

이 사실을 알고 다시 네트워크 요청의 waterfall을 살펴보니 더 명확해졌습니다.

아까 봤던 그림인데 이슈를 알고보니 바로 보이는군요.. ~~(참 야속하네요ㅠㅠ)~~

<img src="https://velog.velcdn.com/images/jhy979/post/30df7bb6-115f-4227-a4cd-18ba91a7c799/image.png" />

좌측 연두색 바를 살펴보니 `427 delete 요청`과 `schedule?startDateTime... get 요청`이 같은 시점에 queue에 들어가고 있군요😂😂

바로 staleTime을 지정해줬어요.
```tsx
const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: 1,
        retryDelay: 0,
        onError,
        // 1분으로 staleTime 지정하기
        staleTime: 1 * 60 * 1000,
      },
      mutations: {
        retry: 1,
        retryDelay: 0,
        onError,
      },
    },
  });
```
참고로 refetchOnMount, refetchOnWindowFocus는 true 설정해줘야 stale해졌을때 제대로 데이터를 가져올 것이라고 판단하여 true인 기본 옵션값으로 건드리지 않았어요.

---
## 확인
> 해결되었는지 확인해봅시다. 삭제 후 onSuccess 내부 로직인 일정 get 요청이 되는지 확인해볼까요?

<img src="https://velog.velcdn.com/images/jhy979/post/4376e31b-b5d8-4b86-bff0-bb3b759c76b9/image.png" width="800"/>

🌈드디어 `238 삭제 요청`이 성공하고 `schedules?startDateTime... get 요청`이 시점에 맞게 들어오네요. 

🌈 오른쪽 waterfall 연두색 바가 보이시죠? 이게 맞지!

---
## 결론
🤦‍♂️ 이번 트러블 슈팅을 통해 라이브러리를 사용하는 의도를 명확히 파악해야한다는 것을 느꼈어요.

- React-Query를 도입한 이유의 가장 큰 이유는 `캐싱에서의 이점`이었는데 이 부분을 개발할 때 크게 생각하지 않고 default option값들을 사용하면 될 줄 알았던 것이 문제였어요.

- 간헐적으로 발생해서 오랜 시간동안 에러를 해결하지 못했으나 덕분에 `라이브러리를 도입할 때 명확한 의도에 맞게` 사용해야한다는 점을 뼈저리게 느끼게 되었어요.
