# TMDOG的微服务之路_07——初入微服务，NestJS微服务快速入门

## 博客地址：[TMDOG的博客](https://blog.tmdog114514.icu)

在前几篇博客中，我们探讨了如何在 NestJS 中的一些基础功能，并可以使用NestJS实现一个简单的单体架构后端应用。本篇博客，我们将进入微服务架构，以一个简单的NestJS示例快速了解微服务架构。

## 1. 什么是微服务？

微服务架构是一种软件开发方法，将应用程序划分为多个独立的小服务，每个服务都执行特定的业务功能。这些服务可以独立部署、更新和扩展，且通常通过轻量级的通信机制（如 HTTP、TCP 等）进行交互。微服务架构的优点在于高可扩展性、灵活性和易于维护。

与传统的单体应用架构相比，微服务架构具有以下特点：

- 模块化：将应用程序拆分为一系列小型服务，每个服务都是独立的模块，易于维护和扩展。
- 独立部署：每个服务都可以独立部署，无需影响其他服务。
- 松耦合：每个服务都使用独立的数据存储，相互之间松耦合，避免了单点故障。
- 高可用性：服务可以水平扩展，以应对高流量和高并发请求。
- 技术多样性：不同的服务可以使用不同的技术栈，例如 Java、Python、Node.js 等，充分利用各种技术的优势。

微服务架构的核心思想是将复杂的系统拆分为多个小型服务，每个服务都有一个明确的责任，从而减少系统的复杂性，并提高开发效率和灵活性。

## 2. NestJS 微服务快速入门

我们以一个简单的微服务架构作为示例

我们将会构建三个模块如图：
![image](https://github.com/user-attachments/assets/954f57fe-3a57-41e0-b3db-a1ce4f6824e6)

- `api-gateway` API Gateway 是整个微服务架构中的入口点。它主要负责接收来自客户端的请求，并将请求分发到对应的微服务。它也可以进行身份验证、请求转发、聚合数据等操作。
-  `service_1` 和 `service_2` 是独立的微服务，它们各自负责特定的业务逻辑。这些微服务通常根据业务需求进行拆分，每个服务专注于处理一个功能或模块。



### 2.1 创建项目

首先，我们先在自己的工作区创建`nestjs_microservice_quickstart`目录作为根目录，并且使用npm初始化
```bash
npm init -y
```
然后创建 microservice 目录作为微服务的目录

再使用 NestJS CLI 分别创建三个新项目，我们创建 `api-gateway` 作为网关服务，并分别创建两个微服务 `service_1` 和 `service_2`。项目的基本结构如下：

```bash
nest new api-gateway
cd microservice
nest new service_1
nest new service_2
```

```
nestjs_microservice_quickstart
|-- api-gateway
|   |-- src
|         |-- 
|
|-- microservice
|   |-- service_1
|   |        |-- src
|   |
|   |-- service_2
|            |-- src
```
创建完成我们给每一个服务安装NestJS的微服务包

```bash
npm i --save @nestjs/microservices
```

### 2.2 编码

#### 2.2.1 编写 `service_1`

在 `service_1` 中，我们首先在 `main.ts` 中设置微服务的传输协议为 TCP，并定义其监听的端口为 3001：

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.TCP,
      options: {
        host: '0.0.0.0',
        port: 3001,
      },
    },
  );
  await app.listen();
}
bootstrap();
```
##### 解释
- 与之前的单体架构项目不同的是我们创建的是微服务，并默认使用NestJS的`TCP`协议进行微服务之间的通信，并指定监听端口`3001`

接下来，我们在 `app.controller.ts` 中使用 `MessagePattern` 装饰器来处理从网关发来的消息：

```typescript
import { Controller } from '@nestjs/common';
import { AppService } from './app.service';
import { MessagePattern } from '@nestjs/microservices';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @MessagePattern({ cmd: 'get_hello' })
  getHello(data: string): string {
    return this.appService.getHello(data);
  }
}
```
##### 解释
- 我们在`MessagePattern`修饰器中设置参数作为标识接收网关的消息，`{ cmd: 'get_hello' }`标识了该方法为`getHello`


`app.service.ts` 提供了一个简单的服务方法：

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(data: string): string {
    return `Hello World! This is service_1 from: ${data}`;
  }
}
```

