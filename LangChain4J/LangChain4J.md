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



# 二、原生入门

## 1、前置约定

- LangChain4J支持的大模型：https://docs.langchain4j.info/integrations/language-models

![大模型调用三件套](图片\大模型调用三件套.png)



## 2、基础包和高阶包的区别

- <font color="red">**langchain4j‑open‑ai：（低阶主包）**</font>

  - 职责：<font color="red">**只负责发 HTTP 请求调用大模型**</font>
  - 类：`OpenAiChatModel`、`OpenAiStreamingChatModel`、`OpenAiEmbeddingModel`
  - 能力：
    - 拼装 http 请求体
    - 签名、携带 apiKey、请求模型接口 `/v1/chat/completions`
    - 解析返回 JSON，转成 LangChain4j 标准对象 `AiMessage`
    - 抛出你刚才遇到的 `HttpException`
  - 只能用低阶用法

  ~~~java
  ChatLanguageModel model = OpenAiChatModel.builder()
          .baseUrl("https://api.deepseek.com")
          .apiKey("sk-xxx")
          .modelName("deepseek-chat")
          .build();
  
  //自己手动拼消息列表、自己维护历史、自己处理工具返回
  Response<AiMessage> resp = model.generate(UserMessage.from("你好"));
  ~~~

  - 这个包<font color="red">**完全没有 AiService、@Tool、ChatMemory**</font>

- <font color="red">**langchain4j（高阶主包）**</font>

  - 它本身**不能直接调用任何大模型接口**！ 

  - 它依赖 `langchain4j‑core` + 模型驱动包（open‑ai /dashscope 等） 

  - 提供上层封装组件：

    - `AiServices` 动态代理（最核心）

    - `@Tool` 工具调用注解

    - `ChatMemory` 对话上下文记忆

    - 文档拆分、RAG 检索组件

    - 结构化输出（直接返回 POJO）

  - 高阶写法：

  ```java
  //1.先用 open‑ai 包构造底层模型
  ChatLanguageModel model = OpenAiChatModel.builder()
          .baseUrl("https://api.deepseek.com")
          .apiKey("sk-xxx")
          .modelName("deepseek-chat")
          .build();
  
  //2 .交给高阶包 AiServices 包装一层
  interface Agent {
      String chat(String msg);
  }
  Agent agent = AiServices.builder(Agent.class)
          .chatLanguageModel(model)
          .chatMemory(MessageWindowChatMemory.withMaxMessages(5))
          .build();
  
  String answer = agent.chat("你好");
  ```

- 对比

| 对比项             | langchain4j‑open‑ai      | langchain4j (高阶主包)             |
| ------------------ | ------------------------ | ---------------------------------- |
| 定位               | 模型网络驱动 / 客户端    | Agent 高层框架                     |
| 核心类             | OpenAiChatModel          | AiServices、ChatMemory、@Tool      |
| 能不能发 http 请求 | ✅可以                    | ❌不可以，必须传入模型实例          |
| AiService          | ❌没有                    | ✅提供                              |
| 工具调用注解 @Tool | ❌不支持                  | ✅支持                              |
| 对话记忆           | 需要自己手动维护消息列表 | 内置 ChatMemory                    |
| 结构化返回 POJO    | 自己解析字符串           | 框架自动序列化                     |
| 依赖               | 只依赖 langchain4j‑core  | 依赖 core + 可以接入任意模型驱动包 |



## 3、入门例子

- POM文件：<font color="red">**原生整合方式**</font>
  - langchain4j-open-ai：基础
  - langchain4j：高阶


~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.4</version>
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
            	.logRequests(true) // 日志级别设置为debug才有效  
                .logResponses(true)// 日志级别设置为debug才有效  
                .listeners(List.of(new TestChatModelListener()))  // 事件监听
                .maxRetries(2) // 最大重试次数
                .timeout(Duration.ofSeconds(5)) // 请求大模型的超时时间
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



## 4、实现多模型共存

