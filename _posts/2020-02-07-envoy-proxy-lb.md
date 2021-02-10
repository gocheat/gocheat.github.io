---
layout: single
title:  "On-Premise 환경에서 Envoy로 GRPC 서비스 부하 분산 구성"
date: 2020-02-07
tags: [envoy, lb, grpc]
categories: envoy

---

해당 포스팅은 MSA로 구성된 서비스 간 GRPC 통신을 Envoy로 부하분산을 어떻게 설정했는지 기록했습니다.

_(본 포스트는 envoy 1.17 버전(V3)으로 작성하였습니다.)_ 

개요
---

우선 개발중인 서비스들의 인터페이스는 아래와 같습니다.
```
유저 --RESTAPI-->서비스--GRPC--->내부서비스
                 ㄴ----GRPC--->내부서비스
                 ㄴ----GRPC--->내부서비스
                 ...
```
기본적으로 MSA 패턴으로 각 주요 도메인이 서비스 단위로 분리 되어 있지만, 
서비스와 내부서비스간의 통신이 단일로 되어 있다보니 하나의 서비스가 죽게 되면 복구될 여지없이 그대로 장애로 이어지게 되었습니다.

또한 컨테이너 운영환경(쿠버네티스)이 아닌 정적인 on-premise 환경이라는 인프라적인 제약 사항이 있어 많은 고민이 있었습니다. 
( 또한 이후에 쿠버네티스 운영환경으로의 전환 계획이 있었기 때문에 많은 시간을 할애하고 싶지 않았음..ㅎ ) 


따라서 한정된 자원에서 고가용성 작업을 해야했었는데 정리하자면 요구사항은 아래와 같았습니다.

1. 서비스간 GRPC 통신에 대해 부하분산이 이루어져야함
2. 서비스의 버전관리를 위해서 Path, header 값을 통한 라우팅이 되어야함 (L7)
3. 죽은 서비스에 대해서 LB 구성에서 자동으로 제외하고 복구되었을때는 다시 LB 구성에 포함이 되는 회복탄력성을 갖추어야 함
4. 부하 분산 구성을 위한 코드 작성이 없어야함

몇가지 방법론을 검색해 보았고 그중에서 위의 요구사항을 충족하는 **Envoy proxy** 라는 오픈소스 활용을 하게 되었습니다.

Envoy proxy 의 설명은 따로 정리하도록 하겠습니다.

기본적인 Envoy LB 설정
---

