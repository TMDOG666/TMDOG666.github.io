# TMDOG的微服务之路_08——使用Docker部署NestJS微服务

## 博客地址：[[TMDOG的博客](https://blog.tmdog114514.icu/)](https://blog.tmdog114514.icu)

在上一篇博客中，我们探讨了如何使用 NestJS 创建一个简单的微服务架构。为了将这些微服务部署到生产环境，我们可以使用 Docker 来打包和管理这些服务。本篇博客将详细介绍如何使用 Docker 和 Docker Compose 部署我们的 NestJS 微服务项目。

## 1. 为什么选择 Docker？

Docker 是一个开源的平台，允许开发者自动化地部署应用程序到轻量级、可移植的容器中。这些容器包含了运行应用所需的所有内容，包括代码、依赖项和系统库。因此，使用 Docker 可以确保在不同环境中运行应用的一致性，同时简化了部署和扩展的过程。

### Docker 的主要优点：

- **环境一致性**：Docker 容器确保在开发、测试和生产环境中运行的应用程序保持一致，减少了“在我这里可以运行”的问题。
- **隔离性**：每个 Docker 容器在独立的环境中运行，避免了依赖冲突和资源争用。
- **可移植性**：Docker 容器可以在任何支持 Docker 的平台上运行，从而提高了应用的可移植性。
- **轻量级**：相比传统的虚拟机，Docker 容器占用资源更少，启动速度更快。

## 2. 使用 Docker 部署 NestJS 微服务

在这一部分，我们将通过 Dockerfile 和 Docker Compose 来打包和部署我们在上一篇博客中创建的三个微服务：`api-gateway`、`service_1` 和 `service_2`。

### 2.1 编写 Dockerfile

我们为每个微服务编写了一个 Dockerfile，以便打包成 Docker 镜像。下面是三个模块的 Dockerfile 示例：

#### API Gateway 的 Dockerfile

```dockerfile
# 使用官方的 Node.js 作为基础镜像
FROM node

# 创建工作目录
WORKDIR /usr/src/app

# 复制 package.json 和 package-lock.json 文件
COPY package*.json ./

# 安装依赖
RUN npm install --production

# 复制项目的所有文件到工作目录
COPY . .

# 编译 TypeScript
RUN npm run build

# 暴露 API 网关的端口
EXPOSE 3000

# 运行 API 网关
CMD ["npm", "run", "start:prod"]
```

#### Service_1 的 Dockerfile

```dockerfile
# 使用官方的 Node.js 版本作为基础镜像
FROM node

# 创建工作目录
WORKDIR /usr/src/app

# 复制 package.json 和 package-lock.json 文件
COPY package*.json ./

# 安装依赖
RUN npm install --production

# 复制项目的所有文件到工作目录
COPY . .

# 编译 TypeScript
RUN npm run build

# 暴露服务端口
EXPOSE 3000

# 运行服务
CMD ["npm", "run", "start:prod"]
```

#### Service_2 的 Dockerfile

```dockerfile
# 使用官方的 Node.js 版本作为基础镜像
FROM node

# 创建工作目录
WORKDIR /usr/src/app

# 复制 package.json 和 package-lock.json 文件
COPY package*.json ./

# 安装依赖
RUN npm install --production

# 复制项目的所有文件到工作目录
COPY . .

# 编译 TypeScript
RUN npm run build

# 暴露服务端口
EXPOSE 3000

# 运行服务
CMD ["npm", "run", "start:prod"]
```

### 2.2 编写 Docker Compose 文件

Docker Compose 允许我们通过一个配置文件同时管理多个 Docker 容器。我们为项目编写了一个 `docker-compose.yaml` 文件，以启动所有微服务。

