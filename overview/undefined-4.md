# 파이프

파이프는 `PipeTransform` 인터페이스를 사용하는 `@Injectable()` 데코레이터가 달린 클래스입니다.

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

파이프에는 두 가지 일반적인 사용 사례가 있습니다:

* **변환**: 입력 데이터를 원하는 형식으로 변환(예: 문자열에서 정수로)&#x20;
* **유효성 검사**: 입력 데이터를 평가하여 유효하면 변경하지 않고 그대로 전달하고, 그렇지 않으면 예외를 던집니다.&#x20;

두 경우 모두 파이프는 [**컨트롤러 라우트 핸들러**](controllers.md)가 처리 중인 `arguments`에 대해 작동합니다. Nest는 메서드가 호출되기 직전에 파이프를 삽입하고, 파이프는 메서드에 전달되는 인수를 받아 해당 인수를 대상으로 작동합니다. 모든 변환, 유효성 검사 작업이 이때 수행되고, 그 후에 라우트 핸들러가 (잠재적으로) 변환된 인수를 사용하여 호출됩니다.

Nest에는 바로 사용할 수 있는 여러 가지 기본 제공 파이프가 있습니다. 직접 사용자 정의 파이프를 만들 수도 있습니다. 이 장에서는 기본 제공 파이프를 소개하고 라우트 핸들러에 바인딩하는 방법에 대해 알려드리겠습니다. 그런 다음 몇 가지 사용자 정의 파이프를 살펴보고 처음부터 파이프를 만드는 방법에 대해서도 알아보겠습니다.

> 힌트
>
> 파이프는 예외 영역 내에서 실행됩니다. 즉, 파이프가 예외를 던지면 예외 레이어(전역 예외 필터 및 현재 컨텍스트에 적용되는 모든 [**예외 필터)**](undefined-3.md)에서 처리됩니다. 위의 내용을 고려할 때, 파이프에서 예외가 발생하면 컨트롤러 메서드가 이후에 실행되지 않는다는 것을 분명히 알 수 있습니다. 이는 시스템 경계에서 외부 소스에서 애플리케이션으로 들어오는 데이터의 유효성을 검사하는 모범 사례 기법을 제공합니다.



### 기본 제공 파이프

Nest는 바로 사용할 수 있는 9가지 파이프를 제공합니다.

* `ValidationPipe`
* `ParseIntPipe`
* `ParseFloatPipe`
* `ParseBoolPipe`
* `ParseArrayPipe`
* `ParseUUIDPipe`
* `ParseEnumPipe`
* `DefaultValuePipe`
* `ParseFilePipe`

이 파이프는 @nestjs/common 패키지에서 임포트됩니다.

`ParseIntPipe`를 사용하는 방법에 대해 간단히 살펴보겠습니다. 이 파이프는 **변환** 역할을 합니다. 메서드 핸들러 매개변수가 JavaScript 정수로 변환되도록 합니다. (변환에 실패할 경우 예외를 던집니다.) 이 장의 뒷부분에서는 `ParseIntPipe`에 대한 간단한 사용자 정의 방법에 대해 알아봅니다. 아래의 예제 기법은 다른 기본 제공 변환 파이프에도 적용됩니다. (`ParseBoolPipe`, `ParseFloatPipe`, `ParseEnumPipe`, `ParseArrayPipe`, `ParseUUIDPipe`. 또한 앞으로 이번장에서는 이런 파이프를  `Parse*` 파이프라고 부르겠습니다)



### 파이프 바인딩하기

파이프를 사용하려면 파이프 클래스의 인스턴스를 적절한 컨텍스트에 바인딩해야 합니다. `ParseIntPipe` 예제에서는 파이프를 특정 라우트 핸들러 메서드와 연결하고 메서드가 호출되기 전에 파이프가 실행되도록 하려고 합니다. 이를 위해 메서드 매개변수 수준에서 파이프를 바인딩하는 다음 구문을 사용합니다:

