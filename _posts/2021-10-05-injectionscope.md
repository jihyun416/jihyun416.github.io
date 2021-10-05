---
layout: post
title:  NestJS - Injection scopes
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-05 18:16 +0900
---

## Injection scopes

For people coming from different programming language backgrounds, it might be unexpected to learn that in Nest, almost everything is shared across incoming requests. We have a connection pool to the database, singleton services with global state, etc. Remember that Node.js doesn't follow the request/response Multi-Threaded Stateless Model in which every request is processed by a separate thread. Hence, using singleton instances is fully **safe** for our applications.

However, there are edge-cases when request-based lifetime may be the desired behavior, for instance per-request caching in GraphQL applications, request tracking, and multi-tenancy. Injection scopes provide a mechanism to obtain the desired provider lifetime behavior.

다른 프로그래밍 언어 배경을 가진 사람들의 경우 Nest에서 거의 모든 것이 들어오는 요청에서 공유된다는 사실을 배우는 것은 예상치 못한 일입니다. 데이터베이스에 대한 연결 풀, 전역 상태의 싱글톤 서비스 등이 있습니다. Node.js는 모든 요청이 별도의 스레드에서 처리되는 요청/응답 다중 스레드 상태 비저장 모델을 따르지 않습니다. 따라서 싱글톤 인스턴스를 사용하는 것은 애플리케이션에 완전히 **안전**합니다.

그러나 GraphQL 애플리케이션의 요청별 캐싱, 요청 추적 및 멀티 테넌시와 같이 요청 기반 수명이 원하는 동작일 수 있는 경우가 있습니다. 주입 범위는 원하는 공급자 수명 동작을 얻기위한 메커니즘을 제공합니다.



## Provider scope

A provider can have any of the following scopes:

프로바이더는 다음 범위중 하나를 가질 수 있습니다.

| `DEFAULT`   | A single instance of the provider is shared across the entire application. The instance lifetime is tied directly to the application lifecycle. Once the application has bootstrapped, all singleton providers have been instantiated. Singleton scope is used by default. | 프로바이더의 단일 인스턴스는 전체 응용 프로그램에서 공유됩니다. 인스턴스 수명은 애플리케이션 수명주기와 직접 연결됩니다. 애플리케이션이 부트스트랩되면 모든 싱글톤 프로바이더가 인스턴스화되었습니다. 기본적으로 싱글톤 범위가 사용됩니다. |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `REQUEST`   | A new instance of the provider is created exclusively for each incoming **request**. The instance is garbage-collected after the request has completed processing. | 프로바이더의 새 인스턴스는 들어오는 각 **요청**에 대해 독점적으로 생성됩니다. 요청이 처리를 완료한 후 인스턴스가 가비지 수집(garbage-collected)됩니다. |
| `TRANSIENT` | Transient providers are not shared across consumers. Each consumer that injects a transient provider will receive a new, dedicated instance. | 일시적인 프로바이더는 소비자간에 공유되지 않습니다. 임시 프로바이더를 삽입하는 각 소비자는 새로운 전용 인스턴스를 받게됩니다. |

> **HINT**
>
> Using singleton scope is **recommended** for most use cases. Sharing providers across consumers and across requests means that an instance can be cached and its initialization occurs only once, during application startup.
>
> 싱글톤 범위 사용은 대부분의 사용 사례에서 **권장**됩니다. 소비자와 요청간에 프로바이더를 공유한다는 것은 인스턴스를 캐시할 수 있고 초기화가 애플리케이션 시작중에 한번만 발생함을 의미합니다.



## Usage

Specify injection scope by passing the `scope` property to the `@Injectable()` decorator options object:

`scope` 속성을 `@Injectable()` 데코레이터 옵션 객체에 전달하여 주입 범위를 지정합니다.

