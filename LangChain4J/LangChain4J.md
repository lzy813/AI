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



## 2、入门例子

- POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.0</version>
        <relativePath/>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>langchin4j-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <langchain4j.version>1.19.0</langchain4j.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--langchain4j-open-ai 基础-->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-open-ai</artifactId>
        </dependency>
        <!--langchain4j 高阶-->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j</artifactId>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>dev.langchain4j</groupId>
                <artifactId>langchain4j-bom</artifactId>
                <version>${langchain4j.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
~~~

- 主启动类

~~~java
package org.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class LangChain4JMainApplication {
    public static void main(String[] args) {
        SpringApplication.run(LangChain4JMainApplication.class, args);
    }
}
~~~

- 模型配置类

~~~java
package org.example.config;

import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.openai.OpenAiChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 模型配置
 */
@Configuration
public class LLMConfig {
    @Bean
    public ChatModel chatModel() {
        return OpenAiChatModel.builder()
                .apiKey(System.getenv("ALIQWEN_API"))  
                .modelName("deepseek-v4-pro")
                .baseUrl("https://api.deepseek.com")
                .build();
    }
}
~~~

- 编写controller

~~~java
package org.example.controller;

import dev.langchain4j.model.chat.ChatModel;
import jakarta.annotation.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LangChai4JController {

    @Resource
    private ChatModel chatModel;

    @GetMapping("/chat")
    public String chat(@RequestParam(value = "message", defaultValue = "你好") String message) {
        String response = chatModel.chat(message);
        System.out.println("response : " + response);
        return response;
    }

}
~~~

- 测试：http://localhost:8080/chat?message=你好