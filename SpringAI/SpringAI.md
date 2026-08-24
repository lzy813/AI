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

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.1.0</version>
        <relativePath/>
    </parent>

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
                <version>2.0.1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-starter-model-openai</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
~~~



## 2、配置文件

~~~yml
spring:
  ai:
    OpenAI:
      base-url: https://ws-zks05buw2zy3a3e4.cn-beijing.maas.aliyuncs.com/compatible-mode/v1
      api-key: sk-ws-H.EYHLXID.94g4.MEYCIQDxvKfua7F_KDMECg_myE86M2VlVnWjFW_MdNY_ZVHE7gIhAPKwAd2FGskYlTDlm3iNyDm-gDLb70sqRLK5SN9iWEBq
      chat:
        options:
          model: qwen3.7-plus
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

    @GetMapping("/chat")
    public String chat(@RequestParam(value = "prompt", defaultValue = "hello")
                           String prompt) {
        String response = chatModel.call(prompt);
        System.out.println("response : "+response);
        return response;
    }
}
~~~

- 测试访问：http://localhost:8080/chat?prompt=hello



# 三、聊天模型

## 1、概述

Spring AI 的聊天模型 API 为开发者提供了一条便捷通道，能够将强大的 AI 驱动的聊天完成功能无缝集成到各类应用中。借助预先训练的语言模型，如广为人知的 GPT，它能够依据用户输入生成自然流畅、类人化的回复。这一 API 不仅工作机制高效，而且设计理念极为先进，旨在实现简单易用与高度可移植性，让开发者能以极少的代码改动在不同 AI 模型间自由切换，充分契合 Spring 框架一贯秉持的模块化与可互换性原则



## 2、ChatClient接口

### 2.1 实现简单的对话

- ChatClient 是一个接口，它定义了一个与聊天服务交互的客户端。这个接口主要用于创建聊天客户端对象，设置请求规范，以及发起聊天请求
- ChatClient 接口提供了构建和配置聊天客户端对象的灵活性，以及发起和处理聊天请求的能力。
- 用户可以通过 ChatClient.Builder 来定制客户端的行为，然后使用 prompt() 和 prompt(Prompt prompt) 方法设置请求规范，最后通过 call() 方法发起聊天请求。

~~~java
package org.example.controller;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class SpringAIController {

    private final ChatClient chatClient;
    public SpringAIController(ChatClient.Builder chatClientBuilder) {
        this.chatClient = chatClientBuilder.build();
    }

    @GetMapping("/chat")
    public String chat(@RequestParam(value = "prompt", defaultValue = "hello")
                       String prompt) {
        //prompt:提示词
        return this.chatClient.prompt()
                //用户输入的信息
                .user(prompt)
                //请求大模型
                .call()
                //返回文本
                .content();
    }
}
~~~

- 测试：http://localhost:8080/chat?prompt=hello



### 2.2 系统提示词，实现角色预设

- 配置类

~~~java
package org.example.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ChatClientConfigs {

    @Bean
    public ChatClient chatClient(OpenAiChatModel chatModel, ChatMemory chatMemory) {
        return ChatClient.builder(chatModel)
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .defaultSystem("你是文档整理助手，可以生成文档、总结文档内容、解析文档属性等等相关功能业务.")
                .build();
    }
}
~~~

- controller使用

~~~java
package org.example.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequiredArgsConstructor // 为所有 final 未初始化字段生成构造器
public class SpringAIController {

    private final ChatClient chatClient;

    @GetMapping("/chat")
    public String chat(@RequestParam(value = "prompt", defaultValue = "hello")
                       String prompt) {
        // prompt:提示词
        return this.chatClient.prompt()
                // 用户输入的信息
                .user(prompt)
                // 请求大模型
                .call()
                // 返回文本
                .content();
    }
}
~~~

- 测试：http://localhost:8080/chat?prompt=你是谁
- 返回：您好！我是您的**文档整理助手**。 我可以为您提供以下文档相关的帮助： - **生成文档**：根据您的需求和主题，快速起草、撰写和生成各类文档（如报告、总结、方案等）。 - **总结文档内容**：帮您快速阅读长篇文章或文件，提取核心要点，生成精炼的摘要或大纲。 - **解析文档属性**：分析文档的结构、字数、格式特征等属性信息。 - **文档优化与处理**：包括文档润色、信息提取、格式调整等。 请问今天有什么文档相关的工作需要我协助您处理吗？随时把需求告诉我！



### 2.3 流式输出

- 非流式输出 call：等待大模型把回答结果全部生成后输出给用户；
- 流式输出stream：逐个字符输出，一方面符合大模型生成方式的本质，另一方面当模型推理效率不是很高时，流式输出比起全部生成后再输出大大提高用户体验

~~~java
package org.example.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
@RequiredArgsConstructor // 为所有 final 未初始化字段生成构造器
public class SpringAIController {

    private final ChatClient chatClient;

