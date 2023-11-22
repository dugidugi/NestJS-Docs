# 파이프

\*validator는 번역하자면 유효성 검사(기)로 할 수 있으나, 실제 많이 쓰이는 용어이니 그대로 validator라고 남겨둡니다.

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

이러한 속성은 현재 처리된 인수를 설명합니다.

<table><thead><tr><th width="137"></th><th></th></tr></thead><tbody><tr><td><code>type</code></td><td>인수가 body @Body(), uqery @Query(), param @Param() 또는 사용자 정의 매개변수인지를 나타냅니다(자세한 내용은 여기를 참조하세요).</td></tr><tr><td><code>metatype</code></td><td>인수의 메타타입(예: <code>String</code>)을 제공합니다. 참고: 라우트 핸들러 메서드 시그니처에서 타입 선언을 생략하거나 바닐라 자바스크립트를 사용하는 경우 이 값은 <code>undefined</code> 입니다</td></tr><tr><td><code>data</code></td><td>데코레이터에 전달된 문자열입니다.(예: <code>@Body('string')</code>). 데코레이터 괄호를 비워두면 <code>undefined</code> 입니다.</td></tr></tbody></table>

> 주의\
> TypeScript 인터페이스는 코드 컴파일 중에 사라집니다. 따라서 메서드 매개 변수의 타입이 클래스 대신 인터페이스로 선언되면 `metatype` 값은 `Object`가 됩니다.



### 스키마 기반 유효성 검사

유효성 검사 파이프를 좀 더 유용하게 만들어 봅시다. 서비스 메서드를 실행하기 전에 포스트 body 객체가 유효한지 확인해야 하는 `CatsController`의 `create()` 메서드를 자세히 살펴봅시다.

```typescript
@Post()
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```

`CreateCatDto` body 매개변수를 집중적으로 살펴봅시다. 이 매개변수의 유형은 `CreateCatDto`입니다:

{% code title="create-cat.dto.ts" %}
```typescript
export class CreateCatDto {
  name: string;
  age: number;
  breed: string;
}
```
{% endcode %}

create 메서드로 들어오는 모든 요청에 유효한 body가 포함되어 있는지 확인하고자 합니다. 따라서 `createCatDto` 객체의 세 멤버의 유효성을 검사해야 합니다. 라우트 핸들러 메서드 내부에서 이 작업을 수행할 수 있지만 **단일 책임 원칙(SRP)**을 위반하므로 이상적이지 않습니다.

또 다른 접근 방식은 **유효성 검사 클래스**를 생성하고 작업을 위임하는 것입니다. 이 방법은 각 메서드의 시작 부분에서 이 **유효성 검사**를 호출하는 것을 기억해야 한다는 단점이 있습니다.

유효성 검사 미들웨어를 만드는 것은 어떨까요? 이 방법도 효과가 있을 수 있지만, 안타깝게도 전체 애플리케이션의 모든 컨텍스트에서 사용할 수 있는 **일반적인 미들웨어**를 만드는 것은 불가능합니다. 미들웨어는 호출될 핸들러와 그 매개변수를 포함한 **실행 컨텍스트**를 인식하지 못하기 때문입니다.

물론 이것이 바로 파이프를 사용하는 이유입니다. 이제 유효성 검사 파이프를 구체화해 보겠습니다.



### 객체 스키마 유효성 검사

