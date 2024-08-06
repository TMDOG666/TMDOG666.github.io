# TMDOG的微服务之路_04——Nest.js 的异常筛选器
## 博客地址：[TMDOG的博客](https://blog.tmdog114514.icu)

在上一篇博客中，我们实现了一个简易的用户管理 API 并添加了中间件功能。本篇博客，我们将探讨如何在 Nest.js 中使用异常筛选器。可以帮助我们更好地处理异常。

## 异常筛选器

### 1. 创建异常筛选器

异常筛选器用于捕获和处理应用程序中的HTTP异常。在 `src\common\filter` 下创建一个异常筛选器 `http-exception.filter.ts`。

```typescript
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();
    
    const errorResponse = {
      statusCode: status,
      timestamp: new Date().toLocaleString(),
      path: request.originalUrl,
      message: exceptionResponse['message'] || exception.message,
    };

    response.status(status).json(errorResponse);
  }
}
```
#### 解释
- `ExceptionFilter`, `Catch`, `ArgumentsHost`, `HttpException`：从 `@nestjs/common` 导入，分别用于定义异常过滤器、捕获特定类型异常的装饰器、获取当前处理上下文和表示 HTTP 异常。
- `@Catch(HttpException)`：这是一个装饰器，用于捕获 `HttpException` 类型的异常。
- `HttpExceptionFilter` 实现了 `ExceptionFilter` 接口。
- `catch` 方法是 `ExceptionFilter` 接口必须实现的方法。它接收一个 `HttpException` 对象和一个 `ArgumentsHost` 对象。
- `ctx`：从 `ArgumentsHost` 中获取 HTTP 上下文。
- `response` 和 `request`：分别获取 `Response` 和 `Request` 对象。
- `status`：获取异常的状态码。
- `exceptionResponse`：获取异常的响应信息。
- `errorResponse`：构造一个包含状态码、时间戳、请求路径和错误信息的对象。
- 使用 `response` 对象将 `errorResponse` 以 JSON 格式发送给客户端。

### 2. 全局挂载异常筛选器

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { HttpExceptionFilter } from './common/filter/http-exception.filter';
require('dotenv').config();

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(3000);
}
bootstrap();
```
- 我们将异常筛选器挂载到全局`app.useGlobalFilters(new HttpExceptionFilter());`


### 3. 修改controller
```typescript
@Post('register')
  register(@Req() req: Request, @Res() res: Response) {
    const { username, password } = req.body;
    try {
      const result = this.userService.register(username, password);
      res.status(HttpStatus.CREATED).send({ result, massage: '注册成功' });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.UNAUTHORIZED);
    }
  }
```
- 在我们之前的代码基础上规范化了`register`的响应并使用`HttpException`抛出异常

### 4.运行效果
正常请求：
![image](https://github.com/user-attachments/assets/5f12f5d2-1487-45c0-9b63-516a0fcc7887)
然后我们再次注册：
![image](https://github.com/user-attachments/assets/93e30d1e-8897-40be-a703-cfc0ad0bd7a4)

我们发现响应了异常

然后我们注释掉挂载的异常筛选器，再次同样的操作
![image](https://github.com/user-attachments/assets/fcb42038-a9b0-4b67-bef7-02f8de6cd76c)
我们发现只响应了`HttpException`的内容

## 结论

通过以上步骤，我们成功地在 Nest.js 应用中添加并使用了异常筛选器。在处理异常时起到了统一对外响应不同的异常的作用。希望这篇博客能帮助你更好地理解和使用 Nest.js 中的这些高级功能。

在下一个博客中，我们将进一步探讨 Nest.js 的其他功能，敬请期待。

如果各位技术大佬有任何问题或建议，欢迎在评论区留言。

感谢阅读！