    @GetMapping(value="/chatStream", produces="text/html;charset=UTF-8")
    public Flux<String> chatStream(@RequestParam(value = "prompt", defaultValue = "hello")
                                   String prompt) {
        // prompt:提示词
        return this.chatClient.prompt()
                // 用户输入的信息
                .user(prompt)
                // 流式输出
                .stream()
                // 返回文本
                .content();
    }
}
~~~

- 测试：http://localhost:8080/chatStream?prompt=写一篇500字作文



## 3、ChatModel接口

### 3.1 介绍

- ChatModel接口作为核心，定义了与AI模型交互的基本方法。它继承自Model<Prompt, ChatResponse>，提供了两个重载的call方法

~~~java
public interface ChatModel extends Model<Prompt, ChatResponse> {
    default String call(String message) {...}
    @Override
    ChatResponse call(Prompt prompt);
}
~~~

- 在 ChatModel 接口中，带有String参数的call()方法简化了实际的使用，避免了更复杂的Prompt和 ChatResponse 类的复杂性。但是在实际应用程序中，更常见的是使用ChatResponse call()方法，该方法采用Prompt实例并返回ChatResponse。
- 我们使用的ChatClient底层是使用ChatModel作为属性的，在初始化ChatClient的时候可以指定ChatModel，这里我们直接看底层源码

~~~java
//ChatClient（部分构造器代码）
static ChatClient create(ChatModel chatModel) {
    return create(chatModel, ObservationRegistry.NOOP);
}
~~~



### 3.2 两种对话实现

~~~java
package org.example.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.Objects;

@RestController
@RequiredArgsConstructor // 为所有 final 未初始化字段生成构造器
public class SpringAIController {

    private final ChatModel chatModel;

    /**
     * 返回String的方法
     */
    @GetMapping(value="/chat1")
    public String chat1(@RequestParam(value = "message", defaultValue = "hello") String message) {
        return chatModel.call(message);
    }

    /**
     * 返回ChatResponse的方法
     */
    @GetMapping(value="/chat2")
    public ChatResponse chat2(@RequestParam(value = "message", defaultValue = "hello") String message) {
        // 提示词配置
        Prompt prompt = new Prompt(message, OpenAiChatOptions.builder()
                .model("deepseek-v4-pro-0813")
                .temperature(0.8)
                .build());
        ChatResponse chatResponse = chatModel.call(prompt);
        System.out.println(Objects.requireNonNull(chatResponse.getResult()).getOutput().getText());
        return chatResponse;
    }
}
~~~

- 测试1：http://localhost:8080/chat2?message=你好

  - 返回：你好

- 测试2：http://localhost:8080/chat2?message=你好

  - 返回：

  ~~~json
  {
  	"metadata": {
  		"empty": false,
  		"id": "chatcmpl-99d2b8a8-e29c-9bd6-b3e6-dc26313d2858",
  		"model": "deepseek-v4-pro-0813",
  		"promptMetadata": [],
  		"rateLimit": {
  			"requestsLimit": 0,
  			"requestsRemaining": 0,
  			"requestsReset": "PT0S",
  			"tokensLimit": 0,
  			"tokensRemaining": 0,
  			"tokensReset": "PT0S"
  		},
  		"usage": {
  			"promptTokens": 84,
  			"completionTokens": 31,
  			"totalTokens": 115,
  			"cacheReadInputTokens": 0,
  			"nativeUsage": {
  				"completion_tokens": 31,
  				"prompt_tokens": 84,
  				"total_tokens": 115,
  				"completion_tokens_details": {
  					"reasoning_tokens": 18,
  					"valid": true
  				},
  				"prompt_tokens_details": {
  					"cached_tokens": 0,
  					"valid": true
  				},
  				"valid": true
  			}
  		}
  	},
  	"result": {
  		"metadata": {
  			"contentFilters": [],
  			"empty": true,
  			"finishReason": "STOP"
  		},
  		"output": {
  			"media": [],
  			"messageType": "ASSISTANT",
  			"metadata": {
  				"role": "assistant",
  				"messageType": "ASSISTANT",
  				"refusal": "",
  				"finishReason": "STOP",
  				"index": 0,
  				"annotations": [{}],
  				"id": "chatcmpl-99d2b8a8-e29c-9bd6-b3e6-dc26313d2858",
  				"reasoningContent": "我们需要回答用户中文“你好”。需要简单友好。不需要分析。直接回复问候。"
  			},
  			"text": "你好！很高兴见到你，有什么可以帮你的吗？",
  			"toolCalls": []
  		}
  	},
  	"results": [{
  		"metadata": {
  			"contentFilters": [],
  			"empty": true,
  			"finishReason": "STOP"
  		},
  		"output": {
  			"media": [],
  			"messageType": "ASSISTANT",
  			"metadata": {
  				"role": "assistant",
  				"messageType": "ASSISTANT",
  				"refusal": "",
  				"finishReason": "STOP",
  				"index": 0,
  				"annotations": [{}],
  				"id": "chatcmpl-99d2b8a8-e29c-9bd6-b3e6-dc26313d2858",
  				"reasoningContent": "我们需要回答用户中文“你好”。需要简单友好。不需要分析。直接回复问候。"
  			},
  			"text": "你好！很高兴见到你，有什么可以帮你的吗？",
  			"toolCalls": []
  		}
  	}]
  }
  ~~~

  

