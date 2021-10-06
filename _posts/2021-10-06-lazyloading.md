---
layout: post
title:  NestJS - Lazy loading modules
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-06 12:11 +0900
---

## Lazy-loading modules

By default, modules are eagerly loaded, which means that as soon as the application loads, so do all the modules, whether or not they are immediately necessary. While this is fine for most applications, it may become a bottleneck for apps/workers running in the **serverless environment**, where the startup latency ("cold start") is crucial.

Lazy loading can help decrease bootstrap time by loading only modules required by the specific serverless function invocation. In addition, you could also load other modules asynchronously once the serverless function is "warm" to speed-up the bootstrap time for subsequent calls even further (deferred modules registration).

기본적으로 모듈은 즉시 로드됩니다. 즉, 애플리케이션이 로드되는 즉시 모든 모듈이 즉시 필요한지 여부에 관계없이 로드됩니다. 이는 대부분의 애플리케이션에 적합하지만 시작 지연('콜드 스타트')이 중요한 **서버리스 환경**에서 실행되는 앱/작업자에게는 병목 현상이 될 수 있습니다.

지연로드는 특정 서버리스 함수 호출에 필요한 모듈만 로드하여 부트스트랩 시간을 줄이는 데 도움이 될 수 있습니다. 또한 서버리스 기능이 "warm"되면 다른 모듈을 비동기적으로 로드하여 후속 호출에 대한 부트스트랩 시간을 훨씬 더 빠르게 할 수 있습니다(모듈 등록 지연).

> **HINT**
>
> If you're familiar with the **Angular** framework, you might have seen the "lazy-loading modules" term before. Be aware that this technique is **functionally different** in Nest and so think about this as an entirely different feature that shares similar naming conventions.
>
> **Angular** 프레임워크에 익숙하다면 이전에 "lazy-loading modules" 용어를 본 적이 있을 것입니다. 이 기술은 Nest에서 **기능적으로 다르므로** 비슷한 이름 지정 규칙을 공유하는 완전히 다른 기능이라고 생각하세요.



## Getting started

To load modules on-demand, Nest provides the `LazyModuleLoader` class that can be injected into a class in the normal way:

주문형 모듈을 로드하기 위해 Nest는 일반적인 방법으로 클래스에 삽입할 수 있는 `LazyModuleLoader` 클래스를 제공합니다.

### cats.service.ts

```typescript
@Injectable()
export class CatsService {
  constructor(private lazyModuleLoader: LazyModuleLoader) {}
}
```

> **HINT**
>
> The `LazyModuleLoader` class is imported from the `@nestjs/core` package.
>
> `LazyModuleLoader` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

Alternatively, you can obtain a reference to the `LazyModuleLoader` provider from within your application bootstrap file (`main.ts`), as follows:

또는 다음과 같이 애플리케이션 부트스트랩 파일(`main.ts`)에서 `LazyModuleLoader` 프로바이더에 대한 참조를 가져올 수 있습니다.

```typescript
// "app" represents a Nest application instance
const lazyModuleLoader = app.get(LazyModuleLoader);
```

With this in place, you can now load any module using the following construction:

이제 다음 구성을 사용하여 모든 모듈을 로드할 수 있습니다.

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);
```

> **HINT**
>
> "Lazy-loaded" modules are cached upon the first `LazyModuleLoader#load` method invocation. That means, each consecutive attempt to load `LazyModule` will be very fast and will return a cached instance, instead of loading the module again.
>
> "Lazy-loaded" 모듈은 첫번째 `LazyModuleLoader#load` 메소드 호출시 **캐시**됩니다. 즉,`LazyModule`을 로드하려는 각 연속 시도는 **매우 빠르며** 모듈을 다시 로드하는 대신 캐시된 인스턴스를 반환합니다.
>
> ```bash
> Load "LazyModule" attempt: 1
> time: 2.379ms
> Load "LazyModule" attempt: 2
> time: 0.294ms
> Load "LazyModule" attempt: 3
> time: 0.303ms
> ```
>
> Also, "lazy-loaded" modules share the same modules graph as those eagerly loaded on the application bootstrap as well as any other lazy modules registered later in your app.
>
> 또한 "lazy-loaded" 모듈은 나중에 앱에 등록된 다른 지연 모듈뿐만 아니라 애플리케이션 부트스트랩에 열심히 로드된 모듈과 동일한 모듈 그래프를 공유합니다.

