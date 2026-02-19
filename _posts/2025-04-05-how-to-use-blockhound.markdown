---
layout: post
title:  "BlockHound 사용 방법 : NonBlocking 스레드 검증하기"
subtitle: "Reactor 애플리케이션에서 블로킹 호출을 탐지하는 기법"
date:   2025-04-05 21:45:00 +0900
categories: java reactor
tags : Java Reactor BlockHound Testing NonBlocking
---

## 1) BlockHound란?

BlockHound는 Reactor 기반 애플리케이션에서 **NonBlocking 스레드에서 blocking 호출이 발생했는지**를 탐지하기 위한 **runtime instrumentation 라이브러리**다.

### 동작 방식
- Reactor/Netty 등에서 "NonBlocking"으로 간주되는 스레드(예: **Netty event loop**, `Schedulers.parallel()` worker)에서
- `Thread.sleep`, 특정 I/O 등과 같이 **블로킹으로 분류되는 호출**이 실행되면
- **예외(예: `BlockingOperationError`)를 던져** 문제를 조기에 발견하게 한다.

### 활용 포인트
- WebFlux(Netty) 환경에서 요청 처리가 주로 **Netty event loop(NonBlocking)** 에서 실행되므로,
- **event loop에서 블로킹 호출이 섞였는지 테스트로 검출**하는 데 유용하다.

---

## 2) BlockHound 사용 방법

### 2.1 의존성 추가(테스트용)
일반적으로 테스트 스코프에 추가해, 개발/테스트 단계에서만 블로킹 호출을 검출한다.

```gradle
testImplementation("io.projectreactor.tools:blockhound:<version>")
```

---

### 2.2 설치(install)
`BlockHound.install()`을 **테스트 JVM에서 1회** 실행한다. 보통 `@BeforeAll` 등에서 설치한다.

```java
import reactor.blockhound.BlockHound;
import org.junit.jupiter.api.BeforeAll;

public class BlockHoundTestSetup {
    @BeforeAll
    static void setup() {
        BlockHound.install();
    }
}
```

---

### 2.3 예시: sanity test (BlockHound가 제대로 동작하는지 확인)
아래 테스트는 **NonBlocking 스레드에서 `Thread.sleep`을 실행**하므로, BlockHound가 예외를 던지는지 확인할 수 있다.

```java
import org.junit.jupiter.api.Test;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;
import reactor.test.StepVerifier;

class BlockHoundUsageTest {

    @Test
    void blockHoundSanityCheck() {
        // NonBlocking 스레드에서 Thread.sleep 실행 시 예외 발생 확인
        Mono<Void> blockingMono = Mono.fromRunnable(() -> {
            try {
                Thread.sleep(10);
            } catch (InterruptedException ignored) {
            }
        }).subscribeOn(Schedulers.parallel()); // NonBlocking 마커 스레드에서 실행

        StepVerifier.create(blockingMono)
            .expectError() // BlockingOperationError 계열
            .verify();
    }
}
```

---

### 2.4 `subscribeOn(Schedulers.parallel())`을 한 이유?  
#### 핵심 개념: "NonBlocking 스레드에서만 예외를 던진다"

- BlockHound는 **아무 스레드에서나** 블로킹 호출을 전부 에러로 만들지 않는다.
- 기본 동작은 **"Reactor가 NonBlocking으로 마킹한 스레드(또는 Netty event loop)에서 블로킹 호출이 발생할 때만"** 예외를 던지는 것이다.

#### 왜 테스트에서는 `subscribeOn`/`publishOn`이 필요한가?
- JUnit 테스트는 보통 **테스트 스레드(main/JUnit thread)** 에서 구독(subscribe)이 일어난다.
- 이 스레드는 기본적으로 **NonBlocking 스레드가 아니다.**
- 따라서 `Thread.sleep()` 같은 블로킹 코드를 넣어도,
  - "NonBlocking 스레드를 막은 게 아니라 테스트 스레드가 잠든 것"으로 간주되어
  - BlockHound가 예외를 던지지 않을 수 있다.

#### 그래서 왜 `Schedulers.parallel()`로 오프로딩하나?
- `Schedulers.parallel()` 워커는 일반적으로 **NonBlocking 성격으로 취급**된다.
- 여기에서 블로킹 호출을 실행하면 BlockHound가 예외를 던져 **"리액티브 규칙 위반"을 확실히 드러낸다.**
