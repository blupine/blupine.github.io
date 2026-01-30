---
layout: post
title:  "[장애 회고] Redis(Lettuce) Executor 스레드 고갈과 Reactor 스레드 모델의 오해"
subtitle: "subscribeOn에 대한 오해"
date:   2025-11-12 22:13:52 +0900
categories: dev
tags : Spring Redis Reactor Lettuce TroubleShooting
---

운영하던 서비스에서 최근 Redis와 관련한 무서운 장애가 있었다.
정확히는 Reactor의 스레드 동작 방식에 대한 이해 부족으로 인해 발생했던 문제인데, 이와 관련해 기록을 남겨보려 한다.

Redis 클라이언트 라이브러리인 Lettuce와 Project Reactor를 함께 사용할 때 발생할 수 있는 문제라 정리해둔다.

---

## 1. 어떤 문제가 발생했나?

어느 날 갑자기 서비스에서 **Redis에 접근하는 모든 API가 응답이 없는(stuck)** 현상이 발생했다.
Redis 서버 자체는 멀쩡했는데, 애플리케이션 로그를 보니 아래와 같은 예외가 반복되고 있었다.

```text
WARN ... io.lettuce.core.protocol.CommandHandler : Unexpected exception during request: java.util.concurrent.RejectedExecutionException: event executor terminated

java.util.concurrent.RejectedExecutionException: event executor terminated
    at io.netty.util.concurrent.SingleThreadEventExecutor.reject(SingleThreadEventExecutor.java:926)
    at io.netty.util.concurrent.SingleThreadEventExecutor.offerTask(SingleThreadEventExecutor.java:353)
    ...
    at io.lettuce.core.RedisPublisher$PublishOnSubscriber.onError(RedisPublisher.java:931)
```

로그의 핵심은 `RejectedExecutionException: event executor terminated` 였다.
Lettuce가 내부적으로 사용하는 Event Executor(Netty 스레드)가 **종료(terminated)** 되었고, 이로 인해 더 이상 작업을 처리할 수 없어 모든 요청이 거절당하고 있었던 것이다.

이상한 건, 모든 API의 응답 코드에는 아래와 같이 `onErrorResume`으로, Redis에 장애가 발생하더라도 응답을 내려주는 코드가 있었지만, 해당 코드조차 실행되지 않았다.

```java
    public Mono<Response> getCachedData() {
      redisClient.getData()
            .map(Response::of)
            .onErrorResume(e -> Mono.just(Response.ofError(e))) // 실행 안됨.. 
    }
 ```
upstream (redisClient.getData())에서 `RejectedExecutionException`이 발생했다면, `onErrorResume`이 실행되어야 했는데, 이상한 일이었다.

---

## 2. 왜 Executor가 죽었을까? (원인 분석)

### [참고] Custom Executor 환경
우선 당시의 환경 설정을 짚고 넘어가야 할 것 같다.
서비스는 Lettuce가 Netty의 기본 EventLoop(Global Resources)를 공유하지 않고, **별도의 독립된 스레드 풀(Executor)**을 사용하도록 구성되어 있었다.

```java
// Lettuce ClientResources 설정 예시
EventLoopGroup eventLoopGroup = ...;
ClientResources res = ClientResources.builder()
    .eventLoopGroupProvider(new EventLoopGroupProvider() { ... })
    .build();
RedisClient redisClient = RedisClient.create(redisURI, res);
```

애플리케이션의 메인 스레드 풀과 격리하여 안정성을 높이려는 의도였으나, 결과적으로는 이 **전용 Executor** 내부에서 발생한 문제가 Redis 클라이언트 전체의 불능으로 이어지게 되었다.

### 로그 분석과 원인

원인을 찾기 위해 타임라인을 거슬러 올라가 보니, 장애 발생 전부터 특이한 로그가 쌓이고 있었다.
바로 `ErrorCallbackNotImplemented` 관련 로그였다.

주기적으로 Redis 데이터를 갱신하는 스케줄러 코드가 있었는데, 대락 아래와 같은 형태였다.

```java
// 문제가 된 코드 패턴 (단순화)
redisClient.getData()
    .map(data -> { /* 1. 데이터 변환 */ })
    .flatMap(x -> { /* 2. 추가 로직 */ })
    .subscribeOn(Schedulers.boundedElastic()) // 여기서 실행하겠지? 라고 기대
    .subscribe(); // Error Handler 없음!
```

여기서 두 가지 치명적인 문제가 겹쳤다.

### 1) subscribe 할 때 에러 처리를 안 했다.
Reactor에서 `subscribe()`를 호출할 때 `onError` 소비자(Consumer)를 등록하지 않으면, 스트림 도중에 발생한 에러가 최종 단계에서 처리되지 못하고 "Unhandled Error"가 된다. 이 경우 스레드 컨텍스트에 따라 심각한 부작용을 낳을 수 있다. 이 경우에는 Lettuce의 EventExecutor 스레드가 종료(Terminated state)되는 현상이 발생했다.

