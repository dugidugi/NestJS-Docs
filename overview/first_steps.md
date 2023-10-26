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

프로젝트 이름 디렉터리가 생성되고, 노드 모듈과 몇 가지 파일이 설치되며, src/ 디렉터리가 생성되어 몇 가지 핵심 파일로 채워집니다.

```
src
- app.controller.spec.ts
- app.controller.ts
- app.module.ts
- app.service.ts
- main.ts
```

핵심 파일에 대한 간략한 개요입니다:

| `app.controller.ts`      | 단일 경로를 가진 기본 컨트롤러입니다.                                                                      |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `app.controller.spec.ts` | 컨트롤러에 대한 유닛 테스트                                                                                |
| `app.module.ts`          | 애플리케이션의 루트 모듈                                                                                   |
| `app.service.ts`         | 단일 메서드가 있는 기본 서비스                                                                             |
| `main.ts`                | 애플리케이션 인스턴스를 생성하기 위해 핵심 함수 `NestFactory`를 사용하는 애플리케이션의 엔트리 파일입니다. |

main.ts에는 애플리케이션을 부트스트랩하는 비동기 함수가 포함되어 있습니다:

```typescript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

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

###

### 플랫폼

Nest는 플랫폼에 구애받지 않는 프레임워크를 지향합니다. 플랫폼 독립성은 개발자가 여러 유형의 애플리케이션에서 활용할 수 있는 재사용 가능한 논리적 부분을 생성할 수 있게 해줍니다. 기술적으로 Nest는 어댑터를 생성하면 모든 Node HTTP 프레임워크에서 작동할 수 있습니다. 기본적으로 지원되는 HTTP 플랫폼은 `express`와 `fastify` 두 가지입니다. 필요에 따라 적합한 HTTP플랫폼을 선택 할 수 있습니다.

<table data-header-hidden><thead><tr><th width="164.5"></th><th></th></tr></thead><tbody><tr><td><code>platform-express</code></td><td>Express는 잘 알려진 노드용 미니멀리즘 웹 프레임워크입니다. 커뮤니티에서 구현한 많은 리소스가 포함되테 있고 실전 테스트를 거친 프로덕션에 바로 사용할 수 있는 라이브러리입니다. 기본적으로 <code>@nestjs/platform-express</code> 패키지가 사용됩니다. 많은 사용자가 Express를 잘 사용하고 있으며, 이를 활성화하기 위해 별도의 조치를 취할 필요가 없습니다.</td></tr><tr><td><code>platform-fastify</code></td><td>Fastify는 최대한의 효율성과 속도를 제공하는 데 중점을 둔 고성능의 낮은 오버헤드 프레임워크입니다. 이곳에서 사용 방법을 알아보세요.</td></tr></tbody></table>

어떤 플랫폼을 사용하든 자체 애플리케이션 인터페이스를 노출합니다. 이는 각각 `NestExpressApplication`과 `NestFastifyApplication`으로 표시됩니다.

아래 예시처럼 `NestFactory.create()` 메서드에 유형을 전달하면 앱 객체에는 해당 특정 플랫폼에서만 사용할 수 있는 메서드가 있습니다. 그러나 실제로 기본 플랫폼 API에 액세스하려는 경우가 아니라면 유형을 지정할 필요가 없습니다.

```typescript
const app = await NestFactory.create<NestExpressApplication>(AppModule);
```

###

### 애플리케이션 실행

설치 프로세스가 완료되면 OS 명령 프롬프트에서 다음 명령을 실행하여 인바운드 HTTP 요청을 수신하는 애플리케이션을 시작할 수 있습니다:

```
$ npm run start
```

> 힌트\
> 개발 프로세스 속도를 높이려면(빌드 속도가 20배 빨라짐) 다음과 같이 시작 스크립트에 -b swc 플래그를 전달하여 SWC 빌더를 사용할 수 있습니다.

이 명령은 `src/main.ts` 파일에 정의된 포트에서 수신 대기 중인 HTTP 서버로 앱을 시작합니다. 애플리케이션이 실행되면 브라우저를 열고 `http://localhost:3000/` 로 이동합니다. `Hello World!` 메시지가 표시됩니다.

파일의 변경 사항을 확인하려면 다음 명령을 실행하여 애플리케이션을 시작할 수 있습니다:

```
$ npm run start:dev
```

이 명령은 파일을 감시하여 서버를 자동으로 다시 컴파일하고 다시 로드합니다.

### 린팅 및 포맷팅

CLI는 대규모로 안정적인 개발 워크플로를 구축하기 위한 최선의 노력을 제공합니다. 따라서 생성된 Nest 프로젝트에는 코드 린터와 포매터가 모두 사전 설치되어 있습니다(각각 eslint와 prettier).

> 힌트\
> 포맷터와 린터의 역할에 대해 잘 모르시나요? [여기에서](https://prettier.io/docs/en/comparison.html) 차이점을 알아보세요.

안정성과 확장성을 극대화하기 위해 기본 `eslint`와 `prettier` cli 패키지를 사용합니다. 이 설정은 깔끔한 IDE 통합을 가능하게 합니다.

IDE가 적합하지 않은 헤드리스 환경(지속적 통합, Git hook 등)의 경우 Nest 프로젝트에는 바로 사용할 수 있는 `npm` 스크립트가 함께 제공됩니다.

```
# Lint and autofix with eslint
$ npm run lint

# Format with prettier
$ npm run format
```
