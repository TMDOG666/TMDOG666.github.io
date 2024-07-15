## 学习构建邮箱验证系统：

突然想尝试一下怎么实现一个邮箱验证的功能

### 遇见技术爬爬虾

一切都始于在B站上看到了技术爬爬虾（一个非常有名的技术 UP 主）的视频。在视频中，爬爬虾详细介绍了如何通过 Cloudflare（赛博活佛） 提供的邮件路由服务获取企业级邮箱。这对我来说是一个全新的知识领域，决定研究研究。爬爬虾还推荐一个名为 Resend 的邮件发送服务商。Resend 提供了简洁易用的 API，使得邮件发送变得非常简单和高效。

根据视频的教程我成功通过官方的代码示例给自己的邮箱发送邮件
![image](https://github.com/user-attachments/assets/64116c47-d738-4a67-abdd-f01993682f1a)
![image](https://github.com/user-attachments/assets/e93f3f3c-16ba-4087-86bd-59e9ef055b78)

### 构建邮箱验证系统的初步尝试

#### 在掌握了 Cloudflare 和 Resend 的基本知识后，我决定用它们作为基础，构建一个邮箱验证系统。
#### 开始的想法就是：
1、用户先通过邮箱参数给后端，后端随机生成一个6位的验证码发送给用户邮箱
2、用户通过邮箱和验证码再次请求，后端去验证

但会出现一些问题，验证码后端存储问题，当然可以存在内存里面，但数据的处理就比较复杂也不灵活，万一服务端炸了数据都没了
最后想到redis，通过键值对去存邮箱验证码，就只需要对redis去做操作方便很多，还可以设定过期的时间很符合邮件的业务需求


#### 环境搭建

首先，我配置了项目的环境变量。在项目根目录下创建 `.env` 文件，并添加以下配置（redis的配置、resend api 配置、自己发送邮箱的配置）：

```env
REDIS_HOST=your_redis_host
REDIS_PORT=your_redis_port
REDIS_PASSWORD=your_redis_password
REDIS_DB=your_redis_db_number
EMAIL_HOST=your_email_host
RESEND_API_KEY=your_resend_api_key
```

然后搭建 express 的脚手架

#### 项目结构

为了保持代码的整洁和可维护性，设计了下面项目结构：

- `controllers/emailVerificationController.js`: 处理验证码发送和验证的控制器。
- `routes/emailVerificationRoutes.js`: 定义 API 路由。
- `utils/redisUtils/redisEmailUtil.js`: 包含与 Redis 相关的邮箱验证实用工具函数。
- `utils/randomCodeUtils/emailVerifyCodeUtil.js`: 生成随机验证码的实用工具函数。
- `utils/emailUtils/verifyCodeSenderUtil.js`: 发送验证邮件的实用工具函数。
- `utils/emailUtils/emailSenderBaseUtil.js`: 基础邮件发送功能。

### 初步成果：
![image](https://github.com/user-attachments/assets/4714414d-3cd0-4344-8b2f-9c311c0bce41)
![image](https://github.com/user-attachments/assets/9f1dd2e3-c542-4b22-a2f8-808139c075a0)
![image](https://github.com/user-attachments/assets/21f75900-524c-4510-97eb-02ad593d425f)


### 优化和完善

在实现了基本功能后，我开始优化和完善项目，以提高系统的安全性和用户体验。


#### 错误计数和邮箱锁定机制

为了防止暴力破解和滥用，我加入了错误计数和邮箱锁定机制。具体实现逻辑如下：

1. **发送验证码邮件**：
   - 通过 POST 请求发送验证码邮件，验证邮箱的接口。
   - 如果邮箱已被锁定，返回 `423 Locked` 状态码。
   - 如果验证码已发送并在10分钟内不能再次发送，返回 `429 Too Many Requests` 状态码。

2. **验证邮箱验证码**：
   - 通过 POST 请求验证用户输入的验证码。
   - 如果邮箱已被锁定，返回 `423 Locked` 状态码。
   - 如果验证码已过期，返回 `410 Gone` 状态码。
   - 如果验证码错误且尝试次数超过限制，返回 `400 Bad Request` 状态码，并锁定邮箱（10分钟后自动解锁）。



### 最终效果
![image](https://github.com/user-attachments/assets/21f7bab7-f67e-4d30-b8b2-9192cd7190c8)
![image](https://github.com/user-attachments/assets/856684b9-d9ea-4faa-99a5-08f743dcb41e)
![image](https://github.com/user-attachments/assets/ec52ad8f-fcd9-4a4d-9ad3-ad9b8b2dc681)


经过一系列的优化和完善，邮箱验证系统能稳定运行，还具备了一定的用户体验和安全机制。这段构建过程让我对邮件服务和系统设计有了更深的理解。

​
### 总结
通过这次学习和实践，我不仅掌握了如何使用 Cloudflare 和 Resend 提供的服务，还学会了如何设计和优化一个邮箱验证系统。感谢 B站技术爬爬虾的视频，视频都非常有意思。

如果你也对这个项目感兴趣，欢迎访问我的查看完整代码，并期待你提出宝贵的意见和建议！


gitee：[码云仓库](https://gitee.com/mbjdot/email-resend-demo.git)

github：  [GitHub 仓库](https://github.com/TMDOG666/email-resend-demo) 

​