- 主要修改配置类：LLMConfig

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
    @Bean("qwen")
    public ChatModel chatModelQwen() {
        return OpenAiChatModel.builder()
                .apiKey("sk-ws-H.EYHLXID.94g4.MEYCIQDxvKfua7F_KDMECg_myE86M2VlVnWjFW_MdNY_ZVHE7gIhAPKwAd2FGskYlTDlm3iNyDm-gDLb70sqRLK5SN9iWEBq")
                .modelName("qwen3.7-plus")
                .baseUrl("https://ws-zks05buw2zy3a3e4.cn-beijing.maas.aliyuncs.com/compatible-mode/v1")
                .build();
    }

    @Bean("deepseek")
    public ChatModel chatModelDeepSeek() {
        return OpenAiChatModel.builder()
                .apiKey("sk-3aa699fb50004c23bcb71282481d3a68")
                .modelName("deepseek-v4-pro")
                .baseUrl("https://api.deepseek.com")
                .build();
    }
}
~~~

- controller

~~~java
package org.example.controller;

import dev.langchain4j.model.chat.ChatModel;
import jakarta.annotation.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LangChai4JController {

    @Resource(name = "qwen")
    private ChatModel chatModelQwen;
    @Resource(name = "deepseek")
    private ChatModel chatModelDeepSeek;

    @GetMapping("/chat/qwen")
    public String chatQwen(@RequestParam(value = "message", defaultValue = "你好") String message) {
        String response = chatModelQwen.chat(message);
        System.out.println("qwen response : " + response);
        return response;
    }

    @GetMapping("/chat/deepseek")
    public String chatDeepSeek(@RequestParam(value = "message", defaultValue = "你好") String message) {
        String response = chatModelDeepSeek.chat(message);
        System.out.println("deepseek response : " + response);
        return response;
    }

}
~~~

- 测试
  - 千问：http://localhost:8080/chat/deepseek?message=你好
  - deepseek：http://localhost:8080/chat/deepseek?message=你好



# 三、整合SpringBoot

## 1、公共部分

- POM文件：<font color="red">**SpringBoot整合方式**</font>

  - langchain4j-open-ai-spring-boot-starter：基础

  - langchain4j-spring-boot-starter：高阶

~~~XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.4</version>
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
        <!-- 基础 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
        </dependency>
        <!-- 高阶 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-spring-boot-starter</artifactId>
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

- 这里需要把配置写在yml里

~~~yaml
langchain4j:
  open-ai:
    chat-model:
      # 是否开启，默认true
      enabled: true
      # 模型名称
      model-name: qwen3.7-plus
      # api‑key
      api-key: sk-ws-H.EYHLXID.94g4.MEYCIQDxvKfua7F_KDMECg_myE86M2VlVnWjFW_MdNY_ZVHE7gIhAPKwAd2FGskYlTDlm3iNyDm-gDLb70sqRLK5SN9iWEBq
      # 自定义地址(DeepSeek/通义都需要改)，官方OpenAI填 https://api.openai.com/v1
      base-url: https://ws-zks05buw2zy3a3e4.cn-beijing.maas.aliyuncs.com/compatible-mode/v1
      # 温度 0~2，越低越确定，越高越发散
      temperature: 0.7
      # 最大返回token
      max-tokens: 2048
      # 超时时间
      timeout: 30s
      # 日志打印完整请求响应
      log-requests: true
      log-responses: true
~~~



## 2、原生操作ChatModel

- 直接使用ChatModel进行交互即可

```java
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

    @GetMapping("/chat/qwen")
    public String chatQwen(@RequestParam(value = "message", defaultValue = "你好") String message) {
        String response = chatModel.chat(message);
        System.out.println("qwen response : " + response);
        return response;
    }

}
```

- 测试：http://localhost:8080/chat/qwen?message=你好



## 3、声明式AiService

- 用注解AiService声明接口

~~~java
package org.example.service;

import dev.langchain4j.service.spring.AiService;

@AiService
public interface ChatAssistant {

    /**
     * 对话接口
     */
    String chat(String prompt);
}
~~~

- controller：使用前面定义的接口

~~~java
package org.example.controller;

import dev.langchain4j.model.chat.ChatModel;
import jakarta.annotation.Resource;
import org.example.service.ChatAssistant;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class LangChai4JController {

    @Resource
    private ChatAssistant chatAssistant;

    @GetMapping("/chat/qwen")
    public String chatQwen(@RequestParam(value = "message", defaultValue = "你好") String message) {
        String response = chatAssistant.chat(message);
        System.out.println("qwen response : " + response);
        return response;
    }

}
~~~

- 测试：http://localhost:8080/chat/qwen?message=你好



# 四、低阶高阶对比

























































































# 五、模型参数介绍













































# 七、流式输出

## 1、是什么

