# 모듈

모듈은 `@Module()` 데코레이터가 붙은 클래스입니다. `@Module()` 데코레이터는 **Nest**가 애플리케이션 구조를 구성하는데 사용하는 메타데이터를 제공합니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

각 애플리케이션은 최소한 하나의 모듈, 즉 **루트 모듈**을 가지고 있습니다. 루트 모듈은 Nest가 **애플리케이션 그래프**를 만드는 시작점입니다. 애플리케이션 그래프는 Nest가 모듈-프로바이더 관계, 의존성을 관리하는데 사용하는 내부 데이터 구조입니다. 매우 작은 애플리케이션은 이론적으로 루트 모듈만 존재할 수도 있습니다만, 이는 일반적인 경우는 아닙니다. 모듈은 컴포넌트를 효과적으로 조직하는 방법으로 강력히 권장됩니다. 따라서 대부분의 애플리케이션의 아키텍처는 여러 개의 모듈을 사용하며, 각 모듈은 밀접하게 관련된 기능들을 캡슐화합니다.

`@Module()` 데코레이터는 모듈을 정의하는 단일 객체를 받습니다.

|               |                                                                                                |
| ------------- | ---------------------------------------------------------------------------------------------- |
| `providers`   | Nest 인젝터에 의해 인스턴스화되고 이 모듈에서 적어도 공유될 수 있는 프로바이더                                                 |
| `controllers` | 이 모듈에서 정의되고 인스턴스화되어야 하는 컨트롤러 집합                                                                |
| `imports`     | 이 모듈에서 필요한 프로바이더를 내보내는 가져온 모듈 목록                                                               |
| `exports`     | 이 모듈에 의해 제공되고 이 모듈을 가져오는 다른 모듈에서 사용 가능해야 하는 프로바이더의 하위 집합입니다. 프로바이더 자체 또는 토큰(제공 값)만 사용할 수 있습니다. |

모듈은 기본적으로 프로바이더를 캡슐화합니다. 즉, 현재 모듈의 직접적인 일부가 아니거나 가져온 모듈에서 내보낸 프로바이더를 주입하는 것은 불가능합니다. 따라서 모듈에서 내보낸 프로바이더는 모듈의 공개 인터페이스 또는 API로 간주할 수 있습니다.



### 피쳐 모듈

`CatsController`와 `CatsService`는 동일한 애플리케이션 도메인에 속합니다. 이들은 서로 밀접하게 연관되어 있기 때문에, 기능 모듈로 이동하는 것이 좋습니다. 기능 모듈은 특정 기능과 관련된 코드를 간단히 정리하여 코드를 체계적으로 유지하고 명확한 경계를 설정합니다. 이는 특히 애플리케이션 및/또는 팀의 규모가 커짐에 따라 복잡성을 관리하고 SOLID 원칙에 따라 개발하는 데 도움이 됩니다.

예시로 `CatsModule`을 만들어 보겠습니다.

