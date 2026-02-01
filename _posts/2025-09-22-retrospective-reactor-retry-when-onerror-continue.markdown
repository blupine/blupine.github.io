---
layout: post
title:  "Reactor retryWhen과 onErrorContinue"
subtitle: "Reactor 개발자도 후회하는 연산자가 있다?"
date:   2025-09-22 19:52:13 +0900
categories: dev
tags : Spring Reactor TroubleShooting
---

Project Reactor를 사용하면서 겪었던 아주 **치명적인(Critical)** 장애 경험을 정리해본다.  
Reactor의 에러 처리 연산자인 `retryWhen`과 `onErrorContinue`를 함께 사용했다가, **스트림이 영원히 종료되지 않는(Hang)** 현상을 겪었다.

---

## 1. 장애 상황 발생

팀에서 운영 중인 배치를 모니터링하다가 이상한 점을 발견했다.

- **상황**: 대용량 데이터를 DB에서 조회(Cursor 방식)하여 처리하는 배치 작업.
- **기대 동작**: 처리 도중 에러가 발생하면, `doFinally` 훅을 통해 로깅을 남기고 **Slack으로 알림**을 보내야 함.
- **실제 동작**: 에러가 발생했는데도 **Slack 알림이 오지 않음**. 게다가 배치 작업이 끝나지도 않고 **무한 대기(Stuck)** 상태에 빠짐.

로그를 확인해보니 에러는 분명 발생했다. 그런데 왜 스트림은 종료되지 않고 알림도 보내지 못했을까?

---

## 2. 원인 분석: `retryWhen` + `onErrorContinue`

범인은 바로 `retryWhen`과 `onErrorContinue`의 잘못된 만남이었다.  
당시 코드는 대략 이런 형태였다.

```java
// ❌ 문제가 된 코드 (Anti-pattern)
return dataSource.findAll(condition)
        .flatMap(items -> processItems(items))
        .retryWhen(Retry.fixedDelay(3, Duration.ofSeconds(1))) // 재시도 로직
        .doFinally(signal -> {
            log.info("Processing completed");
            sendSlackAlert(signal); // 여기가 실행되어야 하는데...
        })
        .onErrorContinue((throwable, object) -> { // 에러가 나면 계속 진행해라?
            log.error("Error ignored: {}", throwable.getMessage());
        })
        .then();
```

### 왜 문제가 될까?

Reactor에서 일반적인 연산자(`map`, `filter` 등)는 **위에서 아래(Upstream -> Downstream)** 로 데이터가 흐른다. 하지만 `onErrorContinue`는 아주 특수한 녀석이다.

1. **`onErrorContinue`의 동작 방식**: 자기보다 **위(Upstream)** 에 있는 연산자들의 에러 처리 방식에 개입한다. "위쪽에서 에러가 나도 스트림을 끊지 말고 다음 데이터를 달라"고 요청한다.
2. **`retryWhen`의 내부 구조**: `retryWhen`은 재시도 로직을 수행하기 위해 내부적으로 `concatMap` 등을 사용해 복잡한 시그널 처리를 한다.
3. **충돌 발생**:
    - `retryWhen`에서 재시도를 모두 실패(Exhausted)하고 에러를 터트리려고 한다.
    - 그런데 밑에 있던 `onErrorContinue`가 "어? 에러 났네? 무시하고 계속 진행해(Swallow Error)"라며 에러를 삼켜버린다.
    - 결과적으로 `retryWhen`은 에러를 밖으로 던지지도 못하고, 그렇다고 정상 완료(Complete) 시그널을 보내지도 못하는 상태가 되어버린다.
    - **결국 시퀀스는 영원히 종료되지 않는 상태(Deadlock/Hang)가 된다.**

이 문제는 Reactor 커뮤니티에서도 꽤 유명한 이슈로, Reactor 메인테이너인 Simon Baslé는 Github 이슈에서 이런 명언(?)을 남기기도 했다.

> *"onErrorContinue is my billion dollar mistake :("* 
> (onErrorContinue는 내 10억 달러짜리 실수다)
> — [reactor-core#2184](https://github.com/reactor/reactor-core/issues/2184#issuecomment-641921007)

---

## 3. 해결 방법

결론은, **`retryWhen`과 `onErrorContinue`는 같이 쓰지 않는 것이 좋다.**  
대신 `onErrorResume`이나 `flatMap` 내부 처리를 통해 명시적으로 핸들링해야 한다.

### 해결책 1: `onErrorContinue` 대신 `onErrorResume` 사용

가장 깔끔한 방법은 에러가 났을 때 "무시하고 계속(Continue)"하는 게 아니라, "대체값으로 복구(Resume)"하여 스트림을 정상 종료시키는 것이다.

```java
// ✅ 올바른 패턴 1
return dataSource.findAll(condition)
        .flatMap(items -> processItems(items))
        .retryWhen(Retry.fixedDelay(3, Duration.ofSeconds(1)))
        .onErrorResume(e -> { // 에러를 잡아서 빈 스트림으로 정상 종료
            log.error("최종 실패: {}", e.getMessage());
            return Mono.empty(); 
        })
        .doFinally(signal -> {
            sendSlackAlert(signal); // 이제 정상적으로 실행됨!
        })
        .then();
```

### 해결책 2: 개별 아이템 실패 무시는 `flatMap` 안에서

만약 "전체 실패"가 아니라 "개별 아이템 실패만 건너뛰고 싶다"면, `onErrorContinue`를 쓰는 대신 `flatMap` 내부에서 에러를 잡으세요.

```java
// ✅ 올바른 패턴 2: 개별 항목 실패 무시
Flux.just(items)
    .flatMap(item -> process(item)
        .doOnError(e -> log.error("개별 실패", e))
        .onErrorResume(e -> Mono.empty()) // 실패한 놈만 빈값 처리하고 나머진 진행
    )
    .retryWhen(...)
    .subscribe();
```

---

## 4. 핵심 요약 (Cheat Sheet)

Reactor 에러 처리가 헷갈릴 때를 위해 정리해둔다.

| 상황 | 사용해야 할 연산자 | 비고 |
|------|-------------------|------|
| 에러가 나면 대체값을 반환하고 싶다 | **onErrorResume** | 가장 안전함 & 권장 |
| 에러가 나면 기본값(상수)을 반환하고 싶다 | **onErrorReturn** | |
| 단순 로깅만 하고 에러를 계속 던지고 싶다 | **doOnError** | 리액티브 스트림 흐름 영향 없음 |
| **`retryWhen`과 함께 쓰고 싶다** | **onErrorResume** | `onErrorContinue` 사용 금지 ❌ |
| 일부 데이터 실패를 무시하고 싶다 | `flatMap` 내부 `onErrorResume` | `onErrorContinue`는 동작 예측이 어려움 |

이번 일을 계기로 "편리해 보이는 마법 같은 연산자(`onErrorContinue`)일수록 부작용을 조심해야 한다"는 교훈을 얻었다. 향후 운영 환경에서도 혹시 이 "위험한 동거"가 일어나고 있지는 않은지 점검해봐야겠다.


 ## 참고 자료
 - [NHN Cloud 기술 블로그 - Reactor retryWhen, onErrorContinue 이슈](https://blog.naver.com/nhncloud_official/223264587752)
 - [GitHub Issue: Retry not playing well with onErrorContinue](https://github.com/reactor/reactor-addons/issues/210)
 - [GitHub Issue: onErrorContinue() design](https://github.com/reactor/reactor-core/issues/2184)
 - [Reactor FAQ: When and how to use onErrorContinue()](https://nurkiewicz.com/2021/08/onerrorcontinue-reactor.html)