```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

Similarly, for [custom providers](https://docs.nestjs.com/fundamentals/custom-providers), set the `scope` property in the long-hand form for a provider registration:

마찬가지로 [사용자 지정 프로바이더](https://docs.nestjs.kr/fundamentals/custom-providers)의 경우 프로바이더 등록을 위해 긴 형식으로 `scope` 속성을 설정합니다.

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

> **HINT**
>
> Import the `Scope` enum from `@nestjs/common`

> **NOTICE**
>
> Gateways should not use request-scoped providers because they must act as singletons. Each gateway encapsulates a real socket and cannot be instantiated multiple times.
>
> 게이트웨이는 싱글톤으로 작동해야하므로 요청 범위 프로바이더를 사용해서는 안됩니다. 각 게이트웨이는 실제 소켓을 캡슐화하며 여러번 인스턴스화 할 수 없습니다.

Singleton scope is used by default, and need not be declared. If you do want to declare a provider as singleton scoped, use the `Scope.DEFAULT` value for the `scope` property.

싱글톤 범위는 기본적으로 사용되며 선언할 필요가 없습니다. 프로바이더를 싱글톤 범위로 선언하려면 `scope` 속성에 `Scope.DEFAULT` 값을 사용하세요.



## Controller scope

Controllers can also have scope, which applies to all request method handlers declared in that controller. Like provider scope, the scope of a controller declares its lifetime. For a request-scoped controller, a new instance is created for each inbound request, and garbage-collected when the request has completed processing.

Declare controller scope with the `scope` property of the `ControllerOptions` object:

컨트롤러는 해당 컨트롤러에서 선언된 모든 요청 메서드 핸들러에 적용되는 범위를 가질 수도 있습니다. 프로바이더 범위와 마찬가지로 컨트롤러의 범위는 수명을 선언합니다. 요청 범위 컨트롤러의 경우 각 인바운드 요청에 대해 새 인스턴스가 생성되고 요청이 처리를 완료하면 가비지 수집됩니다.

`ControllerOptions` 객체의 `scope` 속성으로 컨트롤러 범위를 선언합니다.

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```



## Scope hierarchy

Scope bubbles up the injection chain. A controller that depends on a request-scoped provider will, itself, be request-scoped.

Imagine the following dependency graph: `CatsController <- CatsService <- CatsRepository`. If `CatsService` is request-scoped (and the others are default singletons), the `CatsController` will become request-scoped as it is dependent on the injected service. The `CatsRepository`, which is not dependent, would remain singleton-scoped.

스코프가 주입 체인을 위로 올립니다. 요청 범위 프로바이더에 의존하는 컨트롤러는 자체적으로 요청 범위가 됩니다.

다음 종속성 그래프를 상상해보십시오: `CatsController <- CatsService <- CatsRepository`. `CatsService`가 요청 범위이고 나머지는 기본 싱글톤인 경우 `CatsController`는 삽입된 서비스에 따라 달라지므로 요청 범위가 됩니다. 종속되지 않는 `CatsRepository`는 싱글톤 범위로 유지됩니다.



## Request provider

In an HTTP server-based application (e.g., using `@nestjs/platform-express` or `@nestjs/platform-fastify`), you may want to access a reference to the original request object when using request-scoped providers. You can do this by injecting the `REQUEST` object.

HTTP 서버 기반 애플리케이션 (예: `@nestjs/platform-express` 또는 `@nestjs/platform-fastify` 사용)에서 요청 범위 프로바이더를 사용할 때 원래 요청 객체에 대한 참조에 액세스할 수 있습니다. `REQUEST`개체를 삽입하여 이를 수행할 수 있습니다.

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

Because of underlying platform/protocol differences, you access the inbound request slightly differently for Microservice or GraphQL applications. In [GraphQL](https://docs.nestjs.com/graphql/quick-start) applications, you inject `CONTEXT` instead of `REQUEST`:

기본 플랫폼/프로토콜 차이로 인해 마이크로 서비스 또는 GraphQL 애플리케이션의 경우 인바운드 요청에 약간 다르게 액세스합니다. [GraphQL](https://docs.nestjs.kr/graphql/quick-start) 애플리케이션에서 `REQUEST`대신 `CONTEXT`를 삽입합니다.

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';

@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

You then configure your `context` value (in the `GraphQLModule`) to contain `request` as its property.

그런 다음 `request`을 속성으로 포함하도록 `context`값(`GraphQLModule`에서)을 구성합니다.



## Performance

Using request-scoped providers will have an impact on application performance. While Nest tries to cache as much metadata as possible, it will still have to create an instance of your class on each request. Hence, it will slow down your average response time and overall benchmarking result. Unless a provider must be request-scoped, it is strongly recommended that you use the default singleton scope.

요청 범위 프로바이더를 사용하면 애플리케이션 성능에 영향을 미칩니다. Nest는 가능한 한 많은 메타데이터를 캐시하려고 하지만 각 요청에 대해 클래스의 인스턴스를 만들어야합니다. 따라서 평균 응답 시간과 전체 벤치마킹 결과가 느려집니다. 프로바이더가 요청 범위여야 하는 경우가 아니면 기본 싱글톤 범위를 사용하는 것이 좋습니다.



#### 출처

> https://docs.nestjs.com/fundamentals/injection-scopes
>
> https://docs.nestjs.kr/fundamentals/injection-scopes