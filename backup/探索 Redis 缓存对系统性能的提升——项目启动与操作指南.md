## 探索 Redis 缓存对系统性能的提升——项目启动与操作指南

#### 一、项目简介
Redis是一款高性能的键值存储数据库，以其出色的读写速度和丰富的数据结构著称，被广泛用作应用系统的缓存层。作为缓存，Redis通过将热点数据存储在内存中，显著降低数据库负载，提升响应速度，尤其适用于高并发场景。它支持字符串、哈希、列表等多种数据类型，可灵活应对不同业务需求，并提供了过期机制、持久化选项及集群部署能力，确保数据可靠性与扩展性。结合LRU淘汰策略，Redis能自动管理内存，避免溢出。此外，其原子操作和发布订阅功能进一步扩展了缓存之外的实时数据处理场景。无论是会话缓存、页面缓存还是数据库查询加速，Redis都能通过毫秒级访问大幅优化系统性能，是现代化架构中不可或缺的缓存解决方案。

在这个项目中，我们将探索 Redis 缓存对系统性能的提升作用。项目使用 `Gin` 框架搭建后端服务，结合 MySQL 数据库和 Redis 缓存，同时提供了一个前端仪表盘用于监控系统状态和进行压力测试。

项目地址：https://gitee.com/mbjdot/go_redis
博客地址：https://blog.tmdog114514.icu
#### 二、项目启动步骤

