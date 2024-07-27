# TMDOG的微服务之路_02——Nest.js  的初步使用

在上一篇博客中，我们介绍了如何在 Nest.js 中创建一个简单的应用程序，hello world！今天在这篇博客中，我们将进一步探讨如何使用 Nest.js 的 Controller 来处理 HTTP 请求，并了解service 与 module。我们将通过创建一个用户管理的功能，来展示如何使用各种 HTTP 请求方法。

### 1. 目录结构

首先，让我们看一下项目的目录结构：

```
src
├── app.controller.ts
├── app.module.ts
├── app.service.ts
└── main.ts
```

### 2. 入口文件 (`main.ts`)

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();
```

`main.ts` 文件是应用程序的入口。我们使用 `NestFactory.create` 方法来创建一个 Nest 应用实例，并让它监听 3000 端口。

### 3. 服务 (`app.service.ts`)

接下来，我们在之前的代码基础上对User数组对象的操作 `UsersService`。

```typescript
import { Injectable } from '@nestjs/common';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}

type User = {       //定义一个简单的用户模板
  username: string;
  password: string;
}

@Injectable()
export class UsersService {
  private users: User[]; // 实例化用户数组对象

  constructor() {
    this.users = []; // 初始化 users 数组
  }

// 接下开就是对数组的CRUD了，这里不多赘述
  public register(username: string, password: string): boolean {
    if (this.users.find(user => user.username === username)) throw new Error('User already exists');
    this.users.push({ username, password });
    return true;
  }

  public login(username: string, password: string): boolean {
    const user = this.users.find(user => user.username === username);
    if (!user) throw new Error('User not found');
    if (user.password !== password) throw new Error('Password is incorrect');
    return true;
  }

  public deleteUser(username: string): boolean {
    const user = this.users.find(user => user.username === username);
    if (!user) throw new Error('User not found');
    this.users = this.users.filter(user => user.username !== username);
    return true;
  }

  public changePassword(username: string, password: string): boolean {
    const user = this.users.find(user => user.username === username);
    if (!user) throw new Error('User not found');
    user.password = password;
    return true;
  }

  public getAllUsers(): User[] {
    return this.users;
  }
}
```

- `UsersService` 则负责用户的注册、登录、更改密码、删除用户和获取所有用户。
- `@Injectable()` 装饰器标识了一个可依赖注入的类，这样一来，Nest 的 IoC 容器就能够识别这个类，并将其注入到其他被注册的类或模块中。

### 4. 控制器 (`app.controller.ts`)

接下来来到我们的重点，我们在之前的基础上编写了，实现了对用户的请求处理 `UsersController`。

```typescript
import { Controller, Get, Post, Res, Req, Put, Delete } from '@nestjs/common';
import { AppService, UsersService } from './app.service';
import { Request, Response } from 'express';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) { }

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}

@Controller('users')
export class UsersController {
  constructor(private readonly userService: UsersService) { }

  @Post('register')
  register(@Req() req: Request) {
    const { username, password } = req.body;
    return this.userService.register(username, password);
  }

  @Post('login')
  login(@Req() req: Request) {
    const { username, password } = req.body;
    return this.userService.login(username, password);
  }

  @Put(':username')
  changePassword(@Req() req: Request) {
    const { password } = req.body;
    const { username } = req.params;
    return this.userService.changePassword(username, password);
  }

  @Delete(':username')
  deleteUser(@Req() req: Request) {
    const { username } = req.params;
    return this.userService.deleteUser(username);
  }

