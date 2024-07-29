# TMDOG的微服务之路_03——Nest.js 的中间件
## 博客地址：[TMDOG的博客](https://blog.tmdog114514.icu)

在上一篇博客中，我们实现了一个简易的用户管理api的功能。在此基础上，我们将继续探讨如何在 Nest.js 中使用中间件。中间件是处理 HTTP 请求的一个重要环节，可以在请求到达控制器之前对其进行修改、验证或日志记录等操作。我们将通过两个示例来详细讲解中间件的使用：日志记录中间件和 JWT 身份验证中间件。

## 日志记录中间件

### 1. 创建日志记录中间件
在我们上一篇文章中，我们的api请求发生后并不能显示在控制台上，导致我们调试接口很不方便，所以我们可以通过中间件的方式将日志输出在控制台上。

日志记录中间件用于记录每个 HTTP 请求的信息，例如请求方法、URL、状态码、响应长度和用户代理。我们首先在`src\common\middlewares`下创建一个日志记录中间件`logger.middlewares.ts`。

```typescript
import { Injectable, NestMiddleware, Logger } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction): void {
    const { method, originalUrl } = req;
    const userAgent = req.get('user-agent') || '';

    res.on('finish', () => {
      const { statusCode } = res;
      const contentLength = res.get('content-length');
      this.logger.log(
        `${method} ${originalUrl} ${statusCode} ${contentLength} - ${userAgent}`,
      );
    });

    next();
  }
}
```

#### 解释：

- `LoggerMiddleware` 类实现了 `NestMiddleware` 接口。
- 实例化对于HTTP请求的日志对象`Logger('HTTP')`
- `use` 方法接受 `Request`、`Response` 和 `NextFunction` 参数，并在Reguest对象监听请求事件。
- 中间件记录了请求的 `method`、`originalUrl` 和 `user-agent`，在响应完成后记录 `statusCode` 和 `contentLength`，然后通过 `Logger` 打印在控制台上。

### 2.在`module`中挂载中间件

```typescript
import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { AppController, UsersController } from './app.controller';
import { AppService, UsersService } from './app.service';
import { LoggerMiddleware } from "./common/middlewares/logger.middlewares";

@Module({
  imports: [],
  controllers: [AppController, UsersController],
  providers: [AppService, UsersService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*')
}
```

#### 解释
- `AppModule`实现了` NestModule`接口
- 实现函数`configure`传入`MiddlewareConsumer`对象
- 通过`apply(LoggerMiddleware)`挂载中间件
- 并指定作用的路由`*`通配符，指定所有路由，也可以指定特定路由

同样中间件也可以通过依赖注入的方式去挂载，其他的代码都不用改动

### 简单测试

启动项目