##### 1. 环境准备
确保你已经安装了 `Docker` 和 `Docker Compose`。如果还未安装，可以参考 [Docker 官方文档](https://docs.docker.com/get-docker/) 进行安装。

##### 2. 启动项目
- 打开终端，进入项目目录。

![Image](https://github.com/user-attachments/assets/51aab32a-540b-4809-9b4d-8aff5326115d)

- 执行以下命令启动项目：
```bash
docker-compose up -d --build
```
这个命令会构建项目所需的 Docker 镜像，并在后台启动 MySQL、Redis 和应用服务。`--build` 选项会确保重新构建镜像，以包含最新的代码更改。

##### 3. 验证项目启动
等待一段时间后，打开浏览器访问 `http://localhost:8080`，如果看到“数据库与系统监控仪表盘”页面，说明项目已经成功启动。

![Image](https://github.com/user-attachments/assets/bb4f50f6-c0d8-457f-ab9f-b531bbc292ca)

#### 三、核心代码描述
``` golang
// 用户信息处理函数
func getUserHandler(c *gin.Context) {
	id := c.Param("id")

	// 记录开始时间用于计算性能
	start := time.Now()

	//优先从Redis获取
	user, err := getUserFromCache(id)
	if err == nil {
		// 缓存命中
		atomic.AddUint64(&redisHits, 1)
		c.JSON(http.StatusOK, user)
		return
	}

	// 缓存未命中
	atomic.AddUint64(&redisMisses, 1)

	// 查询数据库
	atomic.AddUint64(&dbQueryCount, 1)
	user, err = getUserFromDB(id)

	// 计算查询耗时，记录慢查询
	elapsed := time.Since(start)
	if elapsed > 100*time.Millisecond {
		atomic.AddUint64(&slowQueryCount, 1)
		log.Printf("慢查询警告: 获取用户ID=%s 耗时: %v", id, elapsed)
	}

	if err != nil {
		atomic.AddUint64(&dbErrorCount, 1)
		if err == sql.ErrNoRows {
			c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// 将结果写入Redis（异步）
	go setUserToCache(user)

	c.JSON(http.StatusOK, user)
}
```
该函数的主要功能是处理用户信息获取请求，优先从 Redis 缓存中获取用户信息，如果缓存未命中，则从数据库中查询，并将查询结果异步写入 Redis 缓存。同时，代码还会记录缓存命中、缓存未命中、数据库查询次数、慢查询次数和数据库错误次数等指标。

``` golang
// 从数据库获取用户信息
func getUserFromDB(userID string) (*User, error) {
	query := `SELECT id, name, email FROM users WHERE id = ?`

	switch rand.Intn(100) {
	case 0: // 1%概率触发锁等待超时
		time.Sleep(1500 * time.Millisecond)
		return nil, &mysql.MySQLError{Number: 1205, Message: "Lock wait timeout"}
	case 1, 2: // 2%概率触发高负载延迟
		time.Sleep(time.Duration(300+rand.Intn(700)) * time.Millisecond) // 300-1000ms
	default: // 正常情况
		base := 15 + rand.Intn(25) // 15-40ms基础延迟
		time.Sleep(time.Duration(base) * time.Millisecond)
	}

	start := time.Now()
	row := db.QueryRow(query, userID)

	var user User
	err := row.Scan(&user.ID, &user.Name, &user.Email)

	// 记录数据库操作耗时
	elapsed := time.Since(start)
	if elapsed > 100*time.Millisecond {
		log.Printf("MySQL慢查询: 获取用户ID=%s 耗时: %v", userID, elapsed)
	}

	if err != nil {
		if err != sql.ErrNoRows {
			log.Printf("数据库查询错误: %v", err)
		}
		return nil, err
	}

	return &user, nil
}
```
我模拟了查新数据库所带来的高负载延迟与触发锁等待超时


#### 四、项目操作指南

##### 1. 系统监控
打开浏览器访问 `http://localhost:8080`，你会看到系统监控仪表盘。仪表盘会显示 Redis 和 MySQL 的状态、缓存命中率、数据库连接数等信息。你可以通过选择不同的刷新间隔，或者点击“刷新数据”按钮来更新页面数据。

![Image](https://github.com/user-attachments/assets/bfdf682d-17e6-4120-a521-e79315342a6b)

##### 2. 压力测试
在仪表盘的“压力测试”部分，你可以进行压力测试。输入测试的持续时间、并发数和测试端点，然后点击“开始测试”按钮，系统会开始进行压力测试。在测试过程中，你可以点击“停止测试”按钮来停止测试。测试结束后，系统会显示测试结果，包括总请求数、成功请求数、平均响应时间等指标。

![Image](https://github.com/user-attachments/assets/efc006c5-13e4-455f-b5b0-2005c45327a9)

我们发现30s的测试每秒请求次数达到了16212次，性能非常强悍。


我们直接修改代码
``` golang
// 用户信息处理函数
func getUserHandler(c *gin.Context) {
	id := c.Param("id")

	// 记录开始时间用于计算性能
	start := time.Now()

	////优先从Redis获取
	//user, err := getUserFromCache(id)
	//if err == nil {
	//	// 缓存命中
	//	atomic.AddUint64(&redisHits, 1)
	//	c.JSON(http.StatusOK, user)
	//	return
	//}

	// 缓存未命中
	atomic.AddUint64(&redisMisses, 1)

	// 查询数据库
	atomic.AddUint64(&dbQueryCount, 1)
	user, err := getUserFromDB(id)

	// 计算查询耗时，记录慢查询
	elapsed := time.Since(start)
	if elapsed > 100*time.Millisecond {
		atomic.AddUint64(&slowQueryCount, 1)
		log.Printf("慢查询警告: 获取用户ID=%s 耗时: %v", id, elapsed)
	}

	if err != nil {
		atomic.AddUint64(&dbErrorCount, 1)
		if err == sql.ErrNoRows {
			c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
			return
		}
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// 将结果写入Redis（异步）
	//go setUserToCache(user)

	c.JSON(http.StatusOK, user)
}
```
注释掉redis相关逻辑

重新部署，先关闭卸载容器`docker-compose down -v `，重新构建`docker-compose up -d --build`

进入仪表盘并测试

![Image](https://github.com/user-attachments/assets/1fcb06f5-f299-432a-8eca-e769e5a9c1f2)

发现30s测试只能请求27407次，性能大幅降低

#### 五、总结
通过启动和操作这个项目，我们可以直观地看到 Redis 缓存对系统性能的提升作用。在后续的博客中，我们将深入分析项目的代码，探讨 Redis 缓存的实现原理和优化策略。

