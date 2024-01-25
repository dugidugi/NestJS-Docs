# 프로바이더

프로바이더는 Nest의 핵심 개념입니다. 서비스, 리포지토리, 팩토리, 헬퍼 등 많은 기본 Nest 클래스가 프로바이더로 취급될 수 있습니다. 프로바이더의 주요 개념은 **의존성으로 주입**될 수 있다는 것입니다. 즉, 객체들이 서로 다양한 관계를 생성할 수 있으며, 이러한 객체를 "연결, 관리"하는 기능은 대부분 Nest 런타임 시스템에 위임할 수 있습니다.

<figure><img src="../.gitbook/assets/image (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

이전 장에서는 간단한 `CatsController`를 만들었습니다. 컨트롤러는 HTTP 요청을 처리하고 더 복잡한 작업을 프로바이더에 맡겨야 합니다. 프로바이더는 **module**에서 `providers`로 선언되는 일반 자바스크립트 클래스입니다.

> **힌트**
>
> Nest를 사용하면 의존성을 보다 객체지향적으로 설계하고 구성할 수 있으므로 [**SOLID**](https://en.wikipedia.org/wiki/SOLID) 원칙을 따를 것을 강력히 권장합니다.



### 서비스 Services&#x20;

간단한 `CatsService`를 만들어 보겠습니다. 이 서비스는 데이터 저장, 읽기를 담당하며 `CatsController`에서 사용하도록 설계되었습니다. provider로 사용하기에 적합합니다.

{% code title="cats.service.ts" %}
```typescript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
  private readonly cats: Cat[] = [];

  create(cat: Cat) {
    this.cats.push(cat);
  }

  findAll(): Cat[] {
    return this.cats;
  }
}  
```
{% endcode %}

> **힌트**
>
> CLI를 사용하여 서비스를 만들려면 `$ nest g service cats` 명령을 실행하세요.

`CatsService`는 하나의 프로퍼티와 두 개의 메서드가 있는 클래스입니다. 새로운 점은 `@Injectable()` 데코레이터를 사용한다는 것입니다. `@Injectable()` 데코레이터는 메타데이터를 첨부하는데, 이 메타데이터는 CatsService가 **Nest IoC 컨테이너**에서 관리할 수 있는 클래스임을 선언합니다.&#x20;

참고로 이 예시에서는 다음과 같이 보이는 `Cat` 인터페이스도 사용합니다:

{% code title="interfaces/cat.interface.ts" %}
```typescript
export interface Cat {
  name: string;
  age: number;
  breed: string;
}
```
{% endcode %}

이제 고양이를 읽어오는 서비스 클래스가 생겼으니 `CatsController` 내부에서 사용해 보겠습니다:

{% code title="cats.controller.ts" %}
```typescript
import { Controller, Get, Post, Body } from '@nestjs/common';
import { CreateCatDto } from './dto/create-cat.dto';
import { CatsService } from './cats.service';
import { Cat } from './interfaces/cat.interface';

@Controller('cats')
export class CatsController {
  constructor(private catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```
{% endcode %}

`CatsService`는 클래스 생성자를 통해 주입됩니다. `private` 구문이 사용된 것을 주목하세요. 이 구문을 사용하면 같은 위치에서 `catsService` 멤버를 즉시 선언하고 초기화할 수 있습니다.



### 의존성 주입&#x20;

Nest는 일반적으로 **의존성 주입**이라고 알려진 강력한 디자인 패턴을 기반으로 구축되었습니다. 이 개념에 대한 자세한 내용은 [**Angular**](https://angular.io/guide/dependency-injection) 공식 문서에서 읽어보시기 바랍니다.

Nest에서는 TypeScript 덕분에 의존성을 매우 쉽게 관리할 수 있습니다. 타입을 사용하기만 하면 되기 때문입니다. 아래 예제에서 Nest는 `CatsService`의 인스턴스를 생성하고 반환합니다.(싱글톤 원칙에 따라 이미 서비스가 요청된 경우 기존에 생성된 인스턴스를 반환합니다.) 이 의존성은 컨트롤러의 생성자에게 전달되거나 지정된 프로퍼티에 할당됩니다:

```typescript
constructor(private catsService: CatsService) {}
```



### Scopes&#x20;

프로바이더는 일반적으로 애플리케이션 수명 주기와 동일한 수명("scope")을 갖습니다. 애플리케이션이 부트스트랩되면 모든 의존성이 해결되어야 하므로 모든 프로바이더가 인스턴스화됩니다. 마찬가지로 애플리케이션이 종료되면 각 프로바이더 인스턴스가 삭제됩니다. 하지만 프로바이더의 수명을 **요청**에 따라 설정할 수 있는 방법도 있습니다. 이러한 기술에 대한 자세한 내용은 여기에서 확인할 수 있습니다.

