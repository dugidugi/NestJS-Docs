# 예외 필터

Nest에는 애플리케이션 전체에서 처리되지 않은 모든 예외를 처리하는 **예외 계층(exceptions layer)**이 내장되어 있습니다. 예외가 애플리케이션 코드에서 처리되지 않으면 이 계층에서 예외를 포착하여 적절한 사용자 친화적인 응답을 자동으로 전송합니다.

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

기본적으로 이 작업은 내장된 전역 예외 필터에 의해 수행되며, 이 필터는 `HttpException` 유형(및 그 하위 클래스)의 예외를 처리합니다. 예외가 인식되지 않는 경우(`HttpException`도 아니고 `HttpException`을 상속하는 클래스도 아닌 경우) 기본 제공 예외 필터는 다음과 같은 기본 JSON 응답을 생성합니다:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> **힌트**
>
> 전역 예외 필터는 부분적으로 `http-errors` 라이브러리를 지원합니다. `statusCode` 및 `message` 프로퍼티를 포함했다면 예외가 적절하게 구성되어 응답으로 전송됩니다. (기본 `InternalServerErrorException` 대신에).



### 표준 예외 던지기

Nest는 `@nestjs/common` 패키지를 활용하여 빌트인 `HttpException` 클래스를 제공합니다. 일반적인 HTTP REST/GraphQL API 기반 애플리케이션의 경우, 특정 오류 조건이 발생하면 표준 HTTP 응답 객체를 전송하는 것이 가장 좋습니다.

예를 들어, `CatsController`에는 `findAll()` 메서드(`GET` 라우트 핸들러)가 있습니다. 이 라우트 핸들러가 어떤 이유로 예외를 던진다고 가정해 봅시다. 이를 보여주기 위해 다음과 같이 하드코딩 해보겠습니다:

{% code title="cats.controller.ts" %}
```typescript
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```
{% endcode %}

> **힌트**
>
> 여기서는 `HttpStatus`를 사용했습니다. 이것은 `@nestjs/common` 패키지에서 가져온 헬퍼 enum입니다.

사용자가 이 엔드포인트에 요청하면, 응답은 다음과 같습니다.

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

`HttpException` 생성자는 응답을 결정하는 두 개의 필수 인수를 받습니다:

