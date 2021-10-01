---
layout: post
title:  NestJS - Middleware
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-01 19:01 +0900
---

Middleware is a function which is called **before** the route handler. Middleware functions have access to the [request](https://expressjs.com/en/4x/api.html#req) and [response](https://expressjs.com/en/4x/api.html#res) objects, and the `next()` middleware function in the application’s request-response cycle. The **next** middleware function is commonly denoted by a variable named `next`.

미들웨어는 라우트 핸들러 **이전에** 호출되는 함수입니다. 미들웨어 기능은 [요청(request)](https://expressjs.com/en/4x/api.html#req) 및 [응답(response)](https://expressjs.com/en/4x/api.html#res) 객체 및 애플리케이션의 요청 - 응답주기에서 `next()` 미들웨어 함수입니다. **다음**(next) 미들웨어 함수는 일반적으로 `next`라는 변수로 표시됩니다.

![img](https://docs.nestjs.com/assets/Middlewares_1.png)

Nest middleware are, by default, equivalent to [express](https://expressjs.com/en/guide/using-middleware.html) middleware. The following description from the official express documentation describes the capabilities of middleware:

> Middleware functions can perform the following tasks:
>
> - execute any code.
> - make changes to the request and the response objects.
> - end the request-response cycle.
> - call the next middleware function in the stack.
> - if the current middleware function does not end the request-response cycle, it must call `next()` to pass control to the next middleware function. Otherwise, the request will be left hanging.

Nest 미들웨어는 기본적으로 [express](https://expressjs.com/en/guide/using-middleware.html) 미들웨어와 동일합니다. 공식 익스프레스 문서의 다음 설명은 미들웨어의 기능을 설명합니다.

> 미들웨어 기능은 다음 작업을 수행할 수 있습니다.
>
> - 모든 코드를 실행하십시오.
> - 요청 및 응답 객체를 변경합니다.
> - 요청-응답주기를 종료합니다.
> - 스택의 next 미들웨어 함수를 호출합니다.
> - 현재 미들웨어 함수가 요청-응답주기를 종료하지 않으면 `next()`를 호출하여 next 미들웨어 기능에 제어를 전달합니다. 그렇지 않으면 요청이 중단됩니다.

You implement custom Nest middleware in either a function, or in a class with an `@Injectable()` decorator. The class should implement the `NestMiddleware` interface, while the function does not have any special requirements. Let's start by implementing a simple middleware feature using the class method.

함수 또는 `@Injectable()` 데코레이터가 있는 클래스에서 커스텀 Nest 미들웨어를 구현합니다. 클래스는 `NestMiddleware` 인터페이스를 구현해야 하지만 함수에는 특별한 요구사항이 없습니다. 클래스 메서드를 사용하여 간단한 미들웨어 기능을 구현하는 것으로 시작하겠습니다.

### logger.middleware.ts

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('Request...');
    next();
  }
}
```

> CLI로 생성 가능! nest g middleware logger



## Dependency injection

Nest middleware fully supports Dependency Injection. Just as with providers and controllers, they are able to **inject dependencies** that are available within the same module. As usual, this is done through the `constructor`.

Nest 미들웨어는 종속성 주입을 완벽하게 지원합니다. 프로바이더 및 컨트롤러와 마찬가지로 동일한 모듈내에서 사용할 수 있는 **종속성을 삽입**할 수 있습니다. 늘 그렇듯이 이것은 `constructor`를 통해 이루어집니다.



## Applying middleware

There is no place for middleware in the `@Module()` decorator. Instead, we set them up using the `configure()` method of the module class. Modules that include middleware have to implement the `NestModule` interface. Let's set up the `LoggerMiddleware` at the `AppModule` level.

`@Module()` 데코레이터에는 미들웨어를 위한 위치가 없습니다. 대신 모듈 클래스의 `configure()` 메소드를 사용하여 설정합니다. 미들웨어를 포함하는 모듈은 `NestModule` 인터페이스를 구현해야 합니다. `AppModule` 레벨에서 `LoggerMiddleware`를 설정해 보겠습니다.

### app.module.ts

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('cats');
  }
}
```

In the above example we have set up the `LoggerMiddleware` for the `/cats` route handlers that were previously defined inside the `CatsController`. We may also further restrict a middleware to a particular request method by passing an object containing the route `path` and request `method` to the `forRoutes()` method when configuring the middleware. In the example below, notice that we import the `RequestMethod` enum to reference the desired request method type.

위의 예에서는 이전에 `CatsController` 내부에 정의된 `/cats` 라우트 핸들러에 대해 `LoggerMiddleware`를 설정했습니다. 또한 미들웨어를 구성할 때 `path` 라우트가 포함된 객체를 전달하고 `method`를 `forRoutes()` 메서드에 요청하여 미들웨어를 특정 요청 메서드로 제한할 수도 있습니다. 아래 예에서 원하는 요청 메서드 타입을 참조하기 위해 `RequestMethod` 열거(enum)형을 가져옵니다.

### app.module.ts

```typescript
import { Module, NestModule, RequestMethod, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'cats', method: RequestMethod.GET });
  }
}
```

> **HINT**
>
> The `configure()` method can be made asynchronous using `async/await` (e.g., you can `await` completion of an asynchronous operation inside the `configure()` method body).
>
> `configure()` 메서드는 `async/await`를 사용하여 비동기식으로 만들 수 있습니다 (예: `configure()` 메서드 본문내에서 비동기 작업의 완료를 `await`할 수 있음).



## Route wildcards

Pattern based routes are supported as well. For instance, the asterisk is used as a **wildcard**, and will match any combination of characters:

패턴 기반 라우트도 지원됩니다. 예를 들어 별표는 **와일드카드**로 사용되며 모든 문자조합과 일치합니다.

```typescript
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

The `'ab*cd'` route path will match `abcd`, `ab_cd`, `abecd`, and so on. The characters `?`, `+`, `*`, and `()` may be used in a route path, and are subsets of their regular expression counterparts. The hyphen ( `-`) and the dot (`.`) are interpreted literally by string-based paths.

`'ab*cd'` 라우트 경로는 `abcd`, `ab_cd`, `abecd` 등과 일치합니다. `?`,`+`,`*`및`()`문자는 라우트 경로에 사용될 수 있으며 해당 정규표현식 대응 부분의 하위집합입니다. 하이픈(`-`)과 점(`.`)은 문자열 기반 경로로 문자 그대로 해석됩니다.

> **WARNING**
>
> The `fastify` package uses the latest version of the `path-to-regexp` package, which no longer supports wildcard asterisks `*`. Instead, you must use parameters (e.g., `(.*)`, `:splat*`).
>
> `fastify` 패키지는 더이상 와일드카드 별표 `*`를 지원하지 않는 `path-to-regexp` 패키지의 최신 버전을 사용합니다. 대신 매개 변수(예: `(.*)`, `:splat*`)를 사용해야 합니다.



## Middleware consumer

The `MiddlewareConsumer` is a helper class. It provides several built-in methods to manage middleware. All of them can be simply **chained** in the [fluent style](https://en.wikipedia.org/wiki/Fluent_interface). The `forRoutes()` method can take a single string, multiple strings, a `RouteInfo` object, a controller class and even multiple controller classes. In most cases you'll probably just pass a list of **controllers** separated by commas. Below is an example with a single controller:

`MiddlewareConsumer`는 헬퍼 클래스입니다. 미들웨어를 관리하기 위한 몇가지 내장된 방법을 제공합니다. 모두 [유창한 스타일](https://en.wikipedia.org/wiki/Fluent_interface)로 간단히 **연결**될 수 있습니다. `forRoutes()` 메소드는 단일 문자열, 여러 문자열, `RouteInfo` 객체, 컨트롤러 클래스 및 여러 컨트롤러 클래스를 사용할 수 있습니다. 대부분의 경우 쉼표로 구분된 **컨트롤러** 목록을 전달합니다. 다음은 단일 컨트롤러의 예입니다.



### app.module.ts

```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller.ts';

@Module({
  imports: [CatsModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes(CatsController);
  }
}
```

> **HINT**
>
> The `apply()` method may either take a single middleware, or multiple arguments to specify [multiple middlewares](https://docs.nestjs.com/middleware#multiple-middleware).
>
> `apply()` 메서드는 단일 미들웨어를 사용하거나 여러 [미들웨어](https://docs.nestjs.kr/middleware#multiple-middleware)를 지정하기 위해 여러 인수를 사용할 수 있습니다.



## Excluding routes

At times we want to **exclude** certain routes from having the middleware applied. We can easily exclude certain routes with the `exclude()` method. This method can take a single string, multiple strings, or a `RouteInfo` object identifying routes to be excluded, as shown below:

때때로 우리는 미들웨어 적용에서 특정 라우트를 **제외**하려고 합니다. `exclude()` 메소드로 특정 라우트를 쉽게 제외할 수 있습니다. 이 메소드는 아래와 같이 제외할 라우트를 식별하는 단일 문자열, 여러 문자열 또는 `RouteInfo` 객체를 사용할 수 있습니다.

```typescript
consumer
  .apply(LoggerMiddleware)
  .exclude(
    { path: 'cats', method: RequestMethod.GET },
    { path: 'cats', method: RequestMethod.POST },
    'cats/(.*)',
  )
  .forRoutes(CatsController);
```

> **HINT**
>
> The `exclude()` method supports wildcard parameters using the [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) package.
>
> `exclude()` 메소드는 [path-to-regexp](https://github.com/pillarjs/path-to-regexp#parameters) 패키지를 사용하여 와일드 카드 매개 변수를 지원합니다.

With the example above, `LoggerMiddleware` will be bound to all routes defined inside `CatsController`**except** the three passed to the `exclude()` method.

위의 예에서 `LoggerMiddleware`는 `exclude()` 메소드에 전달된 세개를 **제외**하고 `CatsController` 내에 정의된 모든 라우트에 바인딩됩니다.



## Functional middleware

The `LoggerMiddleware` class we've been using is quite simple. It has no members, no additional methods, and no dependencies. Why can't we just define it in a simple function instead of a class? In fact, we can. This type of middleware is called **functional middleware**. Let's transform the logger middleware from class-based into functional middleware to illustrate the difference:

우리가 사용해온 `LoggerMiddleware` 클래스는 매우 간단합니다. 멤버, 추가 메서드 및 종속성이 없습니다. 클래스 대신 간단한 함수로 정의할 수 없는 이유는 무엇입니까? 사실 우리는 할 수 있습니다. 이러한 유형의 미들웨어를 **기능적 미들웨어**라고 합니다. 로거 미들웨어를 클래스 기반에서 기능적 미들웨어로 변환하여 차이점을 설명해 보겠습니다.

### logger.middleware.ts

```typescript
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```

And use it within the `AppModule`:

그리고 `AppModule` 내에서 사용합니다.

### app.module.ts

```typescript
consumer
  .apply(logger)
  .forRoutes(CatsController);
```

> **HINT**
>
> Consider using the simpler **functional middleware** alternative any time your middleware doesn't need any dependencies.
>
> 미들웨어에 종속성이 필요하지 않을 때마다 더 간단한 **기능적 미들웨어** 대안을 사용하는 것이 좋습니다.



## Multiple middleware

As mentioned above, in order to bind multiple middleware that are executed sequentially, simply provide a comma separated list inside the `apply()` method:

위에서 언급했듯이 순차적으로 실행되는 여러 미들웨어를 바인딩하려면 `apply()` 메서드내에 쉼표로 구분된 목록을 제공하면 됩니다.

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```



## Global middleware

If we want to bind middleware to every registered route at once, we can use the `use()` method that is supplied by the `INestApplication` instance:

미들웨어를 등록된 모든 경로에 한번에 바인딩하려면 `INestApplication` 인스턴스에서 제공하는 `use()` 메서드를 사용할 수 있습니다.

### main.ts

```typescript
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```

> **HINT**Accessing the DI container in a global middleware is not possible. You can use a [functional middleware](https://docs.nestjs.com/middleware#functional-middleware) instead when using `app.use()`. Alternatively, you can use a class middleware and consume it with `.forRoutes('*')` within the `AppModule` (or any other module).
>
> 글로벌 미들웨어에서 DI 컨테이너에 액세스할 수 없습니다. `app.use()`를 사용할 때 대신 [기능적인 미들웨어](https://docs.nestjs.kr/middleware#functional-middleware)를 사용할 수 있습니다. 또는 클래스 미들웨어를 사용하고 `AppModule`(또는 다른 모듈)내에서 `.forRoutes('*')`와 함께 사용할 수 있습니다.



#### 출처

> https://docs.nestjs.com/middleware
>
> https://docs.nestjs.kr/middleware