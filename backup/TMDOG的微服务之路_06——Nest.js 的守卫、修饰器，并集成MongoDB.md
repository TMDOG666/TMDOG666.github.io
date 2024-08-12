# TMDOG的微服务之路_06——Nest.js 的守卫、修饰器，并集成 MongoDB

## 博客地址：[TMDOG的博客](https://blog.tmdog114514.icu)

在上一篇博客中，我们探讨了如何在 Nest.js 中使用管道进行数据验证和转换。本篇博客，我们将深入了解如何在 Nest.js 中使用守卫和修饰器进行权限控制，并展示如何将 MongoDB 集成到 Nest.js 应用中。

## 集成MongoDB
### 1. 集成 MongoDB

我们之前的数据都只保存在了内存当中，为了持久化存储用户数据，我们将 MongoDB 集成到 Nest.js 应用中。首先，我们在 `AppModule` 中配置 MongoDB 连接：

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { AppController, UsersController } from './app.controller';
import { User, UserSchema } from './schemas/app.schema';
import { AppService, UsersService } from './app.service';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/mydatabase'),
    MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])
  ],
  controllers: [AppController, UsersController],
  providers: [AppService, UsersService],
})
export class AppModule {}
```

#### 解释
- 我们在`app.module.ts`中导入`MongooseModule`模块，并连接mongodb的url
- 并且我们还注册了自己的`UserSchema`

### 2. 定义 User Schema

我们在 `src/schemas/app.schema.ts` 中定义 `User` 的 Mongoose 模型：

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

enum Role {
  Admin = 'admin',
  User = 'user',
  Guest = 'guest'
}

@Schema()
export class User extends Document {
  @Prop({ required: true, unique: true })
  username: string;

  @Prop({ required: true })
  password: string;

  @Prop({ 
    required: true,
    enum: Object.values(Role),
    message: 'Role must be one of admin, user, guest'
  })
  role: Role;

}

export const UserSchema = SchemaFactory.createForClass(User);
```

#### 解释
- mongodb本身并不直接支持枚举类型，作为数据库级别的约束，为了数据的规范化，我们使用枚举类型来验证`role`数据

### 3. 修改用户服务

用户服务负责与数据库交互，完成用户的注册、登录、删除和密码修改操作。

```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { User } from './schemas/app.schema';
import { RegisterDto, LoginDto, UserDto } from './dto/app.dto';

@Injectable()
export class AppService {
  getHello(): string {
    return 'Hello World!';
  }
}

@Injectable()
export class UsersService {
  constructor(@InjectModel(User.name) private userModel: Model<User>) {}

  async register(registerDto:RegisterDto): Promise<boolean> {
    const existingUser = await this.userModel.findOne( {username : registerDto.username} ).exec();
    if (existingUser) throw new Error('User already exists');
    const newUser = new this.userModel(registerDto);
    await newUser.save();
    return true;
  }

  async login(loginDto:LoginDto): Promise<boolean> {
    const user = await this.userModel.findOne({ username:loginDto.username }).exec();
    if (!user) throw new Error('User not found');
    if (user.password !== loginDto.password) throw new Error('Password is incorrect');
    return true;
  }

  async deleteUser(username: string): Promise<boolean> {
    const user = await this.userModel.findOneAndDelete({ username }).exec();
    if (!user) throw new Error('User not found');
    return true;
  }

  async changePassword(username: string, password: string): Promise<boolean> {
    const user = await this.userModel.findOneAndUpdate({ username }, { password }).exec();
    if (!user) throw new Error('User not found');
    return true;
  }

  async getAllUsers(): Promise<UserDto[]> {
    return this.userModel.find().exec();
  }

  // 给jwt数据查询提供服务
  async getUserByUsername(username: string): Promise<UserDto> {
    return this.userModel.findOne({ username }).exec();
  }
}
```
#### 解释
- 我们通过导入User的模型,对数据库进行增删改查
- 我们也可以通过DTO(数据传输对象)对数据库查询的内容进行规范