{% code title="cats/cats.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```
{% endcode %}

> **힌트**
>
> CLI를 사용하여 모듈을 만들려면 `$ nest g module cats` 명령을 실행하세요.

위에서 `cats.module.ts` 파일에 `CatsModule`을 정의하고 이 모듈과 관련된 모든 것을 `cats` 디렉토리로 옮겼습니다. 마지막으로 해야 할 일은 이 모듈을 루트 모듈(`app.module.ts` 파일에 정의된 `AppModule`)로 가져오는 것입니다.

{% code title="app.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```
{% endcode %}

이제 디렉토리 구조는 다음과 같습니다.

<pre><code><strong>src
</strong>├──cats
│  ├─┬ dto
│  │ └── create-cat.dto.ts
│  ├─┬ interfaces
│  │ └── cat.interface.ts
│  ├── cats.controller.ts
│  ├── cats.module.ts
│  └── cats.service.ts
├──app.module.ts
└──main.ts
</code></pre>



### 공유 모듈

Nest에서 모듈은 기본적으로 싱글톤이므로, 여러 모듈 간에 모든 프로바이더의 동일한 인스턴스를 쉽게 공유할 수 있습니다.

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

모든 모듈은 자동으로 공유 모듈입니다. 한 번 생성되면 모든 모듈에서 재사용할 수 있습니다. 다른 여러 모듈에서 `CatsService` 인스턴스를 공유하고 싶다고 가정해보겠습니다. 이를 위해 먼저 아래처럼 모듈의 `exports` 배열에 `CatsService` 프로바이더를 추가하여 내보내야 합니다.

{% code title="cats.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```
{% endcode %}

이제 `CatsModule`을 가져오는하는 모든 모듈은 `CatsService`에 액세스할 수 있으며 `CatsModule`가져오는 다른 모든 모듈과 동일한 인스턴스를 공유하게 됩니다.



### 모듈 **re-exporting**

위에서 보았듯이 모듈은 내부의 프로바이더를 내보낼 수 있습니다. 또한 가져온 모듈을 다시 내보낼 수도 있습니다. 아래 예시에서는 `CommonModule`을 `CoreModule`에서 가져오고 내보내어 이 모듈을 가져오는 다른 모듈에서 사용할 수 있도록 합니다.

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```



### 의존성 주입

모듈 클래스는 프로바이더를 주입할 수도 있습니다(예: 설정을 위한 주입)

{% code title="cats.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private catsService: CatsService) {}
}
```
{% endcode %}

그러나 모듈 클래스 자체는 순환 의존성으로 인해 프로바이더로 주입될 수 없습니다.



### 전역 모듈

모든 곳에서 동일한 모듈을 가져와야 한다면 귀찮을 수 있습니다. Nest와 달리 **Angular** `provider`가 전역 scope에 등록됩니다. 한번 정의하면 어디서나 사용할 수 있습니다. 그러나 Nest는 프로바이더를 모듈 scope 내에 캡슐화합니다. 캡슐화된 모듈을 먼저 가져오지 않으면 모듈의 프로바이더를 다른 곳에서 사용할 수 없습니다.

헬퍼, 데이터베이스 연결 등 모든 곳에서 즉시 사용할 수 있어야 하는 프로바이더를 만들려는 경우 `@Global()` 데코레이터를 사용하여 모듈을 전역으로 설정합니다.

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService],
})
export class CatsModule {}
```

`@Global()` 데코레이터는 모듈을 전역 모듈로 만듭니다. 전역 모듈은 일반적으로 루트 모듈이나 코어 모듈에 의해 한 번만 등록되어야 합니다. 위의 예시에서 `CatsService` 프로바이더는 어디서든 사용할 수 있으며, 이 서비스를 사용하려는 모듈은 imports 배열에서 `CatsModule`을 임포트할 필요가 없습니다.

> 힌트
>
> 모든 것을 글로벌 모듈로 만드는 것은 좋은 설계 결정이 아닙니다. 글로벌 모듈은 필요한 보일러플레이트의 양을 줄이기 위해 사용됩니다. `imports` 를 사용함으로써 모듈의 API를 사용할 수 있습니다.

###

### 다이나믹 모듈

Nest 모듈 시스템에는 다이나믹 모듈이라는 강력한 기능이 포함되어 있습니다. 이 기능을 사용하면 프로바이더를 동적으로 등록하고 구성할 수 있는 사용자 정의 모듈을 쉽게 만들 수 있습니다. 동적 모듈은 여기에서 광범위하게 다룹니다. 이 장에서는 모듈에 대한 소개를 완료하기 위해 간략한 개요를 제공하겠습니다.

다음은 DatabaseModule에 대한 동적 모듈 정의의 예입니다

```typescript
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

> **힌트**
>
> `forRoot()` 메서드는 모듈을 동기적 또는 비동기적으로(즉, Promise를 통해) 동적 모듈을 반환할 수 있습니다.

이 모듈은 기본적으로 `@Module()` 데코레이터 메타데이터에서 `Connection` 프로바이더를 정의하지만, `forRoot()` 메서드에 전달된 엔티티와 옵션 객체에 따라, 예를 들어 리포지토리와 같은 프로바이더 컬렉션을 추가로 노출합니다. 동적 모듈에 의해 반환된 속성들이 `@Module()` 데코레이터에서 정의된 기본 모듈 메타데이터를 확장한다는 점에 유의하세요. (덮어쓰지는 않습니다). 이것이 정적으로 선언된 Connection 프로바이더와 동적으로 생성된 리포지토리 프로바이더가 모듈에서 어떻게 내보내지는지를 설명합니다.

동적 모듈을 전역 범위에 등록하려면 global 속성을 true로 설정하세요.

```typescript
{
  global: true,
  module: DatabaseModule,
  providers: providers,
  exports: providers,
}
```

> **주의**
>
> 위에서 언급했듯이 모든 것을 전역으로 만드는 것은 좋은 설계 결정이 아닙니다.

DatabaseModule은 다음과 같은 방법으로 가져오고 구성할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
})
export class AppModule {}
```

동적 모듈을 다시 내보내고 싶다면, exports 배열에서 `forRoot()` 메서드 호출을 생략할 수 있습니다:

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [DatabaseModule.forRoot([User])],
  exports: [DatabaseModule],
})
export class AppModule {}
```

동적 모듈에 대한 주제는 동적 모듈 장에서 더 자세히 다루며, 실제 작동하는 예제가 포함되어 있습니다.

> **힌트**
>
> **이 장**에서 `ConfigurableModuleBuilder`를 사용하여 깊게 사용자 정의 가능한 동적 모듈을 구축하는 방법을 배워보세요.