### 2) `subscribeOn`에 대한 오해
나는 `subscribeOn(Schedulers.boundedElastic())`을 썼으니, `map`이나 `flatMap` 같은 로직들도 당연히 별도의 작업 스레드(`boundedElastic`)에서 돌 것이라고 생각했고, 그렇기에 Lettuce의 Executor 스레드에 영향을 줄 거라고는 생각을 하지 못했다.

**하지만 Lettuce는 비동기 소스다.**

Lettuce(Netty)는 IO 이벤트를 처리하고 데이터를 발행(`onNext`)하는 시작점이 바로 **Lettuce의 EventExecutor 스레드**다.

`subscribeOn`은 구독이 **시작되는** 시점의 스레드만 결정할 뿐, 이미 Lettuce의 Executor 스레드에서 데이터가 밀려들어오는 상황(Pub/Sub 구조)에서는 연산자들의 실행 스레드를 바꾸지 못하는 경우가 많다.

결국 로직은 여전히 **Lettuce의 EventExecutor 스레드** 위에서 실행되고 있었고, 거기서 에러가 터졌는데 처리가 안 되니(`ErrorCallbackNotImplemented`), **Lettuce의 EventExecutor 스레드 자체가 비정상 종료되는 상황**이 벌어진 것이다.

이게 누적되다 보니 Redis 처리를 담당할 스레드가 하나둘씩 죽어나갔고, 결국 전체 먹통이 된 것이었다.

### 3) `onErrorResume`조차 동작하지 않은 이유

가장 의아했던 점은, 서비스 코드에 `onErrorResume`을 사용하여 Redis 장애 시에도 Default 응답을 내려주도록 처리가 되어있었음에도 불구하고, 실제 장애 상황에서는 이 Fallback 로직조차 실행되지 않았다는 것이다.

```java
public Mono<Response> getCachedData() {
    return redisClient.getData() // 여기서 RejectedExecutionException 발생
        .map(Response::of)
        .onErrorResume(e -> Mono.just(Response.ofError(e))); // 실행되지 않음...
}
```

Upstream(`redisClient.getData()`)에서 에러가 발생했다면 당연히 Downstream의 `onErrorResume`이 실행되어야 정상이다. 하지만 이번 경우는 **에러가 발생한 곳이 바로 에러를 전달해야 할 Executor**였다는 점이 문제였다.

Lettuce가 Downstream으로 에러 시그널(`onError`)을 보내기 위해서는 내부적으로 EventExecutor에 작업을 할당(`execute`)해야 한다.

(스택트레이스의 `io.lettuce.core.RedisPublisher$PublishOnSubscriber.onError` 부분)

하지만 이미 EventExecutor 스레드들이 모두 종료(Terminated)된 상태였기 때문에, "에러를 전달하라"는 작업 요청마저 거절(`RejectedExecutionException`)당한 것이다.

결국 에러 시그널을 배달해 줄 집배원(Executor)이 사라졌으니 Downstream은 에러가 났다는 사실조차 전달받지 못한 채 무한히 데이터를 기다리는 상태(Stuck)가 되어버렸고, 이로 인해 `onErrorResume`도 트리거되지 못했던 것이다.

---

## 3. 해결 방법

원인을 알았으니 해결은 명확하다.

### 1) 스케줄러 구독 시 에러 처리 필수
`fire-and-forget` 방식이라도 에러 처리는 반드시 해야 한다.

```java
@Scheduled(...)
public void updateCache() {
    redisClient.getData()
        .subscribe(
            success -> { ... },
            error -> log.error("Redis 갱신 실패", error) // 필수!
        );
}
```

### 2) `publishOn`으로 스레드 격리하기
Lettuce 스레드(EventLoop)는 매우 소중하다. IO 처리만 빠르게 하고 놔줘야 한다.
데이터 변환이나 비즈니스 로직 같은 무거운 작업은 `publishOn`을 사용하여 확실하게 다른 스레드풀로 넘겨야 한다.
`subscribeOn`으로 구독 스레드를 변경하더라도, 이건 구독을 시작하는 스레드만 바뀔 뿐, 비동기 소스에서 발행한 데이터가 처리되는 스레드는 변하지 않는다.

```java
redisClient.getData()
    .publishOn(Schedulers.boundedElastic()) // 이후 작업은 별도 스레드에서 실행!
    .map(data -> { ... })
    .flatMap(x -> { ... })
    .subscribe(...);
```

---

## 4. 마치며

이번 장애를 통해 **Reactor에서 `subscribeOn` 동작에 대한 오해**를 확실히 정리할 수 있었다.
습관적으로 캐시 갱신을 하는 스케줄러에서 `subscribeOn`을 남발하고 있었는데 이번 기회를 통해 바로잡을 수 있었다.

- **`subscribeOn`**: 구독(Subscribe) 시점의 스레드를 결정.
- **`publishOn`**: 데이터가 흘러가는(Emit) 도중의 실행 스레드를 변경.

특히 DB 드라이버나 네트워크 클라이언트처럼 자체 스레드풀을 가진 비동기 소스를 다룰 때는 이 차이를 명확히 알고 `publishOn`을 적절히 사용해야겠다.
