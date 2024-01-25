# 인터셉터

&#x20;인터셉터는 `@Injectable()` 데코레이터가 달린 클래스로, `NestInterceptor` 인터페이스를 implement합니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

인터셉터는 관점 지향 프로그래밍([Aspect Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented\_programming) - AOP)에 영향을 받은 유용한 기능들이 있습니다.

* 메서드 실행 전/후에 로직을 추가합니다
* 함수에서 반환된 결과를 변환
* 함수에서 던진 예외 변환
* 기본 함수 동작 확장
* 특정 조건에 따라 함수를 오버라이드합니다.(예 - 캐싱)



### 기초

각 인터셉터는 두개의 인수를 받는 `intercept()`를 implement합니다. 첫 번째는 `ExecutionContext` 인스턴스입니다. ([**가드**](undefined-5.md)와 동일) `ExecutionContext`는 ArgumentsHost를 상속합니다. 서 예외 필터 챕터에서 `ArgumentsHost`를 살펴봤습니다. `ArgumentsHost`는 원래의 핸들러로 전달된 인수를 감싸는 wrapper이고, 애플리케이션의 유형에 따라 다른 인수 배열을 포함한다는 것에 대해 알아봤습니다. 이 주제에 대한 자세한 내용은 [**예외 필터**](undefined-3.md)를 다시 살펴보세요.



### 실행 컨텍스트

