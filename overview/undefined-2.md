# 미들웨어

미들웨어는 라우트 핸들러 이전에 호출되는 함수입니다. 미들웨어 함수는 요청(request)과 응답(response) 객체에 접근할 수 있으며, 애플리케이션의 요청-응답 사이클에서 다음 미들웨어 함수인 `next()`에도 접근할 수 있습니다. 다음 미들웨어 함수는 보통 `next`라는 변수로 표시됩니다.

Nest의 미들웨어는 기본적으로 **express** 미들웨어와 동일합니다. 공식 express 문서에서는 미들웨어의 기능을 다음과 같이 설명하고 있습니다:

> 미들웨어 함수는 다음과 같은 작업을 수행할 수 있습니다:
>
> * 어떤 코드든 실행 가능
> * 요청, 응답 객체를 변경
> * 요청-응답 사이클을 종료
> * 스택에서 다음 미들웨어 함수 호출하기.
> * 현재 미들웨어 함수가 요청-응답 사이클을 종료하지 않는 경우, `next()`를 호출하여 제어를 다음 미들웨어 함수로 넘겨야 합니다. 그렇지 않으면 요청이 대기 상태로 남게 됩니다.

사용자 정의 Nest 미들웨어는 함수 형태 혹은`@Injectable()` 데코레이터가 있는 클래스 형태로 구현할 수 있습니다. 클래스는 `NestMiddleware` 인터페이스를 implement 해야하며, 함수는 특별한 요구 사항이 없습니다. 클래스 방식을 사용하여 간단한 미들웨어 기능을 구현해봅시다.

> **주의**\
> `Express`와 `fastify`는 미들웨어를 다르게 처리하며 다른 메소드 시그니처를 제공합니다. 여기에서 자세히 알아보세요.

{% code title="logger.middleware.tsJS" overflow="wrap" %}
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
{% endcode %}



### 의존성 주입

Nest 미들웨어는 의존성 주입을 완 지원합니다. 프로바이더, 컨트롤러와 마찬가지로, 같은 모듈 내에서 사용 가능한 의존성을 주입할 수 있습니다. 이 작업은 평소와 같이 `consturctor` 통해 이루어집니다.



### 미들웨어 적용



`@Module()` 데코레이터에서 미들웨어를 설정하지는 않습니다. 대신, 모듈 클래스의 `configure()` 메소드를 사용하여 설정합니다. 미들웨어를 사용하는 모듈은 `NestModule` 인터페이스를 implement해야 합니다. `AppModule`에서 `LoggerMiddleware`를 설정해봅시다.

{% code title="app.module.ts" overflow="wrap" %}
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
{% endcode %}

앞의 예제에서 우리는 `CatsController` 내부에 이전에 정의된 /cats 라우트 핸들러에 대해 `LoggerMiddleware`를 설정했습니다. 미들웨어를 특정 요청 method에 적용하려면, 라우트 `path`와 요청 `method`을 담은 객체를 `forRoutes()` 메소드에 전달할 수도 있습니다. 아래 예제에서는 원하는 method의 타입을 참조하기 위해 `RequestMethod` enum을 임포트해서 사용합니다.

> **힌트**\
> configure() 메서드는 async/await을 사용하여 비동기적으로 작동할 수 있습니다\
> (예: configure() 메서드 본문 내에서 비동기 연산을 기다리기 위해 `await`을 사용할 수 있습니다.)

> **주의**\
> `express`를 사용한다면 NestJS 앱은 `body-parser` 패키지를 사용하여 `json` `urlencoded`를 등록합니다. 즉, `MiddlewareConsumer`를 활용하여 미들웨어를 커스터마이즈하고 싶다면 `NestFactory.create()`로 애플리케이션을 생성할 때 `bodyParser` 플래그를 `false`로 설정하여 전역 미들웨어를 꺼야 합니다.



### 라우트 와일드카드

패턴 기반 라우트도 지원됩니다. 예를 들어, 별표(\*)는 와일드카드로 사용되며, 어떤 문자 조합과도 일치합니다:

```typescript
forRoutes({ path: 'ab*cd', method: RequestMethod.ALL });
```

`'ab*cd'` 라우트 path는 `abcd`, `ab_cd`, `abecd` 등과 일치합니다. `?`, `+`, `*`, `()` 등의 문자들은 라우트 path에서 사용되며, 정규 표현식의 일부로 해석됩니다. 하이픈(`-`)과 점(`.`)은 문자 기반 경로에서 문자 그대로 해석됩니다.