### 这样我们就完成的对MongoDB的集成

## 守卫与修饰器的概念

### 1. 什么是守卫？

守卫（Guards）是 Nest.js 中用于授权的核心概念。它们在管道之前运行，可以阻止未经授权的请求到达路由处理器。守卫通常用于检查用户是否有权限执行特定操作。

### 2. 什么是修饰器？

修饰器（Decorators）是一个特殊的声明，它可以附加在类、方法、属性或参数上，为它们添加元数据。Nest.js 使用修饰器来处理路由、依赖注入、元数据绑定等。

## 实现自定义守卫与修饰器
### 1. 创建 `Roles` 修饰器

我们在`src.common.decorator`创建`roles.decorator.ts`文件下自定义修饰器 `Roles`。

```typescript
import { Reflector } from '@nestjs/core';

export const Roles = Reflector.createDecorator<string[]>();
```

#### 解释
- 我们通过`Reflector `创建了一个以string数组作为参数的修饰器。

### 2. 修改jwt中间件
```typescript
import { Injectable, NestMiddleware, UnauthorizedException } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { UsersService } from 'src/app.service';
import * as jwt from 'jsonwebtoken';

interface JwtPayload {
    username: string;
    iat:number;
    exp:number;
}

@Injectable()
export class JwtMiddleware implements NestMiddleware {
  constructor(private readonly userService: UsersService) { }
  async use(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization;
    if (!authHeader) {
      throw new UnauthorizedException('Authorization header is missing');
    }

    const token = authHeader.split(' ')[1];
    if (!token) {
      throw new UnauthorizedException('Token is missing');
    }

    try {
      const decoded : JwtPayload = jwt.verify(token, process.env.JWT_SECRET) as JwtPayload;
      const user = await this.userService.getUserByUsername(decoded.username);
      (req as any).role = user.role;
      next();
    } catch (error) {
      throw new UnauthorizedException('Invalid token');
    }
  }
}
```
#### 解释
- 我们通过jwt token 解析获得的`username`，通过注入的service查询到用户的角色信息
- 将角色信息`(req as any).role = user.role;`存入请求体进行下一步操作

### 3. 创建自定义的 `RolesGuard`

我们先创建一个自定义守卫 `RolesGuard`，用于检查用户的角色是否有权限访问某些路由。

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
    console.log(request.role);
    const role = request.role;
    return this.matchRoles(roles, role);
  }

  matchRoles(roles:string[],role:string):boolean{
    return roles.includes(role);
  }
}
```

#### 解释
- `RolesGuard` 使用 `Reflector` 读取被Roles注解的`controller`上的`Roles`数组
- `Roles`数组中是否包含jwt中间件中获得的角色信息

### 4. 应用守卫和修饰器

在控制器中，我们使用 `Roles` 修饰器定义哪些角色可以访问特定路由，并将 `RolesGuard` 应用到整个模块中。

`app.module.ts`中
```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { AppController, UsersController } from './app.controller';
import { User, UserSchema } from './schemas/app.schema';
import { AppService, UsersService } from './app.service';
import { RolesGuard } from './common/guard/auth.guard';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/mydatabase'),
    MongooseModule.forFeature([{ name: User.name, schema: UserSchema }])
  ],
  controllers: [AppController, UsersController],
  providers: [AppService, UsersService,{
    provide: APP_GUARD,
    useClass: RolesGuard,
  },],
})
export class AppModule {}
```


`app.controller.ts`中
```typescript
import { Controller, Get, Post, Res, Req, Put, Delete, HttpException, HttpStatus, Body, Param, ParseIntPipe } from '@nestjs/common';
import { AppService, UsersService } from './app.service';
import { RegisterDto , LoginDto} from './dto/app.dto';
import { Roles } from './common/guard/roles.decorator'; 
import * as jwt from 'jsonwebtoken';
import { Request, Response } from 'express';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) { }

  @Get()
  getHello(@Param('id'/*,new ParseIntPipe()*/) id :number): string {
    return (this.appService.getHello() + id + typeof id);
  }
}

