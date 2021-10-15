---
layout: post
title:  NestJS - Rate Limiting
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-15 17:55 +0900
---

## Rate Limiting

A common technique to protect applications from brute-force attacks is **rate-limiting**. To get started, you'll need to install the `@nestjs/throttler` package.

무차별 대입 공격으로부터 애플리케이션을 보호하는 일반적인 기술은 **속도 제한(rate-limiting)**입니다. 시작하려면 `@nestjs/throttler` 패키지를 설치해야 합니다.

```bash
$ npm i --save @nestjs/throttler
```

Once the installation is complete, the `ThrottlerModule` can be configured as any other Nest package with `forRoot` or `forRootAsync` methods.

설치가 완료되면 `ThrottlerModule`을 `forRoot` 또는 `forRootAsync` 메서드를 사용하여 다른 Nest 패키지로 구성할 수 있습니다.

```typescript
@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60,
      limit: 10,
    }),
  ],
})
export class AppModule {}
```

The above will set the global options for the `ttl`, the time to live, and the `limit`, the maximum number of requests within the ttl, for the routes of your application that are guarded.

Once the module has been imported, you can then choose how you would like to bind the `ThrottlerGuard`. Any kind of binding as mentioned in the [guards](https://docs.nestjs.com/guards) section is fine. If you wanted to bind the guard globally, for example, you could do so by adding this provider to any module:

위의 내용은 보호되는 애플리케이션의 경로에 대해 `ttl`, TTL(Time to Live) 및 `limit`(ttl 내의 최대 요청 수)에 대한 전역 옵션을 설정합니다.

모듈을 가져온 후에는 `ThrottlerGuard`를 바인딩하는 방법을 선택할 수 있습니다. [가드](https://docs.nestjs.kr/guards) 섹션에 언급된 모든 종류의 바인딩은 괜찮습니다. 예를 들어 가드를 전역적으로 바인딩하려는 경우 이 프로바이더를 임의의 모듈에 추가하면 됩니

```typescript
{
  provide: APP_GUARD,
  useClass: ThrottlerGuard
}
```



## Customization

There may be a time where you want to bind the guard to a controller or globally, but want to disable rate limiting for one or more of your endpoints. For that, you can use the `@SkipThrottle()` decorator, to negate the throttler for an entire class or a single route. The `@SkipThrottle()` decorator can also take in a boolean for if there is a case where you want to exclude *most* of a controller, but not every route.

There is also the `@Throttle()` decorator which can be used to override the `limit` and `ttl` set in the global module, to give tighter or looser security options. This decorator can be used on a class or a function as well. The order for this decorator does matter, as the arguments are in the order of `limit, ttl`.

가드를 컨트롤러 또는 전역적으로 바인딩하고 싶지만 하나 이상의 엔드 포인트에 대한 속도 제한을 비활성화하려는 경우가 있을 수 있습니다. 이를 위해 `@SkipThrottle()` 데코레이터를 사용하여 전체 클래스 또는 단일 경로에 대한 스로틀러를 무효화할 수 있습니다. `@SkipThrottle()` 데코레이터는 모든 경로가 아닌 *대부분의* 컨트롤러를 제외하려는 경우에 대해 부울을 사용할 수도 있습니다.

더 엄격하거나 느슨한 보안 옵션을 제공하기 위해 전역 모듈에 설정된 `limit` 및 `ttl`을 재정의하는 데 사용할 수 있는 `@Throttle()` 데코레이터도 있습니다. 이 데코레이터는 클래스나 함수에서도 사용할 수 있습니다. 이 데코레이터의 순서는 인수가 `limit, ttl`의 순서이기 때문에 중요합니다.



## Proxies

If your application runs behind a proxy server, check the specific HTTP adapter options ([express](http://expressjs.com/en/guide/behind-proxies.html) and [fastify](https://www.fastify.io/docs/latest/Server/#trustproxy)) for the `trust proxy` option and enable it. Doing so will allow you to get the original IP address from the `X-Forward-For` header, and you can override the `getTracker()` method to pull the value from the header rather than from `req.ip`. The following example works with both express and fastify:

```ts
// throttler-behind-proxy.guard.ts
import { ThrottlerGuard } from '@nestjs/throttler';
import { Injectable } from '@nestjs/common';

@Injectable()
export class ThrottlerBehindProxyGuard extends ThrottlerGuard {
  protected getTracker(req: Record<string, any>): string {
    return req.ips.length ? req.ips[0] : req.ip; // individualize IP extraction to meet your own needs
  }
}

// app.controller.ts
import { ThrottlerBehindProxyGuard } from './throttler-behind-proxy.guard';

@UseGuards(ThrottlerBehindProxyGuard)
```

> **HINT**
>
> You can find the API of the `req` Request object for express [here](https://expressjs.com/en/api.html#req.ips) and for fastify [here](https://www.fastify.io/docs/latest/Request/).



## Websockets

This module can work with websockets, but it requires some class extension. You can extend the `ThrottlerGuard` and override the `handleRequest` method like so:

이 모듈은 웹 소켓과 함께 작동할 수 있지만 일부 클래스 확장이 필요합니다. 다음과 같이 `ThrottlerGuard`를 확장하고 `handleRequest` 메서드를 재정의할 수 있습니다.

```typescript
@Injectable()
export class WsThrottlerGuard extends ThrottlerGuard {
  async handleRequest(
    context: ExecutionContext,
    limit: number,
    ttl: number,
  ): Promise<boolean> {
    const client = context.switchToWs().getClient();
    const ip = client.conn.remoteAddress;
    const key = this.generateKey(context, ip);
    const ttls = await this.storageService.getRecord(key);

    if (ttls.length >= limit) {
      throw new ThrottlerException();
    }

    await this.storageService.addRecord(key, ttl);
    return true;
  }
}
```

> **HINT**
>
> If you are using the `@nestjs/platform-ws` package you can use `client._socket.remoteAddress` instead.
>
> `@nestjs/platform-ws` 패키지를 사용하는 경우 대신 `client._socket.remoteAddress`를 사용할 수 있습니다



## GraphQL

The `ThrottlerGuard` can also be used to work with GraphQL requests. Again, the guard can be extended, but this time the `getRequestResponse` method will be overridden

`ThrottlerGuard`는 GraphQL 요청 작업에도 사용할 수 있습니다. 다시 말하지만, 가드를 확장할 수 있지만 이번에는 `getRequestResponse` 메서드가 재정의됩니다.

```typescript
@Injectable()
export class GqlThrottlerGuard extends ThrottlerGuard {
  getRequestResponse(context: ExecutionContext) {
    const gqlCtx = GqlExecutionContext.create(context);
    const ctx = gqlCtx.getContext();
    return { req: ctx.req, res: ctx.res }
  }
}
```



## Configuration

The following options are valid for the `ThrottlerModule`:

다음 옵션은 `ThrottlerModule`에 유효합니다.

| `ttl`              | the number of seconds that each request will last in storage | 각 요청이 스토리지에서 지속되는 시간 (초)                 |
| ------------------ | ------------------------------------------------------------ | --------------------------------------------------------- |
| `limit`            | the maximum number of requests within the TTL limit          | TTL 제한 내 최대 요청 수                                  |
| `ignoreUserAgents` | an array of regular expressions of user-agents to ignore when it comes to throttling requests | 요청 조절과 관련하여 무시할 사용자 에이전트의 정규식 배열 |
| `storage`          | the storage setting for how to keep track of the requests    | 요청을 추적하는 방법에 대한 저장소 설정                   |



## Async Configuration

You may want to get your rate-limiting configuration asynchronously instead of synchronously. You can use the `forRootAsync()` method, which allows for dependency injection and `async` methods.

One approach would be to use a factory function:

속도 제한 구성을 동기 대신 비동기 적으로 가져올 수 있습니다. 종속성 주입 및 `async` 메서드를 허용하는 `forRootAsync()` 메서드를 사용할 수 있습니다.

한가지 접근 방식은 팩토리 함수를 사용하는 것입니다.

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        ttl: config.get('THROTTLE_TTL'),
        limit: config.get('THROTTLE_LIMIT'),
      }),
    }),
  ],
})
export class AppModule {}
```

You can also use the `useClass` syntax:

`useClass` 구문을 사용할 수도 있습니다.

```typescript
@Module({
  imports: [
    ThrottlerModule.forRootAsync({
      imports: [ConfigModule],
      useClass: ThrottlerConfigService,
    }),
  ],
})
export class AppModule {}
```

This is doable, as long as `ThrottlerConfigService` implements the interface `ThrottlerOptionsFactory`.

이것은 `ThrottlerConfigService`가 `ThrottlerOptionsFactory` 인터페이스를 구현하는 한 가능합니다.



## Storages

The built in storage is an in memory cache that keeps track of the requests made until they have passed the TTL set by the global options. You can drop in your own storage option to the `storage` option of the `ThrottlerModule` so long as the class implements the `ThrottlerStorage` interface.

내장된 스토리지는 글로벌 옵션에 의해 설정된 TTL을 통과할 때까지 요청을 추적하는 메모리 캐시입니다. 클래스가 `ThrottlerStorage` 인터페이스를 구현하는 한 `ThrottlerModule`의 `storage` 옵션에 자체 스토리지 옵션을 추가할 수 있습니다.

For distributed servers you could use the community storage provider for [Redis](https://github.com/kkoomen/nestjs-throttler-storage-redis) to have a single source of truth.



> **NOTE**
>
> `ThrottlerStorage` can be imported from `@nestjs/throttler`.
>
> `ThrottlerStorage`는 `@nestjs/throttler`에서 가져올 수 있습니다.



#### 출처

> https://docs.nestjs.com/security/rate-limiting
>
> https://docs.nestjs.kr/security/rate-limiting