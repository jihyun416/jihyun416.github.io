---
layout: post
title:  NestJS - Execution context
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-06 12:42 +0900
---

## Execution context

Nest provides several utility classes that help make it easy to write applications that function across multiple application contexts (e.g., Nest HTTP server-based, microservices and WebSockets application contexts). These utilities provide information about the current execution context which can be used to build generic [guards](https://docs.nestjs.com/guards), [filters](https://docs.nestjs.com/exception-filters), and [interceptors](https://docs.nestjs.com/interceptors) that can work across a broad set of controllers, methods, and execution contexts.

We cover two such classes in this chapter: `ArgumentsHost` and `ExecutionContext`.

Nest는 여러 애플리케이션 컨텍스트(예: Nest HTTP 서버 기반, 마이크로 서비스 및 WebSockets 애플리케이션 컨텍스트)에서 작동하는 애플리케이션을 쉽게 작성할 수 있도록 도와주는 여러 유틸리티 클래스를 제공합니다. 이러한 유틸리티는 광범위한 세트에서 작동할 수 있는 일반 [가드](https://docs.nestjs.kr/guards), [필터](https://docs.nestjs.kr/exception-filters) 및 [인터셉터](https://docs.nestjs.kr/interceptors)를 빌드하는 데 사용할 수 있는 현재 실행 컨텍스트에 대한 정보를 제공합니다. 컨트롤러, 메서드 및 실행 컨텍스트.

이 장에서는 두가지 클래스인 `ArgumentsHost`와 `ExecutionContext`를 다룹니다.



## ArgumentsHost class

The `ArgumentsHost` class provides methods for retrieving the arguments being passed to a handler. It allows choosing the appropriate context (e.g., HTTP, RPC (microservice), or WebSockets) to retrieve the arguments from. The framework provides an instance of `ArgumentsHost`, typically referenced as a `host` parameter, in places where you may want to access it. For example, the `catch()` method of an [exception filter](https://docs.nestjs.com/exception-filters#arguments-host) is called with an `ArgumentsHost`instance.

`ArgumentsHost` simply acts as an abstraction over a handler's arguments. For example, for HTTP server applications (when `@nestjs/platform-express` is being used), the `host` object encapsulates Express's `[request, response, next]` array, where `request` is the request object, `response` is the response object, and `next` is a function that controls the application's request-response cycle. On the other hand, for [GraphQL](https://docs.nestjs.com/graphql/quick-start) applications, the `host` object contains the `[root, args, context, info]` array.

`ArgumentsHost` 클래스는 핸들러에 전달되는 인수를 검색하는 메서드를 제공합니다. 인수를 검색할 적절한 컨텍스트(예: HTTP, RPC(마이크로 서비스) 또는 WebSockets)를 선택할 수 있습니다. 프레임 워크는 액세스하려는 위치에 일반적으로 `host` 매개변수로 참조되는 `ArgumentsHost`의 인스턴스를 제공합니다. 예를 들어 [예외필터](https://docs.nestjs.kr/exception-filters#arguments-host)의 `catch()` 메서드는 `ArgumentsHost` 인스턴스와 함께 호출됩니다.

`ArgumentsHost`는 단순히 핸들러의 인수에 대한 추상화 역할을 합니다. 예를 들어 HTTP 서버 애플리케이션의 경우(`@nestjs/platform-express`를 사용하는 경우) `host` 객체는 Express의 `[request, response, next]` 배열을 캡슐화합니다. 여기서 `request`는 요청 객체입니다. `response`는 응답 객체이고 `next`는 애플리케이션의 요청-응답주기를 제어하는 함수입니다. 반면에 [GraphQL](https://docs.nestjs.kr/graphql/quick-start) 애플리케이션의 경우 `host` 객체에는 `[root, args, context, info]` 배열이 포함됩니다.



## Current application context

When building generic [guards](https://docs.nestjs.com/guards), [filters](https://docs.nestjs.com/exception-filters), and [interceptors](https://docs.nestjs.com/interceptors) which are meant to run across multiple application contexts, we need a way to determine the type of application that our method is currently running in. Do this with the `getType()` method of `ArgumentsHost`:

여러 애플리케이션 컨텍스트에서 실행되는 일반 [가드](https://docs.nestjs.kr/guards), [필터](https://docs.nestjs.kr/exception-filters) 및 [인터셉터](https://docs.nestjs.kr/interceptors)를 빌드할 때 다음과 같은 애플리케이션 유형을 결정하는 방법이 필요합니다. 우리의 메서드는 현재 실행중입니다. `ArgumentsHost`의 `getType()` 메서드를 사용하여이 작업을 수행합니다.

```typescript
if (host.getType() === 'http') {
  // do something that is only important in the context of regular HTTP requests (REST)
} else if (host.getType() === 'rpc') {
  // do something that is only important in the context of Microservice requests
} else if (host.getType<GqlContextType>() === 'graphql') {
  // do something that is only important in the context of GraphQL requests
}
```

> **HINT**
>
> The `GqlContextType` is imported from the `@nestjs/graphql` package.
>
> `GqlContextType`은 `@nestjs/graphql` 패키지에서 가져옵니다.

With the application type available, we can write more generic components, as shown below.

사용 가능한 애플리케이션 유형을 사용하면 아래와 같이 보다 일반적인 구성 요소를 작성할 수 있습니다.



## Host handler arguments

To retrieve the array of arguments being passed to the handler, one approach is to use the host object's `getArgs()` method.

핸들러에 전달되는 인수 배열을 검색하려면 호스트 객체의 `getArgs()` 메소드를 사용하는 방법이 있습니다.

```typescript
const [req, res, next] = host.getArgs();
```

You can pluck a particular argument by index using the `getArgByIndex()` method:

`getArgByIndex()` 메소드를 사용하여 색인으로 특정 인수를 가져올 수 있습니다.

```typescript
const request = host.getArgByIndex(0);
const response = host.getArgByIndex(1);
```

In these examples we retrieved the request and response objects by index, which is not typically recommended as it couples the application to a particular execution context. Instead, you can make your code more robust and reusable by using one of the `host` object's utility methods to switch to the appropriate application context for your application. The context switch utility methods are shown below.

이 예제에서 우리는 인덱스별로 요청 및 응답 객체를 검색했는데, 이는 애플리케이션을 특정 실행 컨텍스트에 연결하므로 일반적으로 권장되지 않습니다. 대신 `host` 객체의 유틸리티 메소드중 하나를 사용하여 애플리케이션에 적합한 애플리케이션 컨텍스트로 전환하여 코드를 더 강력하고 재사용할 수 있도록 만들 수 있습니다. 컨텍스트 전환 유틸리티 방법은 다음과 같습니다.

```typescript
/**
 * Switch context to RPC.
 */
switchToRpc(): RpcArgumentsHost;
/**
 * Switch context to HTTP.
 */
switchToHttp(): HttpArgumentsHost;
/**
 * Switch context to WebSockets.
 */
switchToWs(): WsArgumentsHost;
```

Let's rewrite the previous example using the `switchToHttp()` method. The `host.switchToHttp()` helper call returns an `HttpArgumentsHost` object that is appropriate for the HTTP application context. The `HttpArgumentsHost` object has two useful methods we can use to extract the desired objects. We also use the Express type assertions in this case to return native Express typed objects:

`switchToHttp()` 메서드를 사용하여 이전 예제를 다시 작성해 보겠습니다. `host.switchToHttp()` 헬퍼 호출은 HTTP 애플리케이션 컨텍스트에 적합한 `HttpArgumentsHost` 객체를 반환합니다. `HttpArgumentsHost` 객체에는 원하는 객체를 추출하는 데 사용할 수 있는 두가지 유용한 방법이 있습니다. 이 경우 Express 유형 주장(Express type assertions)을 사용하여 네이티브 Express 유형 객체를 반환합니다.

```typescript
const ctx = host.switchToHttp();
const request = ctx.getRequest<Request>();
const response = ctx.getResponse<Response>();
```

Similarly `WsArgumentsHost` and `RpcArgumentsHost` have methods to return appropriate objects in the microservices and WebSockets contexts. Here are the methods for `WsArgumentsHost`:

마찬가지로 `WsArgumentsHost` 및 `RpcArgumentsHost`에는 마이크로 서비스 및 WebSockets 컨텍스트에서 적절한 객체를 반환하는 메서드가 있습니다. 다음은 `WsArgumentsHost`의 메소드입니다.

```typescript
export interface WsArgumentsHost {
  /**
   * Returns the data object.
   */
  getData<T>(): T;
  /**
   * Returns the client object.
   */
  getClient<T>(): T;
}
```

Following are the methods for `RpcArgumentsHost`:

다음은 `RpcArgumentsHost`의 메서드입니다:

```typescript
export interface RpcArgumentsHost {
  /**
   * Returns the data object.
   */
  getData<T>(): T;

  /**
   * Returns the context object.
   */
  getContext<T>(): T;
}
```



## ExecutionContext class

`ExecutionContext` extends `ArgumentsHost`, providing additional details about the current execution process. Like `ArgumentsHost`, Nest provides an instance of `ExecutionContext` in places you may need it, such as in the `canActivate()` method of a [guard](https://docs.nestjs.com/guards#execution-context) and the `intercept()` method of an [interceptor](https://docs.nestjs.com/interceptors#execution-context). It provides the following methods:

`ExecutionContext`는 `ArgumentsHost`를 확장하여 현재 실행 프로세스에 대한 추가 세부 정보를 제공합니다. `ArgumentsHost`와 마찬가지로 Nest는 [가드](https://docs.nestjs.kr/guards#execution-context) 및 [인터셉터](https://docs.nestjs.kr/interceptors#execution-context)의 `intercept()` 메소드. 다음과 같은 방법을 제공합니다.

```typescript
export interface ExecutionContext extends ArgumentsHost {
  /**
   * Returns the type of the controller class which the current handler belongs to.
   */
  getClass<T>(): Type<T>;
  /**
   * Returns a reference to the handler (method) that will be invoked next in the
   * request pipeline.
   */
  getHandler(): Function;
}
```

The `getHandler()` method returns a reference to the handler about to be invoked. The `getClass()` method returns the type of the `Controller` class which this particular handler belongs to. For example, in an HTTP context, if the currently processed request is a `POST` request, bound to the `create()` method on the `CatsController`, `getHandler()` returns a reference to the `create()` method and `getClass()` returns the `CatsController`**type** (not instance).

`getHandler()` 메소드는 호출될 핸들러에 대한 참조를 반환합니다. `getClass()` 메소드는 이 특정 핸들러가 속한 `Controller` 클래스의 유형을 반환합니다. 예를 들어 HTTP 컨텍스트에서 현재 처리된 요청이 `CatsController`의 `create()` 메소드에 바인딩된 `POST` 요청인 경우 `getHandler()`는 `create()`에 대한 참조를 반환합니다. 메소드와 `getClass()`는 `CatsController`**타입**(인스턴스 아님)을 반환합니다.

```typescript
const methodKey = ctx.getHandler().name; // "create"
const className = ctx.getClass().name; // "CatsController"
```

The ability to access references to both the current class and handler method provides great flexibility. Most importantly, it gives us the opportunity to access the metadata set through the `@SetMetadata()` decorator from within guards or interceptors. We cover this use case below.

현재 클래스와 핸들러 메서드 모두에 대한 참조에 액세스하는 기능은 뛰어난 유연성을 제공합니다. 가장 중요한 것은 가드 또는 인터셉터 내에서 `@SetMetadata()` 데코레이터를 통해 메타 데이터 세트에 액세스할 수 있는 기회를 제공한다는 것입니다. 이 사용 사례는 아래에서 다룹니다.



## Reflection and metadata

Nest provides the ability to attach **custom metadata** to route handlers through the `@SetMetadata()` decorator. We can then access this metadata from within our class to make certain decisions.

Nest는 `@SetMetadata()` 데코레이터를 통해 라우트 핸들러에 **커스텀 메타데이터**를 첨부하는 기능을 제공합니다. 그런 다음 클래스 내에서 이 메타데이터에 액세스하여 특정 결정을 내릴 수 있습니다.

cats.controller.tsJS

```typescript
@Post()
@SetMetadata('roles', ['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> **HINT**
>
> The `@SetMetadata()` decorator is imported from the `@nestjs/common` package.
>
> `@SetMetadata()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

With the construction above, we attached the `roles` metadata (`roles` is a metadata key and `['admin']` is the associated value) to the `create()` method. While this works, it's not good practice to use `@SetMetadata()` directly in your routes. Instead, create your own decorators, as shown below:

위의 구성으로 `create()` 메소드에 `roles` 메타데이터(`roles`는 메타데이터 키이고 `['admin']`는 연관된 값임)를 첨부했습니다. 이것이 작동하지만 라우트에서 직접 `@SetMetadata()`를 사용하는 것은 좋지 않습니다. 대신 아래와 같이 자신만의 데코레이터를 만드세요.



### roles.decorator.ts

```typescript
import { SetMetadata } from '@nestjs/common';

export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

This approach is much cleaner and more readable, and is strongly typed. Now that we have a custom `@Roles()` decorator, we can use it to decorate the `create()` method.

이 접근 방식은 훨씬 깔끔하고 읽기 쉬우며 강력하게 입력됩니다. 이제 사용자 정의 `@Roles()` 데코레이터가 있으므로 이를 사용하여 `create()` 메소드를 데코레이션할 수 있습니다.

### cats.controller.ts

```typescript
@Post()
@Roles('admin')
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

To access the route's role(s) (custom metadata), we'll use the `Reflector` helper class, which is provided out of the box by the framework and exposed from the `@nestjs/core` package. `Reflector` can be injected into a class in the normal way:

라우트의 역할(커스텀 메타데이터)에 액세스하기 위해 프레임워크에서 즉시 제공되고 `@nestjs/core` 패키지에서 노출되는 `Reflector` 헬퍼 클래스를 사용합니다. `Reflector`는 일반적인 방법으로 클래스에 삽입될 수 있습니다.



### roles.guard.ts

```typescript
@Injectable()
export class RolesGuard {
  constructor(private reflector: Reflector) {}
}
```

> **HINT**
>
> The `Reflector` class is imported from the `@nestjs/core` package.
>
> `Reflector` 클래스는 `@nestjs/core` 패키지에서 가져옵니다.

Now, to read the handler metadata, use the `get()` method.

이제 핸들러 메타데이터를 읽으려면 `get()` 메소드를 사용하십시오.

```typescript
const roles = this.reflector.get<string[]>('roles', context.getHandler());
```

The `Reflector#get` method allows us to easily access the metadata by passing in two arguments: a metadata **key** and a **context** (decorator target) to retrieve the metadata from. In this example, the specified **key** is `'roles'` (refer back to the `roles.decorator.ts` file above and the `SetMetadata()` call made there). The context is provided by the call to `context.getHandler()`, which results in extracting the metadata for the currently processed route handler. Remember, `getHandler()` gives us a **reference** to the route handler function.

Alternatively, we may organize our controller by applying metadata at the controller level, applying to all routes in the controller class.

`Reflector#get` 메소드를 사용하면 메타데이터를 검색할 메타데이터 **키** 및 **컨텍스트**(데코레이터 대상)의 두 인수를 전달하여 메타데이터에 쉽게 액세스할 수 있습니다. 이 예에서 지정된 **키**는 `'roles'`입니다(위의 `roles.decorator.ts` 파일과 거기에서 이루어진 `SetMetadata()` 호출을 다시 참조하십시오). 컨텍스트는 `context.getHandler()` 호출에 의해 제공되며, 결과적으로 현재 처리된 라우트 핸들러에 대한 메타데이터가 추출됩니다. `getHandler()`는 라우트 핸들러 함수에 대한 **참조**를 제공합니다.

또는 컨트롤러 수준에서 메타데이터를 적용하고 컨트롤러 클래스의 모든 라우트에 적용하여 컨트롤러를 구성할 수 있습니다.

### cats.controller.ts

```typescript
@Roles('admin')
@Controller('cats')
export class CatsController {}
```

In this case, to extract controller metadata, we pass `context.getClass()` as the second argument (to provide the controller class as the context for metadata extraction) instead of `context.getHandler()`:

이 경우 컨트롤러 메타데이터를 추출하려면 `context.getHandler()` 대신 `context.getClass()`를 두번째 인수(컨트롤러 클래스를 메타데이터 추출의 컨텍스트로 제공하기 위해)로 전달합니다.

### roles.guard.ts

```typescript
const roles = this.reflector.get<string[]>('roles', context.getClass());
```

Given the ability to provide metadata at multiple levels, you may need to extract and merge metadata from several contexts. The `Reflector` class provides two utility methods used to help with this. These methods extract **both** controller and method metadata at once, and combine them in different ways.

Consider the following scenario, where you've supplied `'roles'` metadata at both levels.

여러 수준에서 메타 데이터를 제공할 수 있는 기능이 주어지면 여러 컨텍스트에서 메타데이터를 추출하고 병합해야 할 수 있습니다. `Reflector` 클래스는 이를 돕기 위해 사용되는 두가지 유틸리티 메소드를 제공합니다. 이러한 메소드는 컨트롤러와 메소드 메타데이터 **모두**를 한번에 추출하고 서로 다른 방식으로 결합합니다.

두 수준 모두에서 `'roles'` 메타데이터를 제공한 다음 시나리오를 고려하십시오.

### cats.controller.ts

```typescript
@Roles('user')
@Controller('cats')
export class CatsController {
  @Post()
  @Roles('admin')
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }
}
```

If your intent is to specify `'user'` as the default role, and override it selectively for certain methods, you would probably use the `getAllAndOverride()` method.

기본 역할로 `'user'`를 지정하고 특정 메서드에 대해 선택적으로 재정의하려는 경우 `getAllAndOverride()` 메서드를 사용할 수 있습니다.

```typescript
const roles = this.reflector.getAllAndOverride<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

A guard with this code, running in the context of the `create()` method, with the above metadata, would result in `roles` containing `['admin']`.

To get metadata for both and merge it (this method merges both arrays and objects), use the `getAllAndMerge()` method:

위의 메타데이터와 함께 `create()` 메소드의 컨텍스트에서 실행되는 이 코드를 사용하는 가드는 `['admin']`을 포함하는 `roles`를 생성합니다.

둘 다에 대한 메타데이터를 가져와 병합하려면(이 메서드는 배열과 객체를 모두 병합) `getAllAndMerge()` 메서드를 사용합니다.

```typescript
const roles = this.reflector.getAllAndMerge<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

This would result in `roles` containing `['user', 'admin']`.

For both of these merge methods, you pass the metadata key as the first argument, and an array of metadata target contexts (i.e., calls to the `getHandler()` and/or `getClass())` methods) as the second argument.

그러면 `['user', 'admin']`이 포함된 `roles`가 생성됩니다.

이 두가지 병합 메서드 모두에 대해 메타데이터 키를 첫번째 인수로 전달하고 메타데이터 대상 컨텍스트 배열(즉, `getHandler()` 및/ 또는 `getClass()` 메서드를 두번째 인수로 전달합니다.



#### 출처

> https://docs.nestjs.com/fundamentals/execution-context
>
> https://docs.nestjs.kr/fundamentals/execution-context