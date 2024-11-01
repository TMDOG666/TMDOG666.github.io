# TMDOG的Gin学习笔记_01——初识Gin框架

## 博客地址：[[TMDOG的博客](https://blog.tmdog114514.icu/)](https://blog.tmdog114514.icu)

**作者自述：** 
	停更太久了，是因为开学了课太多了，并且我一直在准备上篇文章的内容正在coding，就先搁置了更新博客QAQ，现在终于闲下来了。

​	说实话挺久没有进行golang的编程了，但以前上课系统学习了golang并且期末项目就是使用beego实现的[[CRM管理系统](https://gitee.com/mbjdot/biego-final-project)](https://gitee.com/mbjdot/biego-final-project)（可以康康），所以上手gin还是比较快的。

## 学习目标

使用Gin框架实现一个微服务，并整合到上篇博客的项目中。  
**预期使用的技术栈：** Gin、GORM、Postgres、gRPC、Docker

## Gin快速入门

### 1. 安装Gin

首先，确保你已安装Go。然后使用以下命令安装Gin：

```bash
go get -u github.com/gin-gonic/gin
```

### 2. 创建一个简单的Web应用

在你的项目目录下，创建一个新的Go文件，例如`main.go`，并添加以下代码：

```go
package main

import (
	"github.com/gin-gonic/gin"
)

func main() {
	// 创建一个默认的Gin路由
	r := gin.Default()

	// 设置一个简单的GET路由
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "hello gin",
		})
	})

	// 启动服务器，监听在8080端口
	r.Run(":8080")
}
```

### 3. 运行你的应用

在终端中，运行以下命令启动你的应用：

```bash
go run main.go
```

### 4. 测试API

打开你的浏览器或使用工具（如Postman或curl），访问 `http://localhost:8080/`。你应该会看到以下JSON响应：

![image-20241101093157409](https://github.com/user-attachments/assets/450d77dd-577b-44dc-8c7e-c5f4cfd6695e)

## Gin整合GORM



### 1. 安装GORM依赖

使用`go get`命令安装GORM和Postgres驱动的依赖：

```bash
go get gorm.io/gorm
go get gorm.io/driver/postgres
```

### 2. 初始化数据库连接并构建表模型

以用户表为例进行初始化：

```go
// User模型
type User struct {
	ID    uint   `gorm:"primaryKey;autoIncrement"`
	Name  string `gorm:"not null"`
	Email string `gorm:"not null"`
}

var db *gorm.DB

func init() {
	// 数据库连接信息
	dsn := "host=localhost user=postgres password=123456 dbname=postgres port=5432 sslmode=disable TimeZone=Asia/Shanghai"
	var err error
	// 连接数据库
	db, err = gorm.Open(postgres.Open(dsn), &gorm.Config{})
	if err != nil {
		panic("failed to connect to database")
	}
	// 自动迁移（将在数据库中创建表）
	db.AutoMigrate(&User{})
}
```

### 3. 编写接口

实现对用户表的增删改查接口：

```go
// 创建用户
r.POST("/users", func(c *gin.Context) {
	var user User
	if err := c.ShouldBindJSON(&user); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	db.Create(&user)
	c.JSON(http.StatusCreated, user)
})

// 获取所有用户
r.GET("/users", func(c *gin.Context) {
	var users []User
	db.Find(&users)
	c.JSON(http.StatusOK, users)
})

// 根据ID获取用户
r.GET("/users/:id", func(c *gin.Context) {
	var user User
	id := c.Param("id")
	if err := db.First(&user, id).Error; err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
		return
	}
	c.JSON(http.StatusOK, user)
})

// 更新用户
r.PUT("/users/:id", func(c *gin.Context) {
	var user User
	id := c.Param("id")
	if err := db.First(&user, id).Error; err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
		return
	}
	if err := c.ShouldBindJSON(&user); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	db.Save(&user)
	c.JSON(http.StatusOK, user)
})

// 删除用户
r.DELETE("/users/:id", func(c *gin.Context) {
	id := c.Param("id")
	if err := db.Delete(&User{}, id).Error; err != nil {
		c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
		return
	}
	c.Status(http.StatusNoContent)
})
```

### 4. 测试

使用Postman或curl测试API：

- **创建用户**:

  ```bash
  curl -X POST http://localhost:8080/users -H "Content-Type: application/json" -d '{"name":"Alice", "email":"alice@example.com"}'
  ```

- **获取所有用户**:

  ```bash
  curl -X GET http://localhost:8080/users
  ```

- **获取单个用户**:

  ```bash
  curl -X GET http://localhost:8080/users/1
  ```

- **更新用户**:

  ```bash
  curl -X PUT http://localhost:8080/users/1 -H "Content-Type: application/json" -d '{"name":"Alice Updated", "email":"alice.updated@example.com"}'
  ```

- **删除用户**:

  ```bash
  curl -X DELETE http://localhost:8080/users/1
  ```

### 5. 测试结果

![image-20241101112734821](https://github.com/user-attachments/assets/71ce4d86-c427-4a30-b7c9-f16226629ab4)


## 总结

通过本篇学习笔记，我们初步了解了gin的基本用法，包括如何安装、创建简单的Web应用以及整合gorm进行数据库操作。我们实现了一个用户管理的RESTful API，能够完成基本的增删改查功能。我们发现gin是一个非常简洁的一个框架，几行代码就可以构建一个Web应用，和express.js有着异曲同工之妙。接下来我会继续分享我的学习笔记，尽请期待。