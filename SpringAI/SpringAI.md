# 一、概述

## 1、简介

- 在当今这样一个快速发展的技术时代，人工智能（AI）已经成为各行各业的一种标配。而作为一款主流的 Java 应用开发框架 Spring，肯定会紧跟时代的潮流，所以，推出了 Spring AI 框架。

- 官网：https://spring.io/projects/spring‑ai



## 2、官网描述

- Spring AI 是一个 AI 工程领域的应用程序框架；
- Spring AI 是 AI 工程的应用框架。其目标是将 Spring 生态系统设计原则（如可移植性和模块化设计）应用于 AI 领域，并促进使用 POJO 作为应用程序的构建块到 AI 领域。
- 它的目标是将 Spring 生态系统的设计原则应用于 AI 领域，比如 Spring 生态系统的可移植性和模块化设计，并促进使用 POJO 作为应用程序的构建块到 AI 领域；
- Spring AI 的核心是提供了开发 AI 大模型应用所需的基本抽象模型，这些抽象拥有多种实现方式，使得开发者可以用很少的代码改动就能实现组件的轻松替换；
- 简言之，Spring AI 是一个 AI 工程师的应用框架，它提供了一个友好的 API 和开发 AI 应用的抽象，旨在简化 AI 大模型应用的开发工作。



## 3、主要功能

- 第一、对主流 AI 大模型供应商提供了支持，比如：OpenAI、DeepSeek、Microsoft、Ollama、Amazon、Google HuggingFace 等。
- 第二、支持 AI 大模型类型包括：聊天、文本到图像、文本到声音等。
- 第三、支持主流的 Embedding Models（嵌入模型）和向量数据库，比如：Azure Vector Search、Chroma、Milvus、Neo4j、Redis、PineCone、PostgreSQL/PGVector 等。
- 第四、把 AI 大模型输出映射到简单的 Java 对象 (POJOs) 上。
- 第五、支持了函数调用（Function calling）功能。
- 第六、为数据工程提供 ETL（数据抽取、转换和加载）框架。
- 第七、支持 Spring Boot 自动配置和快速启动，便于运行 AI 模型和管理向量库



# 二、快速入门

## 1、引入依赖

- JDK最少17以上
- springboot最少3.3以上

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>spring-ai-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.ai</groupId>
                <artifactId>spring-ai-bom</artifactId>
                <version>1.1.8</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>3.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <version>3.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-openai</artifactId>
            <version>1.1.8</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>
</project>
~~~



## 2、配置文件

~~~yml
spring:
  ai:
    openai:
      base-url: https://api.deepseek.com
      api-key: sk-3aa699fb50004c23bcb71282481d3a68
      chat:
        options:
          model: deepseek-v4-pro
          temperature: 0.7
~~~

- temperature 参数用于控制生成文本的多样性。具体来说：
  - 值越高，生成的文本越多样化，但也可能包含更多的随机性和不可预测的内容
  - 值越低，生成的文本越接近于确定性的结果，即生成的文本会更加一致和可预测



## 3、启动类

~~~java
package org.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringAIMainApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringAIMainApplication.class, args);
    }
}
~~~



## 4、controller

~~~java
package org.example.controller;

import jakarta.annotation.Resource;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SpringAIController {

    @Resource
    private OpenAiChatModel chatModel;

    @GetMapping("/hello")
    public String generate(@RequestParam(value = "message", defaultValue = "hello")
                           String message) {
        String response = this.chatModel.call(message);
        System.out.println("response : "+response);
        return response;
    }
}
~~~

- 测试访问：http://localhost:8080/hello?message=你好



# 三、聊天模型

## 1、概述

Spring AI 的聊天模型 API 为开发者提供了一条便捷通道，能够将强大的 AI 驱动的聊天完成功能无缝集成到各类应用中。借助预先训练的语言模型，如广为人知的 GPT，它能够依据用户输入生成自然流畅、类人化的回复。这一 API 不仅工作机制高效，而且设计理念极为先进，旨在实现简单易用与高度可移植性，让开发者能以极少的代码改动在不同 AI 模型间自由切换，充分契合 Spring 框架一贯秉持的模块化与可互换性原则



## 2、ChatClient接口

- ChatClient 是一个接口，它定义了一个与聊天服务交互的客户端。这个接口主要用于创建聊天客户端对象，设置请求规范，以及发起聊天请求