- 是一种<font color="red">**逐步返回大模型生成结果**</font>的技术，生成一点返回一点，允许服务器将响应内容<font color="red">**分批次实时传输给客户端**</font>，而不是等待全部内容生成完毕后再一次性返回。
- 这种机制能显著提升用户体验，尤其适用于大模型响应较慢的场景（如生成长文本或复杂推理结果）



## 2、前置准备

- 需要加入新的依赖

~~~xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-reactor</artifactId>
</dependency>
~~~

- 配置文件设置字符集

~~~yaml
# 设置响应的字符编码，避免流式返回输出乱码  
server:
	servlet:
		encoding:
			charset: utf-8  
			enabled: true  
			force: true
~~~

- StreamingChatResponseHandler 处理流式事件
  - 通过实现 StreamingChatResponseHandler，可以为以下事件定义操作：
    - 当生成下一个部分响应时：调用 onPartialResponse(String partialResponse)。可以在标记可用时立即将其发送到 UI。
    - 当 LLM 完成生成时：调用 onCompleteResponse(ChatResponse completeResponse)。 ChatResponse 对象包含完整的响应（AiMessage）以及 ChatResponseMetadata
    - 当发生错误时：调用 onError(Throwable error)

~~~java
public interface StreamingChatResponseHandler {
 
    void onPartialResponse(String partialResponse);
 
    void onCompleteResponse(ChatResponse completeResponse);
 
    void onError(Throwable error);
}
~~~



## 3、完整依赖

- pom文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.4</version>
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
        <!-- 流式 -->
        <dependency>
            <groupId>dev.langchain4j</groupId>
            <artifactId>langchain4j-reactor</artifactId>
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



## 4、配置类

- 修改成StreamingChatModel类返回

~~~java
package org.example.config;

import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.openai.OpenAiStreamingChatModel;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 模型配置
 */
@Configuration
public class LLMConfig {
    @Bean("qwen")
    public StreamingChatModel chatModelQwen() {
        return OpenAiStreamingChatModel.builder()
                .apiKey("sk-ws-H.EYHLXID.94g4.MEYCIQDxvKfua7F_KDMECg_myE86M2VlVnWjFW_MdNY_ZVHE7gIhAPKwAd2FGskYlTDlm3iNyDm-gDLb70sqRLK5SN9iWEBq")
                .modelName("qwen3.7-plus")
                .baseUrl("https://ws-zks05buw2zy3a3e4.cn-beijing.maas.aliyuncs.com/compatible-mode/v1")
                .build();
    }

    @Bean("deepseek")
    public StreamingChatModel chatModelDeepSeek() {
        return OpenAiStreamingChatModel.builder()
                .apiKey("sk-3aa699fb50004c23bcb71282481d3a68")
                .modelName("deepseek-v4-pro")
                .baseUrl("https://api.deepseek.com")
                .build();
    }
}
~~~



## 5、原生方式ChatModel

~~~java
package org.example.controller;

import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.chat.response.ChatResponse;
import dev.langchain4j.model.chat.response.StreamingChatResponseHandler;
import jakarta.annotation.Resource;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
public class LangChai4JStreamController {

    @Resource(name = "qwen")
    private StreamingChatModel streamingChatModel;

    @GetMapping(value = "/chatstream")
    public Flux<String> chat(@RequestParam("prompt") String prompt) {
        return Flux.create(emitter -> {
            streamingChatModel.chat(prompt, new StreamingChatResponseHandler() {
                @Override
                public void onPartialResponse(String partialResponse) {
                    emitter.next(partialResponse);
                }

                @Override
                public void onCompleteResponse(ChatResponse completeResponse) {
                    emitter.complete();
                    // 可以打印此次对话的一些相关的信息
                    System.out.println(completeResponse);
                }

                @Override
                public void onError(Throwable throwable) {
                    emitter.error(throwable);
                }
            });
        });
    }

}
~~~



## 6、高阶方法AiService

- ChatAssistant类

~~~java
package org.example.service;

import reactor.core.publisher.Flux;

public interface ChatAssistant {
    /**
     * 流式对话
     */
    Flux<String> chatFlux(String prompt);
}
~~~

- 配置类LLMConfig

~~~java
package org.example.config;

import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.openai.OpenAiStreamingChatModel;
import dev.langchain4j.service.AiServices;
import org.example.service.ChatAssistant;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 模型配置
 */