Where `lazy.module.ts` is a TypeScript file that exports a **regular Nest module** (no extra changes are required).

The `LazyModuleLoader#load` method returns the [module reference](https://docs.nestjs.com/fundamentals/module-ref) (of `LazyModule`) that lets you navigate the internal list of providers and obtain a reference to any provider using its injection token as a lookup key.

For example, let's say we have a `LazyModule` with the following definition:

여기서 `lazy.module.ts`는 **일반 Nest 모듈**을 내보내는 TypeScript 파일입니다(추가 변경이 필요하지 않음).

For example, let's say we have a `LazyModule` with the following definition: `LazyModuleLoader#load` 메소드는 내부 프로바이더 목록을 탐색하고 주입 토큰을 조회 키로 사용하여 프로바이더에 대한 참조를 얻을 수 있는 [모듈 참조](https://docs.nestjs.kr/fundamentals/module-ref)(`LazyModule`의)를 반환합니다.

예를 들어 다음과 같은 정의를 가진 `LazyModule`이 있다고 가정해 보겠습니다.

```typescript
@Module({
  providers: [LazyService],
  exports: [LazyService],
})
export class LazyModule {}
```

> **HINT**
>
> Lazy-loaded modules cannot be registered as **global modules** as it simply makes no sense (since they are registered lazily, on-demand when all the statically registered modules have been already instantiated). Likewise, registered **global enhancers** (guards/interceptors/etc.) **will not work** properly either.
>
> 지연로드된 모듈은 단순히 의미가 없기 때문에 **전역 모듈**로 등록할 수 없습니다(정적으로 등록된 모든 모듈이 이미 인스턴스화되었을 때 요청에 따라 지연 등록되기 때문에). 마찬가지로 등록된 **글로벌 인핸서**(가드/인터셉터 등)도 **작동하지 않습니다**.

With this, we could obtain a reference to the `LazyService` provider, as follows:

이를 통해 다음과 같이 `LazyService` 프로바이더에 대한 참조를 얻을 수 있습니다.

```typescript
const { LazyModule } = await import('./lazy.module');
const moduleRef = await this.lazyModuleLoader.load(() => LazyModule);

const { LazyService } = await import('./lazy.service');
const lazyService = moduleRef.get(LazyService);
```

> **WARNING**
>
> If you use Webpack, make sure to update your `tsconfig.json` file - setting `compilerOptions.module` to `"esnext"` and adding `compilerOptions.moduleResolution` property with `"node"` as a value:
>
> **Webpack**을 사용하는 경우 `tsconfig.json` 파일을 업데이트해야 합니다.`compilerOptions.module`을 `"esnext"`로 설정하고 값으로 `"node"`를 사용하여 `compilerOptions.moduleResolution` 속성을 추가합니다.
>
> ```json
> {
>   "compilerOptions": {
>     "module": "esnext",
>     "moduleResolution": "node",
>     ...
>   }
> }
> ```
>
> With these options set up, you'll be able to leverage the [code splitting](https://webpack.js.org/guides/code-splitting/) feature.
>
> 이러한 옵션을 설정하면 [코드 분할](https://webpack.js.org/guides/code-splitting/) 기능을 활용할 수 있습니다.



#### Lazy-loading controllers, gateways, and resolvers

Since controllers (or resolvers in GraphQL applications) in Nest represent sets of routes/paths/topics (or queries/mutations), you **cannot lazy load them** using the `LazyModuleLoader` class.

Nest의 컨트롤러(또는 GraphQL 애플리케이션의 리졸버)는 라우트/경로/주제(또는 쿼리/변이) 집합을 나타내므로 `LazyModuleLoader` 클래스를 사용하여 **지연로드할 수 없습니다**.

> **WARNING**
>
> Controllers, [resolvers](https://docs.nestjs.com/graphql/resolvers), and [gateways](https://docs.nestjs.com/websockets/gateways) registered inside lazy-loaded modules will not behave as expected. Similarly, you cannot register middleware functions (by implementing the `MiddlewareConsumer` interface) on-demand.
>
> 컨트롤러, [resolvers](https://docs.nestjs.kr/graphql/resolvers) 및 [gateways](https://docs.nestjs.kr/websockets/gateways)가 지연로드된 모듈에 등록되어 예상대로 작동하지 않습니다. 마찬가지로 주문형 미들웨어 기능( `MiddlewareConsumer` 인터페이스 구현)을 등록할 수 없습니다.

For example, let's say you're building a REST API (HTTP application) with a Fastify driver under the hood (using the `@nestjs/platform-fastify` package). Fastify does not let you register routes after the application is ready/successfully listening to messages. That means even if we analyzed route mappings registered in the module's controllers, all lazy-loaded routes wouldn't be accessible since there is no way to register them at runtime.

Likewise, some transport strategies we provide as part of the `@nestjs/microservices` package (including Kafka, gRPC, or RabbitMQ) require to subscribe/listen to specific topics/channels before the connection is established. Once your application starts listening to messages, the framework would not be able to subscribe/listen to new topics.

Lastly, the `@nestjs/graphql` package with the code first approach enabled automatically generates the GraphQL schema on-the-fly based on the metadata. That means, it requires all classes to be loaded beforehand. Otherwise, it would not be doable to create the appropriate, valid schema.

예를 들어 내부에서 Fastify 드라이버(`@nestjs/platform-fastify` 패키지 사용)를 사용하여 REST API(HTTP 애플리케이션)를 빌드한다고 가정해 보겠습니다. Fastify는 애플리케이션이 준비/성공적으로 메시지를 수신한 후에 라우트를 등록할 수 없도록 합니다. 즉, 모듈의 컨트롤러에 등록된 라우트 매핑을 분석하더라도 런타임에 등록할 방법이 없으므로 모든 지연로드 라우트에 액세스할 수 없습니다.

마찬가지로, `@nestjs/microservices` 패키지(Kafka, gRPC 또는 RabbitMQ 포함)의 일부로 제공하는 일부 전송 전략은 연결이 설정되기 전에 특정 주제/채널을 구독/수신해야 합니다. 애플리케이션이 메시지 수신을 시작하면 프레임워크는 새 주제를 구독하거나 들을 수 없습니다.

마지막으로 코드 우선 접근 방식이 활성화된 `@nestjs/graphql` 패키지는 메타데이터를 기반으로 즉석에서 GraphQL 스키마를 자동으로 생성합니다. 즉, 모든 클래스를 미리 로드해야 합니다. 그렇지 않으면 적절하고 유효한 스키마를 생성할 수 없습니다.



#### Common use-cases

Most commonly, you will see lazy loaded modules in situations when your worker/cron job/lambda & serverless function/webhook must trigger different services (different logic) based on the input arguments (route path/date/query parameters, etc.). On the other hand, lazy-loading modules may not make too much sense for monolithic applications, where the startup time is rather irrelevant.

가장 일반적으로 작업자/크론 작업/람다 및 서버리스 함수/웹 후크가 입력 인수(라우트 경로/날짜/쿼리 매개변수 등)에 따라 다른 서비스(다른 논리)를 트리거해야 하는 상황에서 지연로드된 모듈이 표시됩니다. 반면에 지연로딩 모듈은 시작 시간이 다소 무관한 모놀리식 애플리케이션에는 그다지 의미가 없을 수 있습니다.



#### 출처

> https://docs.nestjs.com/fundamentals/lazy-loading-modules
>
> https://docs.nestjs.kr/fundamentals/lazy-loading-modules