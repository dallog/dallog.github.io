---
title: "서블릿 생명주기와 직접만든 톰캣을 통한 의문점"
date: 2022-09-17 23:00:00
update: 2022-09-17 23:00:00
tags:
  - tomcat
  - servlet
---

> 이 글은 우테코 달록팀 크루 '[리버](https://github.com/gudonghee2000)'가 작성했습니다.

## 글을 쓰게된 계기
우아한 테크코스 레벨4 Tomcat 구현하기 미션을 진행하면서 직접만든 Tomcat과 실제 Tomcat이 다르게 작동하는 부분이 있었다.
Servlet과 ServletContainer, ServletLifeCycle를 알아보고
왜 다르게 작동하는지에 대한 의문점을 공유하고자 글을 작성해본다.

## Servlet이란?
자바로 HTTP 요청을 처리하는 프로그램을 만들 때 사용하는 인터페이스이다.
Servlet은 자바 표준으로 `Jakarta EE`의 표준 API이다.
Servlet은 클라이언트의 `Request`를 받아서 일련의 작업을 처리하고 `Response`에 작업의 결과물을 담아준다.

## Servlet Container란?
서버에서 만들어진 `Servlet`들을 **관리**하고 **중계**하는 역할을 한다.

ServletContainer는 클라이언트의 **요청**을 받았을 때, 요청에 대응할 수 있는 Servlet을 인스턴스화 해주고 **요청**에 대한 **응답**을 해준다.

## ServletContainer의 역할
> 1. 웹서버와의 통신 지원
ServletContainer는 `Servlet`과 `WebServer`가 손쉽게 통신 할 수 있게 중계 해준다. 
ServletContainer는 `ServerSoket`을 만들고 `listen()`, `accept()` 등의 API로 클라이언트 요청이 오면 적절한 `Servlet`에게 요청을 처리하게끔 한다.

> 2. Servlet 생명주기 관리
ServletContainer는 `Servlet`을 관리한다.
-  Servlet 클래스를 로딩하여 인스턴스화
-  Servlet 초기화
- 요청에 적절한 Servlet 할당
- Servlet 제거

Servlet 생명주기를 어떻게 관리하는지 좀 더 자세히 살펴보자.

## Servlet 생명주기를 관리하는 메서드
``` java
public class ExampleServlet extends HttpServlet {

    public static final String 인코딩 = "인코딩";

    @Override
    public void init(final ServletConfig config) throws ServletException {
    }

    @Override
    protected void service(final HttpServletRequest request, final HttpServletResponse response) throws IOException {
    }

    @Override
    public void destroy() {
    }
}
```
ServletContainer는 `Servlet`들의 생명주기를 그림과 같이 `init()`, `service()`, `destroy()` 메서드를 통해 관리해준다.
아래에서 전체적인 과정을 간략하게 살펴보고 `Servlet 생명주기`를 자세히 살펴보자.


## Servlet 생명주기 과정 알아보기
ServletContainer가 `Servlet`을 어떻게 사용하고 관리하는지 그림을 통해 정리해보자.