### 3.3 提示词

- 提示词是引导大模型生成特定输出的输入，提示词的设计和措辞会极大地影响模型的响应结果
- Prompt 提示词是与模型交互的一种输入数据组织方式，本质上是一种复合结构的输入，在 prompt 我们是可以包含多组不同角色（System、User、Aissistant等）的信息。如何管理好 Prompt 是简化 AI 应用开发的关键环节。
- Spring AI 提供了 Prompt Template 提示词模板管理抽象，开发者可以预先定义好模板，并在运行时替换模板中的关键词。在 Spring AI 与大模型交互的过程中，处理提示词首先要创建包含动态内容占位符 {占位符} 的模板，然后，这些占位符会根据用户请求或应用程序中的其他代码进行替换。在提示词模板中，{占位符} 可以用 Map 中的变量动态替换
- 步骤
  - 设置系统提示词
  - 设置用户提示词
  - 替换占位符
  - 调用大模型

~~~Java
package org.example.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.model.Generation;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.SystemPromptTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@RestController
@RequiredArgsConstructor // 为所有 final 未初始化字段生成构造器
public class SpringAIController {

    private final ChatModel chatModel;

    @GetMapping("/prompt")
    public String prompt(@RequestParam("name") String name, @RequestParam("voice") String voice) {
        // 系统提示词
        String systemText= """
            你是一个美食咨询助手，可以帮助人们查询美食信息。
            你的名字是{name},
            你应该用你的名字和{voice}的饮食习惯回复用户的请求。
            """;
        SystemPromptTemplate systemPromptTemplate = new SystemPromptTemplate(systemText);
        // 替换占位符
        Message systemMessage = systemPromptTemplate
                .createMessage(Map.of("name", name, "voice", voice));

        // 用户提示词
        String userText= """
            给我推荐北京的至少三种美食
            """;
        UserMessage userMessage = new UserMessage(userText);
        Prompt prompt = new Prompt(List.of(userMessage, systemMessage));

        // 调用
        List<Generation> results = chatModel.call(prompt).getResults();
        return results.stream().map(x->x.getOutput().getText()).collect(Collectors.joining(""));
    }

}
~~~

- 测试：http://localhost:8080/prompt?name=lzy&voice=中文



# 四、历史会话

- 在构建智能对话系统时，保持对话上下文的连贯性是提升用户体验的关键。Spring AI 框架提供了强大的 Chat Memory 机制，支持多种存储方式来持久化对话历史。本文将深入解析 Spring AI Chat Memory 的核心机制，并通过实际代码演示如何实现基于本地内存（Local）和数据库（JDBC）的两种存储方案



## 1、Chat Memory核心机制

- 核心组件解析

  - **ChatMemory 接口**：提供统一的对话记忆管理抽象

  - **ChatMemoryRepository**：负责底层存储操作

  - **MessageChatMemoryAdvisor**：基于 Advisor 模式的透明化处理

  - **MessageWindowChatMemory**：支持消息窗口限制的实现



## 2、本地存储

- 配置类
  - `MessageChatMemoryAdvisor`：采用 Advisor 模式，自动处理消息的存储和检索
  - `defaultAdvisors`：为 ChatClient 配置默认的 advisor，使 memory 功能透明化

~~~java
package org.example.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.openai.OpenAiChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ChatClientConfigs {

    @Bean
    public ChatClient chatClient(OpenAiChatModel chatModel, ChatMemory chatMemory) {
        return ChatClient.builder(chatModel)
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
                .defaultSystem("你是文档整理助手，可以生成文档、总结文档内容、解析文档属性等等相关功能业务.")
                .build();
    }
}
~~~

- controller
  - 通过 `ChatMemory.CONVERSATION_ID` 参数指定对话会话 ID
  - ChatClient 自动从 memory 中检索历史消息并添加到 prompt 中
  - 响应后自动将对话记录存储到 memory 中

~~~java
package org.example.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
@RequiredArgsConstructor // 为所有 final 未初始化字段生成构造器
public class SpringAIController {

    private final ChatClient chatClient;

    // 模拟一个会话 ID
    private static final String CONVERSATION_ID = "naming-20250528";

    @GetMapping(value="/chatStream", produces="text/html;charset=UTF-8")
    public Flux<String> chatStream(@RequestParam(value = "prompt", defaultValue = "hello") String prompt, @RequestParam(value = "conversationId") String conversationId) {
        return chatClient.prompt()
                .user(prompt)
                // 关键：通过 advisor 参数指定对话ID
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId == null ? CONVERSATION_ID : conversationId))
                .stream()
                .content();
    }
}
~~~

