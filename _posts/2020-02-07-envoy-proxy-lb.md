---
layout: single
title:  "Envoy proxy로 GRPC 서비스 부하 분산"
date: 2020-02-07
tags: [envoy, lb, grpc]
categories: envoy

---

해당 포스팅은 MSA로 구성된 서비스 간 envoy를 통한 부하분산을 어떻게 설정했는지 공유하고자 합니다.

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
- http2_protocol_options.stream_error_on_invalid_http_messaging: grpc stream을 사용할경우 잘못된 HTTP 메시징 또는 헤더의 허용 여부입니다. `false` 지정할 경우 잘못된 HEADERS 프레임이 전달 오면 HTTP/2 연결이 종료 됩니다. 운영환경에서는 true로 처리하여 서비스 내에서 예외를 커스텀하는것이 좋은 전략입니다.
  (참고: https://tools.ietf.org/html/rfc7540#section-8.1)
- **connect_timeout**: 요청에 대한 응답 대기 시간으로 짧은 시간은 실제 운영 환경에선 권장하지 않습니다.
- **type**: 서비스 검색에 따른 유형, 정적 자원에 대한 부하 분산이기 때문에 `static` 으로 설정 하였습니다.
- **lb_policy**: 부하 분산에 대한 정책 (default: ROUND_ROBIN)
- **lb_endpoints**: 부하 분산을 

위 같이 설정할 경우 lb_policy 에 맞게 알아서 아래 endpoints 에 가중치를 두어 부하분산을 처리하게 됩니다. 
그러나 이렇게만 설정할 경우에는 실제로 endpoints 중에 서비스 하나가 죽을 경우에도 감지하지 못하고 분산이 이루어지기 때문에 
부가적인 설정이 필요합니다.

Health Check 를 통한 Endpoint 관리
---
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


Outlier detection 를 통한 Endpoint 관리
---

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