---
layout: post
title:  "[Pinpoint] Docker Compose 환경에서 Pinot 연결 Timeout 트러블 슈팅"
subtitle: "EC2 재시작 시 발생하는 IP 변경 이슈와 Apache Helix 메타데이터 불일치 문제 해결"
date:   2025-05-25 23:04:00 +0900
categories: devops monitoring
tags : Pinpoint Docker Pinot DevOps Troubleshooting
---


EC2 인스턴스에 [pinpoint-docker](https://github.com/pinpoint-apm/pinpoint-docker) 리포지토리를 기반으로 Pinpoint를 구축하여 운영하던 중, 예기치 못한 장애를 겪었다.
잘 동작하던 모니터링 시스템이 EC2 인스턴스를 재시작한 직후, URI Stat 화면 등 주요 기능에 진입이 불가능해졌고 로그에는 알 수 없는 Connection Timeout 오류가 발생했다.

원인을 분석해보니 **EC2나 컨테이너가 재시작되는 과정에서 컨테이너의 IP가 변경**되었으나, **Pinot 관련 컨테이너들이 여전히 이전 IP 기반으로 요청**을 보내고 있는 것이 문제였다.

이번 포스팅에서는 해당 이슈의 원인을 파악하고 해결하기까지의 과정을 트러블 슈팅 형식으로 정리해 본다.

---

## 1. 문제 상황 

EC2 재시작 혹은 Docker 컨테이너 재시작 후, Pinpoint Web의 **Inspector**, **URL Statistics**, **Infrastructure** 등 Pinot을 사용하는 탭 진입 시 데이터가 조회되지 않고 에러가 발생했다.

### 에러 로그
Pinpoint Web 로그를 확인해 본 결과, 특정 IP로의 연결 시도가 타임아웃되고 있었다.

```log
Caused by: java.net.ConnectException: connection timed out after 2000 ms: /172.24.0.9:8099
  at org.asynchttpclient.netty.channel.NettyConnectListener.onFailure(...)
  ...
Caused by: io.netty.channel.ConnectTimeoutException: connection timed out after 2000 ms: /172.24.0.9:8099
```

---

## 2. 원인 분석

### 2.1. 컨테이너 IP 변경 확인

Connection Timeout 에러가 발생하는 `172.24.0.9` IP가 어떤 컨테이너인지 확인이 필요했다. 해당 IP가 사용하는 포트가 `8099`인 점을 미루어보아, `docker-compose` 설정상 `8099` 포트를 사용하는 `pinot-broker-0` 컨테이너가 이전에 할당받았던 IP였음을 짐작할 수 있었다.

```yaml
# docker-compose-metric.yml
pinot-broker-0:
  image: apachepinot/pinot:1.0.0-11-amazoncorretto
  restart: unless-stopped
  command: StartBroker -zkAddress pinot-zoo
  depends_on:
    - pinot-controller
  expose:
    - "8099" # 8099 포트 사용
  networks:
    - pinpoint
```

실제로 `docker inspect` 명령어를 통해 현재 실행 중인 컨테이너들의 네트워크 상태를 점검해보니, `pinot-broker-0`는 이제 다른 IP를 사용하고 있었다.

```bash
$ docker inspect network pinpoint-docker_pinpoint
```

| 컨테이너 | 로그 상의 요청 IP (Stale) | 실제 현재 IP (New) |
|---------|---|---|
| pinot-broker-0 | **172.24.0.9** | **172.24.0.11** |

즉, **컨테이너 재시작 과정에서 IP가 `172.24.0.11`로 변경되었음에도 불구하고, 애플리케이션은 여전히 과거의 IP인 `172.24.0.9`를 찾고 있는 것**이 문제의 핵심이었다.

### 2.2. 왜 과거의 IP를 찾고 있을까? (Apache Pinot의 기본 동작)

단순히 컨테이너를 재시작해도 문제는 해결되지 않았지만, **Docker Volume까지 모두 삭제하고 초기화**하니 정상 동작하는 것을 확인했다. 이는 **"Volume에 저장된 메타데이터가 IP 변경을 반영하지 못하고 있다"**는 것을 의미한다.

Pinot과 관련한 배경 지식도 없었기 때문에 이런 저런 레퍼런스를 찾아봤지만, 결과적으로 이유는 간단했다.
**Apache Pinot은 기본적으로 인스턴스를 등록할 때 IP 주소를 사용**하기 때문이다.

1.  **Pinot의 인스턴스 등록 방식**: Pinot은 클러스터에 참여하는 Broker, Server 등의 컴포넌트를 식별할 때, 별도의 설정이 없으면 **IP 기반의 ID**를 생성하여 등록한다 (예: `Broker_172.24.0.9_8099`).
    *   Pinot 공식 문서에 따르면, `pinot.set.instance.id.to.hostname` 설정이 `false`(기본값)일 경우, **Hostname 대신 IP Address를 식별자로 사용**한다고 명시되어 있다 ([Pinot Controller 문서 참고](https://docs.pinot.apache.org/configuration-reference/controller), [Advanced Pinot Setup 문서 참고](https://docs.pinot.apache.org/developers/advanced/advanced-pinot-setup)).
2.  **메타데이터의 불일치**: 컨테이너가 재시작되면서 IP가 `172.24.0.9` → `172.24.0.11`로 변경되었지만, 메타데이터 저장소(ZooKeeper)에는 여전히 **과거 IP(`172.24.0.9`)로 된 인스턴스 정보**가 남아있게 된다.
3.  **결과**: Pinpoint Web은 이 '죽은' 메타데이터를 참조하여 존재하지 않는 `172.24.0.9`로 연결을 시도하다가 Timeout이 발생한 것이다.

Pinot 커뮤니티에서도 "IP가 변경되면 Instance ID가 달라지므로, 기존 정보를 정리하거나 별도의 조치가 필요하다"고 언급하고 있다 ([GitHub Issue #9793](https://github.com/apache/pinot/issues/9793), [#4525](https://github.com/apache/pinot/issues/4525)). 결국 유동적인 IP 환경인 Docker Compose와, 고정된 정보를 기대하는 Pinot의 기본 설정이 충돌하여 발생한 문제다.

---

## 3. 해결 방법 

이 문제를 근본적으로 해결하기 위해 **Docker Compose 환경에서 각 컨테이너의 IP를 고정(Fixed IP)**하는 방식을 적용했다. IP가 고정되면 재시작 시에도 메타데이터 불일치가 발생하지 않는다.

### 3.1. .env 파일 설정
`docker-compose`에서 사용할 서브넷과 각 컨테이너별 고정 IP를 환경변수로 정의한다.

```env
# .env 파일

# Pinpoint Network Subnet
PINPOINT_NETWORK_SUBNET=172.24.0.0/27

# Collector IP
COLLECTOR_FIXED_IP=172.24.0.30

# 추가된 고정 IP 설정
PINPONT_WEB_FIXED_IP=172.24.0.10
PINPONT_PINOT_BROKER_FIXED_IP=172.24.0.11
PINPONT_PINOT_SERVER_FIXED_IP=172.24.0.12
PINPONT_PINOT_CONTROLLER_FIXED_IP=172.24.0.13
PINPONT_HBASE_FIXED_IP=172.24.0.14
PINPONT_REDIS_FIXED_IP=172.24.0.15
PINPONT_ZOO1_FIXED_IP=172.24.0.16
PINPONT_ZOO2_FIXED_IP=172.24.0.17
PINPONT_ZOO3_FIXED_IP=172.24.0.18
PINPONT_PINOT_ZOO_FIXED_IP=172.24.0.19
PINPONT_KAFKA_FIXED_IP=172.24.0.20
PINPONT_MYSQL_FIXED_IP=172.24.0.21
```

### 3.2. docker-compose.yml 수정
정의한 환경변수를 사용하여 `ipv4_address`를 각 서비스에 할당한다.

```yaml
version: "3.7"

services:
  pinpoint-hbase:
    container_name: "${PINPOINT_HBASE_NAME}"
    image: "pinpointdocker/pinpoint-hbase:${PINPOINT_VERSION}"
    networks:
      pinpoint:
        ipv4_address: ${PINPONT_HBASE_FIXED_IP}  # 고정 IP 할당
    environment:
      # ...

  pinot-controller:
    image: apachepinot/pinot:1.0.0-11-amazoncorretto
    restart: unless-stopped
    command: StartController -zkAddress pinot-zoo
    depends_on:
      - pinot-zoo
    networks:
      pinpoint:
        ipv4_address: ${PINPONT_PINOT_CONTROLLER_FIXED_IP} # 고정 IP 할당
    volumes:
      - pinot-controller-volume:/tmp/data/controller
    # ...
```

위와 같이 설정 후 `docker-compose up -d`를 수행하면, 컨테이너가 아무리 재시작되어도 항상 동일한 IP를 유지하게 되어 Connection Timeout 문제가 해결된다.

---

## 4. 마치며

이번 트러블 슈팅을 통해 **Stateful한 분산 시스템을 컨테이너 환경에서 운영할 때의 주의점**을 다시 한번 상기할 수 있었다.

1.  **IP 의존성 제거**: 가능하다면 IP 대신 **Hostname**을 사용하여 인스턴스를 식별하도록 설정하는 것이 더 유연한 방법이다. Pinot의 경우 `pinot.set.instance.id.to.hostname=true` 옵션을 통해 이를 지원한다.
2.  **환경의 이해**: Docker Compose의 기본 네트워크 동작(IP 유동성)을 이해하고, ZooKeeper와 같이 **상태를 영구 저장**하는 시스템과 결합될 때 발생할 수 있는 부작용을 사전에 고려해야 한다.

결과적으로 고정 IP 설정을 통해 안정적인 운영 환경을 확보할 수 있었지만, 근본적으로는 **단일 인스턴스 운영의 한계**를 느낄 수 있었다.

초기 도입 시에는 빠른 구축(Quick Start)을 위해 편의상 Docker Compose 기반의 단일 인스턴스로 구성했지만, 운영하다 보니 이러한 안정성 이슈들이 발생하고 있다. 앞으로는 안정적인 운영과 확장성을 위해 **Kubernetes(Helm) 기반의 클러스터 환경**으로 이전을 고려해봐야겠다. 이번 경험은 "운영 편의성"과 "시스템 안정성" 사이의 트레이드오프를 다시 한번 고민하게 해 준 계기가 되었다.