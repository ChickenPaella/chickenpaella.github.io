---
title: [Spring WebFlux] 핵심 개념과 올바른 사용 방식 정리
parent: Spring 
nav_order: 1
date: 2025-04-13
tags: [spring, ChatGPT, 공부, 블로그쓰기]
---

> Spring WebFlux를 실무에 도입하면서 생긴 **핵심 궁금증**과 **사용 방식**,  
> 그리고 MVC와의 차이, 스레드 구조, 테스트 전략까지 chatGPT와 대화한 내용을 정리한 글입니다.

---

## 📚 목차

1. WebFlux의 구조와 철학  
2. `Mono.defer`와 `Mono.fromCallable`의 차이  
3. WebFlux에서의 `@Transactional` 적용 원칙  
4. 동기 ORM 사용 시의 절충 설계  
5. Spring MVC vs WebFlux 성능 비교  
6. WebClient와 스레드 흐름  
7. Netty 스레드 vs 워커 스레드  
8. WebSocket의 올바른 구조  
9. Mono/Flux 테스트 방법  
10. 결론: WebFlux를 도입해야 할까?  

---

## 1. WebFlux의 구조와 철학

Spring WebFlux는 **비동기/논블로킹 처리**를 위한 프레임워크로,  
요청과 응답을 **Netty 기반 이벤트 루프**를 통해 처리합니다.

- Spring MVC: 요청마다 스레드를 할당 (thread-per-request)
- WebFlux: **작은 수의 스레드(Netty 이벤트 루프)**로 수천 개의 요청을 처리

---

## 2. `Mono.defer` vs `Mono.fromCallable`

| 항목 | `fromCallable()` | `defer()` |
|------|------------------|-----------|
| 실행 시점 | 구독 시 | 구독 시 |
| 반환 | `Mono<T>` | `Mono<T>` or `Mono<Mono<T>>` |
| 사용 예 | 값 계산 | Mono 생성 자체를 지연하고 싶을 때 |

```java
Mono.fromCallable(() -> doSomething());  // 값 자체를 래핑
Mono.defer(() -> Mono.just(doSomething())); // Mono 생성을 지연
```

## 3. @Transactional은 어디에 써야 할까?
WebFlux에서는 @Transactional이 ThreadLocal 기반이기 때문에 깨질 수 있습니다.  
→ 대신 TransactionalOperator를 사용하거나, 동기 메소드로 분리하고 Mono.fromCallable()로 감싸야 합니다.

```java
Mono.fromCallable(() -> {
    return transactionalService.findUserBlocking();
}).subscribeOn(Schedulers.boundedElastic());
```

## 4. 동기 ORM(MyBatis, JPA) 사용할 때 구조
WebFlux에서는 blocking 코드를 감싸지 않으면 Netty 이벤트 루프가 막힙니다.

```java
Mono.fromCallable(() -> mybatisMapper.findUser())
    .subscribeOn(Schedulers.boundedElastic());
```

Controller, Service, Repository에서 Mono 체인을 유지할 수 없고
모든 계층이 동기라면 사실상 MVC와 동일한 구조입니다.

## 5. Spring MVC vs WebFlux 성능 비교
| 항목|	Spring MVC | Spring WebFlux | 
|-----|------------|----------------|
|모델|	Blocking / 스레드 per 요청|	Non-blocking / Netty 이벤트 루프|
|TPS|	수백~1,000|	수천~수만|
|스레드 사용량|	높음	|낮음|
|적합한 곳	|일반 CRUD, 트랜잭션 중심	|고트래픽, API 게이트웨이, 실시간 통신 등|

## 6. WebClient + 스레드 흐름
```java
Mono.fromCallable(() -> service.blocking())
    .subscribeOn(Schedulers.boundedElastic())
    .flatMap(blocked -> webClient.get().uri(...).retrieve().bodyToMono(...));
```

WebClient는 내부적으로 Netty의 I/O 이벤트 루프를 사용합니다.  
스레드가 boundedElastic이더라도, HTTP 요청은 Netty 이벤트 루프에서 처리됩니다.

## 7. Netty 스레드 vs Worker 스레드
|종류|	설명|
|----|------|
|Netty I/O 스레드|	HTTP 요청 수신/응답 (reactor-http-nio)|
|Worker (boundedElastic)|	Blocking 작업, 연산 (예: DB, 파일 등)|
|Parallel|	CPU 연산 처리용 스레드풀|


## 8. WebSocket 구조 (Spring WebFlux)
WebFlux에서는 WebSocketHandler와 Flux 기반 구조를 사용하는 것이 정석입니다.

```java
@Override
public Mono<Void> handle(WebSocketSession session) {
    Flux<String> in = session.receive().map(...);
    Flux<WebSocketMessage> out = someFlux.map(session::textMessage);

    return session.send(out).and(in.then());
}
```

session.send(out)과 session.receive()를 .and()로 연결해 양방향 통신이 가능합니다.

## 9. Mono/Flux 테스트 방법
### ✅ .block() 사용 가능 (테스트 한정)
```java
String result = service.getUser().block();
assertEquals("John", result);
```

### ✅ StepVerifier 사용 (권장)
```java
StepVerifier.create(service.getUser())
    .expectNextMatches(user -> user.getId() == 1L)
    .expectComplete()
    .verify();
```

## 10. 결론: WebFlux를 도입해야 할까?
Spring WebFlux는 뛰어난 성능을 발휘할 수 있는 구조지만, 다음 조건이 필요합니다:

- 전 계층에서 Mono/Flux 체인 유지

- blocking 작업은 반드시 스케줄러로 분리

- Netty I/O 스레드는 절대 block 금지

- 트랜잭션은 TransactionalOperator로 대체

## ✅ 마무리
WebFlux는 트렌드로 쓰기엔 위험한 프레임워크입니다.  
확실한 목적(고동시성, 스트리밍, 효율적인 API 호출 체인)이 있을 때만 쓰는 것을 추천합니다.

지금 구조가 MVC처럼 느껴진다면, 실제로 WebFlux를 "제대로" 쓰고 있는지 되돌아보는 것이 좋습니다.

💬 이 글은 Spring WebFlux를 실무에서 직접 경험하며 고민하고 정리한 내용을 바탕으로 작성되었습니다.  
틀리거나 오해의 소지가 있는 부분이 있다면 언제든 의견 주세요!
