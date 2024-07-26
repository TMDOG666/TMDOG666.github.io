# TMDOG的微服务之路_01——nest.js 快速入门

## **博客地址：[TMDOG 的博客](https://blog.tmdog114514.icu)**


今天我们开始入门nest.js，学习nest.js项目快速的搭建。

## 关于NestJS

NestJS 是一个用于构建高效、可靠且可扩展的服务器端应用程序的框架。它使用 TypeScript 编写，借鉴了许多成熟的解决方案，例如 Angular 的**模块化**和**依赖注入**系统，提供了一种现代化的开发体验。

### 主要特点

1. **模块化架构**: 允许开发者将应用程序拆分为功能模块，便于维护和扩展。
2. **依赖注入**: 提供了强大的依赖注入机制，简化了服务之间的依赖管理。
3. **装饰器**: 通过 TypeScript 装饰器（decorators）来定义路由、管道、中间件等，提高代码的可读性和可维护性。
4. **与现有技术集成**: 支持与 Express 或 Fastify 进行无缝集成，并提供对多种数据库（如 TypeORM、Sequelize）的内置支持。
5. **开箱即用的测试功能**: 提供了便捷的工具来编写和运行单元测试和端到端测试。
6. **易于使用的 CLI**: Nest CLI 工具可以帮助开发者快速生成和管理项目结构和代码片段。



# 开箱即用的 Nest.js

### 1. 创建项目目录

![image-20240726165115583](https://github.com/user-attachments/assets/5e625b22-7b82-47b8-8aba-7c6750d70347)


并且在vscode上打开文件夹并打开终端
![image-20240726165253023](https://github.com/user-attachments/assets/17c04643-2beb-48c6-bfe3-7ec560bedafd)

### 2. 安装 Nest.js CLI

首先，我们需要安装 Nest.js CLI 来创建和管理 Nest.js 项目。运行以下命令来安装 CLI：

```bash
npm i -g @nestjs/cli
```

![image-20240726164959578](https://github.com/user-attachments/assets/c4975f53-d249-4b52-9266-e995c7a34b7b)


### 3. 创建一个新项目

使用 CLI 创建一个新的 Nest.js 项目：

```bash
nest new project-name
```

系统会提示你选择包管理器，可以选择 `npm` 或 `yarn`。选择后，CLI 会自动安装项目所需的依赖。



我选择的是`npm`


![image-20240726165535673](https://github.com/user-attachments/assets/e46c7319-ced2-40a8-9b70-b8a6cb2b44b2)

![image-20240726165805834](https://github.com/user-attachments/assets/787bb254-7b31-4909-9140-d8cb6919b9f2)

等待安装，成功后结果如下：



![image-20240726165827763](https://github.com/user-attachments/assets/d5d7fa88-eef2-45ac-868c-48016df96be3)



### 4. 启动项目

进入创建的项目根目录

```
cd my-nestjs-demo
```

然后启动项目

```
npm run start
```
![image-20240726171116718](https://github.com/user-attachments/assets/411316b2-8138-4a81-bbe8-74b87ed923d8)

### 5. 验证成果

打开浏览器输入url：`localhost:3000`

![image-20240726171841656](https://github.com/user-attachments/assets/6ef0767d-885f-4c19-8c2b-2d6aaef13d27)


然后我们在vscode上打开另一个终端

输入:

```
npm run test
```


![image-20240726171653550](https://github.com/user-attachments/assets/3b207614-63f8-4e97-a961-2fcd51b1942d)

**恭喜！ 我们的项目就启动成功了！**

并且在浏览器上成功访问！

也通过了nest.js的测试模块测试！



# 项目描述

我们成功的启动了项目，接下来我们了解一下项目结构与代码

### 项目结构

创建的项目会有一个默认的结构，主要包括以下文件和文件夹：

```
src
 ├── app.controller.ts
 ├── app.controller.spec.ts
 ├── app.module.ts
 ├── app.service.ts
 └── main.ts
```

- **app.controller.ts**: 控制器，处理 HTTP 请求。
- **app.service.ts**: 服务，包含业务逻辑。
- **app.module.ts**: 模块，组织应用的结构。
- **main.ts**: 应用入口文件。



### `main.ts`

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

`main.ts`是应用程序的入口文件。

1. `import { NestFactory } from '@nestjs/core';`：导入NestFactory，这是一个用于创建Nest应用程序实例的类。

2. `import { AppModule } from './app.module';`：导入应用的根模块（AppModule）。

3. `async function bootstrap() { ... }`：定义了一个异步函数bootstrap，它是应用程序的启动函数。

4. `const app = await NestFactory.create(AppModule);`：使用NestFactory工厂创建一个Nest应用程序实例，并传入根模块AppModule。

5. `await app.listen(3000);`：让应用监听3000端口。

6. `bootstrap();`：调用bootstrap函数启动应用。

   

### `app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

`app.module.ts`定义了应用程序的根模块（AppModule）。

1. `import { Module } from '@nestjs/common';`：导入Module装饰器。
2. `import { AppController } from './app.controller';`：导入AppController。
3. `import { AppService } from './app.service';`：导入AppService。
4. `@Module({ ... })`：装饰器，定义一个模块。
5. `imports: []`：模块的导入数组，这里为空。
6. `controllers: [AppController]`：控制器的数组，包含AppController。
7. `providers: [AppService]`：提供者的数组，包含AppService。
8. `export class AppModule {}`：定义并导出AppModule类。



### `app.controller.ts`

```typescript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

`app.controller.ts`定义了一个控制器（AppController）。

1. `import { Controller, Get } from '@nestjs/common';`：导入Nest.js的Controller和Get装饰器。

2. `import { AppService } from './app.service';`：导入服务（AppService）。

3. `@Controller()`：装饰器，标识这个类是一个控制器。

4. `export class AppController { ... }`：定义控制器类AppController。

5. `constructor(private readonly appService: AppService) {}`：通过依赖注入将AppService注入到控制器中。

6. `@Get()`：装饰器，定义一个GET请求的路由。

7. `getHello(): string { ... }`：定义一个处理GET请求的方法，调用AppService的getHello方法返回字符串。

   

### `app.service.ts`

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}
```

`app.service.ts`定义了一个服务（AppService）。

1. `import { Injectable } from '@nestjs/common';`：导入Injectable装饰器。
2. `@Injectable()`：装饰器，标识这个类是一个可以被注入的服务。
3. `export class AppService { ... }`：定义并导出AppService类。
4. `getHello(): string { ... }`：定义一个方法，返回字符串“Hello World!”。





通过这些代码，可以看到这是一个简单的Nest.js应用程序，它定义了一个根模块（AppModule），一个控制器（AppController），以及一个服务（AppService）。控制器处理GET请求，并调用服务的方法来返回一个“Hello World!”字符串。

# 总结

我们通过本教程介绍了如何创建一个简单的 Nest.js 项目，包括如何创建控制器、服务和模块。我们可以根据需要扩展这个基础项目，添加更多功能和逻辑。感谢各位技术大佬的观看和点评！！！

