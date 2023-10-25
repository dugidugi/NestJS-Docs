# 첫걸음

이 글에서는 Nest의 핵심 기본 사항을 배웁니다. Nest 애플리케이션의 필수 구성 요소에 익숙해지기 위해 입문 수준에서 많은 부분을 다루는 기능을 갖춘 기본 CRUD 애플리케이션을 빌드해 보겠습니다.

### 언어

저희는 TypeScript를 사랑하지만 무엇보다도 Node.js를 사랑합니다. 그렇기 때문에 Nest는 TypeScript와 순수 JavaScript 모두와 호환됩니다. Nest는 최신 언어 기능을 활용하므로 바닐라 자바스크립트와 함께 사용하려면 바벨 컴파일러가 필요합니다.

저희가 제공하는 예제에서는 대부분 TypeScript를 사용하지만 코드를 언제든지 바닐라 JavaScript 구문으로 전환할 수 있습니다(각 코드 조각의 오른쪽 상단 모서리에 있는 언어 버튼을 클릭하여 전환하기만 하면 됩니다).

### 전제 조건

사용 중인 운영체제에 Node.js(버전 16 이상)가 설치되어 있는지 확인하세요.

### 설정

Nest CLI를 사용하면 새 프로젝트를 쉽게 시작할 수 있습니다. npm이 설치되어 있으면 OS 터미널에서 다음 명령어를 사용하여 새 Nest 프로젝트를 생성할 수 있습니다:

```typescript
$ npm i -g @nestjs/cli
$ nest new project-name
```

> 힌트 \
> TypeScript를 strict 설정으로 프로젝트를 만들려면 `nest new` 명령에 `--strict` 플래그를 추가하세요.

프로젝트 이름 디렉터리가 생성되고, 노드 모듈과 몇 가지  파일이 설치되며, src/ 디렉터리가 생성되어 몇 가지 핵심 파일로 채워집니다.

```
src
- app.controller.spec.ts
- app.controller.ts
- app.module.ts
- app.service.ts
- main.ts
```

핵심 파일에 대한 간략한 개요입니다:

| `app.controller.ts`      | 단일 경로를 가진 기본 컨트롤러입니다.                                             |
| ------------------------ | ----------------------------------------------------------------- |
| `app.controller.spec.ts` | 컨트롤러에 대한 유닛 테스트                                                   |
| `app.module.ts`          | 애플리케이션의 루트 모듈                                                     |
| `app.service.ts`         | 단일 메서드가 있는 기본 서비스                                                 |
| `main.ts`                | 애플리케이션 인스턴스를 생성하기 위해 핵심 함수 `NestFactory`를 사용하는 애플리케이션의 엔트리 파일입니다. |

main.ts에는 애플리케이션을 부트스트랩하는 비동기 함수가 포함되어 있습니다:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

Nest 애플리케이션 인스턴스를 생성하기 위해 `NestFactory` 클래스를 사용합니다. `NestFactory`는 애플리케이션 인스턴스를 생성할 수 있는 몇 가지 정적 메서드를 노출합니다. `create()` 메서드는 `INestApplication` 인터페이스를 충족하는 애플리케이션 객체를 반환합니다. 이 객체는 이후에 설명하는 메서드 집합을 제공합니다. 위의 `main.ts` 예시에서는 애플리케이션이 인바운드 HTTP 요청을 기다릴 수 있도록 HTTP 리스너를 시작하기만 하면 됩니다.

Nest CLI는 모듈을 전용 디렉토리에 보관하는 관례에 따라 프로젝트 구조를 생성합니다.

> 힌트\
> 기본적으로 애플리케이션을 생성하는 동안 오류가 발생하면 앱은 코드 1과 함께 종료됩니다. 오류를 발생시키려면 abortOnError 옵션을 비활성화하세요.\
> (예: `NestFactory.create(AppModule, { abortOnError: false })`)

### 플랫폼

Nest는 플랫폼에 구애받지 않는 프레임워크를 지향합니다. 플랫폼 독립성은 개발자가 여러 유형의 애플리케이션에서 활용할 수 있는 재사용 가능한 논리적 부분을 생성할 수 있게 해줍니다. 기술적으로 Nest는 어댑터를 생성하면 모든 Node HTTP 프레임워크에서 작동할 수 있습니다. 기본적으로 지원되는 HTTP 플랫폼은 `express`와 `fastify` 두 가지입니다. 필요에 따라 적합한 HTTP플랫폼을 선택 할 수 있습니다.

<table data-header-hidden><thead><tr><th width="125.5"></th><th></th></tr></thead><tbody><tr><td><code>platform-express</code></td><td>Express는 잘 알려진 노드용 미니멀리즘 웹 프레임워크입니다. 커뮤니티에서 구현한 많은 리소스가 포함되테 있습니 .트를 거친, 프로덕션에 바로 사용할 수 있는 라이브러리입니다. 기본적으로 @nestjs/platform-express 패키지가 사용됩니다. 많은 사용자가 Express를 잘 사용하고 있으며, 이를 활성화하기 위해 별도의 조치를 취할 필요가 없습니다.</td></tr><tr><td><code>platform-fastify</code></td><td><a href="https://www.fastify.io/">Fastify</a> is a high performance and low overhead framework highly focused on providing maximum efficiency and speed. Read how to use it <a href="https://docs.nestjs.com/techniques/performance">here</a>.</td></tr></tbody></table>