  @Get()
  getAllUsers() {
    return this.userService.getAllUsers();
  }
}
```

- `UsersController` 则定义了多个方法来处理用户的注册、登录、更改密码和删除用户等操作。
- `@Controller()` 装饰器定义了一个控制器类，它同样受到 IoC 容器的管理。我们可以把它理解为一个特殊的 `@Injectable()`，可以处理 HTTP 请求。例如，`@Controller('users')` 会截取到 `"/users"` 路径的请求。
- `constructor(private readonly userService: UsersService) { }` 是控制器类的构造函数。在这里，我们可以初始化注册的服务实例（如 `UsersService`），这些服务实例由 Nest 的 IoC 容器自动注入，从而实现依赖注入。
- `@Get() 、@Post('register')` 我们定义了具体的请求处理的方法，处理具体的HTTP请求，同样`@Post('register')`进一步截取到 `"/users/register"` 路径的请求。
- `@Req()`定义了请求体，Nest 提供对底层平台的请求对象的访问（默认为 Express）。我们可以通过指示 Nest 通过在处理者的签名中添加装饰器来注入它来访问请求对象。我们还可以通过不同的注解获取自己想要的请求对象，以下是官方文档中的内容：
![image](https://github.com/user-attachments/assets/b19e30dc-9e93-4c48-8c92-106d506db281)




### 5. 模块 (`app.module.ts`)

最后，我们定义根模块 `AppModule`。

```typescript
import { Module } from '@nestjs/common';
import { AppController, UsersController } from './app.controller';
import { AppService, UsersService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController, UsersController],
  providers: [AppService, UsersService],
})
export class AppModule {}
```

`AppModule` 将所有的控制器和服务注册到 Nest.js 应用程序中。
在这里我们就可以统一管理需要注册的模块。

### 6. 成果展示

#### 我们使用 Postman 测试 Nest.js 应用

在这部分，我们将展示如何使用 Postman 测试上面的 `UsersController` 中的各个端点。首先确保应用已经运行，监听在 `localhost:3000`。然后按照以下步骤进行测试。

#### 1. 测试注册用户 (`/users/register`)

**请求类型**：POST

**URL**：`http://localhost:3000/users/register`

**请求体**（JSON 格式）：

```json
{
  "username": "tmdog",
  "password": "114514"
}
```

**预期响应**：

```json
true
```

**实际响应**：

![image](https://github.com/user-attachments/assets/87c160d3-de9f-4d43-a672-805947c6c465)


#### 2. 测试用户登录 (`/users/login`)

**请求类型**：POST

**URL**：`http://localhost:3000/users/login`

**请求体**（JSON 格式）：

```json
{
  "username": "tmdog",
  "password": "114514"
}
```

**预期响应**：

```json
true
```

**实际响应**：

![image](https://github.com/user-attachments/assets/b7bcb93d-dddb-479b-84da-f2d34be914ab)


#### 3. 测试修改密码 (`/users/:username`)

**请求类型**：PUT

**URL**：`http://localhost:3000/users/tmdog`

**请求体**（JSON 格式）：

```json
{
    "password":"114515"
}
```

**预期响应**：

```json
true
```

**实际响应**：

![image](https://github.com/user-attachments/assets/ba29494f-f33e-48b9-b77e-2a41dd01558a)

#### 4. 测试删除用户 (`/users/:username`)

**请求类型**：DELETE

**URL**：`http://localhost:3000/users/tmdog`

**预期响应**：

```json
true
```

**实际响应**：

![image](https://github.com/user-attachments/assets/0e211f7a-c737-46b3-bbbb-3421ae05b886)

#### 5. 测试获取所有用户 (`/users`)
我们先添加几个用户

**请求类型**：GET

**URL**：`http://localhost:3000/users`

**预期响应**：

```json
[
    {
        "username": "tmdog",
        "password": "114514"
    },
    {
        "username": "tmdog666",
        "password": "114514"
    },
    {
        "username": "tmdog233",
        "password": "114514"
    }
]
```

**实际响应**：
![image](https://github.com/user-attachments/assets/a2fd2587-7c06-4dc7-9173-1e26e20e14f0)

我们就完成了接口的初步测试，这是一个简单的示例，我们并没有给出一个标准统一的响应，所以结果略为简陋



## 结论

在这篇博客中，我们展示了如何在 Nest.js 中使用控制器来处理各种 HTTP 请求。我们创建了一个简单的用户管理服务，涵盖了用户的注册、登录、更改密码、删除用户和获取所有用户等操作。

当我们具体进入到代码编写之后，学习过Spring框架的小伙伴会非常熟悉，Nest.js和Spring框架是同一种思想——AOP（面向切面编程），即在应用程序的不同部分中分离关注点，尤其是横切关注点，如日志记录、安全性、事务管理等。这种方式提高了代码的可维护性和可读性。

在下一个博客中，我们将进一步探讨 Nest.js 的其他功能，敬请期待。

通过这些简单的示例代码，希望你对 Nest.js 的控制器有了初步的了解。
如果各位技术大佬有任何问题或建议，请在评论区留言。

感谢阅读！
