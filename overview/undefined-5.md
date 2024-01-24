# 가드

가드는 `@Injectable()` 데코레이터가 달린 클래스로, `CanActivate` 인터페이스를 사용할 수 있습니다.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

가드는 한 가지 역할만을 담당합니다. 가드는 런타임에 존재하는 특정 조건(예: 권한, 역할,  ACL 등)에 따라 특정 요청이 라우트 핸들러에 의해 처리될지 말지 결정합니다. 이를 흔히 인가(authorization)이라고 합니다. 인가는 기존 Express 애플리케이션에서는 보통 [미들웨어](undefined-2.md)에서 처리되었습니다. 미들웨어는 인증(authentication)에는 적합한 선택입니다. 토큰 유효성 검사, request 객체에 프로퍼티를 첨부하는 것과 같은 작업은 특정 경로 컨텍스트(및 해당 메타데이터)와 밀접하게 연결되어 있지 않기 때문입니다.

하지만 미들웨어는 본질적으로 멍청합니다. 미들웨어는 `next()` 함수 호출 후 어떤 핸들러가 실행될지 모릅니다. 반면에 가드(Guards)는 `ExecutionContext` 인스턴스에 접근할 수 있어서 다음에 무엇이 실행될지 정확하게 알 수 있습니다. 가드는 예외 필터, 파이프, 인터셉터처럼 요청/응답 사이클의 정확한 지점에 로직을 삽입할 수 있도록 설계되었습니다. 때문에 코드를 DRY하고, 선언적으로 유지하는데 도움이 됩니다.

> **힌트**\
> 가드(Guards)는 모든 **미들웨어 이후**에 실행되고, **인터셉터나 파이프보다 먼저 실행**됩니다.



### 인가 가드 (Authorization guard)

앞서 언급했듯이 인가(authorization)는 가드를 사용하기 적절한 사례입니다. 특정 경로는 충분한 권한이 있는 호출자(인증된 특정 사용자)에게만 열려있어야 하기 때문입니다. 아래에 우리가 만들 `AuthGuard`는 인증된 사용자가 있다고 가정합니다. (인증된 사용자이므로 토큰이 요청 헤더에 첨부되어 있다고 가정합니다.) 가드는 토큰을 추출하고 토큰의 유효성을 검증합니다. 추출된 정보를 활용하여 요청이 진행될지 말지를 결정합니다.

{% code title="auth.guard.ts" %}
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}
```
{% endcode %}

> **힌트**
>
> 애플리케이션에서 인증 메커니즘을 구현하는 방법에 대한 실제 예제를 찾고 있다면 [**이 챕터**](https://docs.nestjs.com/security/authentication)를 참조하세요. 마찬가지로 보다 복잡한 인증 예제를 보려면 [**이 페이지**](https://docs.nestjs.com/security/authorization)를 확인하세요.

`validateRequest()` 함수 내부 로직은 필요에 따라 간단하거나 복잡할 수 있습니다. 이 예제의 초첨은 가드가 요청/응답 사이클에서 어떻게 구성되는지 보여주는데 있습니다.

모든 가드는 `canActivate()` 함수를 implement해야합니다. 이 함수는 현재 요청이 허용되는지 여부를 나타내는 boolean을 반환해야합니다. 이 함수는 응답을 동기식 또는 비동기식(`Promise` 혹은 `Oberservable`)으로 반환할 수 있습니다. Nest는 반환값을 사용하여 다음 작업을 제어합니다.

* `true` : 요청이 계속 처리됩니다
* `false` : Nest는 요청을 거부합니다.



### 실행 컨텍스트 Execution context

`canActivate()` 함수는 단일 인수로 `ExecutionContext` 인스턴스를 받습니다. `ExecutionContext`는 `ArgumentsHost`를 상속합니다. 앞서 예외 필터 챕터에서 `ArgumentsHost`를 살펴봤습니다. 위의 샘플에서는 이전에 사용한 것과 동일한 `ArgumentsHost`에 정의된 핼퍼 메서드를 사용해서 `Request` 객체에 대한 참조를 가져옵니다. 이 주제에 대한 자세한 애용은 [**예외 필터**](undefined-3.md) 챕터의 Argument host 섹션을 참고하세요.&#x20;

`ArgumentsHost`를 extend함으로써, `ExecutionContext`는 현재 실행 프로세스에 대한 추가 세부 정보를 제공하는 새로운 헬퍼 메서드들도 추가합니다. 이러한 세부 정보는 컨트롤러, 메서드, 실행 컨텍스트에서 다양하게 작동할 수 있는 가드를 만드는데 유용합니다. `ExecutionContext`가 궁금하다면 [**이곳**](https://docs.nestjs.com/fundamentals/execution-context)을 참고하세요.



### 역할 기반 인증 (Role-based authentication)

특정 역할을 가진 유저에게만 액세스를 허용하는 보다 기능적인 가드를 만들어봅시다. 일반적인 가드 템플릿으로 만들어 보고 다음 섹션에서 이를 기반으로 가드를 만들어 보겠습니다. 일단 지금은 모든 요청이 진행되도록 허용합니다.

{% code title="roles.guard.ts" %}
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RolesGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    return true;
  }
}
```
{% endcode %}



