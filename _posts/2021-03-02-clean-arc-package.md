---
layout: single
title:  "프로젝트 패키지 구조 어떻게 설계 해야 할까?"
date: 2021-03-02
tags: [cleancode, cleanarchitecture, 마틴파울러, architecture, package]
categories: common
---
_해당 포스팅은 마틴 파울러의 **클린 아키텍처** 서적을 기반으로 정리하였습니다._

## 들어가기 앞서서

우리는 신규 프로젝트를 진행하게 되면 어떤 기술 스택을 선택하든지 늘 맞닥뜨리는 일이 있습니다. 바로 프로젝트 패키지 구조 설계에 대해서 입니다.

대개는 전통적인 MVC 구조를 혹은 DDD 기반의 패키지 구조화를 선택하게 되는데 사실 어떤 동료가 왜 _"이런식으로 설계를 하셨나요?"_ 에 대한 질문에 대해 _"아 디자인 패턴이 어찌구...", "예전부터 쓰는 보편화된 방식 어쩌구.."_ 두리뭉실한 대답을 할때가 많습니다.

따라서 이번 포스팅을 통해 설계에 있어 보다 **명확한 근거에 따른 설계 방식들**에 대해 설명하고자합니다. 

**소프트웨어의 설계에서 중요한 부분**은
- 올바르게 정의된 경계
- 명확한 책임
- 통제된 의존성을 가진 클래스와 컴포넌트

위의 3개로 크게 압축할수 있는데, 사실 이론적인 부분을 알면서도 프로젝트를 중간단계를 거치게 되면서 무너지는 경우가 많이 있습니다.
이러한 소프트웨어의 설계 원칙을 최대한 유지하기 위해 아래 몇가지 프로젝트 패키지의 설계나 코드 조직화와 관련된 접근법을 소개하고자 합니다.   


## 계층 기반 패키지

가장 단순한 첫번째 설계 원칙은 전통적인 수평 계층형 아키텍처입니다.

![<1>](/assets/images/2021-03-02-clean-arc-package/1.png){: width="50%"}{: .center}

기술적인 관점에서 해당 코드가 하는 일에 기반해 그 코드를 분할 합니다. 보통 웹베이스 코딩을 하게 되면 `MVC`, `MVP` 패턴으로 개발을 하게 되는데 이도 '`계층 기반 패키지`' 라고 부르며 각 계층은 유사한 종류의 것들을 묶는 도구로 사용됩니다.

이 전형적인 아키텍처에서는 '사용 인터페이스', '업무규칙', '영속성코드' 를 계층이 각각 하나씩 존재하며, 임의의 N계층은 반드시 N+1 계층에만 의존해야 합니다.

- **OrdersController**: 웹 컨트롤러이며, 웹기반 요청 및 응답을 처리한다.
- **OrdersService**: 주문과 관련된 '업무규칙', 유즈케이스를 정의한다.
- **OrdersRepository**: 주문정보와 관련된 영구 저장된 주문 정보에 접근하는 방법을 정의한다.

> 이 아키텍처는 엄청난 복잡함을 겪지 않고도 무언가를 작동시켜 주는 아주 빠른 방법입니다.

문제는 소프트웨어의 규모가 커지고 복잡해지기 시작하면, 레이어로 분리된 패키지에 모든 코드를 담기 부족하다는 사실을 깨닫고 더 잘게 모듈화 해야 할지를 고민하게 될 상황에 놓이게 됩니다.

계층형 아키텍처는 업무 도메인에 대해 아무것도 말해주고 있지 않아서, 컨트롤러, 서비스, 리포지터리로 구성된 패키지들이 **기분 나쁠정도로 비슷하게 보이게 됩니다.**

또한 수평 계층으로 분리가 되어 있다보니 '**사용 인터페이스(service)**' 구간에서 곧바로 '**영속성코드(repository)**' 에 접근이 가능한데, 이는 특정한 상황(CQRS 정책) 이외에는 **엄격하게 지켜져야 합니다.**

## 기능(도메인) 기반 패키지

기능 기반 패키지는 서로 연관된 **기능, 도메인 개념** 또는 **Aggregate Root(도메인 주도 설계 용어)** 에 기반하여 수직의 얇은 조각으로 코드를 나누는 방식입니다.

특정 기능을 지칭하는 하나의 패키지 내에 모든 타입과,개념을 반영해서 구현합니다.

![<2>](/assets/images/2021-03-02-clean-arc-package/2.png){: width="50%"}{: .center}

위의 그림에서 보듯이 등장하는 인터페이스와 클래스가 모두 **단 하나의 패키지**에 속합니다. 이로 인해 코드의 상위 수준 구조가 업무 도메인에 대해 무언가를 알려주게 됩니다.