```yaml
version: '3'
services:
  service_1:
    build: ./microservice/service_1
    ports:
      - "50001:3000"
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:50001/health"]
      interval: 30s
      retries: 3
      start_period: 10s
      timeout: 10s

  service_2:
    build: ./microservice/service_2
    ports:
      - "50002:3000"
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:50002/health"]
      interval: 30s
      retries: 3
      start_period: 10s
      timeout: 10s

  api-gateway:
    build: ./api-gateway
    ports:
      - "3000:3000"
    depends_on:
      - service_1
      - service_2
    networks:
      - microservices-network

networks:
  microservices-network:
    driver: bridge
```

#### 解释：

- `service_1` 和 `service_2` 分别映射到主机的 50001 和 50002 端口，并通过健康检查来确保服务的可用性。
- `api-gateway` 作为微服务的入口，依赖于 `service_1` 和 `service_2`，并且通过端口 3000 与外部通信。
- `microservices-network` 是一个自定义的桥接网络，用于容器之间的通信。

### 2.3 使用脚本简化部署

为了简化在不同操作系统上的部署，我们编写了 Windows 和 Linux 的脚本：

#### Windows 脚本 (`start.bat`)

```batch
@echo off

REM Navigate to the root directory
cd /d %~dp0

REM Initialize service_1
echo Initializing Service 1...
cd microservice\service_1
call npm install
call npm run build

REM Initialize service_2
echo Initializing Service 2...
cd ..\service_2
call npm install
call npm run build

REM Initialize api-gateway
echo Initializing API Gateway...
cd ..\..\api-gateway
call npm install
call npm run build

REM Return to the root directory
cd /d %~dp0

REM Execute Docker Compose
echo Starting Docker containers...
call docker-compose up -d

echo Deployment completed.
pause
```

#### Linux 脚本 (`start.sh`)

```bash
#!/bin/bash

# Navigate to the script's directory
cd "$(dirname "$0")"

# Initialize service_1
echo "Initializing Service 1..."
cd microservice/service_1
npm install
npm run build

# Initialize service_2
echo "Initializing Service 2..."
cd ../service_2
npm install
npm run build

# Initialize api-gateway
echo "Initializing API Gateway..."
cd ../../api-gateway
npm install
npm run build

# Return to the root directory
cd ../../[nestjs_microservice_quickstart](https://github.com/TMDOG666/nestjs_microservice_quickstart)

# Execute Docker Compose
echo "Starting Docker containers..."
docker compose up -d

echo "Deployment completed."
```

在根目录的 `package.json` 文件中，我们定义了启动命令以方便操作：

```json
{
  "name": "nestjs_microservice_quickstart",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "win": "start.bat",
    "linux": "chmod +x start.sh && start.sh"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "description": ""
}
```

#### 运行命令：

- Windows：`npm run win`
- Linux：`npm run linux`

### 2.4 部署与测试

通过上述步骤，我们可以在本地环境中轻松部署和启动 NestJS 微服务。输入启动命令后，等待自动执行脚本即可。
等所有脚本跑完之后：
输入：

```bash
docker ps
```

我们发现3个容器已经创建好了：
![image](https://github.com/user-attachments/assets/79033137-d075-4a7c-844a-db6f23a9495e)


浏览器中测试：
![image](https://github.com/user-attachments/assets/512a2136-3815-4c30-a0fc-d27fc50f09ba)

![image](https://github.com/user-attachments/assets/1cccaa78-7b5e-4fc4-af24-c11b4bf4f54e)

![image](https://github.com/user-attachments/assets/5cf66613-abb5-43fc-86f0-114ced5cb75e)


### 3. 总结

在本篇博客中，我们探讨了如何使用 Docker 和 Docker Compose 部署 NestJS 微服务架构。通过 Docker，将微服务打包为容器，并通过 Docker Compose 管理多个容器的启动，使得整个部署过程变得简单且高效。

如有任何问题或建议，欢迎在评论区留言。下一篇博客中，我们将继续探讨微服务架构的更多高级实践，敬请期待。

感谢阅读！

## 项目源码

项目源码已经上传至 GitHub，欢迎查看：[nestjs_microservice_quickstart](https://github.com/TMDOG666/nestjs_microservice_quickstart)
