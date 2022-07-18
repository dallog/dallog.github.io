---
title: "달록에 적절한 패키지 구조 고민하기"
date: 2022-07-18 12:00:00
update: 2022-07-18 12:00:00
tags:
  - 패키지 구조
---

> 이 글은 우테코 달록팀 크루 [매트](https://github.com/hyeonic)가 작성했습니다.

## 달록에 적절한 패키지 구조 고민하기

우리는 프로젝트를 진행하며 어떠한 패키지 구조를 구성할지 고민하게 된다. 보통 패키지 구조를 나누는 대표적인 방법으로 `계층별`, `기능별`로 나눌 수 있다.

### 계층별 패키지 구조

```json
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── allog
    │   │           └── dallog
    │   │               ├── application
    │   │               ├── config
    │   │               ├── domain
    │   │               ├── dto
    │   │               ├── exception
    │   │               ├── presentation
    │   │               └── DallogApplication.java
    │   └── resources
    │       └── application.yml
```

계층형 구조는 각 계층을 대표하는 패키지를 기준으로 코드를 구성한다. 계층형 구조의 가장 큰 장점은 해당 프로젝트에 대한 이해도가 낮아도 각 계층에 대한 역할을 충분히 숙지하고 있다면 전체적인 구조를 빠르게 파악할 수 있다.

하지만 단점도 존재한다. 하나의 패키지에 모든 클래스들이 모이게 되기 때문에 규모가 커지면 클래스의 개수가 많아져 구분이 어려워진다. 아래는 이전에 계층형 구조를 기반으로 작성한 프로젝트이다.

```json
└── presentation
    ├── ChampionController.java
    ├── CommentController.java
    ├── PlaylistController.java
    ├── RankingController.java
    ├── SearchController.java
    ├── UserController.java
    ├── WardController.java
    └── ...
```

비교적 적은 코드의 양이지만 규모가 커질수록 애플리케이션에서 presentation 계층에 해당하는 모든 객체가 해당 패키지에 모이게 될 것이다.

### 기능별 패키지 구조

기능별로 패키지를 나눠 구성한다. 기능별 패키지 구조의 장점은 해당 도메인에 관련된 코드들이 응집되어 있다는 점이다. 덕분에 `지역성의 원칙`를 잘 지킬 수 있다고 한다.

> 컴퓨터 과학에서, 참조의 지역성, 또는 지역성의 원칙이란 프로세서가 짧은 시간 동안 동일한 메모리 공간에 반복적으로 접근하는 경향을 의미한다. 참조 지역성엔 두 가지 종류가 있다. 바로 시간적 지역성과 공간적 지역성이다. 시간적 지역성이란 특정 데이터 또는 리소스가 짧은 시간 내에 반복적으로 사용되는 것을 가리킨다. 공간적 지역성이란 상대적으로 가까운 저장 공간에 있는 데이터 요소들이 사용되는 것을 가리킨다.

개발자 역시 복잡하고 거대한 프로젝트의 전체 구조를 모두 인지하는 것은 힘든일이다. 우선 특정 지역의 흐름을 파악할 수 있다면 해당 패키지에 대해서는 마치 캐시에 적재된 데이터에 접근 하듯 빠르게 인지가 가능하다.

```json
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── allog
    │   │           └── dallog
    │   │               ├── auth
    │   │               │   ├── application
    |   |               |   ├── domain
    │   │               │   ├── dto
    │   │               │   ├── exception
    │   │               │   └── presentation
    │   │               ├── category
    │   │               │   ├── application
    |   |               |   ├── domain
    │   │               │   ├── dto
    │   │               │   ├── exception
    │   │               │   └── presentation
    │   │               ├── schedule
    │   │               │   ├── application
    |   |               |   ├── domain
    │   │               │   ├── dto
    │   │               │   ├── exception
    │   │               │   └── presentation
    │   │               ├── global
    │   │               │   ├── config
    │   │               │   ├── dto
    │   │               │   ├── error
    │   │               │   └── exception
    │   │               ├── infrastructure
    │   │               │   └── oauth
    │   │               └── AllogDallogApplication.java
    |   |
    │   └── resources
    │       └── application.yml
```

하지만 각 계층이 기능별로 모여 있기 때문에 프로젝트에 대한 이해도가 낮으면 전체 구조를 파악하는데 오랜 시간이 걸린다.

### 더 나아가기

현재 구조에서는 도메인에 해당하는 다양한 기능들이 패키지 전반적으로 퍼져 있다. 각각의 도메인 기능이 밀집되어 있지 않은 구조를 가지고 있다.

```json
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── allog
    │   │           └── dallog
    │   │               ├── domain
    │   │               │   ├── auth
    │   │               │   │   ├── application
    |   |               │   |   ├── domain
    │   │               │   │   ├── dto
    │   │               │   │   ├── exception
    │   │               │   │   └── presentation
    │   │               │   ├── category
    │   │               │   │   ├── application
    |   |               │   |   ├── domain
    │   │               │   │   ├── dto
    │   │               │   │   ├── exception
    │   │               │   │   └── presentation
    │   │               │   └── schedule
    │   │               │       ├── application
    |   |               │       ├── domain
    │   │               │       ├── dto
    │   │               │       ├── exception
    │   │               │       └── presentation
    │   │               ├── global
    │   │               │   ├── config
    │   │               │   ├── dto
    │   │               │   ├── error
    │   │               │   └── exception
    │   │               ├── infrastructure
    │   │               │   └── oauth
    │   │               └── AllogDallogApplication.java
    |   |
    │   └── resources
    │       └── application.properties
```

### domain

실제 애플리케이션의 핵심이 되는 도메인 로직이 모여 있다. 애플리케이션의 주요 비즈니스 로직이 모여 있기 때문에 `외부와의 의존성을 최소화`해야 한다. 즉 `외부의 변경에 의해 도메인 내부가 변경되는 것을 막아야 한다는 것`을 인지해야 한다.

### global

`global`은 프로젝트 전반에서 사용하는 객체로 구성한다. 공통적으로 사용하는 dto나 error, config에 대한 것들이 모여 있다.

### infrastructure

`infrasturcture`는 외부와의 통신을 담당하는 로직들이 담겨 있다. 이번 프로젝트에서는 OAuth를 활용한 회원 관리를 진행하기 때문에 google의 인증 서버와 통신이 필요해진다. 이 패키지는 우리의 의지와 다르게 외부의 변화에 따라 변경될 여지를 가지고 있다. 즉 변화에 매우 취약한 구조이며 외부 서버에 의존적 이기 때문에 항시 변화에 대응할 수 있도록 대비해야 한다. 이것이 의미하는 바는 결국 `도메인 관련 패키지에서 infrastructure를 직접적으로 의존하는 것`은 도메인 로직을 안전하게 지킬 수 없다는 의미를 내포한다.

## 정리

각각의 방법은 서로 다른 장단점을 가지고 있기 때문에 정답은 없다고 생각한다. 현재 프로젝트의 규모와 요구사항을 고려하여 선택해야 한다. 다만 선택한 패키지 구조에 충분한 근거를 가져야하고, 객체와 패키지 사이의 의존성에 대해 충분히 고민해야 한다.

달록은 현재 기능별 패키지 구조로 진행되고 있다. 모든 팀원들이 처음 부터 기획과 설계에 대한 고민을 진행했기 때문에 프로젝트의 구조에 대해 분석 하는 시간이 불필요했기 때문이다. 하지만 각각의 기능들이 난잡하게 퍼져 있기 때문에 [더 나아가기](#더-나아가기)에서 언급한 것 처럼 좀 더 밀접한 기능들을 모아둘 필요가 있다고 판단한다. 이것은 추후 팀원들과 충분한 논의를 통해 개선해갈 예정이다.

## References.

[Spring Guide - Directory](https://cheese10yun.github.io/spring-guide-directory)<br>
[지역성의 원칙을 고려한 패키지 구조: 기능별로 나누기](https://ahnheejong.name/articles/package-structure-with-the-principal-of-locality-in-mind)