`ArgumentsHost`를 extend함으로써, `ExecutionContext`는 현재 실행 프로세스에 대한 추가 세부 정보를 제공하는 몇 가지 새로운 헬퍼 메서드도 추가합니다. 이러한 세부 정보는 다양한 컨트롤러, 메서드 및 실행 컨텍스트에서 작동할 수 있는 보다 일반적인 인터셉터를 구축하는 데 유용할 수 있습니다. [**여기**](undefined-6.md#undefined)에서 `ExecutionContext`에 대해 자세히 알아보세요.



### **Call handler**

두 번째 인자는 `CallHandler`입니다. `CallHandler` 인터페이스는 `handle()`메서드를 implement합니다. 이 메서드는 인터셉터의 어느 지점에서 route handler 메서드를 호출하는 데 사용할 수 있습니다. `intercept()` 메서드의 구현에서 `handle()` 메서드를 호출하지 않으면 route handler 메서드는 전혀 실행되지 않습니다.

이 접근 방식은 `intercept()` 메서드가 요청/응답 스트림을 효과적으로 **래핑**한다는 것을 의미합니다. 따라서 최종 route handler **실행 전후 모두에** custom 로직을 구현할 수 있습니다. `intercept()` 메서드에 `handle()` 호출 **전**에 실행되는 코드를 작성할 수 있다는 것은 분명합니다. 하지만 그 이후에 일어나는 일에 어떤 영향을 미칠까요? `handle()` 메서드는 `Observable`을 반환하므로 강력한 [**RxJS**](https://github.com/ReactiveX/rxjs) 연산자를 사용하여 응답을 추가로 조작할 수 있습니다. 객체지향 프로그래밍 용어를 사용하면 라우트 핸들러의 호출(즉, `handle()` 호출)을 [**포인트컷**](https://en.wikipedia.org/wiki/Pointcut)이라고 하는데, 이는 추가 로직이 삽입되는 지점을 나타냅니다.

예를 들어 들어오는 `POST /cats` 요청을 생각해 봅시다. 이 요청은 `CatsController` 내부에 정의된 `create()` 핸들러로 향합니다. 도중에 `handle()` 메서드를 호출하지 않는 인터셉터가 호출되면 `create()` 메서드는 실행되지 않습니다. `handle()` 메서드가 호출되고 해당 `Observable`이 반환되면 `create()` 핸들러가 트리거됩니다. 그리고 `Observable`을 통해 응답 스트림이 수신되면 스트림에서 추가 연산을 수행하고 최종 결과를 호출자(caller)에게 반환할 수 있습니다.



### **Aspect interception**

첫 번째 사용 사례는 인터셉터를 사용하여 사용자의 상호 작용(예: 사용자 호출 저장, 비동기 이벤트 디스패치 또는 타임스탬프 계산)을 로깅하는 것입니다. 아래에서 간단한 `LoggingInterceptor`를 보여드리겠습니다:

{% code title="logging.interceptor.ts" overflow="wrap" %}
```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('Before...');

    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => console.log(`After... ${Date.now() - now}ms`)),
      );
  }
}
```
{% endcode %}

> **힌트**
>
> `NestInterceptor<T, R>`은 제네릭 인터페이스로, `T`는 (응답 스트림을 지원하는) `Observable<T>`의 타입을, `R`은 `Observable<R>`로 래핑된 값의 타입을 나타냅니다.

> **주의**
>
> 컨트롤러, 프로바이더, 가드 처럼 인터셉터는 constructor를 통해 의존성을 주입할 수 있습니다.

`handle()`는 RxJS `Observable`을 반환하므로 스트림을 조작하는 데 사용할 수 있는 다양한 연산자를 선택할 수 있습니다. 위의 예시에서는 observable 스트림이 정상적으로 또는 예외적으로 종료될 때 익명 로깅 함수를 호출하지만 그 외에는 응답 주기를 방해하지 않는 `tap()`연산자를 사용했습니다.



### 인터셉터 바인딩

인터셉터를 설정하기 위해 `@nestjs/common` 패키지에서 import한 `@UseInterceptors()` 데코레이터를 사용합니다. [**파이프**](undefined-4.md) 및 [**가드**](undefined-5.md)와 마찬가지로 인터셉터는 컨트롤러 범위, 메서드 범위 또는 전역 범위가 될 수 있습니다.

{% code title="cats.controller.ts" %}
```typescript

@UseInterceptors(LoggingInterceptor)
export class CatsController {}
```
{% endcode %}

위의 구조를 사용하면 `CatsController`에 정의된 각 라우트 핸들러는 `LoggingInterceptor`를 사용합니다. 누군가 `GET /cats` 엔드포인트를 호출하면 표준 출력에서 다음과 같은 출력을 볼 수 있습니다:

```typescript
Before...
After... 1ms
```

인스턴스 대신 `LoggingInterceptor` 타입을 전달하여 인스턴스화에 대한 책임을 프레임워크에 맡기고 의존성 주입을 활성화했습니다. 파이프, 가드 및 예외 필터와 마찬가지로 인스턴스를 전달할 수도 있습니다:

{% code title="cats.controller.ts" %}
```typescript

@UseInterceptors(new LoggingInterceptor())
export class CatsController {}
```
{% endcode %}

앞서 언급했듯이, 위의 구조는 이 컨트롤러가 선언한 모든 핸들러에 인터셉터를 붙입니다. 인터셉터의 범위를 단일 메서드로 제한하려면 메서드 수준에서 데코레이터를 적용하면 됩니다.

전역 인터셉터를 설정하기 위해 Nest 애플리케이션 인스턴스의 `useGlobalInterceptors()` 메서드를 사용합니다.

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalInterceptors(new LoggingInterceptor());
```

글로벌 인터셉터는 모든 컨트롤러와 모든 라우트 핸들러 전체 애플리케이션에 걸쳐 사용됩니다. 모듈 외부에서 등록한 글로벌 인터셉터(위 예제에서와 같이 `useGlobalInterceptors()`를 사용한)는 모듈의 컨텍스트 외부에서 수행되므로 의존성 주입할 수 없습니다. 이 문제를 해결하기 위해 다음 구성을 사용하여 **모든 모듈에서 직접** 인터셉터를 설정할 수 있습니다:

{% code title="app.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
  ],
})
export class AppModule {}
```
{% endcode %}

> **힌트**
>
> 이 접근 방식을 사용하여 인터셉터에 대한 의존성을 주입할 때, 이 구조가 적용되는 모듈에 관계 없이 인터셉터는 실제로 전역이라는 점에 유의하세요. 이 작업을 어디에서 수행해야 할까요? 인터셉터가 정의된 모듈(위 예시에서는 `LoggingInterceptor`)을 선택하면 됩니다. 또한`useClass는` custom provider를 등록하기 위한 유일한 방법이 아닐 수 있습니다. [**여기**](https://docs.nestjs.com/fundamentals/custom-providers)에서 알아보세요.&#x20;



### Response mapping

우리는 이미 `handle()`이 `Observable`을 반환한다는 것을 알고 있습니다. 스트림에는 라우트 핸들러에서 반환된 값이 포함되어 있으므로 RxJS의 `map()` 연산자를 사용하여 쉽게 변경할 수 있습니다.

> **주의**
>
> 응답 매핑 기능은 library-specific한 응답 방법에는 작동하지 않습니다(`@Res()` 객체를 직접 사용하는 것은 금지되어 있습니다.)

프로세스를 보여주기 위해 각 응답을 간단한 방식으로 수정하는 `TransformInterceptor`를 만들어 보겠습니다. 이 함수는 RxJS의 `map()` 연산자를 사용하여 response 객체를 새로 생성된 객체의 `data` 프로퍼티에 할당하고 새 객체를 클라이언트에 반환합니다.

{% code title="transform.interceptor.ts" %}
```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(context: ExecutionContext, next: CallHandler): Observable<Response<T>> {
    return next.handle().pipe(map(data => ({ data })));
  }
}
```
{% endcode %}

> **힌트**
>
> Nest 인터셉터는 동기 및 비동기 `intercept()` 메서드 모두에서 작동합니다. 필요한 경우 메서드를 `async`로 변경하기만 하면 됩니다.

위의 구성을 사용하면 누군가 `GET /cats` 엔드포인트를 호출할 때 응답은 다음과 같이 표시됩니다(route handler가 빈 array `[]`을 반환한다고 가정할 때)

```json
{
  "data": []
}
```

인터셉터는 전체 애플리케이션에서 발생하는 요구사항에 대해 재사용 가능한 솔루션을 만드는 데 큰 가치가 있습니다. 예를 들어, `null` 값이 있을 때마다 빈 문자열 `''`로 변환해야 한다고 가정해 보겠습니다. 한 줄의 코드로 이 작업을 수행하고 인터셉터를 전역적으로 바인딩하여 등록된 각 핸들러에서 자동으로 사용하도록 할 수 있습니다.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => value === null ? '' : value ));
  }
}
```



### **Exception mapping**

또 다른 흥미로운 사용 사례는 RxJS의 `catchError()` 연산자를 활용하여 던져진 예외를 override하는 것입니다.

{% code title="errors.interceptor.ts" %}
```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  BadGatewayException,
  CallHandler,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(
        catchError(err => throwError(() => new BadGatewayException())),
      );
  }
}
```
{% endcode %}



### **Stream overriding**

핸들러 호출을 완전히 차단하는 대신 다른 값을 반환하는 데에는 몇 가지 이유가 있습니다. 대표적인 예가 응답 시간을 개선하기 위해 캐시를 구현하는 것입니다. 캐시에서 응답을 반환하는 간단한 캐시 인터셉터를 살펴보겠습니다. 현실적인 예제에서는 TTL, 캐시 무효화, 캐시 크기 등과 같은 다른 요소를 고려해야 하지만 여기서는 이 논의의 범위를 벗어납니다. 여기서는 주요 개념을 설명하는 기본적인 예제를 제공하겠습니다.

{% code title="cache.interceptor.ts" %}
```typescript

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable, of } from 'rxjs';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const isCached = true;
    if (isCached) {
      return of([]);
    }
    return next.handle();
  }
}
```
{% endcode %}

`CacheInterceptor`에는 하드코딩된  변수와 하드코딩된 응답 `[]`이 있습니다. 여기서 주목해야 할 핵심 사항은 `RxJS의()` 연산자에 의해 생성된 새 스트림을 반환해서 route handler가 **전혀 호출되지 않는다**는 점입니다. 누군가 `CacheInterceptor`를 사용하는 엔드포인트를 호출하면 응답(하드코딩된 빈 배열)이 즉시 반환됩니다. 넓게 통용되는 솔루션을 만들려면 `Reflector`를 활용하고 custom 데코레이터를 만들 수 있습니다. `Reflector`는 [**가드**](undefined-5.md) 챕터에 잘 설명되어 있습니다.



### 더 많은 연산자

RxJS 연산자를 사용하여 스트림을 조작할 수 있기 때문에 많은 기능을 할 수 있습니다. 또 다른 일반적인 사용 사례를 살펴보겠습니다. 경로 요청에 대한 시간 초과(timeout)를 처리하고 싶다고 가정해 봅시다. 일정 시간이 지나도 엔드포인트가 아무 것도 반환하지 않으면 오류 응답으로 종료하고 싶다면. 아래와 같이 하면 됩니다.

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, RequestTimeoutException } from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(5000),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException());
        }
        return throwError(() => err);
      }),
    );
  };
};
```

5초가 지나면 요청 처리가 취소됩니다. `RequestTimeoutException` 예외가 발생하기 전에 custom 로직을 추가할 수도 있습니다. (예: 리소스 릴리스)



