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

