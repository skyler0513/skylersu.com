---
title: MCP协议
date: 2025-08-07T20:56:49+08:00
lastmod: 2025-08-07T20:56:49+08:00
cover: https://img.sysummery.top/mcp_cover.webp
avatar: https://img.sysummery.top/coffee.jpg
authorlink: https:www.skylersu.com
categories:
  - AI
tags:
  - MCP
---
MCP（Model Context Protocol，模型上下文协议） ，2024年11月底，由 Anthropic 推出的一种开放标准，旨在统一大模型与外部数据源和工具之间的通信协议。MCP 的主要目的在于解决当前 AI 模型因数据孤岛限制而无法充分发挥潜力的难题，MCP 使得 AI 应用能够安全地访问和操作本地及远程数据，为 AI 应用提供了连接万物的接口。
<!--more-->
### 为什么要有MCP协议
#### 大模型需要API
大模型类似于人的大脑，经过训练后可以思考。但是有些问题是思考不出来的，比如：北京今天天气如何？
![](https://img.sysummery.top/mcp%E4%B8%8D%E4%BD%BF%E7%94%A8%E5%B7%A5%E5%85%B7.jpg)

为了扩展大模型的能力，必须让它能够调用外部的API获取额外信息。还是上面同样的问题，只不过我在问问题的时候同时提供给大模型一个**高德地图**服务，因为这个服务中有根据城市名称获取天气的接口
![](https://img.sysummery.top/%E9%80%89%E5%8F%96mcp%E5%B7%A5%E5%85%B7.jpg)
这次得到的结果就完全不一样了
![](https://img.sysummery.top/mcp%E8%B0%83%E7%94%A8%E7%BB%93%E6%9E%9C.jpg)
从图中可以看出，大模型先是调用了**高德地图:map_whether**之后才给用户产生的结果。
有了调用外部的API的能力，大模型就如虎添翼了！

#### Function Call
上面通过API获取北京的天气就是个function call的例子，或者tool call的例子，function和tool是一个概念。
![](https://img.sysummery.top/function-calling.png)
1. 用户发出promot给AI Application
2. AI Application把用户的promot和函数的定义一起传递给大模型
3. 大模型根据用户的问题和函数定义告诉AI Application应该调用哪个函数以及参数是什么
4. AI Application根据大模型返回的结果去调用外部API获取函数调用结果
5. AI Application将函数调用的结果作为参数再次请求大模型
6. 大模型收到函数调用的结果后输出这次交互的最终结果


从上面的过程我们可以看出AI Application与大模型交互了2次，分别是步骤2和步骤5。在这2步中AI Application与大模型交互的协议是由大模型来决定的，而不同的大模型是不同的公司出品的，所以一个AI Application要想同时集成多个大模型的话，对于同一个外部API可能需要多个adaptor兼容。

#### 统一的协议
因此需要一个协议来统一不同的大模型调用工具的方式，这点很类似于HTTP协议在Web中的作用。这个协议就是[MCP协议](https://modelcontextprotocol.io/overview)。目前该协议还在不断地更新中。它是由Claude模型的母公司Anthropic提出的。MCP协议是建立在不同厂商的大模型与MCP Servers之间的桥梁！

#### MCP Server
一个MCP server可以有多个工具。
通过上面的步骤5可以看出，真正与MCP server交互的是AI Application，他们之间是通过client/server模式进行通信的，通信的协议又分为两种：

1. **Stdio**: AI Application与mcp server在同一台机器上，他们之间通过标准输入和输出通信。
2. **Streamable HTTP**: AI Application与mcp server之间通过http协议通信，并且server可以不断的向client流式输送信息。

### MCP中的重要概念
这部分内容是基于
#### 角色
在MCP协议中，角色有3个
* AI Application，又叫MCP Host
* MCP Client
* MCP Server

#### 服务端

* Tools，用来提供一个的功能，这个功能可以很简单也可以很复杂
* Resources，用来提供一个静态的资源，比如图片、文档等
* Prompts，用来提供请求大模型的promot

#### 客户端

* Sampling，是一个工作流，它用来定义server怎么通过client来与AI Application中集成的大模型交互的
![](https://img.sysummery.top/mcp_sampling_flow.jpg)

* Roots，如果server需要访问文件系统，那么client在请求server的时候会通过Roots限制server访问文件系统的边界
* Elicitation，同样是一个工作流，server在执行的过程中可以通过这个工作流来向用户请求额外的信息
![](https://img.sysummery.top/mcp_elicitation_flow.jpg)