```typescript
@Get(':id')
async findOne(@Param('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

이렇게 하면 다음 두 가지 조건 중 하나가 참인지 확인합니다.\
`findOne()` 메서드에서 받은 매개 변수가 숫자인지(`this.catsService.findOne()` 호출에서 예상한 대로),\
라우트 핸들러가 호출되기 전에 예외가 발생하는지.

예를 들어 경로가 다음과 같이 호출된다고 가정해 보겠습니다:

```bash
GET localhost:3000/abc
```

Nest는 다음과 같은 예외를 던질 것입니다.

```json
{
  "statusCode": 400,
  "message": "Validation failed (numeric string is expected)",
  "error": "Bad Request"
}
```

예외는 `findOne()` 메서드의 본문이 실행되지 않도록 처리합니다.

위의 예제에서는 인스턴스가 아닌 클래스(`ParseIntPipe`)를 전달하여 인스턴스화에 대한 책임을 프레임워크에 맡기고 의존성 주입을 가능하게 합니다. 파이프 및 가드와 마찬가지로, 대신 인스턴스를 전달할 수 있습니다. 인스턴스를 전달하면 옵션을 전달할 수 있어서 내장된 파이프의 동작을 커스터마이즈 하려고 할 때 유용합니다:

```typescript
@Get(':id')
async findOne(
  @Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE }))
  id: number,
) {
  return this.catsService.findOne(id);
}
```

다른 변환 파이프(모든 **Parse\*** 파이프)를 바인딩하는 것도 비슷하게 작동합니다. 이러한 파이프는 모두 경로 매개변수, 쿼리 문자열 매개변수 및 request body 값의 유효성을 검사하는 컨텍스트에서 작동합니다.

예를 들어 쿼리 문자열 매개변수가 있습니다:

```typescript
@Get()
async findOne(@Query('id', ParseIntPipe) id: number) {
  return this.catsService.findOne(id);
}
```

다음은 문자열 매개변수를 구문 분석하고 해당 매개변수가 UUID인지 확인하는 `ParseUUIDPipe`의 예제입니다.

```typescript
@Get(':uuid')
async findOne(@Param('uuid', new ParseUUIDPipe()) uuid: string) {
  return this.catsService.findOne(uuid);
}
```

> **힌트**
>
> `ParseUUIDPipe()`를 사용할 때 버전 3, 4 또는 5의 UUID를 구문 분석하는 경우 특정 버전의 UUID만 필요한 경우 파이프 옵션에 버전을 전달할 수 있습니다.

위에서 다양한 `Parse*` 내장 파이프 제품군을 바인딩하는 예제를 살펴봤습니다. 유효성 검사 파이프를 바인딩하는 것은 조금 다르므로 다음 섹션에서 이에 대해 설명하겠습니다.

> **힌트**
>
> 유효성 검사 파이프의 광범위한 예제는 **유효성 검사 테크닉**을 참조하세요.



### 사용자 정의 파이프

앞서 언급했듯이 자신만의 사용자 정의 파이프를 만들 수 있습니다. Nest는 강력한 기본 제공 `ParseIntPipe` 및 `ValidationPipe`를 제공하지만, 사용자 정의 파이프가 어떻게 구성되는지 알아보기 위해 각각의 간단한 사용자 정의 버전을 처음부터 만들어 보겠습니다.

간단한 `ValidationPipe`부터 시작하겠습니다. 처음에는 단순히 입력값을 받고 즉시 동일한 값을 반환하여 항등 함수처럼 동작하도록 하겠습니다.

{% code title="validation.pipe.ts" %}
```typescript
import { PipeTransform, Injectable, ArgumentMetadata } from '@nestjs/common';

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    return value;
  }
}
```
{% endcode %}

> **힌트**
>
> `PipeTransform<T, R>`은 모든 파이프에서 적용해야하는 하는 제네릭 인터페이스입니다. 제레닉 인터페이스는 `T`를 사용하여 입력 값의 타입을 나타내고 `R`을 사용하여 `transform()` 메서드의 반환 타입을 명시합니다.

모든 파이프는 `PipeTransform` 인터페이스 조건을 이행하기 위해 `transform()` 메서드를 구현해야 합니다. 이 메서드에는 두 개의 매개변수가 있습니다:

* `value`
* `metadata`&#x20;

`value` 매개변수는 현재 처리된 메서드 인자(경로 처리 메서드에서 수신하기 전)이고 `metadata`는 현재 처리된 메서드 인자의 메타데이터입니다. 메타데이터 객체에는 이러한 속성이 있습니다:

```typescript
export interface ArgumentMetadata {
  type: 'body' | 'query' | 'param' | 'custom';
  metatype?: Type<unknown>;
  data?: string;
}
```







