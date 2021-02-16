---
layout: single
title:  "앤서블(Ansible)을 통한 인프라 자동화 하기"
date: 2021-02-15
tags: [ansible, infra, provisioning]
categories: ansible
---

해당 포스팅은 Ansible 의 전반적인 사전 지식을 익히기 위해 기록했습니다.

Ansible이란?
---
python 으로 구현된 오픈소스로써 서버의 프로비저닝, SW 배포 등의 자동화를 관리해주는 도구(Infrastructure as Code)입니다.

![2](/assets/images/2021-02-15-ansible-basic/1.png){: width="50%" }{: .center}

> **Infrastructure as a Code(IaC)란?** 컴퓨터의 인프라 구성을 소프트웨어를 개발 하듯이 코드로 작성하는 것을 의미합니다. Ansible 또한 이런 개념이 도입 되어 규격화된 환경과 묘듈 구성을 통해 배포되는 모든 서버가 동일한 환경을 유지할 수 있도록 해줍니다.

Ansible의 특징
---
- **Idempotency(멱등성)** — 어떤 연산이 여러번 수행되더라도 결과가 달라지지 않는 성질을 의미합니다. Ansible 또한 동일한 모듈을 반복 실행해도 결과가 동일하게 출력시켜, 결과가 달라지지 않도록 구성되어 일관되게 수행할 수 있습니다.
- **Agentless** — 타 자동화 도구들은 타겟 대상들에 Agent 설치 기반 PULL 방식으로 동작 하는 것에 비해 Ansible은 타겟 대상들에 Agentless 기반의 PUSH 방식으로 동작하기 때문에 기술적, 지리적 제한이 보다 넓다는 장점이 있습니다.

> 물론 **Agentless방식**이 **Agent방식**에 비해 장점만 있는것은 아닙니다. **Agentless방식**은 PUSH 기반으로 동작하기 때문에 관리,사용성 측면은 올라가지만 전체 서버들을 접근 하는 master 역할을 하는 서버가 존재함으로써 보안적인 측면에서는 좋지 않을수 있습니다. 반대로 **Agent방식**의 puppet 나 chef 같은 도구들은 관리 대상 서버에 별도의 프로그램을 설치하며 강력한 기능을 제공하지만 반대로 복잡한 설치 및 관리를 해야합니다.  

Ansible의 구성 요소
---
Ansible은 인프라 구성을 위해 아래와 같이 3가지의 논리적 요소를 코드로 구현하게끔 되어 있습니다.

- 어디서(Where): `인벤토리(Inventory)`
- 무엇을(What): `플레이북(Playbook)` 
- 어떻게(How): `모듈(Module)`

### 1. 인벤토리 (Inventory)

기본 파일명은 `hosts.ini`이며 프로비저닝,배포등의 제어될 대상을 정의한 파일입니다. 대상들에게 별명을 붙이거나 그룹으로 묶을수도 있고 ssh 접근방식(`IP`,`PORT`,`USER`)을 설정 할 수 있습니다.

**[예제 코드]**

```yaml
[WebServers] # 호스트 그룹핑
server0 ansible_host=127.0.0.1  ip=182.162.176.170  
server1 ansible_host=172.18.0.1  ansible_port=20101

[DBServers] # 호스트 그룹핑
server3 ansible_host=172.22.233.1  ansible_port=20101

[Group:children] # 위의 두개 그룹 Group란 이름으로 병합 
WebServers
DBServers
```

### 2. 플레이북 (Playbook)
인벤토리에 작성된 서버들을 대상으로 특정 행위(프로비저닝,배포등등)에 대해 정의한 파일입니다. 보통 플레이북을 작성한다고 하면 YAML 파일에 작성하는 것인데 인벤토리에 기록된 그룹 및 별명을 통해 여러 Task를 묶어서 사용 가능합니다.

**[예제 코드]**
```yaml
# 웹 서비스 재시작
- name: WebService Restart
  hosts: WebServers # 인벤토리에서 설정한 그룹명 혹은 호스트명 지정
  gather_facts: no # 위의 host의 접근이 실제로 가능한지 여부 ( no로 정의할 경우 host 체크를 하지 않아서 성능향상을 높일 수 있다. ) 
  roles:
    - webservice/stop
    - webservice/start

# 전체 서버 프로비저닝 
- hosts: Group
  become: true // (true|false|yes|no) 특정 사용자로 전환 여부
  become_user: other_user
  gather_facts: no
  roles:
    - common/install_plugin
    - common/script_deploy
    - common/other_task
```

### 3. 모듈(Module)

모듈은 플레이북에서 실행되는 `roles`에서 실행되는 Task 단위입니다. 
예를 들어서 쉘 스크립트를 동작하거나 웹서비스를 실행한다거나, 혹은 apt 패키지를 설치하는 등의 단일 작업을 의미합니다.