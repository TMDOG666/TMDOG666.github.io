# 语言大模型进行意图分析驱动后端实践

## 项目概述

该项目通过语言大模型，通过**分析用户意图**、**拆分任务**、**构建API调用链**来驱动后端实践。

以一个简单的教务系统后端为例，将教务系统后端接口文档作为知识库，精确分析用户意图，自动执行业务流程。

使得用户可以在聊天交互页面实现较为复杂的业务操作，简化用户操作，并与后端接口解耦，具有良好的灵活性。

### **操作示例**

<img width="734" alt="Image" src="https://github.com/user-attachments/assets/2a1ffceb-85d4-4368-9123-3e1eb137a5ef" />

## 核心架构

<img width="654" alt="Image" src="https://github.com/user-attachments/assets/d0a157e0-6e1e-404e-a381-22d6138adf9a" />

1. **意图分析层** - 核心处理用户输入的自然语言意图
2. **知识库检索层** - 通过RAG技术检索相关API文档
3. **任务分解层** - 将复杂请求拆分为可执行的API调用序列
4. **执行引擎层** - 实际调用后端API并处理响应

## 意图分析层深度解析

### 1. 意图识别技术栈

	**Prompt工程**：精心设计的提示模板引导模型准确理解意图
	**RAG（检索增强生成）**：将后端API文档和业务调用逻辑文档作为知识库


### 2. 多阶段意图分析流程

1. **初级意图分类**：
   • 将用户输入内容并检索检索知识库，分析意图拆分任务
   • 使用轻量级模型（Qwen2.5-14b）提高响应速度

2. **细粒度意图解析**：
   • 将拆分步骤检索知识库获取精确的API信息
   • 根据意图分析结果生成API调用计划链

### 3. 知识库增强的意图分析

• **API文档向量化**：
  • 使用嵌入模型进行文本向量化
  • 使用ChromaDB存储和检索API文档片段
  • 查询与用户意图最相关的API描述

## 执行引擎优化

1. **智能重试机制**：
   • 处理API失败情况

2. **响应后处理**：
   • 自然语言生成

## 性能优化策略

1. **意图缓存**：
   • 缓存常见意图的解析结果


## 总结与展望

**优点**

- 该架构通过多层次的意图分析，实现了从自然语言到系统API的精准转换

- 意图分析并不依赖重量级参数的模型，即使是参数规模较小的模型也可以实现功能

- 与后端解耦，不需要为意图分析层修改后端逻辑，仅需提供API文档与操作逻辑文档作为知识库

**缺陷**

- 毕竟是一个简易的DEMO，并不支持上下文，如果支持上下文可以实现更复杂、更流畅的用户交互流程

- 性能问题，使用的是硅基流动大模型服务商，由于响应延迟，处理用户输入延迟很大

- 交互过于简单，博主想的是能不能和前端联动，实现意图分析驱动前端，就可以实现更复杂的业务逻辑



这种意图驱动的后端实践为构建智能交互系统提供了可扩展的框架，特别适合需要将自然语言转换为复杂系统操作的场景。