![](https://velog.velcdn.com/images/gudonghee2000/post/1993301b-561f-419d-9082-a5e668e0c13f/image.png)


1. 톰캣이 가지고 있는 Connector 중 하나를 통해 `Request`를 전달 받는다.

2. 톰캣은 `Request`의 `url`을 기반으로 적절한 `Servlet`을 찾아내 매핑한다.

3. 요청이 적절한 `Servlet`에게 매핑되면 메모리에 해당 `Servlet` 인스턴스가 올라와있는지 확인한다. 만약 존재하지 않으면 `Servlet` 인스턴스를 생성한다.

4. 톰캣이 `init()` 메서드를 호출하여 `Servlet`을 초기화한다.

5. `Servlet`이 초기화 되면 톰캣은 `service()` 메서드를 호출하여 `Request`를 처리한다.

6. 마지막으로 톰캣을 종료하거나 `Servlet`을 수정하게되면 `destroy()` 메서드를 호출하여 메모리에 올라와있던 `Servlet`을 제거한다.

## Servlet 생명주기 조금 더 자세히 살펴보기
![](https://velog.velcdn.com/images/gudonghee2000/post/02b01768-d52a-4820-b829-9361d1328b4c/image.JPG)

Servlet은 처음 호출되어 메모리에 올라가면 해당 인스턴스를 재활용한다.
**그래서 객체가 생성되는 순간부터 자원이 해제되는 순간까지가 서블릿의 라이프사이클이다.** 

과정을 하나씩 자세히 살펴보자.

#### 1. Servlet 객체 생성
먼저 Servlet이 최초에 호출되면 톰캣에서 서블릿 파일(.java)을 불러와 컴파일한 후 메모리에 로드시킨다. 톰캣은 서블릿 파일이 변경되지 않는 이상 서버가 종료될 때까지 계속해서 해당 객체를 재활용한다.

#### 2. init() 메서드 호출 (객체 생성시 최초 1회만 실행)
Servlet에서 기본 상속받는 HttpServlet의 부모 추상 클래스인 GenericServlet에 아래와 같이 정의하고 있는 메서드이다.

``` java
public abstract class GenericServlet implements Servlet, ServletConfig, java.io.Serializable {

	...
    
	@Override
    public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init();
    }
}

```
Servlet 객체 생성시 초기화 작업을 수행하는데, `ServerConfig` 객체를 넣어 실행한다.

`ServerConfig`객체는 아래에서 다시살펴보자.


#### 3. service() 메서드 호출
클라이언트의 요청에 따라 생성된 `Request`객체를 통해 요청을 처리하고 `Response` 객체에 적절한 응답을 추가해준다.
이때, 쓰레드풀의 쓰레드 하나를 사용한다.

#### 4. destroy() 메서드 호출 (자원 해제시에 1번만 실행)
Servlet에 수정사항이 생겨 Servlet을 다시 로드해야하거나 서버가 정상적으로 종료되는 경우에 호출된다.

![](https://velog.velcdn.com/images/gudonghee2000/post/93a0c18e-4db4-4538-ad85-4b291322b430/image.JPG)
실제 톰캣의 Servlet을 호출할 때의 로그이다.
처음 Servlet이 호출됬을 때만 `init()` 메서드가 호출됨을 알 수 있다.

## 직접 구현한 톰캣과 달랐던 의문점들

우아한 테크코스 레벨4 저번미션을 진행하면서 직접 Tomcat을 구현해보았었다.
직접만든 Tomcat과 달랐던 부분이있어서 이를 공유하고자 한다.

### 1. Servlet 실행시점에 Servlet 객체 생성

직접 구현해본 톰캣에서는 다음과 같이 `Tomcat` 실행시점에 `Servlet` 인스턴스를 생성하였다.

``` java
public class Tomcat {
	...

    public void start() {
        ControllerContainer container = new ControllerContainer(configuration);
        var connector = new Connector(container);
      	...
    }
}
```
`ControllerContainer`가 ServletCotainer이고 `ControllerContainer` 생성자 파라미터의 `Configuration` 객체에 `Servlet`
 인스턴스 생성에 대한 코드가 담겨있다.

그런데, 실제 톰캣은 클라이언트의 `Request`가 들어오고 매핑할 `Servlet`이 필요해질 때 `Servlet` 인스턴스를 생성한다😮😮

![](https://velog.velcdn.com/images/gudonghee2000/post/93a0c18e-4db4-4538-ad85-4b291322b430/image.JPG)

실제 테스트 코드에서 `쉐어드 서블릿`이라는 객체에 요청을 보냈을 때 확인해 본 결과이다.

이런 결과를 보고 

**톰캣을 실행 할 때, Servlet 객체를 모두 생성 해놓으면 클라이언트 요청이 들어올때 더 빨리 응답을 해줄 수 있지않나? 라는 의문이 들었다.**

아마, `Servlet`에 변경사항이 있을 때, 실제 Server를 종료시키지 않고 `Servlet` 변경사항을 적용하기 위해서 클라이언트 **요청**이 들어올 때, `Servlet` 인스턴스를 생성하도록 구현 한게 아닐까 조심스럽게 추측한다🙄 

### 2. Servlet 객체 생성과 init()을 따로하는 이유
두번째는 `Servlet`을 생성 하고 `init()` 메서드를 호출한다는 점이다.
**Servlet을 생성 할 때 초기화도 같이 하면 되는거 아닌가? 왜 과정을 나눠놨지? 라는 의문이 들었다.**

찾아본 자료에서 몇가지 이유를 보았는데 일리가 있다고 생각하는 이유는 다음과 같다.

- Interface는 생성자를 가질수 없기 때문이다.
`Servlet` 초기화를 위해서는 앞에서 이야기한 `ServletConfig` 객체를 `Servlet`에게 전달해주어야 하는데 Interface는 생성자를 가질 수 없다.
그래서 `Java EE Servlet API` 표준에 `Servlet` 객체를 초기화하기 위해서는 `ServletConfig` 객체가 필요함을 명시해주기 위해서 `init()`메서드를 통해 초기화 하도록 한것이 아닐까 한다.   
> 
**💡ServletConfig란?**
하나의 Servlet 초기화에 필요한 정보를 전달하기 위한 Config 객체를 의미한다.

- 생성자는 복잡한 논리/비즈니스 처리가 없어야 하기때문이다. `ServletConfig`객체를 생성하는 것은 비용이 비싼작업이기 때문에 생성자가 아닌 `init()` 메서드를 통해 처리해야 한다는 의견도 있다.

## 마치면서
`Servlet`과 `ServletContainer`, `Servlet` 생명주기에 대해서 살표보았다. 
`Servlet` 생명주기에 대해서 같은 의문점이 있었다면 의문을 해소하는데 도움이 되었으면 좋겠다😊

## 참고한 자료
https://stackoverflow.com/questions/7580771/can-we-replace-the-purpose-of-init-method-to-the-servlet-constructor

https://codevang.tistory.com/193

https://velog.io/@jihoson94/Servlet-Container-%EC%A0%95%EB%A6%AC

https://bashbeta.com/ko/%EC%84%9C%EB%B8%94%EB%A6%BF%EC%97%90%EC%84%9C-%EC%83%9D%EC%84%B1%EC%9E%90%EB%A5%BC-%ED%98%B8%EC%B6%9C%ED%95%98%EB%8A%94-%EC%9D%B4%EC%9C%A0%EB%8A%94-%EB%AC%B4%EC%97%87%EC%9E%85%EB%8B%88%EA%B9%8C/