### **가드 연결하기**

파이프나 예외 필터처럼 가드는 컨트롤러 범위, 메소드 범위, 전역 범위로 설정될 수 있습니다. 아래에서는 `@UseGuards` 데코레이터를 이용하여 컨트롤러 범위 가드를 만듭니다. 이 데코레이터는 단일 인수를 받거나, 쉼표로 구분된 인수 리스트를 받을 수 있습니다. 이를 통해 하나의 선언으로 적절한 가드들을 쉽게 적용할 수 있습니다.

```typescript
@Controller('cats')
@UseGuards(RolesGuard)
export class CatsController {}
```

> **힌트**\
> `@UseGuards()` 데코레이터는 `@nestjs/common` 패키지에서 임포트합니다.

위 코드에서는 인스턴스 대신 `RolesGuard` 클래스를 전달해서 인스턴스화에 대한 책임을 프레임워크에 맡기고 의존성을 주입했습니다. 파이프, 예외 필터와 마찬가지로 인스턴스를 전달할 수도 있습니다.&#x20;

```typescript
@Controller('cats')
@UseGuards(new RolesGuard())
export class CatsController {}
```

위 구조는 이 컨트롤러가 선언한 모든 핸들러에 가드를 붙입니다. 가드를 메서드 하나에만 적용하려면 메서드 레벨에서 `@UseGuards()`를 붙이면 됩니다.



글로벌 가드를 설정하려면 Nest 애플리케이션 인스턴스의 `useGlobalGuards()` 메서드를 사용합니다.

```typescript
const app = await NestFactory.create(AppModule);
app.useGlobalGuards(new RolesGuard());
```