깔끔하고 [**DRY**](https://en.wikipedia.org/wiki/Don't\_repeat\_yourself)한 방식으로 객체 유효성 검사를 수행하는 데 사용할 수 있는 몇 가지 접근 방식이 있습니다. 한 가지 일반적인 접근 방식은 **스키마 기반** 유효성 검사를 사용하는 것입니다. 이 접근 방식을 사용해 보겠습니다.

[**Zod**](https://zod.dev/) 라이브러리를 사용하면 읽기 쉬운 API를 사용하여 간단한 방식으로 스키마를 만들 수 있습니다. Zod 기반 스키마를 사용하는 유효성 검사 파이프를 구축해 보겠습니다.

먼저 필요한 패키지를 설치합니다:

```bash
$ npm install --save zod
```

아래 코드 샘플에서는 스키마를 생성자 인수로 받는 간단한 클래스를 만들었습니다. 그런 다음 제공된 스키마에 대해 들어오는 인수의 유효성을 검사하는 `schema.parse()` 메서드를 적용합니다.

앞서 언급했듯이 **유효성 검사 파이프**는 변경되지 않은 값을 반환하거나 예외를 던집니다.

다음 섹션에서는 `@UsePipes()` 데코레이터를 사용하여 주어진 컨트롤러 메서드에 적절한 스키마를 제공하는 방법을 살펴보겠습니다. 이렇게 하면 의도한 대로 여러 컨텍스트에서 유효성 검사 파이프를 재사용할 수 있습니다.

```typescript
import { PipeTransform, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { ZodObject } from 'zod';

export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodObject<any>) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      this.schema.parse(value);
    } catch (error) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }
}
```

### 유효성 검사 파이프 바인딩

앞서 변환 파이프를 바인딩하는 방법(예: `ParseIntPipe` 및 나머지 `Parse*` 파이프)을 살펴보았습니다.

유효성 검사 파이프를 바인딩하는 방법도 매우 간단합니다.

이 경우 메서드 호출 수준에서 파이프를 바인딩하고 싶습니다. 현재 예제에서는 `ZodValidationPipe`를 사용하려면 다음을 수행해야 합니다:

1. `ZodValidationPipe`의 인스턴스를 생성합니다.&#x20;
2. 파이프의 클래스 생성자에서 컨텍스트별 Zod 스키마를 전달합니다.&#x20;
3. 파이프를 메서드에 바인딩합니다.&#x20;

Zod 스키마 예제:

```typescript
import { z } from 'zod';

export const createCatSchema = z
  .object({
    name: z.string(),
    age: z.number(),
    breed: z.string(),
  })
  .required();

export type CreateCatDto = z.infer<typeof createCatSchema>;
```

아래 그림과 같이 `@UsePipes()` 데코레이터를 사용하여 이를 수행합니다:

{% code title="cats.controller.ts" %}
```typescript
@Post()
@UsePipes(new ZodValidationPipe(createCatSchema))
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```
{% endcode %}

> **힌트**
>
> `UsePipes()` 데코레이터는 @nestjs/common 패키지에서 가져옵니다.

> **주의**
>
> `zod` 라이브러리를 사용하려면 `tsconfig.json` 파일에서 `strictNullChecks`을 활성화해야 합니다.



### 클래스 validator

> **주의**
>
> 이 섹션의 기술은 TypeScript가 필요하며 바닐라 JavaScript를 사용하여 앱을 작성하는 경우에는 사용할 수 없습니다.&#x20;

유효성 검사 기법에 대한 대체 구현 방법을 살펴보겠습니다.

Nest는 클래스 [**class-validator**](https://github.com/typestack/class-validator) 라이브러리와 잘 작동합니다. 이 강력한 라이브러리를 사용하면 데코레이터 기반 유효성 검사를 사용할 수 있습니다. 데코레이터 기반 유효성 검사는 특히 처리된 프로퍼티의 `metatype`에 액세스할 수 있기 때문에 Nest의 파이프 기능과 결합할 때 매우 강력합니다. 시작하기 전에 필요한 패키지를 설치해야 합니다:

```bash
$ npm i --save class-validator class-transformer
```

설치가 완료되면 `CreateCatDto` 클래스에 몇 가지 데코레이터를 추가할 수 있습니다. 이 기법의 중요한 장점은 (별도의 유효성 검사 클래스를 만들지 않고도) `CreateCatDto` 클래스가 Post body 객체에 대한 단일 소스로 유지된다는 점입니다.

{% code title="create-cat.dto.ts" %}
```typescript
import { IsString, IsInt } from 'class-validator';

export class CreateCatDto {
  @IsString()
  name: string;

  @IsInt()
  age: number;

  @IsString()
  breed: string;
}
```
{% endcode %}

> **힌트**
>
> class-validator 데코레이터에 대해서는 [**이곳에서**](https://github.com/typestack/class-validator#usage) 더 읽어보세요.

이제 이러한 어노테이션을 사용하는 ValidationPipe 클래스를 만들 수 있습니다.

{% code title="validation.pipe.ts" %}
```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    const object = plainToInstance(metatype, value);
    const errors = await validate(object);
    if (errors.length > 0) {
      throw new BadRequestException('Validation failed');
    }
    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```
{% endcode %}

> **힌트**
>
> 다시 한 번 말씀드리자면, `ValidationPipe`는 Nest에서 기본으로 제공되므로 일반적인 validation 파이프는 직접 만들 필요가 없습니다. 기본 제공 `ValidationPipe`는 이 장에서 만든 샘플보다 더 많은 옵션을 제공하지만, 사용자 정의 파이프의 메커니즘을 설명하기 위해 기본으로 유지했습니다. 자세한 내용은 여기에서 많은 예제와 함께 확인할 수 있습니다.&#x20;

> **참고** \
> 위의 [**class-transformer**](https://github.com/typestack/class-transformer) 라이브러리는 **class-validator** 라이브러리와 같은 사람이 만든 라이브러리로, 결과적으로 두 라이브러리가 매우 잘 어울립니다.





