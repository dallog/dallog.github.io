---
title: "React에서 무한 스크롤 구현하기"
date: 2022-07-18 08:00:00
tags:
  - react
---

> 이 글은 우테코 달록팀 크루 '[나인](https://github.com/jhy979)'이 작성했습니다.

🎯 "무한 스크롤을 구현해보세요!"

어떻게 구현하실 건가요?

## 무한 스크롤을 처음 마주했을때

🤔 저는 처음 무한 스크롤을 구현할 때 다음과 같은 방식을 사용했어요.

```
1. scroll 이벤트를 감지한다.

2. 현재 스크롤 영역의 `위치를 계산`한다.

3. 영역 계산을 통해 페이지 아래에 위치하면 API 요청을 진행한다.

4. 받아온 데이터를 추가하여 다시 렌더링한다.

5. 무한 반복
```

---
## scroll event 사용하기

[우테코 레벨1 유튜브 미션](https://github.dev/jhy979/javascript-youtube-classroom/tree/jhy979-step2)

처음 제가 무한 스크롤을 구현했던 방법은 다음과 같습니다.

바로 스크롤 이벤트와 offset을 이용한 방식이죠!

```js
  scrollToBottom(callback) {
    const isScrollBottom =
      this.$videoList.scrollHeight - this.$videoList.scrollTop <=
      this.$videoList.offsetHeight + EVENT.SCROLL.OFFSET;

    if (isScrollBottom) {
      callback(this.$searchInput.value);
    }
  }
```
메서드 네이밍을 통해서도 알 수 있듯이, 화면 하단까지 내려갔을 경우 (offset 정도를 감안하여) 인자로 받은 함수를 실행시켜주었습니다.

아 물론, 스크롤 이벤트는 워낙 많이 발생하기 때문에 throttle을 걸어주었습니다. (이건 필수죠)

😢 하지만, `documentElement.scrollTop`, `documentElement.scrollHeight`, `documentElement.offsetHeight`는 리플로우(Reflow)가 발생합니다.

확실히 비효율적이겠죠!

---

## IntersectionObserver 사용하기

달록에서는 무한 스크롤을 구현할 때 [Intersection Observer](https://developer.mozilla.org/ko/docs/Web/API/Intersection_Observer_API)를 사용했습니다. 

> Intersection Observer는 쉽게 말해 지정한 대상이 화면에 보이는지 감시하고 판단하는 도구입니다.

브라우저 Viewport와 Target으로 설정한 요소의 교차점을 관찰하여 그 Target이 Viewport에 포함되는지 구별하는 기능을 제공합니다. 

<img src="https://velog.velcdn.com/images/jhy979/post/19500233-65fc-4ba9-b421-81516700c00b/image.png" />


### useIntersect 커스텀훅

> 가장 먼저 useIntersect 라는 커스텀훅을 제작했습니다. 

이 커스텀훅은 `인자로 intersect시 실행할 함수`를 받고 `ref를 제공`하여 관찰할 대상을 지정할 수 있습니다.

```ts
type IntersectHandler = (entry: IntersectionObserverEntry, observer: IntersectionObserver) => void;

// 인자로 onIntersect와 options를 받습니다.
// onIntersect는 intersect 발생 시 실행하고 싶은 함수입니다.
function useIntersect(onIntersect: IntersectHandler, options?: IntersectionObserverInit) {
  // 관찰하고 싶은 친구를 잡기 위해 ref를 만들어주세요.
  const ref = useRef<HTMLDivElement>(null); 
  
  // intersect 시 실행할 함수를 만들어줍시다.
  const callback = useCallback(
    (entries: IntersectionObserverEntry[], observer: IntersectionObserver) => {
      entries.forEach((entry) => {
        if (entry.isIntersecting) {
          onIntersect(entry, observer);
        }
      });
    },
    [onIntersect]
  );

  // 🔨 옵저버에게 일을 시켜봅시다.
  useEffect(() => {
    
  	// 우리가 관찰하고 싶은 친구가 없으면 그냥 return 해버려요.
    if (!ref.current) {
      return;
    }
	
    // 관찰할 대상이 있으면 옵저버 데꼬 와야죠!
    const observer = new IntersectionObserver(callback, options);
	
    // 이 옵저버한테 감시를 시킵시다.
    observer.observe(ref.current);

    // 할 일 끝나면 고생한 옵저버도 쉬게 해줍시다!
    return () => {
      observer.disconnect();
    };
    
  }, [ref, options, callback]);

  return ref;
}

export default useIntersect;
```

### 실제 사용
useIntersect 커스텀훅을 잘 만들었으니 이제 이 커스텀훅을 무한 스크롤에 사용해 봅시다.

![](https://velog.velcdn.com/images/jhy979/post/4643727c-852d-4f23-ab4f-44ce79e2e3b2/image.gif)

다음은 카테고리 목록을 계속 불러와 리스트를 보여주는 컴포넌트입니다.
```tsx
function CategoryList({ categoryList, getMoreCategories, hasNextPage }: CategoryListProps) {
  
  // useIntersect 커스텀훅의 인자로 (교차 시) 실행할 함수를 넣어줍시다.
  const ref = useIntersect(() => {
    hasNextPage && getMoreCategories();
  });

  return (
    <div css={categoryTable}>
      <div css={categoryTableHeader}>
        <span> 생성 날짜 </span>
        <span> 카테고리 이름 </span>
      </div>
      
      {categoryList.map((category) => (
        <CategoryItem key={category.id} category={category} />
      ))}
      
      // 페이지 하단까지 내리면 이 친구가 등장하여 옵저버에게 감지될 거예요.
      <div ref={ref} css={intersectTarget}></div>
    </div>
  );
}

```
💪 무한 스크롤함에 따라 props로 받아오는 categoryList가 길어지게 될텐데요, 다행히 React에서는 key값으로 변경 여부를 확인하기 때문에 새롭게 추가된 리스트들만 리렌더링해주었습니다.
