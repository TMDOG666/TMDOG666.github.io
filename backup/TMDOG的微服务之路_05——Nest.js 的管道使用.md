# TMDOG的微服务之路_05——Nest.js 的管道使用
## 博客地址：[TMDOG的博客](https://blog.tmdog114514.icu)

在上一篇博客中，我们探讨了如何在 Nest.js 中使用异常筛选器。本篇博客，我们将深入了解如何在 Nest.js 中使用管道进行数据验证和转换。

## 使用内置管道

### 1. 在控制器中使用内置管道
我们在原有的代码基础上修改：
```typescript
@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get(':id')
  getHello(@Param('id', new ParseIntPipe()) id: number): string {
    return (this.appService.getHello() + id + typeof id);
  }
}
```

#### 解释
- `ParseIntPipe`：这是一个内置的管道，用于将参数转换为整数类型。如果转换失败，将抛出一个 `BadRequestException`。

### 2. 测试效果
正常请求：
![image](https://github.com/user-attachments/assets/178d4ea7-3595-4cbd-a34c-7e19fe0991e8)
我们发现成功响应了请求param的值还有值的类型

异常请求：
![image](https://github.com/user-attachments/assets/9346a23a-3be9-48ed-8ec4-2d819955bc72)
我们换成`abc`后发生报错，值的类型不匹配

目前我们就简单使用了管道帮我们进行数据验证
Nest 配有九个开箱即用的管道：
- `ValidationPipe`
- `ParseIntPipe`
- `ParseFloatPipe`
- `ParseBoolPipe`
- `ParseArrayPipe`
- `ParseUUIDPipe`
- `ParseEnumPipe`
- `DefaultValuePipe`
- `ParseFilePipe`


## 使用自定义的全局管道

### 1. 创建自定义验证管道

在 `src\common\pipes` 下创建一个自定义验证管道 `validation.pipe.ts`。

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException, Type } from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToInstance } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    console.log(metatype)
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

  private toValidate(metatype: Type): boolean {
    const types = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype as any);
  }
}
```

#### 解释
- `validate`：从 `class-validator` 导入，用于验证对象。
- `metatype` 会自动匹配请求中使用的DTO（数据传输对象），如果没有使用DTO，或者DTO对象在`toValidate`存在就返回值
- `plainToInstance`：从 `class-transformer` 导入，用于将普通对象转换为类实例，如果值与metatype匹配不上直接抛出异常
- `transform` 方法：将传入的数据转换为类实例并进行验证。
- `toValidate` 方法：检查元类型是否需要验证。

### 2. 全局挂载验证管道

我们在 `main.ts` 中全局挂载自定义验证管道。

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { HttpExceptionFilter } from './common/filter/http-exception.filter';
import { ValidationPipe } from './common/pipes/validation.pipe';
require('dotenv').config();


async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  app.useGlobalPipes(new ValidationPipe());

  await app.listen(3000);
}
bootstrap();

```

### 3. 使用 DTO 进行验证

我们在src/dto/app.dto.ts下创建两个 DTO，`RegisterDto` 和 `LoginDto`，用于用户注册和登录。

```typescript
import { IsString, MinLength, MaxLength } from 'class-validator';

export class RegisterDto {
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  username: string;

  @IsString()
  @MinLength(6)
  password: string;
}

export class LoginDto {
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  username: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

### 4. 修改控制器

我们修改 `UsersController`，使用 DTO 进行数据验证。

```typescript
import { Controller, Post, Body, Res, HttpException, HttpStatus } from '@nestjs/common';
import { UsersService } from './users.service';
import { RegisterDto, LoginDto } from './dto/app.dto';
import * as jwt from 'jsonwebtoken';
import { Response } from 'express';

@Controller('users')
export class UsersController {
  constructor(private readonly userService: UsersService) {}

  @Post('register')
  register(@Body() registerDto: RegisterDto, @Res() res: Response) {
    const { username, password } = registerDto;
    try {
      const result = this.userService.register(username, password);
      res.status(HttpStatus.CREATED).send({ result, message: '注册成功' });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.UNAUTHORIZED);
    }
  }

  @Post('login')
  login(@Body() loginDto: LoginDto, @Res() res: Response) {
    const { username, password } = loginDto;
    try {
      const isLogin = this.userService.login(username, password);
      const jwtToken = jwt.sign({ username }, process.env.JWT_SECRET, { expiresIn: '1h' });
      res.status(HttpStatus.OK).send({ result: isLogin, message: '登陆成功', token: jwtToken });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.UNAUTHORIZED);
    }
  }
}
```

### 5. 测试效果
注册：
![image](https://github.com/user-attachments/assets/78e787b4-9644-402f-8eb4-218b2ff7ae41)
正常登录：
![image](https://github.com/user-attachments/assets/e8a515ce-ddc0-4216-b0c0-c8df269daa1e)
异常登录：
![image](https://github.com/user-attachments/assets/98e1e956-8465-43b4-8859-c85231489670)

当我们发送注册请求时，如果传入的数据不符合 DTO 中的验证规则，将会抛出 `BadRequestException` 并返回验证失败的消息。

## 结论

通过以上步骤，我们成功地在 Nest.js 应用中添加并使用了自定义验证管道，实现了数据验证和转换的功能。希望这篇博客能帮助你更好地理解和使用 Nest.js 中的管道功能。

在下一个博客中，我们将进一步探讨 Nest.js 的其他功能，敬请期待。

如果各位技术大佬有任何问题或建议，欢迎在评论区留言。

感谢阅读！