따라서 이 코드베이스가 계층단위(웹, 서비스, 리포지토리)가 아니라 **주문(업무 도메인)과 관련된 무언가**를 한다는 것을 볼 수 있습니다.

또 다른 이점으로는 '주문 조회하기' 유스케이스가 변경될 경우 변경해야 할 코드를 모두 찾는 작업이 쉬워질 수 있습니다. 
> 변경해야 할 코드가 모두 한 패키지 내에 응집력을 갖고 있기 때문입니다.

## 포트와 어댑터

**엉클 밥**에 따르면 `포트와 어댑터` 혹은 `육각형 아키텍처`, `'경계, 컨트롤러, 엔티티'`, 등의 방식으로 접근하는 이유는 업무/도메인에 초점을 둔 코드가 **기술적인 세부 구현과 독립적이며 분리된 아키텍처**를 만들기 위해서 입니다.

![<2>](/assets/images/2021-03-02-clean-arc-package/3.png){: width="50%"}{: .center}

'**내부**' 영역은 도메인 개념을 모두 포함하는 반면, '**외부**'영역은 UI, 데이터베이스, 서드파티등과의 상호작용을 포함합니다.

여기서 중요한 규칙은 '외부'가 '내부'에 의존하며, **절대 그 반대로는 안된다는 점입니다.** `( 의존 방향: 도메인 <-- 인프라 )`

![<2>](/assets/images/2021-03-02-clean-arc-package/3-2.png){: width="50%"}{: .center}

도메인 주도 설계에서는 OrderRepository 는 Orders 라는 명칭으로 바꿔줘야 합니다. 그 이유는 도메인에 대해 논의할때 우리는 '**주문**'에 대해 말하는것이지 '**주문 리포지토리**'에 대해 말하는것이 아니기 때문입니다.

또한 도메인 주도 설계에서는 '내부'에 존재하는 모든 것의 이름은 반드시 '**유비쿼터스 도메인 언어**' 관점에서 기술하라고 조언합니다.

> **유비쿼터스 도메인 언어란?** 업무 간 도메인 전문가, 아키텍트, 개발자 등 프로젝트 구성원 모두에게 공유된 언어를 뜻합니다.


## 컴포넌트 기반 패키지

**컴포넌트 기반 패키지**는 저자인 마틴 파울러가 제시하는 방법입니다. 

지금까지 위에서 설명한 방식을 혼합한 것으로 큰 단위의 단일 컴포넌트와 관련된 모든 책임을 하나의 패키지로 묶는데 주안점을 두고 있습니다.

이 접근법은 **서비스 중심적인 시각**으로 소프트웨어 시스템을 바라 보며, 마이크로서비스 아키텍처가 가진 시각과도 동일합니다.

![<2>](/assets/images/2021-03-02-clean-arc-package/4.png){: width="50%"}{: .center}

여기서 말하는 '**컴포넌트**'에 대해 마틴파울러는 **_"컴포넌트는 멋지고 깔끔한 인터페이스로 감싸진 연관된 기능들의 묶음으로, 애플리케이션과 같은 실행 환경 내부에 존재한다."_** 로 정의하고 있습니다.

컴포넌트 기반 패키지 접근법의 주된 이점은 주문과 관련된 무언가를 코딩해야 할 때 오직, **OrdersComponent만** 둘러보면 된다는 점입니다. 이 컴포넌트 내부에서 관심사의 분리는 여전히 유효하며, 컴포넌트 구현과 관련된 세부사항은 사용자가 알 필요가 없습니다.

이는 **마이크로 서비스**나 **서비스 지향 아키텍처**를 적용했을 때 얻는 이점과도 유사합니다. 따라서 모노리틱 애플리케이션에서 컴포넌트를 잘 정의하면 **결합 분리 모드**를 통해 마이크로 서비스 아키텍처로 가기 위한 발판으로 삼을 수 있습니다.

## 결론

해당 포스팅은 최적의 설계를 했다고 하더래도, 구현 전략에 얽힌 복잡함을 고려하지 않으면 **순식간에 망가질 수 있다는 사실을 강조**하는데 그 목적이 있으며,

소프트웨어의 구조를 유지하기 위해 설계를 어떻게 해야만 원하는 코드 구조로 매핑할 수 있을지, 그 코드를 어떻게 조직화할지, 런타임, 컴파일타임에 어떤 결합 분리 모드를 적용할지 고민해야 합니다.

가능하다면 **선택사항을 열어두되, 실용주의적으로 행해야 합니다.**
`(프로젝트의 팀의 규모, 기술 수준, 일정, 예산이라는 제약을 늘 동시에 고려해야 합니다.)`

**구현 세부사항에는 항상 문제가 있는 법 입니다.**