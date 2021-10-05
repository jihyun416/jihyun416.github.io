---
layout: post
title:  NestJS - Custom providers
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-05 17:07 +0900
---

### Custom providers

In earlier chapters, we touched on various aspects of **Dependency Injection (DI)** and how it is used in Nest. One example of this is the [constructor based](https://docs.nestjs.com/providers#dependency-injection) dependency injection used to inject instances (often service providers) into classes. You won't be surprised to learn that Dependency Injection is built into the Nest core in a fundamental way. So far, we've only explored one main pattern. As your application grows more complex, you may need to take advantage of the full features of the DI system, so let's explore them in more detail.

이전 장에서 **DI 종속성 주입**의 다양한 측면과 Nest에서 사용되는 방법에 대해 설명했습니다. 이에 대한 한가지 예는 인스턴스(종종 서비스 프로바이더)를 클래스에 주입하는데 사용되는 [생성자 기반](https://docs.nestjs.kr/providers#dependency-injection) 종속성 주입입니다. 의존성 주입이 기본적인 방식으로 Nest 코어에 내장되어 있다는 사실에 놀라지 않을 것입니다. 지금까지 우리는 하나의 주요 패턴만 살펴보았습니다. 애플리케이션이 더 복잡해짐에 따라 DI 시스템의 전체기능을 활용해야할 수도 있으므로 더 자세히 살펴 보겠습니다.



## DI fundamentals

Dependency injection is an [inversion of control (IoC)](https://en.wikipedia.org/wiki/Inversion_of_control) technique wherein you delegate instantiation of dependencies to the IoC container (in our case, the NestJS runtime system), instead of doing it in your own code imperatively. Let's examine what's happening in this example from the [Providers chapter](https://docs.nestjs.com/providers).

First, we define a provider. The `@Injectable()` decorator marks the `CatsService` class as a provider.

종속성 주입은 대신 IoC 컨테이너(이 경우 NestJS 런타임 시스템)에 종속성 인스턴스화를 위임하는 [IoC(inversion of control)](https://en.wikipedia.org/wiki/Inversion_of_control) 기술입니다. 자신의 코드에서 명령적으로 수행하는 것입니다. [프로바이더 장](https://docs.nestjs.kr/providers)에서 이 예제에서 어떤 일이 발생하는지 살펴보겠습니다.

먼저 프로바이더를 정의합니다. `@Injectable()` 데코레이터는 `CatsService` 클래스를 프로바이더로 표시합니다.

### cats.service.ts

```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  findAll(): Cat[] {
    return this.cats;
  }
}
```

Then we request that Nest inject the provider into our controller class:

그런 다음 Nest가 프로바이더를 컨트롤러 클래스에 주입하도록 요청합니다.

### cats.controller.ts

```typescript
import { Controller, Get } from '@nestjs/common';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

Finally, we register the provider with the Nest IoC container:

마지막으로 Nest IoC 컨테이너에 프로바이더를 등록합니다.

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats/cats.controller';
import { CatsService } from './cats/cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class AppModule {}
```

What exactly is happening under the covers to make this work? There are three key steps in the process:

1. In `cats.service.ts`, the `@Injectable()` decorator declares the `CatsService` class as a class that can be managed by the Nest IoC container.
2. In `cats.controller.ts`, `CatsController` declares a dependency on the `CatsService` token with constructor injection:

이 작업을 수행하기 위해 정확히 어떤 일이 일어나고 있습니까? 프로세스에는 세가지 주요 단계가 있습니다.

1. `cats.service.ts`에서 `@Injectable()` 데코레이터는 `CatsService` 클래스를 Nest IoC 컨테이너에서 관리할 수 있는 클래스로 선언합니다.
2. `cats.controller.ts`에서 `CatsController`는 생성자(constructor) 주입으로 `CatsService` 토큰에 대한 종속성을 선언합니다.

```typescript
  constructor(private catsService: CatsService)
```

1. In `app.module.ts`, we associate the token `CatsService` with the class `CatsService` from the `cats.service.ts` file. We'll [see below](https://docs.nestjs.com/fundamentals/custom-providers#standard-providers) exactly how this association (also called *registration*) occurs.

   app.module.ts`에서 `CatsService` 토큰을 `cats.service.ts` 파일의 `CatsService` 클래스와 연관시킵니다. 이 연결 (_registration_이라고도 함)이 어떻게 발생하는지 정확히 [아래를 참조](https://docs.nestjs.kr/fundamentals/custom-providers#standard-providers)합니다.

When the Nest IoC container instantiates a `CatsController`, it first looks for any dependencies*. When it finds the `CatsService` dependency, it performs a lookup on the `CatsService` token, which returns the `CatsService` class, per the registration step (#3 above). Assuming `SINGLETON` scope (the default behavior), Nest will then either create an instance of `CatsService`, cache it, and return it, or if one is already cached, return the existing instance.

*This explanation is a bit simplified to illustrate the point. One important area we glossed over is that the process of analyzing the code for dependencies is very sophisticated, and happens during application bootstrapping. One key feature is that dependency analysis (or "creating the dependency graph"), is **transitive**. In the above example, if the `CatsService` itself had dependencies, those too would be resolved. The dependency graph ensures that dependencies are resolved in the correct order - essentially "bottom up". This mechanism relieves the developer from having to manage such complex dependency graphs.

Nest IoC 컨테이너가 `CatsController`를 인스턴스화할 때 먼저 종속성*을 찾습니다. `CatsService` 종속성을 찾으면 등록단계(위 #3)에 따라 `CatsService` 클래스를 반환하는 `CatsService` 토큰에 대한 조회를 수행합니다. `SINGLETON` 범위(기본 동작)를 가정하면 Nest는 `CatsService`의 인스턴스를 만들고 캐시한 다음 반환하거나 이미 캐시된 경우 기존 인스턴스를 반환합니다.

*이 설명은 요점을 설명하기 위해 약간 단순화되었습니다. 우리가 살펴본 중요한 영역중 하나는 종속성에 대한 코드 분석 프로세스가 매우 정교하고 애플리케이션 부트스트랩중에 발생한다는 것입니다. 한가지 주요 기능은 종속성 분석(또는 "종속성 그래프 생성")이 **전이적**(transitive)이라는 것입니다. 위의 예에서 `CatsService` 자체에 종속성이 있으면 이것도 해결됩니다. 종속성 그래프는 종속성이 올바른 순서(기본적으로 "상향(bottom up)")로 해결되도록 합니다. 이 메커니즘은 개발자가 복잡한 종속성 그래프를 관리할 필요가 없도록 합니다.



## Standard providers

Let's take a closer look at the `@Module()` decorator. In `app.module`, we declare:

`@Module()` 데코레이터를 자세히 살펴 보겠습니다. `app.module`에서 다음을 선언합니다.

```typescript
@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
```

The `providers` property takes an array of `providers`. So far, we've supplied those providers via a list of class names. In fact, the syntax `providers: [CatsService]` is short-hand for the more complete syntax:

`providers` 속성은 `providers`의 배열을 받습니다. 지금까지 클래스 이름 목록을 통해 이러한 프로바이더를 제공했습니다. 실제로 `providers: [CatsService]` 구문은 보다 완전한 구문을 위한 축약형입니다.

```typescript
providers: [
  {
    provide: CatsService,
    useClass: CatsService,
  },
];
```

Now that we see this explicit construction, we can understand the registration process. Here, we are clearly associating the token `CatsService` with the class `CatsService`. The short-hand notation is merely a convenience to simplify the most common use-case, where the token is used to request an instance of a class by the same name.

이제 이 명시적인 구성을 보았으므로 등록 프로세스를 이해할 수 있습니다. 여기서는 `CatsService` 토큰을 `CatsService` 클래스와 명확하게 연관시킵니다. 축약 표기법은 가장 일반적인 사용사례를 단순화하기 위한 편의일뿐입니다. 토큰은 동일한 이름으로 클래스의 인스턴스를 요청하는 데 사용됩니다.



## Custom providers

What happens when your requirements go beyond those offered by *Standard providers*? Here are a few examples:

- You want to create a custom instance instead of having Nest instantiate (or return a cached instance of) a class
- You want to re-use an existing class in a second dependency
- You want to override a class with a mock version for testing

Nest allows you to define Custom providers to handle these cases. It provides several ways to define custom providers. Let's walk through them.

귀하의 요구 사항이 *표준 프로바이더* 에서 제공하는 것 이상으로 넘어가면 어떻게됩니까? 다음은 몇가지 예입니다.

- Nest가 클래스를 인스턴스화(또는 캐시된 인스턴스 반환)하는 대신 사용자 지정 인스턴스를 만들고 싶습니다.
- 두번째 종속성에서 기존 클래스를 재사용하려는 경우
- 테스트를 위해 모의(mock) 버전으로 클래스를 재정의하려는 경우

Nest를 사용하면 이러한 경우를 처리할 사용자 지정 프로바이더를 정의할 수 있습니다. 사용자 지정 프로바이더를 정의하는 여러 방법을 제공합니다. 그것들을 살펴 보겠습니다.



## Value providers: `useValue`

The `useValue` syntax is useful for injecting a constant value, putting an external library into the Nest container, or replacing a real implementation with a mock object. Let's say you'd like to force Nest to use a mock `CatsService` for testing purposes.

`useValue` 구문은 상수값을 삽입하거나 Nest 컨테이너에 외부 라이브러리를 넣거나 실제 구현을 모의 객체로 대체하는 데 유용합니다. Nest가 테스트 목적으로 모의 `CatsService`를 사용하도록 강제하고 싶다고 가정해 보겠습니다.

```typescript
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

In this example, the `CatsService` token will resolve to the `mockCatsService` mock object. `useValue` requires a value - in this case a literal object that has the same interface as the `CatsService` class it is replacing. Because of TypeScript's [structural typing](https://www.typescriptlang.org/docs/handbook/type-compatibility.html), you can use any object that has a compatible interface, including a literal object or a class instance instantiated with `new`.

이 예에서 `CatsService` 토큰은 `mockCatsService` 모의 객체로 확인됩니다. `useValue`에는 값이 필요합니다. 이 경우 교체 할 `CatsService` 클래스와 동일한 인터페이스를 가진 리터럴 객체입니다. TypeScript의 [구조적 유형](https://www.typescriptlang.org/docs/handbook/type-compatibility.html)때문에 리터럴 객체 또는 `new`로 인스턴스화된 클래스 인스턴스를 포함하여 호환되는 인터페이스가 있는 모든 객체를 사용할 수 있습니다.



## Non-class-based provider tokens

So far, we've used class names as our provider tokens (the value of the `provide` property in a provider listed in the `providers` array). This is matched by the standard pattern used with [constructor based injection](https://docs.nestjs.com/providers#dependency-injection), where the token is also a class name. (Refer back to [DI Fundamentals](https://docs.nestjs.com/fundamentals/custom-providers#di-fundamentals) for a refresher on tokens if this concept isn't entirely clear). Sometimes, we may want the flexibility to use strings or symbols as the DI token. For example:

지금까지 우리는 클래스 이름을 프로바이더 토큰(`providers` 배열에 나열된 프로바이더의 `provide` 속성 값)으로 사용했습니다. 이는 토큰이 클래스 이름이기도 한 [생성자 기반 주입](https://docs.nestjs.kr/providers#dependency-injection)과 함께 사용되는 표준 패턴과 일치합니다. (이 개념이 완전히 명확하지 않은 경우 토큰에 대한 복습은 [DI 기초](https://docs.nestjs.kr/fundamentals/custom-providers#di-fundamentals)를 다시 참조하세요). 때때로 우리는 DI 토큰으로 문자열이나 기호를 사용할 수 있는 유연성을 원할 수 있습니다. 예를 들면:

```typescript
import { connection } from './connection';

@Module({
  providers: [
    {
      provide: 'CONNECTION',
      useValue: connection,
    },
  ],
})
export class AppModule {}
```

In this example, we are associating a string-valued token (`'CONNECTION'`) with a pre-existing `connection` object we've imported from an external file.

이 예에서는 문자열 값 토큰 (`'CONNECTION'`)을 외부 파일에서 가져온 기존 `connection` 객체와 연결합니다.

> **NOTICE**
>
> In addition to using strings as token values, you can also use JavaScript [symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) or TypeScript [enums](https://www.typescriptlang.org/docs/handbook/enums.html).
>
> 문자열을 토큰 값으로 사용하는 것 외에도 자바스크립트 [심볼 symbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) 또는 TypeScript [열거형 enums](https://www.typescriptlang.org/docs/handbook/enums.html)를 사용할 수 있습니다.

We've previously seen how to inject a provider using the standard [constructor based injection](https://docs.nestjs.com/providers#dependency-injection) pattern. This pattern **requires** that the dependency be declared with a class name. The `'CONNECTION'` custom provider uses a string-valued token. Let's see how to inject such a provider. To do so, we use the `@Inject()` decorator. This decorator takes a single argument - the token.

이전에 표준 [생성자 기반 주입](https://docs.nestjs.kr/providers#dependency-injection) 패턴을 사용하여 공급자를 주입하는 방법을 살펴 보았습니다. 이 패턴은 종속성이 클래스 이름으로 선언되도록 **요구**합니다. `'CONNECTION'` 커스텀 프로바이더는 문자열 값 토큰을 사용합니다. 그러한 프로바이더를 주입하는 방법을 살펴 보겠습니다. 이를 위해 `@Inject()` 데코레이터를 사용합니다. 이 데코레이터는 단일인수인 토큰을 받습니다.

```typescript
@Injectable()
export class CatsRepository {
  constructor(@Inject('CONNECTION') connection: Connection) {}
}
```

> **HINT**
>
> The `@Inject()` decorator is imported from `@nestjs/common` package.
>
> `@Inject()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

While we directly use the string `'CONNECTION'` in the above examples for illustration purposes, for clean code organization, it's best practice to define tokens in a separate file, such as `constants.ts`. Treat them much as you would symbols or enums that are defined in their own file and imported where needed.

위의 예에서는 설명 목적으로 `'CONNECTION'` 문자열을 직접 사용하지만 깔끔한 코드 구성을 위해 `constants.ts`와 같은 별도의 파일에 토큰을 정의하는 것이 가장 좋습니다. 자체 파일에 정의되고 필요한 곳에 가져오는 기호 또는 열거형처럼 처리하십시오.



## Class providers: `useClass`

The `useClass` syntax allows you to dynamically determine a class that a token should resolve to. For example, suppose we have an abstract (or default) `ConfigService` class. Depending on the current environment, we want Nest to provide a different implementation of the configuration service. The following code implements such a strategy.

`useClass` 구문을 사용하면 토큰이 확인해야 하는 클래스를 동적으로 결정할 수 있습니다. 예를 들어 추상(또는 기본) `ConfigService` 클래스가 있다고 가정합니다. 현재 환경에 따라 Nest가 구성 서비스의 다른 구현을 제공하기를 원합니다. 다음 코드는 이러한 전략을 구현합니다.

```typescript
const configServiceProvider = {
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};

@Module({
  providers: [configServiceProvider],
})
export class AppModule {}
```

Let's look at a couple of details in this code sample. You'll notice that we define `configServiceProvider` with a literal object first, then pass it in the module decorator's `providers` property. This is just a bit of code organization, but is functionally equivalent to the examples we've used thus far in this chapter.

Also, we have used the `ConfigService` class name as our token. For any class that depends on `ConfigService`, Nest will inject an instance of the provided class (`DevelopmentConfigService` or `ProductionConfigService`) overriding any default implementation that may have been declared elsewhere (e.g., a `ConfigService` declared with an `@Injectable()` decorator).

이 코드 샘플에서 몇가지 세부 사항을 살펴 보겠습니다. 먼저 리터럴 객체로 `configServiceProvider`를 정의한 다음 모듈 데코레이터의 `providers` 속성에 전달합니다. 이것은 약간의 코드 구성에 불과하지만 이 장에서 지금까지 사용한 예제와 기능적으로 동일합니다.

또한 토큰으로 `ConfigService` 클래스 이름을 사용했습니다. `ConfigService`에 의존하는 모든 클래스의 경우 Nest는 제공된 클래스(`DevelopmentConfigService` 또는 `ProductionConfigService`)의 인스턴스를 삽입하여 다른 곳에서 선언되었을 수 있는 기본 구현(예: `@Injectable()` 데코레이터로 선언된 `ConfigService`)을 재정의합니다.



## Factory providers: `useFactory`

The `useFactory` syntax allows for creating providers **dynamically**. The actual provider will be supplied by the value returned from a factory function. The factory function can be as simple or complex as needed. A simple factory may not depend on any other providers. A more complex factory can itself inject other providers it needs to compute its result. For the latter case, the factory provider syntax has a pair of related mechanisms:

1. The factory function can accept (optional) arguments.
2. The (optional) `inject` property accepts an array of providers that Nest will resolve and pass as arguments to the factory function during the instantiation process. The two lists should be correlated: Nest will pass instances from the `inject` list as arguments to the factory function in the same order.

The example below demonstrates this.

`useFactory` 구문을 사용하면 **동적으로** 프로바이더를 만들 수 있습니다. 실제 프로바이더는 팩토리 함수에서 반환된 값으로 제공됩니다. 팩토리 함수는 필요에 따라 간단하거나 복잡할 수 있습니다. 단순한 팩토리는 다른 프로바이더에 의존하지 않을 수 있습니다. 더 복잡한 팩토리는 결과를 계산하는 데 필요한 다른 프로바이더를 자체적으로 주입할 수 있습니다. 후자의 경우 팩토리 프로바이더 구문에는 한 쌍의 관련 메커니즘이 있습니다.

1. 팩토리 함수는 (선택 사항) 인수를 받을 수 있습니다.
2. (선택 사항) `inject` 속성은 Nest가 인스턴스화 프로세스중에 확인하고 팩토리 함수에 인수로 전달할 프로바이더 배열을 허용합니다. 두 목록은 상호 연관되어야 합니다. Nest는 `inject` 목록의 인스턴스를 동일한 순서로 팩토리 함수에 대한 인수로 전달합니다.

아래 예가 이를 보여줍니다.

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
})
export class AppModule {}
```



## Alias providers: `useExisting`

The `useExisting` syntax allows you to create aliases for existing providers. This creates two ways to access the same provider. In the example below, the (string-based) token `'AliasedLoggerService'` is an alias for the (class-based) token `LoggerService`. Assume we have two different dependencies, one for `'AliasedLoggerService'` and one for `LoggerService`. If both dependencies are specified with `SINGLETON` scope, they'll both resolve to the same instance.

`useExisting` 구문을 사용하면 기존 프로바이더의 별칭을 만들 수 있습니다. 이것은 동일한 프로바이더에 액세스하는 두가지 방법을 만듭니다. 아래 예에서 (문자열 기반) 토큰 `'AliasedLoggerService'`는 (클래스 기반) 토큰 `LoggerService`의 별칭입니다. `'AliasedLoggerService'`와 `LoggerService`에 대한 두개의 서로 다른 종속성이 있다고 가정합니다. 두 종속성이 `SINGLETON` 범위로 지정되면 둘 다 동일한 인스턴스로 확인됩니다.

```typescript
@Injectable()
class LoggerService {
  /* implementation details */
}

const loggerAliasProvider = {
  provide: 'AliasedLoggerService',
  useExisting: LoggerService,
};

@Module({
  providers: [LoggerService, loggerAliasProvider],
})
export class AppModule {}
```



## Non-service based providers

While providers often supply services, they are not limited to that usage. A provider can supply **any** value. For example, a provider may supply an array of configuration objects based on the current environment, as shown below:

프로바이더가 서비스를 제공하는 경우가 많지만 해당 용도에 국한되지는 않습니다. 프로바이더는 **모든** 값을 제공할 수 있습니다. 예를 들어 프로바이더는 아래와 같이 현재 환경을 기반으로 구성 객체의 배열을 제공할 수 있습니다.

```typescript
const configFactory = {
  provide: 'CONFIG',
  useFactory: () => {
    return process.env.NODE_ENV === 'development' ? devConfig : prodConfig;
  },
};

@Module({
  providers: [configFactory],
})
export class AppModule {}
```



## Export custom provider

Like any provider, a custom provider is scoped to its declaring module. To make it visible to other modules, it must be exported. To export a custom provider, we can either use its token or the full provider object.

The following example shows exporting using the token:

다른 프로바이더와 마찬가지로 커스텀 프로바이더는 선언 모듈로 범위가 지정됩니다. 다른 모듈에 표시하려면 내보내야 합니다. 커스텀 프로바이더를 내보내려면 해당 토큰 또는 전체 프로바이더 객체를 사용할 수 있습니다.

다음 예제는 토큰을 사용한 내보내기를 보여줍니다.

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: ['CONNECTION'],
})
export class AppModule {}
```

Alternatively, export with the full provider object:

또는 전체 프로바이더 객체를 사용하여 내 보냅니다.

```typescript
const connectionFactory = {
  provide: 'CONNECTION',
  useFactory: (optionsProvider: OptionsProvider) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider],
};

@Module({
  providers: [connectionFactory],
  exports: [connectionFactory],
})
export class AppModule {}
```



#### 출처

> https://docs.nestjs.com/fundamentals/custom-providers
>
> https://docs.nestjs.kr/fundamentals/custom-providers

