---
layout: post
title:  NestJS - Subscription
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-18 18:36 +0900
---

## Subscriptions

In addition to fetching data using queries and modifying data using mutations, the GraphQL spec supports a third operation type, called `subscription`. GraphQL subscriptions are a way to push data from the server to the clients that choose to listen to real time messages from the server. Subscriptions are similar to queries in that they specify a set of fields to be delivered to the client, but instead of immediately returning a single answer, a channel is opened and a result is sent to the client every time a particular event happens on the server.

A common use case for subscriptions is notifying the client side about particular events, for example the creation of a new object, updated fields and so on (read more [here](https://www.apollographql.com/docs/react/data/subscriptions)).

쿼리를 사용하여 데이터를 가져오고 변형을 사용하여 데이터를 수정하는 것 외에도 GraphQL 사양은 `구독 subscription`이라는 세번째 작업 유형을 지원합니다. GraphQL 구독은 서버에서 실시간 메시지를 수신하도록 선택한 클라이언트로 서버에서 데이터를 푸시하는 방법입니다. 구독은 클라이언트에 전달할 필드 집합을 지정한다는 점에서 쿼리와 유사하지만 단일 응답을 즉시 반환하는 대신 채널이 열리고 서버에서 특정 이벤트가 발생할 때마다 결과가 클라이언트에 전송됩니다. .

구독의 일반적인 사용 사례는 새 객체 생성, 업데이트된 필드 등과 같은 특정 이벤트에 대해 클라이언트 측에 알리는 것입니다 (자세한 내용은 [여기](https://www.apollographql.com/docs/react/data/subscriptions/) 참조).



## Enable subscriptions

To enable subscriptions, set the `installSubscriptionHandlers` property to `true`.

구독을 활성화하려면 `installSubscriptionHandlers` 속성을 `true`로 설정합니다.

```typescript
GraphQLModule.forRoot({
  installSubscriptionHandlers: true,
}),
```

> **WARNING**
>
> The `installSubscriptionHandlers` configuration option has been removed from the latest version of Apollo server and will be soon deprecated in this package as well. By default, `installSubscriptionHandlers` will fallback to use the `subscriptions-transport-ws` ([read more](https://github.com/apollographql/subscriptions-transport-ws)) but we strongly recommend using the `graphql-ws`([read more](https://github.com/enisdenjo/graphql-ws)) library instead.
>
> `PubSub`는 간단한 `publish` 및 `subscribe API`를 노출하는 클래스입니다. [여기](https://www.apollographql.com/docs/graphql-subscriptions/setup.html)에서 자세히 알아보세요. Apollo 문서는 기본 구현이 프로덕션에 적합하지 않다고 경고합니다 ([여기](https://github.com/apollographql/graphql-subscriptions#getting-started-with-your-first-subscription)). 프로덕션 앱은 외부 저장소에서 지원하는 `PubSub`구현을 사용해야 합니다 ([여기](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations)에서 자세히 알아보기).

To switch to use the `graphql-ws` package instead, use the following configuration:

```typescript
GraphQLModule.forRoot({
  subscriptions: {
    'graphql-ws': true
  },
}),
```

> **HINT**
>
> You can also use both packages (`subscriptions-transport-ws` and `graphql-ws`) at the same time, for example, for backward compatibility.



## Code first

To create a subscription using the code first approach, we use the `@Subscription()` decorator and the `PubSub` class from the `graphql-subscriptions` package, which provides a simple **publish/subscribe API**.

The following subscription handler takes care of **subscribing** to an event by calling `PubSub#asyncIterator`. This method takes a single argument, the `triggerName`, which corresponds to an event topic name.

코드 우선 접근 방식을 사용하여 구독을 생성하려면 간단한 **publish/subscribe API**를 제공하는 `graphql-subscriptions` 패키지의 `@Subscription()` 데코레이터와 `PubSub` 클래스를 사용합니다.

다음 구독 핸들러는 `PubSub#asyncIterator`를 호출하여 이벤트에 대한 **구독**을 처리합니다. 이 메소드는 이벤트 주제 이름에 해당하는 `triggerName`이라는 단일 인수를 사용합니다.

```typescript
const pubSub = new PubSub();

@Resolver((of) => Author)
export class AuthorResolver {
  // ...
  @Subscription((returns) => Comment)
  commentAdded() {
    return pubSub.asyncIterator('commentAdded');
  }
}
```

> **HINT**
>
> All decorators are exported from the `@nestjs/graphql` package, while the `PubSub` class is exported from the `graphql-subscriptions` package.

> **NOTE**
>
> `PubSub` is a class that exposes a simple `publish` and `subscribe API`. Read more about it [here](https://www.apollographql.com/docs/graphql-subscriptions/setup.html). Note that the Apollo docs warn that the default implementation is not suitable for production (read more [here](https://github.com/apollographql/graphql-subscriptions#getting-started-with-your-first-subscription)). Production apps should use a `PubSub` implementation backed by an external store (read more [here](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations)).
>
> `PubSub`는 간단한 `publish` 및 `subscribe API`를 노출하는 클래스입니다. [여기](https://www.apollographql.com/docs/graphql-subscriptions/setup.html)에서 자세히 알아보세요. Apollo 문서는 기본 구현이 프로덕션에 적합하지 않다고 경고합니다 ([여기](https://github.com/apollographql/graphql-subscriptions#getting-started-with-your-first-subscription)). 프로덕션 앱은 외부 저장소에서 지원하는 `PubSub`구현을 사용해야 합니다 ([여기](https://github.com/apollographql/graphql-subscriptions#pubsub-implementations)에서 자세히 알아보기).

This will result in generating the following part of the GraphQL schema in SDL:

그러면 SDL에서 GraphQL 스키마의 다음 부분이 생성됩니다:

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

Note that subscriptions, by definition, return an object with a single top level property whose key is the name of the subscription. This name is either inherited from the name of the subscription handler method (i.e., `commentAdded` above), or is provided explicitly by passing an option with the key `name` as the second argument to the `@Subscription()` decorator, as shown below.

정의에 따라 구독은 키가 구독의 이름인 단일 최상위 속성이 있는 객체를 반환합니다. 이 이름은 구독 핸들러 메서드의 이름 (즉, 위의 `commentAdded`)에서 상속되거나 `@Subscription()` 데코레이터에 두번째 인수로 `name` 키가 있는 옵션을 전달하여 명시적으로 제공됩니다. 아래와 같이.

```typescript
  @Subscription(returns => Comment, {
    name: 'commentAdded',
  })
  addCommentHandler() {
    return pubSub.asyncIterator('commentAdded');
  }
```

This construct produces the same SDL as the previous code sample, but allows us to decouple the method name from the subscription.

이 구조는 이전 코드 샘플과 동일한 SDL을 생성하지만 구독에서 메서드 이름을 분리할 수 있습니다.



## Publishing

Now, to publish the event, we use the `PubSub#publish` method. This is often used within a mutation to trigger a client-side update when a part of the object graph has changed. For example:

이제 이벤트를 게시하기 위해 `PubSub#publish` 메소드를 사용합니다. 이것은 객체 그래프의 일부가 변경되었을 때 클라이언트측 업데이트를 트리거하기 위해 뮤테이션내에서 자주 사용됩니다. 예를 들면:

### posts/posts.resolver.ts

```typescript
@Mutation(returns => Post)
async addComment(
  @Args('postId', { type: () => Int }) postId: number,
  @Args('comment', { type: () => Comment }) comment: CommentInput,
) {
  const newComment = this.commentsService.addComment({ id: postId, comment });
  pubSub.publish('commentAdded', { commentAdded: newComment });
  return newComment;
}
```

The `PubSub#publish` method takes a `triggerName` (again, think of this as an event topic name) as the first parameter, and an event payload as the second parameter. As mentioned, the subscription, by definition, returns a value and that value has a shape. Look again at the generated SDL for our `commentAdded` subscription:

`PubSub#publish` 메소드는 첫번째 매개 변수로 `triggerName` (다시 말하지만 이벤트 주제 이름으로 생각)을, 두번째 매개 변수로 이벤트 페이로드를 사용합니다. 언급했듯이 구독은 정의에 따라 값을 반환하고 해당 값은 모양(shape)을 갖습니다. `commentAdded` 구독에 대해 생성된 SDL을 다시 살펴보십시오.

```graphql
type Subscription {
  commentAdded(): Comment!
}
```

This tells us that the subscription must return an object with a top-level property name of `commentAdded` that has a value which is a `Comment` object. The important point to note is that the shape of the event payload emitted by the `PubSub#publish` method must correspond to the shape of the value expected to return from the subscription. So, in our example above, the `pubSub.publish('commentAdded', { commentAdded: newComment })` statement publishes a `commentAdded` event with the appropriately shaped payload. If these shapes don't match, your subscription will fail during the GraphQL validation phase.

이것은 구독이 `Comment` 객체인 값을 가진 `commentAdded`라는 최상위 속성 이름을 가진 객체를 반환해야 함을 알려줍니다. 주목해야 할 중요한 점은 `PubSub#publish` 메서드에서 내보낸 이벤트 페이로드의 모양(shape)이 구독에서 반환될 것으로 예상되는 값의 모양과 일치해야 한다는 것입니다. 따라서 위의 예에서 `pubSub.publish('commentAdded', { commentAdded: newComment })`문은 적절한 모양의 페이로드와 함께 `commentAdded` 이벤트를 게시합니다. 이러한 모양(shape)가 일치하지 않으면 GraphQL 유효성 검사 단계에서 구독이 실패합니다.



## Filtering subscriptions

To filter out specific events, set the `filter` property to a filter function. This function acts similar to the function passed to an array `filter`. It takes two arguments: `payload` containing the event payload (as sent by the event publisher), and `variables` taking any arguments passed in during the subscription request. It returns a boolean determining whether this event should be published to client listeners.

특정 이벤트를 필터링하려면 `filter` 속성을 필터 함수로 설정하세요. 이 함수는 배열 `filter`에 전달된 함수와 유사하게 작동합니다. 이벤트 페이로드 (이벤트 게시자가 보낸대로)를 포함하는 `payload`와 구독 요청 중에 전달된 모든 인수를 받는 `variables`의 두 가지 인수가 필요합니다. 이 이벤트가 클라이언트 리스너에 게시되어야 하는지 여부를 결정하는 부울(boolean)을 반환합니다.

```typescript
@Subscription(returns => Comment, {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded(@Args('title') title: string) {
  return pubSub.asyncIterator('commentAdded');
}
```



## Mutating subscription payloads

To mutate the published event payload, set the `resolve` property to a function. The function receives the event payload (as sent by the event publisher) and returns the appropriate value.

게시된 이벤트 페이로드를 변경하려면 `resolve` 속성을 함수로 설정합니다. 이 함수는 이벤트 게시자가 보낸 이벤트 페이로드를 수신하고 적절한 값을 반환합니다.

```typescript
@Subscription(returns => Comment, {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

> **NOTE**
>
> If you use the `resolve` option, you should return the unwrapped payload (e.g., with our example, return a `newComment` object directly, not a `{ commentAdded: newComment }` object).

If you need to access injected providers (e.g., use an external service to validate the data), use the following construction.

삽입된 공급자에 액세스해야 하는 경우 (예: 외부 서비스를 사용하여 데이터 유효성 검사) 다음 구성을 사용합니다.

```typescript
@Subscription(returns => Comment, {
  resolve(this: AuthorResolver, value) {
    // "this" refers to an instance of "AuthorResolver"
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

The same construction works with filters:

동일한 구성이 필터에서도 작동합니다.

```typescript
@Subscription(returns => Comment, {
  filter(this: AuthorResolver, payload, variables) {
    // "this" refers to an instance of "AuthorResolver"
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```



## Schema first

To create an equivalent subscription in Nest, we'll make use of the `@Subscription()` decorator.

Nest에서 동등한 구독을 생성하기 위해 `@Subscription()` 데코레이터를 사용할 것입니다.

```typescript
const pubSub = new PubSub();

@Resolver('Author')
export class AuthorResolver {
  // ...
  @Subscription()
  commentAdded() {
    return pubSub.asyncIterator('commentAdded');
  }
}
```

To filter out specific events based on context and arguments, set the `filter` property.

컨텍스트 및 인수를 기반으로 특정 이벤트를 필터링하려면 `filter` 속성을 설정하세요.

```typescript
@Subscription('commentAdded', {
  filter: (payload, variables) =>
    payload.commentAdded.title === variables.title,
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

To mutate the published payload, we can use a `resolve` function.

게시된 페이로드를 변경(mutate)하려면 `resolve` 함수를 사용할 수 있습니다.

```typescript
@Subscription('commentAdded', {
  resolve: value => value,
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

If you need to access injected providers (e.g., use an external service to validate the data), use the following construction:

삽입된 프로바이더에 액세스해야 하는 경우 (예: 외부 서비스를 사용하여 데이터 유효성 검사) 다음 구성을 사용합니다.

```typescript
@Subscription('commentAdded', {
  resolve(this: AuthorResolver, value) {
    // "this" refers to an instance of "AuthorResolver"
    return value;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

The same construction works with filters:

동일한 구성이 필터에서도 작동합니다.

```typescript
@Subscription('commentAdded', {
  filter(this: AuthorResolver, payload, variables) {
    // "this" refers to an instance of "AuthorResolver"
    return payload.commentAdded.title === variables.title;
  }
})
commentAdded() {
  return pubSub.asyncIterator('commentAdded');
}
```

The last step is to update the type definitions file.

마지막 단계는 유형 정의 파일을 업데이트하는 것입니다.

```graphql
type Author {
  id: Int!
  firstName: String
  lastName: String
  posts: [Post]
}

type Post {
  id: Int!
  title: String
  votes: Int
}

type Query {
  author(id: Int!): Author
}

type Comment {
  id: String
  content: String
}

type Subscription {
  commentAdded(title: String!): Comment
}
```

With this, we've created a single `commentAdded(title: String!): Comment` subscription. You can find a full sample implementation [here](https://github.com/nestjs/nest/blob/master/sample/12-graphql-schema-first).

이를 통해 단일 `commentAdded(title: String!): Comment` 구독을 만들었습니다. 전체 샘플 구현은 [여기](https://github.com/nestjs/nest/blob/master/sample/12-graphql-schema-first)에서 확인할 수 있습니다.



## PubSub

We instantiated a local `PubSub` instance above. The preferred approach is to define `PubSub` as a [provider](https://docs.nestjs.com/fundamentals/custom-providers) and inject it through the constructor (using the `@Inject()` decorator). This allows us to re-use the instance across the whole application. For example, define a provider as follows, then inject `'PUB_SUB'` where needed.

위에서 로컬 `PubSub` 인스턴스를 인스턴스화했습니다. 선호되는 접근 방식은 `PubSub`를 [provider](https://docs.nestjs.kr/fundamentals/custom-providers)로 정의하고 생성자(constructor)를 통해 삽입하는 것입니다 (`@Inject()` 데코레이터 사용). 이를 통해 전체 애플리케이션에서 인스턴스를 재사용할 수 있습니다. 예를 들어 다음과 같이 프로바이더를 정의한 다음 필요한 곳에 `'PUB_SUB'`를 삽입합니다.

```typescript
{
  provide: 'PUB_SUB',
  useValue: new PubSub(),
}
```



## Customize subscriptions server

To customize the subscriptions server (e.g., change the path), use the `subscriptions` options property.

구독 서버를 사용자 지정하려면 (예: 리스너 포트 변경) `subscriptions` 옵션 속성을 사용합니다 ([자세한 것을 읽어보세요](https://www.apollographql.com/docs/apollo-server/v2/api/apollo-server.html#constructor-options-lt-ApolloServer-gt)).

```typescript
GraphQLModule.forRoot({
  installSubscriptionHandlers: true,
  subscriptions: {
    'subscriptions-transport-ws': {
      path: '/graphql'
    },
  }
}),
```

If you're using the `graphql-ws` package for subscriptions, replace the `subscriptions-transport-ws` key with `graphql-ws`, as follows:

```typescript
GraphQLModule.forRoot({
  installSubscriptionHandlers: true,
  subscriptions: {
    'graphql-ws': {
      path: '/graphql'
    },
  }
}),
```



## Authentication over WebSocket

Checking that the user is authenticated should be done inside the `onConnect` callback function that you can specify in the `subscriptions` options.

The `onConnect` will receive as a first argument the `connectionParams` passed to the `SubscriptionClient` (read [more](https://www.apollographql.com/docs/react/data/subscriptions/#5-authenticate-over-websocket-optional)).

```typescript
GraphQLModule.forRoot({
  subscriptions: {
    'subscriptions-transport-ws': {
      onConnect: (connectionParams) => {
        const authToken = connectionParams.authToken;
        if (!isValid(authToken)) {
          throw new Error('Token is not valid');
        }
        // extract user information from token
        const user = parseToken(authToken);
        // return user info to add them to the context later
        return { user };
      },
    }
  },
  context: ({ connection }) => {
    // connection.context will be equal to what was returned by the "onConnect" callback
  },
}),
```

The `authToken` in this example is only sent once by the client, when the connection is first established. All subscriptions made with this connection will have the same `authToken`, and thus the same user info.

> **NOTE**
>
> There is a bug in `subscriptions-transport-ws` that allows connections to skip the `onConnect` phase (read [more](https://github.com/apollographql/subscriptions-transport-ws/issues/349)). You should not assume that `onConnect` was called when the user starts a subscription, and always check that the `context` is populated.

If you're using the `graphql-ws` package, the signature of the `onConnect` callback will be slightly different:

```typescript
subscriptions: {
  'graphql-ws': {
    onConnect: (context: Context<any>) => {
      const { connectionParams } = context;
      // the rest will remain the same as in the example above
    },
  },
},
```



#### 출처

> https://docs.nestjs.com/graphql/subscriptions
>
> https://docs.nestjs.kr/graphql/subscriptions