Envoy의 filters - router 부분에 매핑된 cluster 설정을 따라 요청에 따른 분기 처리를 합니다.
```yaml
static_resources:
    ...
  
    clusters:
      - name: internal_service_A
        connect_timeout: 30s
        type: static
        http2_protocol_options:
          stream_error_on_invalid_http_messaging: true
        lb_policy: ROUND_ROBIN
        load_assignment:
            cluster_name: internal_service_A
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: 10.0.0.1
                          port_value: 80
                  - endpoint:
                      address:
                        socket_address:
                          address: 10.0.0.2
                          port_value: 80
```
아래는 몇가지 중요한 설정에 대해 공유 드립니다.
- http2_protocol_options.stream_error_on_invalid_http_messaging: grpc stream을 사용할경우 잘못된 HTTP 메시징 또는 헤더의 허용 여부입니다. `false` 지정할 경우 잘못된 HEADERS 프레임이 전달 오면 HTTP/2 연결이 종료 됩니다. 운영환경에서는 true로 처리하여 서비스 내에서 예외를 커스텀하도록 처리하였습니다. 
  (참고: https://tools.ietf.org/html/rfc7540#section-8.1)
- **connect_timeout**: 요청에 대한 응답 대기 시간으로 짧은 시간은 실제 운영 환경에선 권장하지 않습니다.
- **type**: 서비스 검색에 따른 유형, 정적 자원에 대한 부하 분산이기 때문에 `static` 으로 설정 하였습니다.
- **lb_policy**: 부하 분산에 대한 정책 (종류: `ROUND_ROBIN`, `LEAST_REQUEST`, `RING_HASH`, `RANDOM` 등이 있습니다.)
> 각 부하 분산에 대한 알고리즘은 https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers#arch-overview-load-balancing-types 참조 

- **lb_endpoints**: lb_policy 옵션에 따라 부하분산 할 업스트림 서비스 입니다.

위 같이 설정할 경우 lb_policy 에 맞게 알아서 아래 endpoints 에 가중치를 두어 부하분산을 처리하게 됩니다. 
그러나 endpoints 중에 서비스 하나가 죽거나 혹은 복구 가능성에 대해 감지하지 못하기 때문에 서비스 에러 감지와 회복탄력성을 부여하기 위해선 추가적인 설정이 필요합니다.


Health Check 를 통한 Endpoint 관리
---
envoy에서는 Health Check 설정을 통해 LB의 endpoint 설정을 제거 혹은 복구 할 수 있게 구성을 지원하고 있습니다.
크게 `HTTP`, `L3/L4`, `Redis` 등을 지원하고 있으며 운영중인 서비스는 HTTP를 통해 200 응답을 예상할수 있습니다.

아래는 Health Check 예제 입니다.
```yaml
clusters:
    - name: internal_service_A
      health_checks:
        - http_health_check:
            path: '/heathcheck'
          timeout: 2s
          interval: 2s
          unhealthy_threshold: 1
          healthy_threshold: 1
          always_log_health_check_failures: true
          event_log_path: /var/log/envoy/health_check.log
      load_assignment:
        cluster_name: internal_service_A
        endpoints:
          - lb_endpoints:
              - endpoint:
                  health_check_config:
                    port_value: 8282
                  address:
                    socket_address:
                      address: 10.0.0.1
                        port_value: 80
              - endpoint:
                  health_check_config:
                    port_value: 8282
                  address:
                    socket_address:
                      address: 10.0.0.2
                      port_value: 80
```

- **health_checks.http_health_check**: 서비스의 상태 체크를 할 HTTP Path 입니다.
- **health_checks.interval**: 클러스터의 상태체크의 반복 주기입니다. 위의 옵션에 따르면 2초 단위로 업스트림 호스트를 체크합니다. 
- **health_checks.unhealthy_threshold**: interval 내의 비정상 상태 확인 수입니다. 해당 임계치를 넘으면 LB 구성에 제외시킵니다.
- **health_checks.healthy_threshold**: interval 내의 정상 상태 확인 수입니다. 해당 임계치를 넘으면 LB 구성에서 포함시킵니다.
  
상태 체크를 할 업스트림의 호스트를 별도로 구성할수도 있습니다.
- **health_check_config**: 위의 `health_checks` 설정을 적용할 업스트림의 host 정보


Outlier detection 를 통한 Endpoint 관리
---
Outlier detection 옵션을 통해 특정 업스트림의 오류에 대해 사전에 감지하고 예방할수 있는 기능을 제공합니다.

Envoy 문서의 이상 감지 방출 알고리즘에 따르면 아래와 같습니다.

1. `outlier_detection` 설정된 값에 따라 이상 감지를 판단합니다. (`consecutive_5xx`)
2. 방출된 호스트가 없으면  `max_ejection_percent` 에 따라 방출 할지 여부를 판단합니다. (그러나 설정상 90퍼라고 해도 최소 1개의 호스트는 무조건 배출합니다.)
3. 최초 호스트가 방출되면 `base_ejection_time` * 연속적으로 배출된 횟수를 곱한 값입니다.
4. 방출 시간이 `max_ejection_time` 에 도달하면 더 이상 시간이 증가하지 않습니다. (기본 300초)
5. 방출 시간 중에 호스트가 이상 감지에서 정상 상태로 확인 된다고 가정하면 배출 시간을 최소로 낮추는데 아래와 같은 계산식을 갖게 됩니다.

**_( outlier_detection.max_ejection_time / outlier_detection.base_ejection_time * outlier_detection.interval )_**

```yaml
clusters:
    - name: internal_service_A
      outlier_detection:
        consecutive_5xx: 3
        interval: 10s
        base_ejection_time: 10s
        max_ejection_percent: 50
      load_assignment:
        cluster_name: internal_service_A
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: 10.0.0.1
                      port_value: 80
              - endpoint:
                  address:
                    socket_address:
                      address: 10.0.0.2
                      port_value: 80
              ... 
```

아래는 탐지유형 몇가지를 소개드리고자 합니다.

- Consecutive 5xx: 업스트림 호스트의 응답 코드가 5xx 유형이 발생할 경우 이상으로 판단합니다.
- Consecutive Gateway Failure: 게이트웨이에서 연속적으로 에러(5xx 오류 하위집합, 시간초과, TCP 재설정등)가 발생할 경우 이상으로 판단합니다.
- Consecutive Local Origin Failure: 업스트림 호스트의 시간초과, TCP 재설정, ICMP 오류 등 로컬에서 발생합니다.
- Success Rate, Failure Percentage: 호스트에 대한 데이터를 집계한 후, 주어진 간격으로 통계값 기준으로 성공률과 실패율을 결정한 뒤 이상으로 판단합니다. 

이상으로 위와 같이 envoy를 통한 LB 구성 및 호스트 제거/복구 프로세스를 통해 실제 프로젝트에 적용하였습니다.

적용하면서 우여곡절이 많았던것은 오픈소스를 사용함에 있어 버전별로 설정의 이름이라던지 제공여부등 일부 상이한 부분이 있어서 본인이 사용하고 있는 버전에서 제공하는 기능 및 옵션을 꼼꼼히 파악해야함을 느낄수 있었습니다. (실제 운영할때도 잘못된 설정으로 장애가 몇번 나서 아찔한 기억이..)

참고문서
---
- <https://www.envoyproxy.io/docs/envoy/v1.17.0>