> **주의**\
> fastify 패키지는 최신 버전의 `path-to-regexp` 패키지를 사용하는데, 이 패키지는 더 이상 와일드카드 `*`를 지원하지 않습니다. 대신, 파라미터(예: `(.*)`, `:splat*`)를 사용해야 합니다.



### 미들웨어 컨슈머

`MiddlewareConsumer`는 헬퍼 클래스로, 미들웨어를 관리하기 위한 몇 가지 내장 메서드를 제공합니다. 모든 메서드는 [**fluent 스타일**](https://en.wikipedia.org/wiki/Fluent\_interface)로 간단히 연결할 수 있습니다. `forRoutes()` 메서드는 단일 문자열, 여러 문자열, `RouteInfo` 객체, 컨트롤러 클래스, 심지어 여러 컨트롤러 클래스를 받을 수 있습니다. 대부분의 경우 쉼표로 구분된 컨트롤러 목록을 전달할 것입니다. 아래는 단일 컨트롤러를 사용한 예제입니다:

{% code title="app.module.ts" %}
```typescript
import { Module, NestModule, MiddlewareConsumer } from '@nestjs/common';
import { LoggerMiddleware } from './common/middleware/logger.middleware';
import { CatsModule } from './cats/cats.module';
import { CatsController } from './cats/cats.controller';

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
{% endcode %}

> **힌트**\
> apply() 메서드는 단일 미들웨어를 받거나 [여러 미들웨어를 지정](undefined-2.md#undefined-5)하기 위해 여러 인수를 받을 수 있습니다.



### routes 제외하기

때때로 특정 경로를 미들웨어에가 적용되지 않도록 제외시키고 싶을 때가 있습니다. `exclude()` 메서드를 사용하면 특정 경로를 쉽게 제외할 수 있습니다. 이 메서드는 아래와 같이 제외할 경로를 식별하는 단일 문자열, 여러 문자열 또는 `RouteInfo` 객체를 받을 수 있습니다:

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

> **힌트**
>
> `exclude()`메서드는 `path-to-regexp` 패키지를 활용하여 와일드카드 파라미터를 지원합니다. &#x20;

위의 예제에서 `LoggerMiddleware`는 `exclude()` 메서드에 전달된 세 가지를 제외하고 `CatsController` 내부에 정의된 모든 경로에 적용됩니다.



### 함수형 미들웨어

우리가 사용한 `LoggerMiddleware` 클래스는 매우 간단합니다. 멤버도 없고, 추가 메서드도 없으며, 의존성도 없습니다. 클래스 대신 간단한 함수로 미들웨어를 만들 수는 없을까요? 사실 가능합니다. 이러한 유형의 미들웨어를 함수형 미들웨어라고 합니다. 위에 만든 미들웨어를 클래스 기반에서 함수형 미들웨어로 바꾸고, 차이점에 대해 알아봅시다.

{% code title="logger.middleware.ts" %}
```typescript
import { Request, Response, NextFunction } from 'express';

export function logger(req: Request, res: Response, next: NextFunction) {
  console.log(`Request...`);
  next();
};
```
{% endcode %}

그리고 AppModule에서는 이렇게 사용합니다.

{% code title="app.module.ts" %}
```typescript
consumer
  .apply(logger)
  .forRoutes(CatsController);
```
{% endcode %}

> 힌트
>
> 만들려고 하는 미들웨어가 의존성이 필요하지 않다면 언제든 더 간단한 함수형 미들웨어를 사용하는 것을 고려하세요.



### 여러 미들웨어 사용하기

위에서 언급했듯이 순차적으로 실행되는 여러 미들웨어를 바인딩하려면 apply() 메서드 안에 쉼표로 구분된 목록을 제공하기만 하면 됩니다:

```typescript
consumer.apply(cors(), helmet(), logger).forRoutes(CatsController);
```



### 글로벌 미들웨어

등록된 모든 경로에 미들웨어를 한 번에 적용하려면 `INestApplication` 인스턴스에서 제공하는 `use()` 메서드를 사용할 수 있습니다:

{% code title="main.ts" %}
```typescript
const app = await NestFactory.create(AppModule);
app.use(logger);
await app.listen(3000);
```
{% endcode %}

> **힌트**
>
> 글로벌 미들웨어에서는 DI 컨테이너에 액세스할 수 없습니다. `app.use()`를 사용할 때는 함수형 미들웨어를 대신 사용할 수 있습니다. 또는 클래스 미들웨어를 사용하고 `AppModule`(또는 다른 모듈) 내에서 `.forRoutes('*')`로 적용할 수 있습니다
