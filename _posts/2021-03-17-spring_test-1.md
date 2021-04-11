---
layout: single
title:  "Spring Boot 에서 Mock을 통한 단위 테스트 적용하기"
date: 2021-04-11
tags: [java, spring, boot, tdd, testing, mockito, mock, stub]
categories: monitoring
---

이번 포스팅에서는 Spring F/W 내에서 Mock을 통한 단위테스트 구현에 대해서 간단하게 알아봅니다.

들어가기 전에 Mock이 무엇인지에 대해 간략하게 설명합니다.

## Mock Test란?

네트워크, 데이터베이스등 제어하기 어려운 것들에 의존하고 있는 메서드를 테스트 하고 검증하기 위해 가짜 객체를 만들어 사용하는 방법입니다.

## Mock 객체는 언제 필요한가?

- 테스트 작성을 위한 환경 구축이 어려운 경우
- 테스트가 필요한 특정 메소드 내에 타 외부 서비스 혹은 미들웨어에 의존적일때
- 단위 테스트의 특성상 반복적이며, 자가검증이 가능해야하기 위해 테스트 시간을 단축시키기 위해

> 보통 **Stub 객체**와 **Mock 객체**가 자주 혼용되서 사용되는데, **Stub 객체**의 경우 상태검증에 사용되고 **Mock 객체**는 행위 검증에 사용 되기 때문에 용어 사용 시 구별지어야 한다.

## Mockito 프레임워크
그러나 실제 프로젝트 시에 단위테스트를 위한 Mock, Stub 객체를 만드는 일은 굉장히 번거롭습니다. 때문에 Java 진영에서는 Mockito 프레임워크를 통해 편리하게 작성할수 있도록 도와줍니다.

Mockito는 Spring boot 기준으로 `spring-boot-starter-test` 의존성 추가해 주면 됩니다.

mock에는 크게 2개의 어노테이션인 `@Mock`, `@InjectMocks`를 제공하고 있습니다.

- `@ExtendWith(MockitoExtension.class)`: Mockito의 Mock 객체를 사용하기 위한 어노테이션으로 Test class 위에 작성
- `@Mock`: mock 객체를 만들어 반환
- `@InjectMocks`: @Mock 객체를 주입할 주체 객체로 선언

## Mockito 테스트 코드 예제

예를 들어서 투자 서비스의 Service Layer를 테스트 한다고 가정했을때 실제 예제 코드입니다.
```java
...
import static org.mockito.BDDMockito.given;

@ExtendWith(MockitoExtension.class)
public class InvestServiceTest {
    
    @InjectMocks
    private InvestService investService;
    
    @Mock
    private InvestRepository investRepository;
    
    @Test
    public void 전체_투자_상품_조회() throws Exception {
        //given
        Pageable pageable =  PageRequest.of(0, 10, Sort.by("createdAt").descending());
        SearchFilter<Product> filter = new SearchFilter();
        Product product = new Product(1L, "test_product", 1000L, 50L, 0L, ProductType.Recruiting, new Date(), new Date(),new Date());
        List<Product> products = new ArrayList<>();
        products.add(product);
        Page<Product> d = new PageImpl<>(products);
        given(investRepository.findAll(filter.build(), pageable)).willReturn(d);
        
        //when
        Page<Product> result = investService.getInvestProducts(pageable, filter.build());

        //then
        System.out.println("first product = " + result.getContent().get(0));
        Assertions.assertEquals(result.getContent().get(0), products.get(0));
    }
...
```
테스트 성공 후 콘솔 정보입니다.
```java
first product = Product(id=1, title=test_product, totalInvestingAmount=1000, currentInvestingAmount=50, userCount=0, type=Recruiting, startedAt=Sun Apr 11 22:56:42 KST 2021, finishedAt=Sun Apr 11 22:56:42 KST 2021, createdAt=Sun Apr 11 22:56:42 KST 2021)
```
`@InjectMocks` 어노테이션을 통해 InvestService 맴버변수 내에 가짜 객체를 주입하기 위해 선언합니다.
그런 다음 실제 테스트 대상이 아닌 InvestRepository 맴버변수는 `@Mock` 어노테이션을 통해 가짜 객체로 사용합니다.

given 함수를 통해 `investRepository.findAll` 행위에 대해 InvestService 내부에서 호출 될때 리턴되는 값을 생성합니다.

이로 인해 실제 검증하고자 하는 `investService.getInvestProducts()` 함수에 대해 특정 네트워크 및 DB 연결및 데이터등의 의존성과 상관 없이 테스트가 가능해졌습니다.

## 결론 
요즘 같은 오픈소스 시대에서는 정말 유용하고 다양한 라이브러리,F/W 넘쳐나고 있습니다. 이때 중요한건 무엇을 사용할지에 대한 **수단** 보다 근본적으로 문제가 되는 **원인**과 해결하고자하는 **목적**가 분명해야 합니다.
테스트도 마찬가지라고 생각합니다. 

어떤 도구를 사용해서 테스트 코드를 작성할지 보다, 그전에 프로젝트를 하는 과정에서 <U>무엇을 테스트 할지</U>, 해당 테스트 코드를 통한 검증이 <U>코드 커버리지와 비례해 서비스의 신뢰성을 높여주는가</U>등의 이야기가 더 중요하지 않을까 싶습니다.