@Configuration
public class LLMConfig {
    @Bean("qwen")
    public StreamingChatModel chatModelQwen() {
        return OpenAiStreamingChatModel.builder()
                .apiKey("sk-ws-H.EYHLXID.94g4.MEYCIQDxvKfua7F_KDMECg_myE86M2VlVnWjFW_MdNY_ZVHE7gIhAPKwAd2FGskYlTDlm3iNyDm-gDLb70sqRLK5SN9iWEBq")
                .modelName("qwen3.7-plus")
                .baseUrl("https://ws-zks05buw2zy3a3e4.cn-beijing.maas.aliyuncs.com/compatible-mode/v1")
                .build();
    }

    @Bean("deepseek")
    public StreamingChatModel chatModelDeepSeek() {
        return OpenAiStreamingChatModel.builder()
                .apiKey("sk-3aa699fb50004c23bcb71282481d3a68")
                .modelName("deepseek-v4-pro")
                .baseUrl("https://api.deepseek.com")
                .build();
    }

    @Bean
    public ChatAssistant chatAssistant(@Qualifier("qwen") StreamingChatModel streamingChatModel) {
        return AiServices.create(ChatAssistant.class, streamingChatModel);
    }
}
~~~

- controller

~~~java
package org.example.controller;

import jakarta.annotation.Resource;
import org.example.service.ChatAssistant;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
public class LangChai4JStreamController {

    @Resource
    private ChatAssistant chatAssistant;

    @GetMapping(value = "/chatstream")
    public Flux<String> chat(@RequestParam("prompt") String prompt) {
        return chatAssistant.chatFlux(prompt);
    }
}
~~~



# 八、记忆功能

## 1、聊天记忆含义

- LangChain4j 提供的聊天记忆的抽象容器，用于简化手动维护和管理聊天消息的繁琐工作。

- **记忆 vs 历史的区别**：
  - **历史**保留用户与 AI 之间所有消息的完整记录。历史是用户在 UI 中看到的内容，它代表了实际发生的事情。
  - **记忆**保留一些信息，并将其呈现给 LLM（大型语言模型），使其表现得好像 “记得” 了对话。记忆与历史有很大的不同。 根据使用的记忆算法，它可以以各种方式修改历史：删除某些消息、总结多条消息、单独总结消息、从消息中移除不重要的细节、向消息中注入额外信息（例如，用于 RAG) 或指令（例如，用于结构化输出）等等。
- LangChain4j 目前只提供 “记忆”，而不提供 “历史”。如果你需要保留完整的记录，请手动进行

- 记忆缓存是聊天系统中的一个重要组件，用于存储和管理对话的上下文信息。它的主要作用是让 AI 助手能够 “记住” 之前的对话内容，从而提供连贯和个性化的回复



## 2、淘汰策略

- 清除策略是出于以下几个原因而必要的：

  - 为了适应 LLM（大型语言模型）的上下文窗口。LLM 一次可以处理的令牌数量有限制。在某些时候，对话可能会超过这个限制。在这种情况下，应该清除一些消息。通常，最旧的消息会被清除，但如果需要，也可以实现更复杂的算法。

  - 为了控制成本。每个Token都有成本，使得每次调用 LLM 的成本逐渐增加。清除不必要的消息可以降低成本。

  - 为了控制延迟。发送给 LLM 的Token越多，处理它们所需的时间就越长。

- 目前，LangChain4j 提供了两种现成的实现：

  - <font color="red">**`MessageWindowChatMemory`（消息计数）：保留最近的N条消息**</font>，作为一个滑动窗口工作，<font color="red">**保留最近的 N 条消息，并清除不再适合的较旧消息**</font>。然而，由于每条消息可能包含不同数量的令牌，`MessageWindowChatMemory`主要用于快速原型设计。

  - <font color="red">**`TokenWindowChatMemory` （`Tokenizer` 计数）：保留最近的N个 Token**</font>，它也作为滑动窗口工作，<font color="red">**但专注于保留最近的 N 个 Token，根据需要清除较旧的消息**</font>。消息是不可分割的。如果一条消息不适合，它将被完全清除。`TokenWindowChatMemory` 需要一个`TokenCountEstimator` 来计算每条 `ChatMessage` 中的Token数。



## 3、主要特性

- 淘汰策略：自动管理消息数量以适应LLM上下文窗口限制
  - `MessageWindowChatMemory`（消息计数）：保留最近的N条消息
  - `TokenWindowChatMemory` （`Tokenizer` 计数）：保留最近的N个 Token