> 주의
>
> 하이브리드 앱의 경우 `useGlobalGuards()` 메서드는 기본적으로 게이트웨이 및 마이크로 서비스에 대한 가드를 설정하지 않습니다. (이 동작을 변경하는 방법에 대한 자세한 내용은 [하이브리드 애플리케이션](https://docs.nestjs.com/faq/hybrid-application) 참조.) '표준(하이브리드가 아닌) 마이크로서비스 앱의 경우 `useGlobalGuards()`는 가드를 전역으로 적용합니다.

글로벌 가드는 전체 애플리케이션의 모든 컨트롤러와 모든 라우트 핸들러에 사용됩니다. 모듈 외부에서 등록한 글로벌 가드(`useGlobalGuards()`를 사용해서 등록한)는 모듈 외부 컨텍스트 외부에서 수행되므로 의존성을 주입할 수 없습니다. 이 문제를 해결하기 위해 다음 구성을 사용하여 모든 모듈에서 직접 가드를 설정할 수 있습니다.&#x20;

{% code title="app.module.ts" %}
```typescript
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_GUARD,
      useClass: RolesGuard,
    },
  ],
})
export class AppModule {}
```
{% endcode %}

> **힌트**
>
> 이 접근 방식을 사용하여 가드에 대한 의존성을 주입할 때, 이 구조가 적용되는 모듈에 관계 없이 가드는 실제로는 전역이라는 점에 유의하세요. 이 작업은 어디서 수행되어야 할까요? 가드가 정의된 모듈(위 예시에서는 `RolesGuard`)을 선택합니다. 또한 `useClass는` custom provider를 등록하기 위한 유일한 방법이 아닐 수 있습니다. [**여기**](https://docs.nestjs.com/fundamentals/custom-providers)에서 알아보세요.&#x20;



### handler에 역할 설정하기

`RolesGuard`가 작동하고 있지만 아직은 그다지 똑똑하지 않습니다. 가장 중요한 가드 기능인 [**실행 컨텍스트**](https://docs.nestjs.com/fundamentals/execution-context)를 아직 활용하지 못하고 있습니다. 아직 역할에 대해 알지 못하고, 각 핸들러에 어떤 역할이 허용될지 알지 못합니다. 예를 들어, `CatsController`는 경로에 따라 서로 다른 권한 체계를 가질 수 있습니다. 어떤 것은 관리자 사용자만 사용할 수 있고 어떤 것은 모든 사람에게 열려 있을 수 있습니다. 어떻게 하면 유연하고 재사용 가능한 방식으로 역할을 경로에 적용할 수 있을까요?

이곳이 [**custom metadata**](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata)가 필요한 영역입니다. Nest는 데코레이터를 통해 라우팅 핸들러에 custom metadata를 첨부하는 기능을 제공합니다. 데코레이터는 `Reflector#createDecorator` 정적 메서드, 또는 빌트인 `@Setmetadata()` 데코레이터로 생성될 수 있습니다.&#x20;

{% code title="roles.decorator.ts" %}
```ts
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```
{% endcode %}

여기서 `Roles` 데코레이터는 `string[]` 타입의 인수를 받는 함수입니다.

이제 이 데코레이터를 핸들러에 달기만 하면 됩니다:

{% code title="cats.controller.ts" %}
```typescript
@Post()
@Roles(['admin'])
async create(@Body() createCatDto: CreateCatDto) {
  this.catsService.create(createCatDto);
}
```
{% endcode %}

여기서는 `admin` 역할이 있는 사용자만 이 경로에 접근할 수 있게 `create()` 메서드에 `Roles` 데코레이터 메타데이터를 붙였습니다.&#x20;

`Reflector#createDecorator` 메서드를 사용하는 대신 기본 제공 `@SetMetadata()` 데코레이터를 사용할 수 있습니다. [**여기**](https://docs.nestjs.com/fundamentals/execution-context#low-level-approach)에서 자세히 알아보세요.



### 종합 정리하기

이제 돌아가서 이 모든 것을 `RolesGuard`와 연결해 보겠습니다. 지금까지는 모든 경우에 `true`를 반환하여 모든 요청이 진행되도록 허용합니다. 사용자에게 할당된 역할과 현재 처리 중인 경로에 필요한 실제 역할을 비교하여 조건에 따라 다른 값을 반환하게 하려고 합니다. 경로의 역할(custom metadata)에 액세스 하기 위해 다음과 같이 Reflector 헬퍼 클래스를 다시 사용하겠습니다.

{% code title="roles.guard.ts" %}
```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Roles } from './roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const roles = this.reflector.get(Roles, context.getHandler());
    if (!roles) {
      return true;
    }
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    return matchRoles(roles, user.roles);
  }
}
```
{% endcode %}

> **힌트**
>
> node.js 세계에서는 권한이 있는 사용자를 `request` 객체에 첨부하는 것이 관행입니다. 따라서 위의 샘플 코드에서는 `request.user`에 사용자 인스턴스와 허용된 역할이 포함되어 있다고 가정합니다. 앱에서는 custom 인증 가드(또는 미들웨어)에서 이러한 작업을 할 것입니다. 이 주제에 대한 자세한 내용은 [**이 챕터**](https://docs.nestjs.com/security/authentication)를 확인하세요.

> **주의**
>
> `matchRoles()` 함수 내부의 로직은 필요에 따라 간단하거나 복잡할 수 있습니다. 이 예제의 초점은 가드가 요청/응답 사이클에 어떻게 구성될 수 있는지 보여주는 것입니다.

context에 민감한 방식으로 `Reflector`를 활용하는 방법에 대한 자세한 내용은 실행 컨텍스트 챕터의 [**Reflection and metadata**](https://docs.nestjs.com/fundamentals/execution-context#reflection-and-metadata) 섹션을 참조하세요.

권한이 충분하지 않은 사용자가 엔드포인트를 요청하면 Nest는 자동으로 다음과 같은 응답을 반환합니다:

```typescript
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "error": "Forbidden"
}
```

가드가 `false`을 반환하면 프레임워크는 `ForbiddenException`을 던집니다. 다른 오류 응답을 반환하려면 직접 예외를 던져야 합니다. 예를 들어

```typescript
throw new UnauthorizedException();
```

가드가 던지는 모든 예외는 [**예외 계층**](https://docs.nestjs.com/exception-filters)(전역 예외 필터 및 현재 컨텍스트에 적용되는 모든 예외 필터)에서 처리합니다.

> **힌트**&#x20;
>
> 인증을 구현하는 방법에 대한 실제 사례를 찾고 있다면 [**이 챕터**](https://docs.nestjs.com/security/authorization)를 확인하세요.
