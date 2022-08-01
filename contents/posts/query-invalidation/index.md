---
title: "React-Query에서의 데이터 최신화 (Query Invalidation)"
date: 2022-08-01 10:15:00
tags:
  - react
  - react-query
---

> 이 글은 우테코 달록팀 크루 [나인](https://github.com/jhy979)이 작성했습니다.

React-Query의 캐싱개념은 stale과 cacheTime을 통해 이루어집니다.

## Stale
> 사전적 의미로 '신선하지 않은' 입니다. react query는 기본적으로 캐싱된 데이터를 stale하다고 생각합니다.

react query에서는 stale time의 default이 0입니다. (즉, 캐싱이 되지 않는다고 볼 수 있겠죠?)

### staleTime
> 데이터가 fresh한 상태에서 stale한 상태로 변하는 시간입니다.

- fresh 상태일때는 쿼리 인스턴스가 새롭게 mount 되어도 fetch가 일어나지 않습니다. 

- 데이터가 fetch 되고 나서 staleTime이 지나지 않았다면 unmount 후 mount 되어도 fetch가 일어나지 않습니다.

### cacheTime
> 데이터가 inactive 상태일 때 캐싱된 상태로 남아있는 시간입니다.

- 쿼리 인스턴스가 unmount 되면 데이터는 inactive 상태로 변경되며, 캐시는 cacheTime만큼 유지됩니다.

- cacheTime은 staleTime과 관계없이, 무조건 `inactive된 시점`을 기준으로 캐싱을 결정합니다.

---
## 달록에서의 데이터 최신화

> 달록은 일정, 카테고리, 구독 등의 작업으로 인해 데이터의 변화가 잦은 어플리케이션입니다. 따라서 저희 팀은 staleTime을 지정해주지 않았습니다.

그렇다면 stale한 데이터는 늘 최신화가 필요할 것입니다.

그래서 저는 매번 새로운 데이터가 필요할 때마다 `useQuery의 refetch`를 강제적으로 실행시켜주는 방식을 생각했었습니다.

예를 들면 일정을 추가(post)한 후에 일정을 다시 조회(get)하는 경우가 있을겁니다.

![](https://velog.velcdn.com/images/jhy979/post/287f66a2-8b7e-49c8-b623-c05a491db600/image.png)

```tsx
// refetch 함수를 부모로 부터 주입 받아야 합니다.
function ScheduleAddModal({ closeModal, refetch }: ScheduleAddModalProps) {
  const onSuccessPostSchedule = () => {
    refetch();
  };

```

😢 하지만 이 방식이 우아하지는 않더라구요. 

만약 부모와 자식 간의 props전달이 아니라 조부모와 자식 간의 props 전달이라면 어떨까요? 벌써 머리가 아파옵니다. (props hell)


## Query Invalidation 도입
🤔 다른 방법이 있을텐데.. 어떤 방법이 있을까? 고민하며 공식문서를 읽던 도중 `Query Invalidation`을 발견하게 되었습니다.

```tsx
// Invalidate every query in the cache
queryClient.invalidateQueries()
// Invalidate every query with a key that starts with `todos`
queryClient.invalidateQueries(['todos'])
```
> The QueryClient has an invalidateQueries method that lets you intelligently mark queries as stale and potentially refetch them too!

❗ 캐싱키로 관련된 stale 쿼리들을 체크하고 refetch할 수 있는 메서드가 존재했습니다.

```tsx
function ScheduleAddModal({ closeModal }: ScheduleAddModalProps) {
  const queryClient = useQueryClient();

  const onSuccessPostSchedule = () => {
    // 일정 post 성공 시 데이터 최신화를 invalidateQueries메서드를 통해 수행합니다.
    queryClient.invalidateQueries(CACHE_KEY.SCHEDULES);
  };
```
💪 이로써 useQuery의 refetch함수를 넘겨줄 필요없이 어디서든 캐싱키로 fresh한 데이터를 보장할 수 있게 되었습니다.



#### 참고자료
https://tanstack.com/query/v4/docs/guides/query-invalidation