- **持久化支持**：可将聊天消息存储到持久化存储（`ChatMemoryStore`）中
- **SystemMessage特殊处理**：系统消息的专门管理机制
- **工具消息特殊处理**：避免孤立工具执行结果消息导致的问题













































































































# 十四、MCP

## 1、基本概念

- MCP 是一种开放协议，它标准化了应用程序如何向大型语言模型（LLMs）提供上下文。可以将 MCP 想象成 AI 应用的 USB‑C 端口。就像 USB‑C 提供了一种标准化的方式将你的设备连接到各种外围设备和配件一样，<font color="red">**MCP 提供了一种标准化的方式将 AI 模型连接到不同的数据源和工具**</font>

- MCP 让模型可以调用<font color="red">**远程或本地的各种工具与资源**</font>，如果模型是大脑，MCP 则赋予了大脑手脚，使它可以与外界世界进行交互。

- MCP协议指定了两种传输类型：

  - HTTP：客户端请求一个 SSE 通道来接收来自服务器的事件，然后通过 HTTP POST 请求发送命令。

  - STDIO：客户端可以将 MCP 服务器作为本地子进程运行，并通过标准输入/输出直接与其通信。

  | 特性     | SSE                              | STDIO                                |
  | -------- | -------------------------------- | ------------------------------------ |
  | 传输协议 | HTTP（长连接）                   | 操作系统级文件描述符                 |
  | 方向     | 服务器 → 客户端（单向推送）      | 双向流（stdin，stdout）              |
  | 保持连接 | 长连接（Connection: keep‑alive） | 不保证长时间打开，取决于进程生命周期 |
  | 数据格式 | 文本流（EventStream 格式）       | 原始字节流                           |
  | 异常处理 | 可通过 HTTP 状态码或重连机制     | 进程退出或管道断裂                   |

- 大模型版的OpenFeign，OpenFeign用于微服务之间的通讯，MCP用于大模型之间通讯



## 2、能干嘛

- 提供了一种标准化的方式来连接 LLMs 需要的上下文，MCP 就类似于一个 Agent 时代的 Type‑C 协议，<font color="red">**希望能将不同来源的数据、工具、服务统一起来供大模型调用**</font>

- MCP 就是比 FunctionCalling 的更高一级抽象，也是实现智能体 Agent 的基础

~~~bash
假设你正在使用一个 AI 编程助手来帮助你写代码。这个 AI 助手就是一个 MCP 主机。它需要访问一些外部资源，比如代码库、文档或者调试工具。MCP 服务器就像是一个中介，它连接了这些资源和 AI 助手。

- 当你需要查找某个函数的用法时，AI 助手通过 MCP 客户端向 MCP 服务器发送请求。
- MCP 服务器接收到请求后，去代码库或文档中查找相关信息。
- 找到信息后，MCP 服务器将结果返回给 AI 助手。
- AI 助手根据返回的信息，生成一段代码或解释，展示给你。

使用 MCP 后，你直接对 AI 说：“帮我查一下最近数学考试的平均分，把不及格的同学名单整理到值日表里，并在微信群提醒他们补考。”AI 会自动完成：用 “万能插头” MCP 连接你的电脑，读取 Excel 成绩。用 MCP 连接微信，找到相关聊天记录。用 MCP 修改在线文档，更新值日表。整个过程不需要你手动操作，数据也不会离开你的设备，安全又高效
~~~



## 3、架构和交互方式

- <font color="red">**客户端-服务器架构**</font>
- 核心组件
  - MCP主机（MCP Hosts）：发起请求的 AI 应用程序，比如聊天机器人、AI 驱动的 IDE 等。 
  - MCP客户端（MCP Clients）：在主机程序内部，与 MCP 服务器保持 1:1 的连接。 
  - MCP服务器（MCP Servers）：为 MCP 客户端提供上下文、工具和提示信息。
  - 本地资源（Local Resource）：本地计算机中可供 MCP 服务器安全访问的资源，如文件、数据库。
  - 远程资源（Remote Resource）：MCP 服务器可以连接到的远程资源，如通过 API 提供的数据

- 用户与 MCP 和大模型的交互方式如下图所示

![交互方式1](图片\交互方式1.png)

- 流程图

![交互方式2](图片\交互方式2.png)



## 4、MCP工具网站

- 统一提供官网：https://mcp.so/zh
