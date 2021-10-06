---
layout: post
title:  NestJS - Database
author: Jihyun
category: nestjs
tags:
- nestjs
date: 2021-10-06 18:04 +0900
---

## Configuration

Applications often run in different **environments**. Depending on the environment, different configuration settings should be used. For example, usually the local environment relies on specific database credentials, valid only for the local DB instance. The production environment would use a separate set of DB credentials. Since configuration variables change, best practice is to [store configuration variables](https://12factor.net/config) in the environment.

Externally defined environment variables are visible inside Node.js through the `process.env` global. We could try to solve the problem of multiple environments by setting the environment variables separately in each environment. This can quickly get unwieldy, especially in the development and testing environments where these values need to be easily mocked and/or changed.

In Node.js applications, it's common to use `.env` files, holding key-value pairs where each key represents a particular value, to represent each environment. Running an app in different environments is then just a matter of swapping in the correct `.env` file.

A good approach for using this technique in Nest is to create a `ConfigModule` that exposes a `ConfigService` which loads the appropriate `.env` file. While you may choose to write such a module yourself, for convenience Nest provides the `@nestjs/config` package out-of-the box. We'll cover this package in the current chapter.

애플리케이션은 종종 서로 다른 **환경**에서 실행됩니다. 환경에 따라 다른 구성 설정을 사용해야 합니다. 예를 들어, 일반적으로 로컬 환경은 로컬 DB 인스턴스에만 유효한 특정 데이터베이스 자격증명에 의존합니다. 프로덕션 환경은 별도의 DB 자격증명 세트를 사용합니다. 구성 변수가 변경되므로 환경에서 [구성 변수 저장](https://12factor.net/config)을 사용하는 것이 가장 좋습니다.

외부에서 정의된 환경 변수는 `process.env` 전역을 통해 Node.js 내부에서 볼 수 있습니다. 각 환경에서 개별적으로 환경 변수를 설정하여 여러 환경의 문제를 해결할 수 있습니다. 이는 특히 이러한 값을 쉽게 모의하거나 변경해야 하는 개발 및 테스트 환경에서 빠르게 다루기 어려울 수 있습니다.

Node.js 애플리케이션에서는 각 환경을 나타내기 위해 각 키가 특정 값을 나타내는 키-값 쌍을 보유한 `.env` 파일을 사용하는 것이 일반적입니다. 다른 환경에서 앱을 실행하는 것은 올바른 `.env` 파일에서 스와핑하면 됩니다.

Nest에서이 기술을 사용하는 좋은 방법은 적절한 `.env` 파일을 로드하는 `ConfigService`를 노출하는 `ConfigModule`을 만드는 것입니다. 이러한 모듈을 직접 작성하도록 선택할 수 있지만 편의를 위해 Nest는 즉시 사용 가능한 `@nestjs/config` 패키지를 제공합니다. 이 패키지는 현재 장에서 다룰 것입니다.



## Installation

To begin using it, we first install the required dependency.

사용을 시작하려면 먼저 필요한 종속성을 설치합니다.

```bash
$ npm i --save @nestjs/config
```

> **HINT**
>
> The `@nestjs/config` package internally uses [dotenv](https://github.com/motdotla/dotenv).
>
> `@nestjs/config` 패키지는 내부적으로 [dotenv](https://github.com/motdotla/dotenv)를 사용합니다.



## Getting started

Once the installation process is complete, we can import the `ConfigModule`. Typically, we'll import it into the root `AppModule` and control its behavior using the `.forRoot()` static method. During this step, environment variable key/value pairs are parsed and resolved. Later, we'll see several options for accessing the `ConfigService` class of the `ConfigModule` in our other feature modules.

설치 프로세스가 완료되면 `ConfigModule`을 가져올 수 있습니다. 일반적으로 루트 `AppModule`로 가져오고 `.forRoot()` 정적 메서드를 사용하여 동작을 제어합니다. 이 단계에서 환경 변수 키/값 쌍이 구문 분석되고 해결됩니다. 나중에 다른 기능 모듈에서 `ConfigModule`의 `ConfigService` 클래스에 액세스하기 위한 몇가지 옵션을 볼 수 있습니다.

### app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [ConfigModule.forRoot()],
})
export class AppModule {}
```

The above code will load and parse a `.env` file from the default location (the project root directory), merge key/value pairs from the `.env` file with environment variables assigned to `process.env`, and store the result in a private structure that you can access through the `ConfigService`. The `forRoot()` method registers the `ConfigService` provider, which provides a `get()` method for reading these parsed/merged configuration variables. Since `@nestjs/config` relies on [dotenv](https://github.com/motdotla/dotenv), it uses that package's rules for resolving conflicts in environment variable names. When a key exists both in the runtime environment as an environment variable (e.g., via OS shell exports like `export DATABASE_USER=test`) and in a `.env` file, the runtime environment variable takes precedence.

A sample `.env` file looks something like this:

위의 코드는 기본 위치(프로젝트 루트 디렉터리)에서 `.env` 파일을 로드 및 구문 분석하고 `.env` 파일의 키/값 쌍을 `process.env`에 할당된 환경 변수와 병합하고 결과적으로 `ConfigService`를 통해 액세스할 수 있는 개인 구조가 됩니다. `forRoot()` 메서드는 이러한 파싱/병합된 구성 변수를 읽기 위한 `get()` 메서드를 제공하는 `ConfigService` 프로바이더를 등록합니다. `@nestjs/config`는 [dotenv](https://github.com/motdotla/dotenv)에 의존하므로 환경 변수 이름의 충돌을 해결하기 위해 해당 패키지의 규칙을 사용합니다. 키가 런타임 환경에 환경 변수(예: `export DATABASE_USER=test`와 같은 OS 셸 내보내기를 통해)와 `.env` 파일로 모두 존재하는 경우 런타임 환경 변수가 우선합니다.

샘플 `.env` 파일은 다음과 같습니다.

```json
DATABASE_USER=test
DATABASE_PASSWORD=test
```



## Custom env file path

By default, the package looks for a `.env` file in the root directory of the application. To specify another path for the `.env` file, set the `envFilePath` property of an (optional) options object you pass to `forRoot()`, as follows:

기본적으로 패키지는 애플리케이션의 루트 디렉토리에서 `.env` 파일을 찾습니다. `.env` 파일의 다른 경로를 지정하려면 다음과 같이 `forRoot()`에 전달하는(선택 사항) 옵션 객체의 `envFilePath` 속성을 설정합니다.

```typescript
ConfigModule.forRoot({
  envFilePath: '.development.env',
});
```

You can also specify multiple paths for `.env` files like this:

다음과 같이 `.env` 파일에 여러 경로를 지정할 수도 있습니다.

```typescript
ConfigModule.forRoot({
  envFilePath: ['.env.development.local', '.env.development'],
});
```

If a variable is found in multiple files, the first one takes precedence.

변수가 여러 파일에서 발견되면 첫번째 파일이 우선합니다.



## Disable env variables loading

If you don't want to load the `.env` file, but instead would like to simply access environment variables from the runtime environment (as with OS shell exports like `export DATABASE_USER=test`), set the options object's `ignoreEnvFile` property to `true`, as follows:

`.env` 파일을 로드하지 않고 대신 런타임 환경에서 환경 변수에 액세스하려는 경우(`export DATABASE_USER=test`와 같은 OS 셸 내보내기와 마찬가지로) 옵션 객체의 `ignoreEnvFile`을 설정합니다. 다음과 같이 속성을 `true`로 설정합니다.

```typescript
ConfigModule.forRoot({
  ignoreEnvFile: true,
});
```



## Use module globally

When you want to use `ConfigModule` in other modules, you'll need to import it (as is standard with any Nest module). Alternatively, declare it as a [global module](https://docs.nestjs.com/modules#global-modules) by setting the options object's `isGlobal` property to `true`, as shown below. In that case, you will not need to import `ConfigModule` in other modules once it's been loaded in the root module (e.g., `AppModule`).

다른 모듈에서 `ConfigModule`을 사용하려면 가져와야합니다 (모든 Nest 모듈의 표준과 동일). 또는 아래와 같이 옵션 객체의 `isGlobal` 속성을 `true`로 설정하여 이를 [global module](https://docs.nestjs.kr/modules#global-modules)로 선언합니다. 이 경우 루트 모듈(예: `AppModule`)에 로드되면 다른 모듈에서 `ConfigModule`을 가져올 필요가 없습니다.

```typescript
ConfigModule.forRoot({
  isGlobal: true,
});
```



## Custom configuration files

For more complex projects, you may utilize custom configuration files to return nested configuration objects. This allows you to group related configuration settings by function (e.g., database-related settings), and to store related settings in individual files to help manage them independently.

A custom configuration file exports a factory function that returns a configuration object. The configuration object can be any arbitrarily nested plain JavaScript object. The `process.env` object will contain the fully resolved environment variable key/value pairs (with `.env` file and externally defined variables resolved and merged as described [above](https://docs.nestjs.com/techniques/configuration#getting-started)). Since you control the returned configuration object, you can add any required logic to cast values to an appropriate type, set default values, etc. For example:

더 복잡한 프로젝트의 경우 사용자 지정 구성 파일을 사용하여 중첩된 구성 객체를 반환할 수 있습니다. 이를 통해 관련 구성 설정을 기능(예: 데이터베이스 관련 설정)별로 그룹화하고 관련 설정을 개별 파일에 저장하여 독립적으로 관리할 수 있습니다.

사용자 지정 구성 파일은 구성 개체를 반환하는 팩토리 함수를 내보냅니다. 구성 객체는 임의로 중첩된 일반 JavaScript 객체일 수 있습니다. `process.env` 객체에는 [위](https://docs.nestjs.kr/techniques/configuration#getting-started)에서 설명한대로 해결 및 병합 된 `.env` 파일 및 외부 정의 변수와 함께 완전히 해결된 환경 변수 키/값 쌍이 포함됩니다. 반환된 구성 객체를 제어하므로 값을 적절한 유형으로 캐스팅하고 기본값을 설정하는 데 필요한 논리를 추가할 수 있습니다. 예를 들면 다음과 같습니다.

### config/configuration.ts

```typescript
export default () => ({
  port: parseInt(process.env.PORT, 10) || 3000,
  database: {
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10) || 5432
  }
});
```

We load this file using the `load` property of the options object we pass to the `ConfigModule.forRoot()` method:

이 파일은 `ConfigModule.forRoot()` 메서드에 전달하는 옵션 객체의 `load` 속성을 사용하여 로드합니다.

```typescript
import configuration from './config/configuration';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [configuration],
    }),
  ],
})
export class AppModule {}
```

> **NOTICE**
>
> The value assigned to the `load` property is an array, allowing you to load multiple configuration files (e.g. `load: [databaseConfig, authConfig]`)
>
> `load` 속성에 할당된 값은 배열이므로 여러 구성 파일을 로드할 수 있습니다 (예: `load: [databaseConfig, authConfig]`).

With custom configuration files, we can also manage custom files such as YAML files. Here is an example of a configuration using YAML format:

사용자 지정 구성 파일을 사용하면 YAML 파일과 같은 사용자 지정 파일도 관리할 수 있습니다. 다음은 YAML 형식을 사용하는 구성의 예입니다.

```yaml
http:
  host: 'localhost'
  port: 8080

