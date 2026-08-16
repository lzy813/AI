# 一、概述

## 1、是什么

- <font color="red">**等价于 LangChain For Java，专门为 Java程序员而生的LangChain大模型应用开发框架**</font>
- 随着人工智能（AI）技术的迅猛发展，越来越多的开发者开始将目光投向 AI 应用的开发。然而，目前市场上大多数 AI 框架和工具如 LangChain、PyTorch 等主要支持 Python，而 Java 开发者常常面临工具缺乏和学习门槛较高的问题，但是不用担心，任何一个框架 / XXX 云服务器，想要大面积推广，应该不会忘记庞大的 Spring 社区和 Java 程序员

![定位](图片\定位.png)

- 官网：https://docs.langchain4j.info/

![官网](图片\官网.png)

## 2、与传统开发类比

- 模型开发和常规开发层级对比
  - Prompt：好比UI层，进行用户交互
  - LangChain4J, Spring AI：好比Controller，调用AI大模型
  - 各类AI大模型：好比Service，提供人工智能服务
  - 向量数据库：好比Mapper，进行数据存储

![能干什么](图片\能干什么.png)



# 二、入门例子

## 1、前置约定

- LangChain4J支持的大模型：https://docs.langchain4j.info/integrations/language-models

![大模型调用三件套](图片\大模型调用三件套.png)