postman中测试：
![image](https://github.com/user-attachments/assets/07e52784-162a-4d5e-a42d-57792190c53c)

控制台显示：
![image](https://github.com/user-attachments/assets/8ff3d83f-3183-4d4b-8e1b-4eaa833576ef)


恭喜！我们成功使用中间件实现了请求日志的展示！


##  JWT 身份验证中间件
JWT 身份验证中间件用于验证请求中的 JWT，并将解码后的用户信息添加到请求对象中。我们需要 `jsonwebtoken` 库来生成和验证 JWT。


### 1. 准备工作
我们在项目根目录下创建`.env`文件，并写入`JWT_SECRET="your-secret"`,来配置jwt的密钥（可以随便写）

```
JWT_SECRET="your-secret"
```

安装依赖：

```bash
npm install jsonwebtoken @nestjs/jwt dotenv
```

然后在`main.ts`上加入一行：`require('dotenv').config();`。允许读取.env文件中的内容

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
require('dotenv').config();


async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}
bootstrap();

```

### 2.  创建 JWT 中间件

```typescript
import { Injectable, NestMiddleware, UnauthorizedException } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import * as jwt from 'jsonwebtoken';

interface JwtPayload {
    username: string;
    iat: number;
    exp: number;
}

@Injectable()
export class JwtMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization;
    if (!authHeader) {
      throw new UnauthorizedException('Authorization header is missing');
    }

    const token = authHeader.split(' ')[1];
    if (!token) {
      throw new UnauthorizedException('Token is missing');
    }

    try {
      const decoded: JwtPayload = jwt.verify(token, process.env.JWT_SECRET) as JwtPayload;
      (req as any).user = decoded;
      next();
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```

#### 解释：

- `JwtPayload`对于jwt解码数据接口的定义
- `JwtMiddleware` 类实现了 `NestMiddleware` 接口。
- `use` 方法中，从请求头中提取 `Authorization` 字段，如果不存在则抛出 `UnauthorizedException` 异常。
- 从 `Authorization` 头中提取出 JWT（通常以 `Bearer token` 的形式传递）。
- 使用 `jsonwebtoken` 库的 `verify` 方法验证 JWT 的有效性，并解码出 JWT 的负载部分（`payload`）。
- 将解码出的用户信息（负载部分）添加到请求对象的 `user` 属性中。
- 调用 `next` 函数将请求传递给下一个中间件或控制器。

#### 配置中间件

我们需要在应用程序模块中使用这些中间件。在 `app.module.ts` 中注册中间件：

```typescript
import { MiddlewareConsumer, Module, NestModule, RequestMethod } from '@nestjs/common';
import { AppController, UsersController } from './app.controller';
import { AppService, UsersService } from './app.service';
import { LoggerMiddleware } from "./common/middlewares/logger.middlewares";
import { JwtMiddleware } from './common/middlewares/jwt.middlewares';

@Module({
  imports: [],
  controllers: [AppController, UsersController],
  providers: [AppService, UsersService],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*')
    consumer
      .apply(JwtMiddleware)
      .exclude(
        {path:'users/register', method: RequestMethod.POST},
        {path:'users/login', method: RequestMethod.POST}
      )
      .forRoutes(UsersController)
  }
}
```

#### 解释
- 同样我们使用`consumer`对象挂载中间件，但我们使用了`exclude`去剔除不需要的jwt作用的路由


### 3. 修改登录的逻辑生成jwt token

我们在原有的代码上添加以下代码：

```typescript
import * as jwt from 'jsonwebtoken';

@Controller()
export class UsersController {
  constructor(private readonly userService: UsersService) { }

 @Post('login')
  login(@Req() req: Request) {
    const { username, password } = req.body;
    const isLogin = this.userService.login(username, password);
    const jwtToken = jwt.sign({ username }, process.env.JWT_SECRET, { expiresIn: '1h' });
    return {result:isLogin, token:jwtToken};
  }

@Get()
  getAllUsers(@Req() req: Req) {
    const { user } =req;
    console.log(user);
    return this.userService.getAllUsers();
  }
}
```

#### 解释
- `const jwtToken = jwt.sign({ username }, process.env.JWT_SECRET, { expiresIn: '1h' });`注册jwt token过期时长1h
- 在`getAllUsers`中打印jwt中间件加入的`req.user`对象

### 测试
#### 注册：
![image](https://github.com/user-attachments/assets/942f465b-630d-4e13-b975-1a0bbf5f5bec)
#### 登录：
![image](https://github.com/user-attachments/assets/d2d8ea7d-ed2b-4c7a-b425-7b036e17f859)
我们看见，成功返回了jwt token
#### 无token操作：
![image](https://github.com/user-attachments/assets/fec88ba6-4557-4d89-8cda-6d4342c52d90)
#### 有token操作：
我们将登录获得的jwt token复制到 authorization header中
![image](https://github.com/user-attachments/assets/8773f536-2caf-4421-af6f-8e5c62d47db8)
#### 控制台中我们看见打印的对象：
![image](https://github.com/user-attachments/assets/3a207f28-2b6a-4d39-a32f-a0bb0df44986)

恭喜！我们完成了jwt中间件的使用！

## 结论
通过以上步骤，我们成功地在 Nest.js 应用中添加并使用了日志记录中间件和 JWT 身份验证中间件。中间件在处理请求时起到了重要的作用，可以用于记录日志、验证身份、处理错误等。希望这篇博客能帮助你更好地理解和使用 Nest.js 中间件。

在下一个博客中，我们将进一步探讨 Nest.js 的其他功能，敬请期待。

如果各位技术大佬有任何问题或建议，请在评论区留言。

感谢阅读！