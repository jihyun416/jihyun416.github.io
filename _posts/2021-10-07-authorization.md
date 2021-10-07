---
layout: post
title:  NestJS - Authorization
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-07 19:03 +0900
---

## Authorization

**Authorization** refers to the process that determines what a user is able to do. For example, an administrative user is allowed to create, edit, and delete posts. A non-administrative user is only authorized to read the posts.

Authorization is orthogonal and independent from authentication. However, authorization requires an authentication mechanism.

There are many different approaches and strategies to handle authorization. The approach taken for any project depends on its particular application requirements. This chapter presents a few approaches to authorization that can be adapted to a variety of different requirements.

**권한 부여(authorization)**은 사용자가 수행할 수 있는 작업을 결정하는 프로세스를 나타냅니다. 예를 들어, 관리 사용자는 게시물을 작성, 편집 및 삭제할 수 있습니다. 관리자가 아닌 사용자는 게시물을 읽을 수만 있습니다.

권한 부여는 직각이며 인증(authentication)과 독립적입니다. 그러나 권한 부여에는 인증 메커니즘이 필요합니다.

권한 부여를 처리하기 위한 다양한 접근 방식과 전략이 있습니다. 모든 프로젝트에 적용되는 접근 방식은 특정 애플리케이션 요구사항에 따라 다릅니다. 이 장에서는 다양한 요구사항에 적용할 수 있는 권한 부여에 대한 몇가지 접근 방식을 제공합니다.



## Basic RBAC implementation

Role-based access control (**RBAC**) is a policy-neutral access-control mechanism defined around roles and privileges. In this section, we'll demonstrate how to implement a very basic RBAC mechanism using Nest [guards](https://docs.nestjs.com/guards).

First, let's create a `Role` enum representing roles in the system:

역할 기반 액세스 제어 (**RBAC(Role-based access control**)는 역할 및 권한에 대해 정의된 정책 중립적 액세스 제어 메커니즘입니다. 이 섹션에서는 Nest [가드](https://docs.nestjs.kr/guards)를 사용하여 매우 기본적인 RBAC 메커니즘을 구현하는 방법을 보여줍니다.

먼저 시스템에서 역할을 나타내는 `Role` 열거(enum)형을 만들어 보겠습니다.

### role.enum.ts

```typescript
export enum Role {
  User = 'user',
  Admin = 'admin',
}
```

> **HINT**
>
> In more sophisticated systems, you may store roles within a database, or pull them from the external authentication provider.
>
> 보다 정교한 시스템에서는 데이터베이스내에 역할을 저장하거나 외부 인증 프로바이더로부터 역할을 가져올 수 있습니다.

With this in place, we can create a `@Roles()` decorator. This decorator allows specifying what roles are required to access specific resources.

이것으로 `@Roles()` 데코레이터를 만들 수 있습니다. 이 데코레이터를 사용하면 특정 리소스에 액세스하는 데 필요한 역할을 지정할 수 있습니다.

### roles.decorator.ts

```typescript
import { SetMetadata } from '@nestjs/common';
import { Role } from '../enums/role.enum';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]) => SetMetadata(ROLES_KEY, roles);
```

Now that we have a custom `@Roles()` decorator, we can use it to decorate any route handler.

이제 사용자 정의 `@Roles()` 데코레이터가 있으므로 이를 사용하여 모든 라우트 핸들러를 데코레이션할 수 있습니다.

### cats.controller.ts

```typescript
@Post()
@Roles(Role.Admin)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

Finally, we create a `RolesGuard` class which will compare the roles assigned to the current user to the actual roles required by the current route being processed. In order to access the route's role(s) (custom metadata), we'll use the `Reflector` helper class, which is provided out of the box by the framework and exposed from the `@nestjs/core` package.

마지막으로 현재 사용자에게 할당된 역할을 처리중인 현재 경로에 필요한 실제 역할과 비교하는 `RolesGuard` 클래스를 만듭니다. 경로의 역할 (사용자 지정 메타 데이터)에 액세스하기 위해 프레임 워크에서 즉시 제공되고 `@nestjs/core` 패키지에서 노출되는 `Reflector` 도우미 클래스를 사용합니다.

### roles.guard.ts

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (!requiredRoles) {
      return true;
    }
    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

> **HINT**
>
> Refer to the [Reflection and metadata](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata) section of the Execution context chapter for more details on utilizing `Reflector` in a context-sensitive way.
>
> `Reflector`를 상황에 맞는 방식으로 활용하는 방법에 대한 자세한 내용은 실행 컨텍스트 장의 [Reflection and metadata](https://docs.nestjs.kr/fundamentals/execution-context#reflection-and-metadata) 섹션을 참조하세요.

> **NOTICE**
>
> This example is named "**basic**" as we only check for the presence of roles on the route handler level. In real-world applications, you may have endpoints/handlers that involve several operations, in which each of them requires a specific set of permissions. In this case, you'll have to provide a mechanism to check roles somewhere within your business-logic, making it somewhat harder to maintain as there will be no centralized place that associates permissions with specific actions.
>
> 이 예는 경로 핸들러 수준에서 역할의 존재 여부만 확인하므로 "**basic**"으로 이름이 지정됩니다. 실제 애플리케이션에서는 여러 작업을 포함하는 엔드 포인트/핸들러가 있을 수 있으며 각 작업에는 특정 권한 집합이 필요합니다. 이 경우 비즈니스 로직내 어딘가에서 역할을 확인하는 메커니즘을 제공해야 하므로 권한을 특정 작업과 연결하는 중앙 집중식 장소가 없기 때문에 유지 관리가 다소 어려워집니다.

In this example, we assumed that `request.user` contains the user instance and allowed roles (under the `roles` property). In your app, you will probably make that association in your custom **authentication guard** - see [authentication](https://docs.nestjs.com/security/authentication) chapter for more details.

To make sure this example works, your `User` class must look as follows:

이 예에서는 `request.user`에 사용자 인스턴스와 허용된 역할 (`roles`속성 아래)이 포함되어 있다고 가정했습니다. 앱에서 사용자 정의 **인증 가드**에서 해당 연결을 만들 수 있습니다. 자세한 내용은 [인증](https://docs.nestjs.kr/security/authentication) 장을 참조하십시오.

이 예제가 작동하는지 확인하려면 `User` 클래스가 다음과 같아야 합니다.

```typescript
class User {
  // ...other properties
  roles: Role[];
}
```

Lastly, make sure to register the `RolesGuard`, for example, at the controller level, or globally:

마지막으로 `RolesGuard`를 컨트롤러 수준에서 등록하거나 전역적으로 등록해야 합니다.

```typescript
providers: [
  {
    provide: APP_GUARD,
    useClass: RolesGuard,
  },
],
```

When a user with insufficient privileges requests an endpoint, Nest automatically returns the following response:

권한이 부족한 사용자가 엔드 포인트를 요청하면 Nest는 자동으로 다음 응답을 반환합니다.

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

> **HINT**
>
> If you want to return a different error response, you should throw your own specific exception instead of returning a boolean value.
>
> 다른 오류 응답을 반환하려면 부울 값을 반환하는 대신 고유한 예외를 발생시켜야 합니다.



## Claims-based authorization

When an identity is created it may be assigned one or more claims issued by a trusted party. A claim is a name-value pair that represents what the subject can do, not what the subject is.

To implement a Claims-based authorization in Nest, you can follow the same steps we have shown above in the [RBAC](https://docs.nestjs.com/security/authorization#basic-rbac-implementation) section with one significant difference: instead of checking for specific roles, you should compare **permissions**. Every user would have a set of permissions assigned. Likewise, each resource/endpoint would define what permissions are required (for example, through a dedicated `@RequirePermissions()` decorator) to access them.

ID가 생성되면 신뢰할 수 있는 당사자가 발행한 하나 이상의 클레임이 할당될 수 있습니다. 클레임은 주체가 아닌 주체가 할 수 있는 일을 나타내는 이름-값 쌍입니다.

Nest에서 클레임 기반 승인을 구현하려면 위의 [RBAC](https://docs.nestjs.kr/security/authorization#basic-rbac-implementation) 섹션에서 설명한 것과 동일한 단계를 수행 할 수 있습니다. 한 가지 중요한 차이점이 있습니다. 특정 역할을 확인하는 대신 **권한**을 비교해야합니다. 모든 사용자에게는 일련의 권한이 할당됩니다. 마찬가지로 각 리소스/엔드 포인트는 액세스하는 데 필요한 권한 (예: 전용 `@RequirePermissions()` 데코레이터를 통해)을 정의합니다.

### cats.controller.ts

```typescript
@Post()
@RequirePermissions(Permission.CREATE_CAT)
create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

> **HINT**
>
> In the example above, `Permission` (similar to `Role` we have shown in RBAC section) is a TypeScript enum that contains all the permissions available in your system.
>
> 위의 예에서 `Permission` (RBAC 섹션에 표시된 `Role`과 유사)은 시스템에서 사용 가능한 모든 권한을 포함하는 TypeScript 열거(enum) 형입니다.



## Integrating CASL

[CASL](https://casl.js.org/) is an isomorphic authorization library which restricts what resources a given client is allowed to access. It's designed to be incrementally adoptable and can easily scale between a simple claim based and fully featured subject and attribute based authorization.

To start, first install the `@casl/ability` package:

[CASL](https://casl.js.org/)은 주어진 클라이언트가 액세스할 수 있는 리소스를 제한하는 동형 승인 라이브러리입니다. 점진적으로 채택할 수 있도록 설계되었으며 단순 클레임 기반과 완전한 기능을 갖춘 주제(subject) 및 속성(attribute) 기반 권한간에 쉽게 확장할 수 있습니다.

시작하려면 먼저 `@casl/ability` 패키지를 설치하십시오.

```bash
$ npm i @casl/ability
```

> **HINT**
>
> In this example, we chose CASL, but you can use any other library like `accesscontrol` or `acl`, depending on your preferences and project needs.
>
> 이 예에서는 CASL을 선택했지만 기본 설정과 프로젝트 요구 사항에 따라 `accesscontrol` 또는 `acl`과 같은 다른 라이브러리를 사용할 수 있습니다.

Once the installation is complete, for the sake of illustrating the mechanics of CASL, we'll define two entity classes: `User` and `Article`.

설치가 완료되면 CASL의 메커니즘을 설명하기 위해 두 개의 엔티티 클래스 인 `User`와 `Article`을 정의합니다.

```typescript
class User {
  id: number;
  isAdmin: boolean;
}
```

`User` class consists of two properties, `id`, which is a unique user identifier, and `isAdmin`, indicating whether a user has administrator privileges.

`User` 클래스는 고유한 사용자 식별자인 `id`와 사용자에게 관리자 권한이 있는지 여부를 나타내는 `isAdmin`의 두 가지 속성으로 구성됩니다.

```typescript
class Article {
  id: number;
  isPublished: boolean;
  authorId: number;
}
```

`Article` class has three properties, respectively `id`, `isPublished`, and `authorId`. `id` is a unique article identifier, `isPublished` indicates whether an article was already published or not, and `authorId`, which is an ID of a user who wrote the article.

Now let's review and refine our requirements for this example:

- Admins can manage (create/read/update/delete) all entities
- Users have read-only access to everything
- Users can update their articles (`article.authorId === userId`)
- Articles that are published already cannot be removed (`article.isPublished === true`)

With this in mind, we can start off by creating an `Action` enum representing all possible actions that the users can perform with entities:

`Article` 클래스에는 각각 `id`, `isPublished` 및 `authorId`의 세가지 속성이 있습니다. `id`는 고유한 기사(Article) 식별자이고 `isPublished`는 기사(article)가 이미 게시되었는지 여부를 나타내고 `authorId`는 기사를 작성한 사용자의 ID입니다.

이제이 예제에 대한 요구 사항을 검토하고 구체화 해 보겠습니다.

- 관리자는 모든 엔티티를 관리 (생성/읽기/업데이트/삭제)할 수 있습니다.
- 사용자는 모든 것에 대한 읽기 전용 액세스 권한이 있습니다.
- 기사 업데이트 가능 (`article.authorId === userId`)
- 이미 게시된 기사는 삭제할 수 없습니다 (`article.isPublished === true`).

이를 염두에 두고 사용자가 엔티티로 수행할 수 있는 모든 가능한 작업을 나타내는 `Action` 열거(enum)형을 생성하여 시작할 수 있습니다.

```typescript
export enum Action {
  Manage = 'manage',
  Create = 'create',
  Read = 'read',
  Update = 'update',
  Delete = 'delete',
}
```

> **NOTICE**
>
> `manage` is a special keyword in CASL which represents "any" action.
>
> `manage`는 CASL에서 "모든" 작업을 나타내는 특수 키워드입니다.

To encapsulate CASL library, let's generate the `CaslModule` and `CaslAbilityFactory` now.

CASL 라이브러리를 캡슐화하기 위해 이제 `CaslModule`과 `CaslAbilityFactory`를 생성해 보겠습니다.

```bash
$ nest g module casl
$ nest g class casl/casl-ability.factory
```

With this in place, we can define the `createForUser()` method on the `CaslAbilityFactory`. This method will create the `Ability` object for a given user:

이것으로 우리는 `CaslAbilityFactory`에 `createForUser()` 메소드를 정의할 수 있습니다. 이 메소드는 주어진 사용자에 대한 `Ability` 객체를 생성합니다 :

```typescript
type Subjects = InferSubjects<typeof Article | typeof User> | 'all';

export type AppAbility = Ability<[Action, Subjects]>;

@Injectable()
export class CaslAbilityFactory {
  createForUser(user: User) {
    const { can, cannot, build } = new AbilityBuilder<
      Ability<[Action, Subjects]>
    >(Ability as AbilityClass<AppAbility>);

    if (user.isAdmin) {
      can(Action.Manage, 'all'); // read-write access to everything
    } else {
      can(Action.Read, 'all'); // read-only access to everything
    }

    can(Action.Update, Article, { authorId: user.id });
    cannot(Action.Delete, Article, { isPublished: true });

    return build({
      // Read https://casl.js.org/v5/en/guide/subject-type-detection#use-classes-as-subject-types for details
      detectSubjectType: item => item.constructor as ExtractSubjectType<Subjects>
    });
  }
}
```

> **NOTICE**
>
> `all` is a special keyword in CASL that represents "any subject".
>
> `all`은 CASL에서 "모든 주제"를 나타내는 특수 키워드입니다.

> **HINT**
>
> `Ability`, `AbilityBuilder`, `AbilityClass`, and `ExtractSubjectType` classes are exported from the `@casl/ability` package.
>
> `Ability`, `AbilityBuilder`, `AbilityClass` 및 `ExtractSubjectType` 클래스는 `@casl/ability` 패키지에서 내보내집니다.

> **HINT**
>
> `detectSubjectType` option let CASL understand how to get subject type out of an object. For more information read [CASL documentation](https://casl.js.org/v5/en/guide/subject-type-detection#use-classes-as-subject-types) for details.
>
> `detectSubjectType` 옵션을 사용하면 CASL이 객체에서 주제 유형을 가져 오는 방법을 이해할 수 있습니다. 자세한 내용은 [CASL 문서](https://casl.js.org/v5/en/guide/subject-type-detection#use-classes-as-subject-types)를 참조하세요.

In the example above, we created the `Ability` instance using the `AbilityBuilder` class. As you probably guessed, `can` and `cannot` accept the same arguments but has different meanings, `can` allows to do an action on the specified subject and `cannot` forbids. Both may accept up to 4 arguments. To learn more about these functions, visit the official [CASL documentation](https://casl.js.org/v5/en/guide/intro).

Lastly, make sure to add the `CaslAbilityFactory` to the `providers` and `exports` arrays in the `CaslModule` module definition:

위의 예에서는 `AbilityBuilder` 클래스를 사용하여 `Ability` 인스턴스를 생성했습니다. 짐작했듯이 `can`과 `cannot`은 동일한 인수를 받아들이지만 다른 의미를 가지고 있습니다. `can`은 지정된 주제에 대한 작업을 수행하고 `cannot`은 금지합니다. 둘 다 최대 4 개의 인수를 허용할 수 있습니다. 이러한 기능에 대해 자세히 알아 보려면 공식 [CASL 문서](https://casl.js.org/v5/en/guide/intro)를 방문하세요.

마지막으로, `CaslModule` 모듈 정의에서 `providers` 및 `exports` 배열에 `CaslAbilityFactory`를 추가해야 합니다.

```typescript
import { Module } from '@nestjs/common';
import { CaslAbilityFactory } from './casl-ability.factory';

@Module({
  providers: [CaslAbilityFactory],
  exports: [CaslAbilityFactory],
})
export class CaslModule {}
```

With this in place, we can inject the `CaslAbilityFactory` to any class using standard constructor injection as long as the `CaslModule` is imported in the host context:

이 기능을 사용하면 호스트 컨텍스트에서 `CaslModule`을 가져오는 한 표준 생성자 주입을 사용하여 모든 클래스에 `CaslAbilityFactory`를 주입할 수 있습니다.

```typescript
constructor(private caslAbilityFactory: CaslAbilityFactory) {}
```

Then use it in a class as follows.

그런 다음 다음과 같이 클래스에서 사용하십시오.

```typescript
const ability = this.caslAbilityFactory.createForUser(user);
if (ability.can(Action.Read, 'all')) {
  // "user" has read access to everything
}
```

> **HINT**
>
> Learn more about the `Ability` class in the official [CASL documentation](https://casl.js.org/v5/en/guide/intro).
>
> 공식 [CASL 문서](https://casl.js.org/v5/en/guide/intro)에서 `Ability` 클래스에 대해 자세히 알아보세요.

For example, let's say we have a user who is not an admin. In this case, the user should be able to read articles, but creating new ones or removing the existing articles should be prohibited.

예를 들어 관리자가 아닌 사용자가 있다고 가정해 보겠습니다. 이 경우 사용자는 기사를 읽을 수 있어야 하지만 새로운 기사를 작성하거나 기존 기사를 제거하는 것은 금지되어야합니다.

```typescript
const user = new User();
user.isAdmin = false;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Read, Article); // true
ability.can(Action.Delete, Article); // false
ability.can(Action.Create, Article); // false
```

> **HINT**
>
> Although both `Ability` and `AbilityBuilder` classes provide `can` and `cannot` methods, they have different purposes and accept slightly different arguments.
>
> `Ability`와 `AbilityBuilder` 클래스는 모두 `can` 및 `cannot` 메소드를 제공하지만 용도가 다르고 인수도 약간 다릅니다.

Also, as we have specified in our requirements, the user should be able to update its articles:

또한 요구 사항에서 지정한대로 사용자는 기사를 업데이트할 수 있어야 합니다.

```typescript
const user = new User();
user.id = 1;

const article = new Article();
article.authorId = user.id;

const ability = this.caslAbilityFactory.createForUser(user);
ability.can(Action.Update, article); // true

article.authorId = 2;
ability.can(Action.Update, article); // false
```

As you can see, `Ability` instance allows us to check permissions in pretty readable way. Likewise, `AbilityBuilder` allows us to define permissions (and specify various conditions) in a similar fashion. To find more examples, visit the official documentation.

보시다시피 `Ability` 인스턴스를 사용하면 읽기 쉬운 방식으로 권한을 확인할 수 있습니다. 마찬가지로 `AbilityBuilder`를 사용하면 비슷한 방식으로 권한을 정의하고 다양한 조건을 지정할 수 있습니다. 더 많은 예제를 찾으려면 공식 문서를 방문하십시오.



## Advanced: Implementing a `PoliciesGuard`

In this section, we'll demonstrate how to build a somewhat more sophisticated guard, which checks if a user meets specific **authorization policies** that can be configured on the method-level (you can extend it to respect policies configured on the class-level too). In this example, we are going to use the CASL package just for illustration purposes, but using this library is not required. Also, we will use the `CaslAbilityFactory` provider that we've created in the previous section.

First, let's flesh out the requirements. The goal is to provide a mechanism that allows specifying policy checks per route handler. We will support both objects and functions (for simpler checks and for those who prefer more functional-style code).

Let's start off by defining interfaces for policy handlers:

이 섹션에서는 사용자가 메서드 수준에서 구성할 수 있는 특정 **권한 부여 정책**을 충족하는지 확인하는 좀 더 정교한 가드를 구축하는 방법을 보여줄 것입니다 (이를 확장하여 클래스 수준도). 이 예제에서는 설명 목적으로 CASL 패키지를 사용하지만 이 라이브러리를 사용할 필요는 없습니다. 또한 이전 섹션에서 만든 `CaslAbilityFactory` 프로바이더를 사용합니다.

먼저 요구 사항을 구체화합시다. 목표는 경로 핸들러별로 정책 검사를 지정할 수 있는 메커니즘을 제공하는 것입니다. 우리는 객체와 함수를 모두 지원할 것입니다 (더 간단한 검사와 더 기능적인 스타일의 코드를 선호하는 사람들을 위해).

정책 핸들러를 위한 인터페이스를 정의하는 것으로 시작하겠습니다.

```typescript
import { AppAbility } from '../casl/casl-ability.factory';

interface IPolicyHandler {
  handle(ability: AppAbility): boolean;
}

type PolicyHandlerCallback = (ability: AppAbility) => boolean;

export type PolicyHandler = IPolicyHandler | PolicyHandlerCallback;
```

As mentioned above, we provided two possible ways of defining a policy handler, an object (instance of a class that implements the `IPolicyHandler` interface) and a function (which meets the `PolicyHandlerCallback` type).

With this in place, we can create a `@CheckPolicies()` decorator. This decorator allows specifying what policies have to be met to access specific resources.

위에서 언급했듯이, 우리는 정책 핸들러를 정의하는 두가지 가능한 방법, 객체(`IPolicyHandler` 인터페이스를 구현하는 클래스의 인스턴스)와 함수(`PolicyHandlerCallback` 타입을 충족)를 제공했습니다.

이것으로 `@CheckPolicies()` 데코레이터를 만들 수 있습니다. 이 데코레이터를 사용하면 특정 리소스에 액세스하기 위해 충족해야하는 정책을 지정할 수 있습니다.

```typescript
export const CHECK_POLICIES_KEY = 'check_policy';
export const CheckPolicies = (...handlers: PolicyHandler[]) =>
  SetMetadata(CHECK_POLICIES_KEY, handlers);
```

Now let's create a `PoliciesGuard` that will extract and execute all the policy handlers bound to a route handler.

이제 경로 핸들러에 바인딩된 모든 정책 핸들러를 추출하고 실행할 `PoliciesGuard`를 만들어 보겠습니다.

```typescript
@Injectable()
export class PoliciesGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private caslAbilityFactory: CaslAbilityFactory,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const policyHandlers =
      this.reflector.get<PolicyHandler[]>(
        CHECK_POLICIES_KEY,
        context.getHandler(),
      ) || [];

    const { user } = context.switchToHttp().getRequest();
    const ability = this.caslAbilityFactory.createForUser(user);

    return policyHandlers.every((handler) =>
      this.execPolicyHandler(handler, ability),
    );
  }

  private execPolicyHandler(handler: PolicyHandler, ability: AppAbility) {
    if (typeof handler === 'function') {
      return handler(ability);
    }
    return handler.handle(ability);
  }
}
```

> **HINT**
>
> In this example, we assumed that `request.user` contains the user instance. In your app, you will probably make that association in your custom **authentication guard** - see [authentication](https://docs.nestjs.com/security/authentication) chapter for more details.
>
> 이 예에서는 `request.user`에 사용자 인스턴스가 포함되어 있다고 가정했습니다. 앱에서 사용자 정의 **인증 가드**에서 해당 연결을 만들 수 있습니다. 자세한 내용은 [인증](https://docs.nestjs.kr/security/authentication) 장을 참조하십시오.

Let's break this example down. The `policyHandlers` is an array of handlers assigned to the method through the `@CheckPolicies()` decorator. Next, we use the `CaslAbilityFactory#create` method which constructs the `Ability` object, allowing us to verify whether a user has sufficient permissions to perform specific actions. We are passing this object to the policy handler which is either a function or an instance of a class that implements the `IPolicyHandler`, exposing the `handle()` method that returns a boolean. Lastly, we use the `Array#every` method to make sure that every handler returned `true` value.

Finally, to test this guard, bind it to any route handler, and register an inline policy handler (functional approach), as follows:

이 예제를 분해해 보겠습니다. `policyHandlers`는 `@CheckPolicies()` 데코레이터를 통해 메소드에 할당된 핸들러의 배열입니다. 다음으로 `Ability` 객체를 구성하는 `CaslAbilityFactory#create` 메소드를 사용하여 사용자가 특정 작업을 수행할 수 있는 충분한 권한이 있는지 확인할 수 있습니다. 이 객체를 `IPolicyHandler`를 구현하는 클래스의 함수 또는 인스턴스인 정책 핸들러에 전달하여 부울을 반환하는 `handle()` 메서드를 노출합니다. 마지막으로 `Array#every` 메소드를 사용하여 모든 핸들러가 `true` 값을 반환하는지 확인합니다.

마지막으로 이 가드를 테스트하려면 다음과 같이 경로 핸들러에 바인딩하고 인라인 정책 핸들러 (기능적 접근 방식)를 등록합니다.

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies((ability: AppAbility) => ability.can(Action.Read, Article))
findAll() {
  return this.articlesService.findAll();
}
```

Alternatively, we can define a class which implements the `IPolicyHandler` interface:

또는 `IPolicyHandler` 인터페이스를 구현하는 클래스를 정의할 수 있습니다.

```typescript
export class ReadArticlePolicyHandler implements IPolicyHandler {
  handle(ability: AppAbility) {
    return ability.can(Action.Read, Article);
  }
}
```

And use it as follows:

다음과 같이 사용하십시오.

```typescript
@Get()
@UseGuards(PoliciesGuard)
@CheckPolicies(new ReadArticlePolicyHandler())
findAll() {
  return this.articlesService.findAll();
}
```

> **NOTICE**
>
> Since we must instantiate the policy handler in-place using the `new` keyword, `ReadArticlePolicyHandler` class cannot use the Dependency Injection. This can be addressed with the `ModuleRef#get` method (read more [here](https://docs.nestjs.com/fundamentals/module-ref)). Basically, instead of registering functions and instances through the `@CheckPolicies()` decorator, you must allow passing a `Type<IPolicyHandler>`. Then, inside your guard, you could retrieve an instance using a type reference: `moduleRef.get(YOUR_HANDLER_TYPE)` or even dynamically instantiate it using the `ModuleRef#create` method.
>
> `new` 키워드를 사용하여 정책 핸들러를 제자리에서 인스턴스화해야 하므로 `ReadArticlePolicyHandler` 클래스는 Dependency Injection을 사용할 수 없습니다. 이 문제는 `ModuleRef#get` 메소드로 해결할 수 있습니다 ([여기](https://docs.nestjs.kr/fundamentals/module-ref)에 대해 자세히 알아보기). 기본적으로 `@CheckPolicies()` 데코레이터를 통해 함수와 인스턴스를 등록하는 대신 `Type<IPolicyHandler>` 전달을 허용해야 합니다. 그런 다음 가드내에서 타입 참조 `moduleRef.get(YOUR_HANDLER_TYPE)`을 사용하여 인스턴스를 검색하거나 `ModuleRef#create` 메소드를 사용하여 동적으로 인스턴스화 할 수도 있습니다.



#### 출처

> https://docs.nestjs.com/security/authorization
>
> https://docs.nestjs.kr/security/authorization