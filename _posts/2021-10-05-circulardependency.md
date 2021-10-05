---
layout: post
title:  NestJS - Circular dependency
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-05 18:25 +0900
---

## Circular dependency

A circular dependency occurs when two classes depend on each other. For example, class A needs class B, and class B also needs class A. Circular dependencies can arise in Nest between modules and between providers.

While circular dependencies should be avoided where possible, you can't always do so. In such cases, Nest enables resolving circular dependencies between providers in two ways. In this chapter, we describe using **forward referencing** as one technique, and using the **ModuleRef** class to retrieve a provider instance from the DI container as another.

We also describe resolving circular dependencies between modules.

순환 종속성은 두 클래스가 서로 의존할 때 발생합니다. 예를 들어, 클래스 A에는 클래스 B가 필요하고 클래스 B에는 클래스 A도 필요합니다. 순환 종속성은 모듈간 및 프로바이더간에 Nest에서 발생할 수 있습니다.

순환 종속성은 가능한 한 피해야 하지만 항상 그렇게 할 수는 없습니다. 이러한 경우 Nest는 두가지 방법으로 프로바이더간의 순환 종속성을 해결할 수 있습니다. 이 장에서는 **순방향 참조**를 한 기술로 사용하고 **ModuleRef** 클래스를 사용하여 DI 컨테이너에서 프로바이더 인스턴스를 다른 기술로 검색하는 방법을 설명합니다.

또한 모듈간의 순환 종속성 해결에 대해서도 설명합니다.

> **WARNING**
>
> A circular dependency might also be caused when using "barrel files"/index.ts files to group imports. Barrel files should be omitted when it comes to module/provider classes. For example, barrel files should not be used when importing files within the same directory as the barrel file, i.e. `cats/cats.controller` should not import `cats` to import the `cats/cats.service` file. For more details please also see [this github issue](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191).
>
> "barel files"/index.ts 파일을 사용하여 가져오기를 그룹화할 때 순환 종속성이 발생할 수도 있습니다. 모듈/프로바이더 클래스와 관련하여 배럴 파일은 생략해야 합니다. 예를 들어, 배럴 파일과 동일한 디렉토리 내에서 파일을 가져올 때는 배럴 파일을 사용해서는 안됩니다. 자세한 내용은 [이 github 이슈](https://github.com/nestjs/nest/issues/1181#issuecomment-430197191)를 참조하세요.



## Forward reference

A **forward reference** allows Nest to reference classes which aren't yet defined using the `forwardRef()` utility function. For example, if `CatsService` and `CommonService` depend on each other, both sides of the relationship can use `@Inject()` and the `forwardRef()` utility to resolve the circular dependency. Otherwise Nest won't instantiate them because all of the essential metadata won't be available. Here's an example:

**포워드 참조**를 사용하면 Nest가 아직 `forwardRef()` 유틸리티 함수를 사용하여 정의되지 않은 클래스를 참조할 수 있습니다. 예를 들어 `CatsService`와 `CommonService`가 서로 의존하는 경우 관계의 양쪽 모두 `@Inject()` 및 `forwardRef()` 유틸리티를 사용하여 순환 종속성을 해결할 수 있습니다. 그렇지 않으면 모든 필수 메타 데이터를 사용할 수 없기 때문에 Nest가 인스턴스화하지 않습니다. 예를 들면 다음과 같습니다.

### cats.service.ts

```typescript
@Injectable()
export class CatsService {
  constructor(
    @Inject(forwardRef(() => CommonService))
    private commonService: CommonService,
  ) {}
}
```

> **HINT**
>
> The `forwardRef()` function is imported from the `@nestjs/common` package.
>
> `forwardRef()` 함수는 `@nestjs/common` 패키지에서 가져옵니다.

That covers one side of the relationship. Now let's do the same with `CommonService`:

그것은 관계의 한쪽을 다룹니다. 이제 `CommonService`로 똑같이 합시다.

### common.service.ts

```typescript
@Injectable()
export class CommonService {
  constructor(
    @Inject(forwardRef(() => CatsService))
    private catsService: CatsService,
  ) {}
}
```

> **WARNING**
>
> The order of instantiation is indeterminate. Make sure your code does not depend on which constructor is called first.
>
> 인스턴스화 순서가 결정되지 않았습니다. 코드가 먼저 호출되는 생성자(constructor)에 의존하지 않는지 확인하십시오.



## ModuleRef class alternative

An alternative to using `forwardRef()` is to refactor your code and use the `ModuleRef` class to retrieve a provider on one side of the (otherwise) circular relationship. Learn more about the `ModuleRef` utility class [here](https://docs.nestjs.com/fundamentals/module-ref).

`forwardRef()` 사용에 대한 대안은 코드를 리팩터링하고 `ModuleRef` 클래스를 사용하여(그렇지 않은 경우) 순환 관계의 한쪽에서 프로바이더를 검색하는 것입니다. `ModuleRef` 유틸리티 클래스에 대한 자세한 내용은 [여기](https://docs.nestjs.kr/fundamentals/module-ref)를 참조하세요.



## Module forward reference

In order to resolve circular dependencies between modules, use the same `forwardRef()` utility function on both sides of the modules association. For example:

`forwardRef()` 사용에 대한 대안은 코드를 리팩터링하고 `ModuleRef` 클래스를 사용하여(그렇지 않은 경우) 순환 관계의 한쪽에서 프로바이더를 검색하는 것입니다. `ModuleRef` 유틸리티 클래스에 대한 자세한 내용은 [여기](https://docs.nestjs.kr/fundamentals/module-ref)를 참조하세요.

### common.module.ts

```typescript
@Module({
  imports: [forwardRef(() => CatsModule)],
})
export class CommonModule {}
```



#### 출처

> https://docs.nestjs.com/fundamentals/circular-dependency
>
> https://docs.nestjs.kr/fundamentals/circular-dependency