* `reponse` 인수는 JSON response body를 정의합니다. 아래 설명된 대로 `string` 또는 `object`일 수 있습니다.&#x20;
* `status` 인수는 [HTTP 상태 코드](https://developer.mozilla.org/ko/docs/Web/HTTP/Status)를 정의합니다.&#x20;

기본적으로 JSON response body에는 두 가지 속성이 포함됩니다:

* `statusCode`: 기본적으로 `status` 인수에 제공된 HTTP 상태 코드입니다.&#x20;
* `message` : `status`에 따른 HTTP 오류에 대한 간단한 설명입니다.&#x20;

JSON response body의 message 부분만 바꾸려면 `response` 인수에 문자열을 입력합니다.&#x20;



전체 JSON response body를 재정의하려면 response 인수에 `obejct`를 전달합니다. Nest는 객체를 직렬화하여 JSON response body로 반환합니다.

두 번째 생성자 인자인 `status`는 유효한 HTTP 상태 코드여야 합니다. 가장 좋은 방법은 `@nestjs/common`에서 가져온 `HttpStatus` enum을 사용하는 것입니다.

(선택 사항)오류 원인을 제공하는 데 사용할 수 있는 세 번째 생성자 인자인 `options`가 있습니다. 이 `cause` 객체는 응답 객체로 직렬화되지는 않지만 로깅 목적으로 유용할 수 있으며, `HttpException`을 발생시킨 내부 오류에 대한 중요한 정보를 제공합니다.

다음은 전체 응답 본문을 재정의하고 오류 원인을 제공하는 예제입니다:

{% code title="cats.controller.ts" %}
```typescript
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) { 
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```
{% endcode %}

위의 코드에 따라, 응답은 다음과 같습니다.&#x20;

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```



### 사용자 정의 예외

대부분의 경우 사용자 정의 예외를 작성할 필요가 없으며, 다음 섹션에 설명된 대로 기본 제공 Nest HTTP 예외를 사용할 수 있습니다. 사용자 정의 예외를 작성해야 하는 경우, 사용자 정의 예외가 기본 `HttpException` 클래스를 상속하는 자체 **예외 계층**을 만드는 것이 좋습니다. 이 접근 방식을 사용하면 Nest가 예외를 인식하고 오류 응답을 자동으로 처리합니다. 이러한 사용자 정의 예외를 구현해 보겠습니다:

{% code title="forbidden.exception.ts" %}
```typescript
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```
{% endcode %}

`ForbiddenException`은 기본 `HttpException`을 extends하기 때문에 기본 제공 예외 처리기와 원활하게 작동하므로 `findAll()` 메서드 내에서 사용할 수 있습니다.

{% code title="cats.controller.ts" %}
```typescript
@Get()
async findAll() {
  throw new ForbiddenException();
}
```
{% endcode %}

### 기본 제공 HTTP 예외&#x20;

Nest는 기본 `HttpException`을 상속하는 표준 예외 목록을 제공합니다. 이러한 예외는 `@nestjs/common` 패키지에서 제공되며, 가장 일반적인 HTTP 예외 중 다수를 나타냅니다:

* `BadRequestException`
* `UnauthorizedException`
* `NotFoundException`
* `ForbiddenException`
* `NotAcceptableException`
* `RequestTimeoutException`
* `ConflictException`
* `GoneException`
* `HttpVersionNotSupportedException`
* `PayloadTooLargeException`
* `UnsupportedMediaTypeException`
* `UnprocessableEntityException`
* `InternalServerErrorException`
* `NotImplementedException`
* `ImATeapotException`
* `MethodNotAllowedException`
* `BadGatewayException`
* `ServiceUnavailableException`
* `GatewayTimeoutException`
* `PreconditionFailedException`

모든 기본 제공 예외는 `options` 매개 변수를 사용하여 오류 `cause`과 오류 설명을 모두 제공할 수도 있습니다:

```typescript
throw new BadRequestException('Something bad happened', { cause: new Error(), description: 'Some error description' })
```

위의 코드에 따라, 응답은 다음과 같습니다.&#x20;

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400,
}
```



### 예외 필터

기본 예외 필터가 많은 경우를 자동으로 처리할 수 있지만 예외 계층을 **완전히 제어**하고 싶을 수도 있습니다. 예를 들어 로깅을 추가하거나 동적 요인에 따라 다른 JSON 스키마를 사용하고 싶을 수 있습니다. 예외 필터는 바로 이러한 목적을 위해 설계되었습니다. 예외 필터를 사용하면 정확한 제어 흐름과 클라이언트에 다시 전송되는 응답의 내용을 제어할 수 있습니다.

`HttpException` 클래스의 인스턴스인 예외를 포착하고 이에 대한 사용자 정의 응답 로직을 구현하는 예외 필터를 만들어 보겠습니다. 이를 위해서는 플랫폼의 기본 `Request` 및 `Response` 객체에 액세스해야 합니다. `Request` 객체에 액세스하여 원본 `url`을 가져와 로깅 정보에 포함할 수 있습니다. `Response` 객체를 활용하여 `response.json()` 메서드로 응답을 직접 제어할 것입니다.

{% code title="http-exception.filter.ts" %}
```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```
{% endcode %}

> **힌트**
>
> 모든 예외 필터는 제네릭 `ExceptionFilter<T>` 인터페이스를 implement해야 합니다. 이를 위해서는 `catch(exception: T, host: ArgumentsHost)` 메서드에 지정된 시그니처를 제공해야 합니다. `T`는 예외의 유형을 나타냅니다.

> **주의**&#x20;
>
> `nestjs/platform-fastify`를 사용하는 경우 `response.json()` 대신 `response.send()`를 사용할 수 있습니다. `fastify`에서 올바른 타입을 임포트하는 것을 잊지 마세요.

`@Catch(HttpException)` 데코레이터는 필요한 메타데이터를 예외 필터에 바인딩하여 이 특정 필터가 `HttpException` 유형의 예외만 찾고 있음을 Nest에 알려줍니다. `@Catch()` 데코레이터는 단일 매개변수 또는 쉼표로 구분된 목록을 받을 수 있습니다. 이를 통해 한 번에 여러 유형의 예외에 대한 필터를 설정할 수 있습니다.



**Arguments host**

`catch()` 메서드의 매개변수를 살펴봅시다. `exception` 매개변수는 현재 처리 중인 예외 객체입니다. `host` 매개변수는 `ArgumentsHost` 객체입니다. `ArgumentsHost`는 **실행 컨텍스트 챕터**\*에서 자세히 살펴볼 강력한 유틸리티 객체입니다. 이 코드 샘플에서는 이 객체를 사용하여 원래 요청 처리기(예외가 발생한 컨트롤러에서)로 전달되는 `Request` 및 `Response` 객체에 대한 참조를 얻습니다. 이 코드 샘플에서는 `ArgumentsHost`의 몇 가지 헬퍼 메서드를 사용하여 원하는 `Request` 및 `Response` 객체를 가져왔습니다. `ArgumentsHost`에 대해서는 이곳에서 자세히 알아보세요.

\*이렇게 추상화 수준을 높은 이유는 `ArgumentsHost`가 모든 컨텍스트(예: 지금 작업 중인 HTTP 서버 컨텍스트뿐만 아니라 마이크로서비스와 웹소켓 등)에서 작동하기 때문입니다. 실행 컨텍스트 챕터에서는 `ArgumentsHost`와 헬퍼 함수를 이용해 모든 실행 컨텍스트에 적합한 기본 인자에 접근하는 방법을 살펴볼 것입니다. 이를 통해 모든 컨텍스트에서 작동하는 일반 예외 필터를 작성할 수 있습니다.



### 바인딩 필터

새로운 `HttpExceptionFilter`를 `CatsController`의 `create()` 메서드에 연결해 보겠습니다.

{% code title="cats.controller.ts" %}
```typescript
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```
{% endcode %}

> 힌트
>
> `UseFilters()` 데코레이터는 `@nestjs/common` 패키지에서 가져옵니다.

여기서는 `@UseFilters()` 데코레이터를 사용했습니다. `@Catch()` 데코레이터와 마찬가지로 단일 필터 인스턴스 또는 쉼표로 구분된 필터 인스턴스 목록을 받을 수 있습니다. 여기서는 `HttpExceptionFilter` 인스턴스를 생성했습니다. 또는 인스턴스 대신 클래스를 전달하여 인스턴스화에 대한 책임을 프레임워크에 맡기고 **의존성 주입**을 활성화할 수 있습니다.

{% code title="cats.controller.ts" %}
```typescript
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```
{% endcode %}

> **힌트**
>
> 가능하면 인스턴스 대신 클래스를 사용하여 필터를 적용하는 것이 좋습니다. Nest는 전체 모듈에서 동일한 클래스의 인스턴스를 쉽게 재사용할 수 있으므로 **메모리 사용량**을 줄일 수 있습니다.

위의 예제에서 `HttpExceptionFilter`는 단일 `create()` 라우트 핸들러에만 적용되어 메소드에 범위가 한정되어 있습니다. 예외 필터는 다양한 수준에서 범위가 지정 수 있습니다. 컨트롤러/리졸버/게이트웨이의 메서드 범위, 컨트롤러 범위 또는 전역 범위 등. \
예를 들어 컨트롤러 범위로 필터를 설정하려면 다음과 같이 하면 됩니다:

{% code title="cats.controller.ts" %}
```typescript
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```
{% endcode %}

이 구조는 `CatsController` 내부에 정의된 모든 라우트 핸들러에 대해 `HttpExceptionFilter`를 설정합니다.

전역 범위 필터를 만들려면 다음을 수행합니다:

{% code title="main.ts" %}
```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```
{% endcode %}

> **주의**
>
> `useGlobalFilters()` 메서드는 게이트웨이 또는 하이브리드 애플리케이션에 대한 필터를 설정하지 않습니다.

전역 범위 필터는 모든 컨트롤러와 모든 라우트 핸들러에 대해 전체 애플리케이션에서 사용됩니다. 의존성 주입과 관련하여 모듈 외부에서 등록한 전역 필터(위 예제에서의 `useGlobalFilters()`)는 모듈의 컨텍스트 외부에서 수행되므로 의존성을 주입할 수 없습니다. 이 문제를 해결하기 위해 다음 구성을 사용하여 모든 **모듈에서 직접 전역 범위 필터를 등록**할 수 있습니다:

{% code title="app.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```
{% endcode %}

> **힌트**&#x20;
>
> 이 접근 방식을 사용하여 필터에 대한 의존성 주입을 수행할 때는 이 구조가 사용되는 모듈에 관계없이 필터가 실제로는 전역이라는 점에 유의하세요. 이 작업을 어디에서 수행해야 할까요? 필터가 정의된 모듈(위 예제에서는 `HttpExceptionFilter`)을 선택하면 됩니다. 또한 `useClass`가 사용자 정의 프로바이더를 등록하는 유일한 방법은 아닙니다. 여기에서 자세히 알아보세요.

이 기법을 사용하여 필요한 만큼 필터를 추가할 수 있으며, 각 필터를 providers 배열에 추가하기만 하면 됩니다.

### 모든 예외 잡기

처리되지 않은 모든 예외를 잡으려면(예외 유형에 관계없이) `@Catch()` 데코레이터의 매개 변수 목록을 비워두세요(예: `@Catch()`).

아래 예시 코드는 플랫폼에 구애받지 않습니다. HTTP 어댑터를 사용하여 응답을 전달하고 플랫폼의 객체(`Request` 및 `Response`)를 직접 사용하지 않기 때문이죠.

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    // In certain situations `httpAdapter` might not be available in the
    // constructor method, thus we should resolve it here.
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> **주의**\
> 모든 것을 캐치하는 예외 필터와 특정 유형에 바인딩된 필터를 결합하는 경우, 특정 필터가 바인딩된 유형을 올바르게 처리할 수 있도록 '무엇이든 캐치하는 필터'를 먼저 선언해야 합니다.



### 상속

일반적으로 애플리케이션 요구 사항을 충족하기 위해 완전히 사용자 정의된 예외 필터를 만듭니다. 그러나 기본 제공되는 기본 전역 예외 필터를 간단히 확장하고 특정 요인에 따라 동작을 재정의하려는 사용 사례가 있을 수 있습니다.

예외 처리를 기본 필터에 위임하려면 `BaseExceptionFilter`를 확장하고 상속된 `catch()` 메서드를 호출해야 합니다.

{% code title="all-exceptions.filter.ts" %}
```typescript
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```
{% endcode %}

> **주의**\
> `BaseExceptionFilter`를 extend하는 메서드 범위 및 컨트롤러 범위의 필터는 `new`로 인스턴스화해서는 안 됩니다. 대신 프레임워크가 자동으로 인스턴스화하도록 하세요.&#x20;

위의 구현은 접근 방식을 보여주는 셸일 뿐입니다. 확장 예외 필터의 구현에는 맞춤형 비즈니스 로직(예: 다양한 조건 처리)이 포함될 수 있습니다.

전역 필터는 기본 필터를 extend할 수 있습니다. 이 작업은 두 가지 방법 중 하나로 수행할 수 있습니다.

첫 번째 방법은 사용자 정의 전역 필터를 인스턴스화할 때 `HttpAdapter` 참조를 삽입하는 것입니다:

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

두 번째 방법은 [**여기에**](undefined-3.md#undefined-3) 표시된 것처럼 `APP_FILTER` 토큰을 사용하는 것입니다.