interface Req extends Request {
  user: any;
}

@Controller('users')
export class UsersController {
  constructor(private readonly userService: UsersService) { }

  @Post('register')
  async register(@Body() registerDto : RegisterDto, @Res() res: Response) {
    try {
      const result = await this.userService.register(registerDto);
      res.status(HttpStatus.CREATED).send({ result, massage: '注册成功' });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.UNAUTHORIZED);
    }
  }


  @Post('login')
  async login(@Body() loginDto : LoginDto, @Res() res: Response) {
    try {
      const isLogin = await this.userService.login(loginDto);
      const jwtToken = jwt.sign({ username:loginDto.username }, process.env.JWT_SECRET, { expiresIn: '1h' });
      res.status(HttpStatus.OK).send({ result: isLogin, massage: '登陆成功', token: jwtToken });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.UNAUTHORIZED);
    }
  }

  @Roles(['user','admin'])
  @Put(':username')
  async changePassword(@Req() req: Request, @Res() res: Response) {
    const { password } = req.body;
    const { username } = req.params;
    try {
      const result = await this.userService.changePassword(username, password);
      res.status(HttpStatus.OK).send({ result, massage: '修改成功' });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.NOT_FOUND);
    }
  }

  @Roles(['user','admin'])
  @Delete(':username')
  async deleteUser(@Req() req: Request, @Res() res: Response) {
    const { username } = req.params;
    try {
      const result = await this.userService.deleteUser(username);
      res.status(HttpStatus.OK).send({ result, massage: '删除成功' });
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.NOT_FOUND);
    }
  }

  @Roles(['admin'])
  @Get()
  async getAllUsers(@Req() req: Req, @Res() res: Response) {
    const { user } = req;
    console.log(user);
    try {
      const result = await this.userService.getAllUsers();
      res.status(HttpStatus.OK).send(result);
    } catch (e) {
      throw new HttpException(e.message, HttpStatus.NOT_FOUND);
    }
  }
}
```

#### 解释
- 我们通过自定义的修饰器`Roles`,去定义可以通过守卫的角色`@Roles(['user','admin'])`(允许角色为user和admin的访问)

### 这样我们就使用了自定义的修饰器与守卫对请求路由的保护


## 测试与总结
#### 1.证错误的角色信息
![image](https://github.com/user-attachments/assets/48d7cbce-6ea4-4d90-bce6-39ba975b22f1)

我们看到响应错误：`User validation failed: role: `admin1` is not a valid enum value for path `role`.`

#### 2.正常注册
![image](https://github.com/user-attachments/assets/c7f98210-a91c-476f-a32a-c1cff699a5a9)

#### 3.登录
![image](https://github.com/user-attachments/assets/88b365f3-8122-4629-a53e-405f5519c16b)

#### 4.使用获得的jwt token 进行进一步操作
![image](https://github.com/user-attachments/assets/a8fc019d-d9a4-4b21-a0d4-67fb3b58f5a1)

![image](https://github.com/user-attachments/assets/a216097f-4b6f-47a9-8a6f-7d8cf420f20e)

#### 5.切换角色
获得所有用户信息的只有`admin`才可以访问，我们重新注册一个`user`的角色
![image](https://github.com/user-attachments/assets/28ce849d-f2d7-47e4-8334-7c860f9d91d3)
登陆后使用jwt token进行获得所有用户信息的操作报错：
![image](https://github.com/user-attachments/assets/3e3ca205-05f6-4d49-8378-d599bb3c91d6)


## 结论

在本篇博客中，我们学习了如何使用 Nest.js 的守卫和修饰器进行权限管理，并将 MongoDB 集成到项目中进行数据持久化。通过这些工具，我们能够有效地管理用户权限，确保应用的安全性。在下一篇博客中，我们将继续探讨 Nest.js 的更多高级功能，敬请期待。

如果各位技术大佬有任何问题或建议，欢迎在评论区留言。

感谢阅读！