`service_2` 的结构与 `service_1` 类似，但监听端口为 3002。

#### 2.2.2 编写 `api-gateway`

`api-gateway` 作为微服务的入口，通过 `ClientsModule` 配置与 `service_1` 和 `service_2` 的通信：

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'SERVICE_1',
        transport: Transport.TCP,
        options: { host: 'localhost', port: 3001 },
      },
      {
        name: 'SERVICE_2',
        transport: Transport.TCP,
        options: { host: 'localhost', port: 3002 },
      },
    ]),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`app.controller.ts` 中使用 `ClientProxy` 来与微服务通信：

```typescript
import { Controller, Get, Inject, UseInterceptors } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { LoggingInterceptor } from './common/interceptor/logger.interceptor';

@Controller()
@UseInterceptors(LoggingInterceptor)
export class AppController {
  constructor(
    @Inject('SERVICE_1') private readonly service1: ClientProxy,
    @Inject('SERVICE_2') private readonly service2: ClientProxy,
  ) {}

  @Get()
  async getHello(): Promise<string> {
    const service1Response = await this.service1.send({ cmd: 'get_hello' }, 'API Gateway').toPromise();
    const service2Response = await this.service2.send({ cmd: 'get_hello' }, 'API Gateway').toPromise();
    return `${service1Response}\n${service2Response}`;
  }
}
```

### 2.3 运行测试

分别进入对应的服务启动命令：
```bash
npm run start
```

完成编码后，我们可以分别运行 `service_1`、`service_2` 和 `api-gateway`，并通过浏览器或 Postman 访问 `http://localhost:3000`、 `http://localhost:3000/service1`、 `http://localhost:3000/service2`，测试微服务之间的通信是否正常。

运行截图：
![image](https://github.com/user-attachments/assets/7df51aaf-f9fa-4290-a6a1-ff20cccc1a74)
![image](https://github.com/user-attachments/assets/4902428f-4d1a-4a09-bef8-733d75e1f7ec)
![image](https://github.com/user-attachments/assets/cb74aa38-3c15-4882-b647-6270f1a0b877)



### 我们还可以在API网关中使用拦截器构建日志功能：
api-gateway中`src/common/interceptor`的logger.interceptor.ts文件下：
```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler, Logger } from '@nestjs/common';
import { Observable  } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const methodKey = context.getHandler().name;
    const now = Date.now();
    return next
      .handle()
      .pipe(
        tap(() => {
          this.logger.log(`method:${methodKey}-耗时： ${Date.now() - now}ms`)
        })
      );
  }
}
```

并在controller中使用

```typescript
@Controller()
@UseInterceptors(LoggingInterceptor)//使用
export class AppController {
  constructor(
    @Inject('SERVICE_1') private readonly service1: ClientProxy,
    @Inject('SERVICE_2') private readonly service2: ClientProxy,
  ) {}

  @Get()
  async getHello(): Promise<string> {
    const service1Response = await this.service1.send({ cmd: 'get_hello' }, 'API Gateway').toPromise();
    const service2Response = await this.service2.send({ cmd: 'get_hello' }, 'API Gateway').toPromise();
    return `${service1Response}\n${service2Response}`;
  }
```

![image](https://github.com/user-attachments/assets/b54cc41c-90b7-440a-a26f-514791628c18)


## 结论

在本篇博客中，我们初步探讨了微服务架构，并且使用 NestJS 快速创建微服务应用的示例。相信我们对微服务架构有了初步的了解。在下一篇博客中，我们将继续探讨更多高级的微服务架构实践，敬请期待。

如有任何问题或建议，欢迎在评论区留言。

感谢阅读！