db:
  postgres:
    url: 'localhost'
    port: 5432
    database: 'yaml-db'

  sqlite:
    database: 'sqlite.db'
```

To read and parse YAML files, we can leverage the `js-yaml` package.

YAML 파일을 읽고 파싱하기 위해 `js-yaml` 패키지를 활용할 수 있습니다.

```bash
$ npm i js-yaml
$ npm i -D @types/js-yaml
```

Once the package is installed, we use `yaml#load` function to load YAML file we just created above.

패키지가 설치되면 `yaml#load` 함수를 사용하여 방금 만든 YAML 파일을 로드합니다.

### config/configuration.ts

```typescript
import { readFileSync } from 'fs';
import * as yaml from 'js-yaml';
import { join } from 'path';

const YAML_CONFIG_FILENAME = 'config.yaml';

export default () => {
  return yaml.load(
    readFileSync(join(__dirname, YAML_CONFIG_FILENAME), 'utf8'),
  ) as Record<string, any>;
};
```

> **NOTE**
>
> Nest CLI does not automatically move your "assets" (non-TS files) to the `dist` folder during the build process. To make sure that your YAML files are copied, you have to specify this in the `compilerOptions#assets` object in the `nest-cli.json` file. As an example, if the `config` folder is at the same level as the `src` folder, add `compilerOptions#assets` with the value `"assets": [{"include": "../config/*.yaml", "outDir": "./dist/config"}]`. Read more [here](https://docs.nestjs.com/cli/monorepo#assets).
>
> Nest CLI는 빌드 프로세스 중에 "자산"(비 TS 파일)을 `dist`폴더로 자동으로 이동하지 않습니다. YAML 파일이 복사되었는지 확인하려면 `nest-cli.json` 파일의 `compilerOptions#assets` 객체에 이를 지정해야 합니다. 예를 들어,`config` 폴더가 `src` 폴더와 동일한 수준에 있는 경우 `"assets": [{"include": "../config/*.yaml", "outDir": "./dist/config"}]`를 추가합니다. [여기](https://docs.nestjs.kr/cli/monorepo#assets)에서 자세히 알아보세요.



## Using the `ConfigService`

To access configuration values from our `ConfigService`, we first need to inject `ConfigService`. As with any provider, we need to import its containing module - the `ConfigModule` - into the module that will use it (unless you set the `isGlobal` property in the options object passed to the `ConfigModule.forRoot()` method to `true`). Import it into a feature module as shown below.

`ConfigService`에서 구성 값에 액세스하려면 먼저 `ConfigService`를 삽입해야합니다. 다른 프로바이더와 마찬가지로 포함 모듈인 `ConfigModule`을 사용할 모듈로 가져와야 합니다 (`ConfigModule.forRoot()` 메소드에 전달된 옵션 객체의 `isGlobal` 속성을 `true`로 설정하지 않는 한). 아래와 같이 기능 모듈로 가져옵니다.

### feature.module.ts

```typescript
@Module({
  imports: [ConfigModule],
  // ...
})
```

Then we can inject it using standard constructor injection:

그런 다음 표준 생성자 주입을 사용하여 주입할 수 있습니다.

```typescript
constructor(private configService: ConfigService) {}
```

> **HINT**
>
> The `ConfigService` is imported from the `@nestjs/config` package.
>
> `ConfigService`는 `@nestjs/config` 패키지에서 가져옵니다.

And use it in our class:

리고 우리 클래스에서 사용하십시오.

```typescript
// get an environment variable
const dbUser = this.configService.get<string>('DATABASE_USER');

// get a custom configuration value
const dbHost = this.configService.get<string>('database.host');
```

As shown above, use the `configService.get()` method to get a simple environment variable by passing the variable name. You can do TypeScript type hinting by passing the type, as shown above (e.g., `get<string>(...)`). The `get()` method can also traverse a nested custom configuration object (created via a [Custom configuration file](https://docs.nestjs.com/techniques/configuration#custom-configuration-files)), as shown in the second example above.

You can also get the whole nested custom configuration object using an interface as the type hint:

위와 같이 `configService.get()` 메소드를 사용하여 변수 이름을 전달하여 간단한 환경 변수를 가져옵니다. 위에 표시된대로 타입을 전달하여 TypeScript 타입 힌트를 수행할 수 있습니다 (예: `get<string>(...)`). `get()`메소드는 위의 두번째 예와 같이 중첩된 사용자 정의 구성 객체 ([사용자 정의 구성 파일](https://docs.nestjs.kr/techniques/configuration#custom-configuration-files)을 통해 생성됨)를 탐색할 수도 있습니다.

인터페이스를 타입 힌트로 사용하여 전체 중첩된 사용자 지정 구성 객체를 가져올 수도 있습니다.

```typescript
interface DatabaseConfig {
  host: string;
  port: number;
}

const dbConfig = this.configService.get<DatabaseConfig>('database');

// you can now use `dbConfig.port` and `dbConfig.host`
const port = dbConfig.port;
```

The `get()` method also takes an optional second argument defining a default value, which will be returned when the key doesn't exist, as shown below:

`get()` 메서드는 또한 기본값을 정의하는 두번째 인수를 선택적으로 받습니다. 기본값은 아래와 같이 키가 존재하지 않을 때 반환됩니다.

```typescript
// use "localhost" when "database.host" is not defined
const dbHost = this.configService.get<string>('database.host', 'localhost');
```

`ConfigService` has an optional generic (type argument) to help prevent accessing a config property that does not exist. Use it as shown below:

`ConfigService`에는 존재하지 않는 구성 속성에 대한 액세스를 방지하는 데 도움이 되는 선택적 일반(타입 인수)이 있습니다. 아래와 같이 사용하십시오.

```typescript
interface EnvironmentVariables {
  PORT: number;
  TIMEOUT: string;
}

// somewhere in the code
constructor(private configService: ConfigService<EnvironmentVariables>) {
  const port = this.configService.get('PORT', { infer: true });

  // Error: this is invalid as the URL property is not defined
  const url = this.configService.get('URL', { infer: true });
}
```

With the `infer` property set to `true`, the `ConfigService#get` method will automatically infer the property type based on the interface, so for example, `typeof port === "number"` since `PORT` has a `number` type in the `EnvironmentVariables` interface.

Also, with the `infer` feature, you can infer the type of a nested custom configuration object's property, even when using dot notation, as follows:

`infer` 속성을 `true`로 설정하면 `ConfigService#get` 메서드가 인터페이스를 기반으로 속성 유형을 자동으로 추론합니다. 예를 들어 `PORT`는 `EnvironmentVariables` 인터페이스에 `number` 유형을 가지므로 `typeof port === "number"`입니다.

또한 `infer` 기능을 사용하면 다음과 같이 점 표기법을 사용하는 경우에도 중첩된 사용자 지정 구성 객체의 속성 유형을 유추할 수 있습니다.

```typescript
constructor(private configService: ConfigService<{ database: { host: string } }>) {
  const dbHost = this.configService.get('database.host', { infer: true });
  // typeof dbHost === "string"
}
```



## Configuration namespaces

The `ConfigModule` allows you to define and load multiple custom configuration files, as shown in [Custom configuration files](https://docs.nestjs.com/techniques/configuration#custom-configuration-files) above. You can manage complex configuration object hierarchies with nested configuration objects as shown in that section. Alternatively, you can return a "namespaced" configuration object with the `registerAs()` function as follows:

`ConfigModule`을 사용하면 위의 [커스텀 구성 파일](https://docs.nestjs.kr/techniques/configuration#custom-configuration-files)에 표시된대로 여러 사용자 정의 구성 파일을 정의하고 로드할 수 있습니다. 해당 섹션에 표시된 것처럼 중첩된 구성 개체를 사용하여 복잡한 구성 객체 계층을 관리할 수 있습니다. 또는 다음과 같이 `registerAs()` 함수를 사용하여 "네임스페이스" 구성 객체를 반환할 수 있습니다.

### config/database.config.ts

```typescript
export default registerAs('database', () => ({
  host: process.env.DATABASE_HOST,
  port: process.env.DATABASE_PORT || 5432
}));
```

As with custom configuration files, inside your `registerAs()` factory function, the `process.env` object will contain the fully resolved environment variable key/value pairs (with `.env` file and externally defined variables resolved and merged as described [above](https://docs.nestjs.com/techniques/configuration#getting-started)).

커스텀 구성 파일과 마찬가지로 `registerAs()` 팩토리 함수 내에서 `process.env` 객체는 완전히 해결된 환경변수 키/값 쌍을 포함합니다 ([위](https://docs.nestjs.kr/techniques/configuration#getting-started)에 설명된대로 해결 및 병합된 `.env` 파일 및 외부 정의 변수 포함).

> **HINT**
>
> The `registerAs` function is exported from the `@nestjs/config` package.
>
> `registerAs` 함수는 `@nestjs/config` 패키지에서 내보냅니다.

Load a namespaced configuration with the `load` property of the `forRoot()` method's options object, in the same way you load a custom configuration file:

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [
    ConfigModule.forRoot({
      load: [databaseConfig],
    }),
  ],
})
export class AppModule {}
```

Now, to get the `host` value from the `database` namespace, use dot notation. Use `'database'` as the prefix to the property name, corresponding to the name of the namespace (passed as the first argument to the `registerAs()` function):

이제 `database` 네임스페이스에서 `host` 값을 가져 오려면 점 표기법(dot notation)을 사용하십시오. 네임 스페이스의 이름에 해당하는 속성 이름의 접두사로 `'database'`를 사용합니다 (`registerAs()` 함수의 첫번째 인수로 전달됨).

```typescript
const dbHost = this.configService.get<string>('database.host');
```

A reasonable alternative is to inject the `database` namespace directly. This allows us to benefit from strong typing:

합리적인 대안은 `database` 네임스페이스를 직접 삽입하는 것입니다. 이를 통해 강력한 타이핑의 이점을 얻을 수 있습니다.

```typescript
constructor(
  @Inject(databaseConfig.KEY)
  private dbConfig: ConfigType<typeof databaseConfig>,
) {}
```

> **HINT**
>
> The `ConfigType` is exported from the `@nestjs/config` package.
>
> `ConfigType`은 `@nestjs/config` 패키지에서 내보내집니다.



## Cache environment variables

As accessing `process.env` can be slow, you can set the `cache` property of the options object passed to `ConfigModule.forRoot()` to increase the performance of `ConfigService#get` method when it comes to variables stored in `process.env`.

`process.env`에 대한 액세스 속도가 느릴 수 있으므로 `ConfigModule.forRoot()`에 전달된 옵션 객체의 `cache` 속성을 설정하여 `process.env`에 저장된 변수에 대해 `ConfigService#get` 메서드의 성능을 높일 수 있습니다.

```typescript
ConfigModule.forRoot({
  cache: true,
});
```



## Partial registration

Thus far, we've processed configuration files in our root module (e.g., `AppModule`), with the `forRoot()` method. Perhaps you have a more complex project structure, with feature-specific configuration files located in multiple different directories. Rather than load all these files in the root module, the `@nestjs/config` package provides a feature called **partial registration**, which references only the configuration files associated with each feature module. Use the `forFeature()` static method within a feature module to perform this partial registration, as follows:

지금까지 `forRoot()` 메서드를 사용하여 루트 모듈(예: `AppModule`)에서 구성 파일을 처리했습니다. 여러 다른 디렉토리에 기능별 구성 파일이 있는 더 복잡한 프로젝트 구조가 있을 수 있습니다. 이러한 모든 파일을 루트 모듈에 로드하는 대신 `@nestjs/config` 패키지는 각 기능 모듈과 관련된 구성 파일만 참조하는 **부분 등록**이라는 기능을 제공합니다. 기능 모듈 내에서 `forFeature()` 정적 메서드를 사용하여 다음과 같이 부분 등록을 수행합니다.

```typescript
import databaseConfig from './config/database.config';

@Module({
  imports: [ConfigModule.forFeature(databaseConfig)],
})
export class DatabaseModule {}
```

> **WARNING**
>
> In some circumstances, you may need to access properties loaded via partial registration using the `onModuleInit()` hook, rather than in a constructor. This is because the `forFeature()` method is run during module initialization, and the order of module initialization is indeterminate. If you access values loaded this way by another module, in a constructor, the module that the configuration depends upon may not yet have initialized. The `onModuleInit()` method runs only after all modules it depends upon have been initialized, so this technique is safe.
>
> 경우에 따라 생성자가 아닌 `onModuleInit()` 후크를 사용하여 부분 등록을 통해 로드된 속성에 액세스해야 할 수 있습니다. 모듈 초기화 과정에서 `forFeature()` 메서드가 실행되고 모듈 초기화 순서가 정해져 있지 않기 때문입니다. 다른 모듈에 의해 이러한 방식으로 로드된 값에 액세스하는 경우 생성자에서 구성이 의존하는 모듈이 아직 초기화되지 않았을 수 있습니다. `onModuleInit()`메서드는 종속된 모든 모듈이 초기화된 후에 만 실행되므로 이 기술은 안전합니다.



## Schema validation

It is standard practice to throw an exception during application startup if required environment variables haven't been provided or if they don't meet certain validation rules. The `@nestjs/config` package enables two different ways to do this:

- [Joi](https://github.com/sideway/joi) built-in validator. With Joi, you define an object schema and validate JavaScript objects against it.
- A custom `validate()` function which takes environment variables as an input.

To use Joi, we must install Joi package:

필수 환경 변수가 제공되지 않았거나 특정 유효성 검사 규칙을 충족하지 않는 경우 애플리케이션 시작 중에 예외를 던지는(throw) 것이 표준 관행입니다. `@nestjs/config` 패키지를 사용하면 두가지 방법으로 이를 수행할 수 있습니다.

- [Joi](https://github.com/sideway/joi) 내장 검사기. Joi를 사용하여 객체 스키마를 정의하고 이에 대해 JavaScript 객체의 유효성을 검사합니다.
- 환경 변수를 입력으로 받는 사용자 정의 `validate()` 함수.

Joi를 사용하려면 Joi 패키지를 설치해야합니다.

```bash
$ npm install --save joi
```

> **NOTICE**
>
> The latest version of `joi` requires you to be running Node v12 or later. For older versions of node, please install `v16.1.8`. This is mainly after the release of `v17.0.2` which causes errors during build time. For more information, please refer to [their 17.0.0 release notes](https://github.com/sideway/joi/issues/2262).
>
> 최신 버전의 `joi`를 사용하려면 Node v12 이상을 실행해야 합니다. 이전 버전의 노드의 경우 `v16.1.8`을 설치하십시오. 이는 주로 빌드 시간에 오류가 발생하는 `v17.0.2`출시 이후입니다. 자세한 내용은 [17.0.0 출시 노트](https://github.com/sideway/joi/issues/2262)를 참조하세요.

Now we can define a Joi validation schema and pass it via the `validationSchema` property of the `forRoot()` method's options object, as shown below:

이제 Joi 유효성 검사 스키마를 정의하고 아래와 같이 `forRoot()` 메서드의 옵션 객체의 `validationSchema` 속성을 통해 전달할 수 있습니다.

### app.module.ts

```typescript
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

By default, all schema keys are considered optional. Here, we set default values for `NODE_ENV` and `PORT` which will be used if we don't provide these variables in the environment (`.env` file or process environment). Alternatively, we can use the `required()` validation method to require that a value must be defined in the environment (`.env` file or process environment). In this case, the validation step will throw an exception if we don't provide the variable in the environment. See [Joi validation methods](https://joi.dev/api/?v=17.3.0#example) for more on how to construct validation schemas.

By default, unknown environment variables (environment variables whose keys are not present in the schema) are allowed and do not trigger a validation exception. By default, all validation errors are reported. You can alter these behaviors by passing an options object via the `validationOptions` key of the `forRoot()` options object. This options object can contain any of the standard validation options properties provided by [Joi validation options](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options). For example, to reverse the two settings above, pass options like this:

기본적으로 모든 스키마 키는 선택사항으로 간주됩니다. 여기서는 환경 (`.env` 파일 또는 프로세스 환경)에서 이러한 변수를 제공하지 않을 경우 사용할 `NODE_ENV` 및 `PORT`에 대한 기본값을 설정합니다. 또는 `required()` 유효성 검사 메서드를 사용하여 환경 (`.env` 파일 또는 프로세스 환경)에서 값을 정의해야합니다. 이 경우 환경에서 변수를 제공하지 않으면 유효성 검사 단계에서 예외가 발생합니다. 유효성 검사 스키마를 구성하는 방법에 대한 자세한 내용은 [Joi 유효성 검사 방법](https://joi.dev/api/?v=17.3.0#example)을 참조하세요.

기본적으로 알 수 없는 환경변수 (스키마에 키가 없는 환경 변수)가 허용되며 유효성 검사 예외를 트리거하지 않습니다. 기본적으로 모든 유효성 검사 오류가 보고 됩니다. `forRoot()` 옵션 객체의 `validationOptions` 키를 통해 옵션 객체를 전달하여 이러한 동작을 변경할 수 있습니다. 이 옵션 개체는 [Joi 유효성 검사 옵션](https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options)에서 제공하는 표준 유효성 검사 옵션 속성을 포함할 수 있습니다. 예를 들어 위의 두 설정을 되돌리려면 다음과 같은 옵션을 전달합니다.

### app.module.ts

```typescript
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        NODE_ENV: Joi.string()
          .valid('development', 'production', 'test', 'provision')
          .default('development'),
        PORT: Joi.number().default(3000),
      }),
      validationOptions: {
        allowUnknown: false,
        abortEarly: true,
      },
    }),
  ],
})
export class AppModule {}
```

The `@nestjs/config` package uses default settings of:

- `allowUnknown`: controls whether or not to allow unknown keys in the environment variables. Default is `true`
- `abortEarly`: if true, stops validation on the first error; if false, returns all errors. Defaults to `false`.

Note that once you decide to pass a `validationOptions` object, any settings you do not explicitly pass will default to `Joi` standard defaults (not the `@nestjs/config` defaults). For example, if you leave `allowUnknowns` unspecified in your custom `validationOptions` object, it will have the `Joi` default value of `false`. Hence, it is probably safest to specify **both** of these settings in your custom object.

`@nestjs/config` 패키지는 다음의 기본 설정을 사용합니다.

- `allowUnknown`: 환경변수에서 알 수 없는 키를 허용할지 여부를 제어합니다. 기본값은 `true`입니다.
- `abortEarly`: `true`이면 첫번째 오류에서 유효성 검사를 중지합니다. `false`이면 모든 오류를 반환합니다. 기본값은 `false`입니다.

일단 `validationOptions` 객체를 전달하기로 결정하면 명시적으로 전달하지 않은 모든 설정은 `Joi` 표준 기본값 (`@nestjs/config` 기본값이 아님)으로 기본 설정됩니다. 예를 들어 사용자 정의 `validationOptions` 객체에서 `allowUnknowns`를 지정하지 않은 상태로 두면 `Joi` 기본값인 `false`가 됩니다. 따라서 사용자 지정 객체에서 이러한 설정을 **둘 다** 지정하는 것이 가장 안전할 수 있습니다.



## Custom validate function

Alternatively, you can specify a **synchronous**`validate` function that takes an object containing the environment variables (from env file and process) and returns an object containing validated environment variables so that you can convert/mutate them if needed. If the function throws an error, it will prevent the application from bootstrapping.

In this example, we'll proceed with the `class-transformer` and `class-validator` packages. First, we have to define:

- a class with validation constraints,
- a validate function that makes use of the `plainToClass` and `validateSync` functions.

또는 환경변수 (env 파일 및 프로세스에서)를 포함하는 객체를 가져와 필요한 경우 변환/변경할 수 있도록 검증된 환경변수가 포함된 객체를 반환하는 **동기**`validate` 함수를 지정할 수 있습니다. 함수에서 오류가 발생하면 애플리케이션이 부트스트랩되지 않습니다.

이 예에서는 `class-transformer` 및 `class-validator` 패키지를 진행합니다. 먼저 다음을 정의해야 합니다.

- 유효성 검사 제약이 있는 클래스,
- `plainToClass` 및 `validateSync` 함수를 사용하는 유효성 검사 함수.

### env.validation.ts

```typescript
import { plainToClass } from 'class-transformer';
import { IsEnum, IsNumber, validateSync } from 'class-validator';

enum Environment {
  Development = "development",
  Production = "production",
  Test = "test",
  Provision = "provision",
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment;

  @IsNumber()
  PORT: number;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToClass(
    EnvironmentVariables,
    config,
    { enableImplicitConversion: true },
  );
  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }
  return validatedConfig;
}
```

With this in place, use the `validate` function as a configuration option of the `ConfigModule`, as follows:

이 상태에서 다음과 같이 `ConfigModule`의 구성 옵션으로 `validate` 함수를 사용합니다.

### app.module.ts

```typescript
import { validate } from './env.validation';

@Module({
  imports: [
    ConfigModule.forRoot({
      validate,
    }),
  ],
})
export class AppModule {}
```



## Custom getter functions

`ConfigService` defines a generic `get()` method to retrieve a configuration value by key. We may also add `getter` functions to enable a little more natural coding style:

`ConfigService`는 키로 구성값을 검색하는 일반적인 `get()` 메소드를 정의합니다. 좀 더 자연스러운 코딩 스타일을 사용하기 위해 `getter` 함수를 추가할 수도 있습니다.

```typescript
@Injectable()
export class ApiConfigService {
  constructor(private configService: ConfigService) {}

  get isAuthEnabled(): boolean {
    return this.configService.get('AUTH_ENABLED') === 'true';
  }
}
```

Now we can use the getter function as follows:

이제 다음과 같이 getter 함수를 사용할 수 있습니다.

### app.service.ts

```typescript
@Injectable()
export class AppService {
  constructor(apiConfigService: ApiConfigService) {
    if (apiConfigService.isAuthEnabled) {
      // Authentication is enabled
    }
  }
}
```



## Expandable variables

The `@nestjs/config` package supports environment variable expansion. With this technique, you can create nested environment variables, where one variable is referred to within the definition of another. For example:

`@nestjs/config` 패키지는 환경변수 확장을 지원합니다. 이 기술을 사용하면 한 변수가 다른 정의내에서 참조되는 중첩 환경변수를 만들 수 있습니다. 예를 들면:

```json
APP_URL=mywebsite.com
SUPPORT_EMAIL=support@${APP_URL}
```

With this construction, the variable `SUPPORT_EMAIL` resolves to `'support@mywebsite.com'`. Note the use of the `${...}` syntax to trigger resolving the value of the variable `APP_URL` inside the definition of `SUPPORT_EMAIL`.

이 구성에서 `SUPPORT_EMAIL` 변수는 `'support@mywebsite.com'`으로 해석됩니다. `${...}` 구문을 사용하여 `SUPPORT_EMAIL` 정의내에서 `APP_URL` 변수의 값을 확인합니다.

> **HINT**
>
> For this feature, `@nestjs/config` package internally uses [dotenv-expand](https://github.com/motdotla/dotenv-expand).
>
> 이 기능을 위해 `@nestjs/config` 패키지는 내부적으로 [dotenv-expand](https://github.com/motdotla/dotenv-expand)를 사용합니다.

Enable environment variable expansion using the `expandVariables` property in the options object passed to the `forRoot()` method of the `ConfigModule`, as shown below:

아래와 같이 `ConfigModule`의 `forRoot()` 메소드에 전달된 옵션 객체의 `expandVariables` 속성을 사용하여 환경변수 확장을 활성화합니다.

### app.module.ts

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      // ...
      expandVariables: true,
    }),
  ],
})
export class AppModule {}
```



## Using in the `main.ts`

While our config is a stored in a service, it can still be used in the `main.ts` file. This way, you can use it to store variables such as the application port or the CORS host.

To access it, you must use the `app.get()` method, followed by the service reference:

우리의 설정은 서비스에 저장되지만 `main.ts` 파일에서 계속 사용할 수 있습니다. 이런식으로 애플리케이션 포트 또는 CORS 호스트와 같은 변수를 저장하는 데 사용할 수 있습니다.

액세스하려면 `app.get()` 메소드와 서비스 참조를 사용해야합니다.

```typescript
const configService = app.get(ConfigService);
```

You can then use it as usual, by calling the `get` method with the configuration key:

그런 다음 구성 키로 `get` 메소드를 호출하여 평소처럼 사용할 수 있습니다.

```typescript
const port = configService.get('PORT');
```



#### 출처

> https://docs.nestjs.com/techniques/configuration
>
> https://docs.nestjs.kr/techniques/configuration