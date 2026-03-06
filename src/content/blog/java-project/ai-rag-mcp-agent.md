---
title: spring-ai练习
publishDate: 2026-03-06
description: '介绍java项目中的各种环境配置步骤'
tags:
  - java
  - project
  - ai
language: '中文'
---

## Chat流程

新版本spring-ai（1.1.2）中，ChatClient底层依托ChatModel发送请求和接收响应

与chatModel相关的主要包括call()和stream()，前者是一次性返回对话的所有的内容，后者是流式的（类似一个字一个字）返回对话的内容

其他的调用与上下文信息（prompt）、RAG（advisors）、MCP（tools）、AGENT相关

总览图：

```
只需要记住这张图：
ChatClient.Builder（Spring自动注入，前提是配置里只有一个Client）
    │
    ├── .defaultSystem(...)       → 默认系统提示
    ├── .defaultAdvisors(...)     → 默认拦截器（记忆/RAG）
    ├── .defaultTools(...)        → 默认工具（MCP/Function）
    └── .build()
            │
            ▼
        ChatClient（你拿来用的）
            │
            └── .prompt()
                    ├── .system(...)    → 覆盖默认系统提示
                    ├── .user(...)      → 用户输入
                    ├── .advisors(...)  → 本次请求的拦截器参数
                    ├── .call()         → 普通调用
                    └── .stream()       → 流式调用
```

### POM

```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring-ai.version>1.1.2</spring-ai.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 单模型

POM中导入open-ai模型：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>


    <!--   使用openai模型  -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-openai</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

分别使用ChatModel和ChatClient发送请求（以deepseek api为例）：

1. 配置文件（application.yml）

   ```yaml
   server:
     port: 8090
   
   spring:
     ai:
       openai:
         base-url: https://api.deepseek.com
         api-key: sk-xxx
         chat:
           options:
             model: deepseek-chat
   ```

2. 配置类

   ```java
   @Configuration
   public class AiConfig {
       @Bean
       public ChatClient chatClient(ChatClient.Builder builder) {
           return builder
                   .defaultSystem("你是一个专业的技术助手，请用中文回答。")
                   .build();
       }
   }
   ```

3. 测试

   ```java
   @SpringBootTest
   public class ChatTest {
       // 第三方Bean自动注入
       @Resource
       private ChatModel chatModel;
       // 配置类注入
       @Resource
       private ChatClient chatClient;
       // ==================== ChatModel 测试 ====================
       /**
        * 测试：ChatModel 使用 Prompt，携带系统提示
        */
       @Test
       public void test_chatModel_withPrompt() {
           Prompt prompt = new Prompt(List.of(
                   new SystemMessage("你是一个专业的 Java 开发助手，回答请简洁专业"),
                   new UserMessage("Spring AI 和 LangChain 有什么区别？")
           ));
   
           ChatResponse response = chatModel.call(prompt);
   
           // 获取回复文本
           String content = response.getResult().getOutput().getText();
           System.out.println("回复：" + content);
   
           // 打印 token 用量
           var usage = response.getMetadata().getUsage();
           System.out.println("输入 tokens：" + usage.getPromptTokens());
           System.out.println("输出 tokens：" + usage.getCompletionTokens());
       }
       // ==================== ChatClient 测试 ====================
   
       /**
        * 测试：ChatClient 链式调用流式输出（最常用写法）
        */
       @Test
       public void test_chatClient_stream() throws InterruptedException {
           Flux<String> flux = chatClient.prompt()
               	.system("你是一个幽默的程序员，回答时适当加入编程梗")
                   .user("解释一下什么是递归")
                   .stream()
                   .content();
   
           // 同步阻塞订阅，测试环境用 blockLast()
           flux.doOnNext(token -> System.out.print(token))
                   .blockLast();
   
           System.out.println(); // 换行
       }
   }
   ```

解释一下需要了解的类和api

- Prompt：提示词，由单条或多条有序的Message组成，基本可以看做是对话内容
- Message：根据类型分为System（系统设置），User（用户消息），Assistant（模型回复消息）
- ChatModel：对话模型客户端，负责给大模型发送消息和接收响应
  - call()
  - stream()
- ChatClient：封装了ChatModel和prompt以及其他的一些可能用到的工具流程（比如设置，RAG等），使其支持链式调用，而不必单独手动组合在一起
  - prompt()：输入的提示词
  - system()：语法糖，就是输入的System Message
  - user()：语法糖，就是输入的User Message
  - call()
  - stream()

### 多模型

当需要使用多个厂商的模型时（或者不同厂商提供的不同组件），在配置上以及第三方Bean的使用上与单模型有差别：

POM引入：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>
```

1. 配置文件需要配置多个厂商：

   ```yaml
   spring:
     ai:
       ollama:
         base-url: http://127.0.0.1:11434
         embedding:
           options:
             num-batch: 512
           model: nomic-embed-text	# 本地ollama提供的嵌入模型, 见RAG流程
       openai:
         base-url: https://api.deepseek.com
         api-key: sk-xxx
         chat:
           options:
             model: deepseek-chat	# deepseek api提供的对话模型
   ```

2. 第三方Bean导入：

   使用各厂商的ChatModel作为参数，然后导入对应的ChatClient：

   ```java
   @Configuration
   public class AiConfig {
       @Bean("openAiChatClient")
       public ChatClient chatClient(OpenAiChatModel model) {
           return ChatClient.builder(model)
                   .defaultSystem("你是 OpenAI 驱动的助手")
                   .build();
       }
   
       @Bean("ollamaChatClient")
       public ChatClient ollamaChatClient(OllamaChatModel model) {
           return ChatClient.builder(model)
                   .defaultSystem("你是本地 Ollama 驱动的助手")
                   .build();
       }
   }
   ```

3. 可以使用chatOption指定各厂商提供的模型：

   ```java
   @Test
   public void test_chatClientOption() {
       String content = ollamaChatClient.prompt()
           .options(OllamaChatOptions.builder()
                    .model("deepseek-r1:1.5b")	// 指定本地ollama提供的1.5b模型
                    .build())
           .user("你是什么助手？")
           .call()
           .content();
       System.out.println("ollama助手回答:" +  content);
   }
   ```

## RAG流程

以对话模型为例，当发送的消息比如“帮我写xxx算法的代码”，对话模型内部处理并不是自然语言，而是一堆数据（向量），也就是先有一个模型将各种自然语言转为[0.1,0.2,0.3....]之类的向量，然后对话模型内部才去处理，这个将自然语言转为向量的模型叫嵌入模型（embedding-model）

RAG（Retrieval-Augmented-Generation）提前将资料（文本文档，图片，音频等）通过各类嵌入模型转为向量，然后存入专门的向量数据库中，在用户向对话模型提问时，将问题也经过嵌入模型转为向量后，去向量数据库中去比对，然后找到相似的信息，最后将数据库中的信息和用户消息同时作为prompt交给对话模型，以此让对话模型能够获取外部资料增强输出

POM引入：

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-advisors-vector-store</artifactId>
</dependency>
```

介绍上述提到的概念对应到spring-ai中的类：

1. 嵌入模型：EmbeddingModel，使用方式跟ChatModel类似，不过返回的内容是数组
2. 上传以及检索出来的资料：Document，文件资料可能以各种方式存在（txt，md，pdf），Document负责将其统一
3. 文本分割器：TokenTextSplitter，一个文件可能过大，需要将内容按指定规则分割多个小块，这些小块以Document的形式存在
4. 向量数据库：PgVectorStore，负责连接向量数据库并进行操作，其需要指定一个嵌入模型，之后在上传和检索时都会使用该模型进行文本转向量的处理

### 配置

配置文件：

```yaml
spring:
  datasource:
    driver-class-name: org.postgresql.Driver
    username: postgres
    password: postgres
    url: jdbc:postgresql://127.0.0.1:15432/ai-rag-mcp
    type: com.zaxxer.hikari.HikariDataSource
    # hikari连接池配置
    hikari:
      #连接池名
      pool-name: HikariCP
      #最小空闲连接数
      minimum-idle: 5
      # 空闲连接存活最大时间，默认10分钟
      idle-timeout: 600000
      # 连接池最大连接数，默认是10
      maximum-pool-size: 10
      # 此属性控制从池返回的连接的默认自动提交行为,默认值：true
      auto-commit: true
      # 此属性控制池中连接的最长生命周期，值0表示无限生命周期，默认30分钟
      max-lifetime: 1800000
      # 数据库连接超时时间,默认30秒
      connection-timeout: 30000
      # 连接测试query
      connection-test-query: SELECT 1
  ai:
    ollama:
      base-url: http://127.0.0.1:11434
      embedding:
        options:
          num-batch: 512
        model: nomic-embed-text	# ollama的嵌入模型
```

配置类：

```java
@Bean
public PgVectorStore pgVectorStore(JdbcTemplate jdbcTemplate, OllamaEmbeddingModel ollamaEmbeddingModel) {
    return PgVectorStore.builder(jdbcTemplate, ollamaEmbeddingModel)
        .initializeSchema(true)
        .dimensions(768)
        .distanceType(PgVectorStore.PgDistanceType.COSINE_DISTANCE)
        .indexType(PgVectorStore.PgIndexType.HNSW)
        .build();
}

@Bean
public TokenTextSplitter  tokenTextSplitter() {
    return new TokenTextSplitter();
}
```

向量数据库：

使用docker启动一个vector_db，这里启动的是PGVector

在docker终端创建并连接数据库，创建表：

```bash
# 连接
psql -h 127.0.0.1 -p 5432 -U postgres

# 创建数据库
\l # 查看数据库列表
create databse ai-rag-mcp;
\c ai-rag-mcp # 使用对应数据库

# 创建表
CREATE EXTENSION IF NOT EXISTS vector; # 安装pgvector扩展
SELECT * FROM pg_extension WHERE extname = 'vector'; # 验证是否安装成功（应该能看到一条记录）
\dt vector_store	# 检查表是否存在
DROP TABLE IF EXISTS vector_store;	# 删除表

CREATE TABLE IF NOT EXISTS vector_store (
    id UUID PRIMARY KEY,
    content TEXT,
    metadata JSONB,
    embedding vector(768)
);
```

### 上传

1. reader读取文档
2. tokenTextSplitter分割文档
3. 给文档打标签（也叫元信息，会一起存储到数据库里，可以根据这个进行检索前的过滤）
4. vectorstore.add()将文档上传到向量数据库

```java
@Resource
private PgVectorStore pgVectorStore;
@Resource
private TokenTextSplitter tokenTextSplitter;
@Test
public void test_upload_file_to_db() {
    
    TikaDocumentReader reader = new TikaDocumentReader("./data/env-build.md");
    List<Document> documents = reader.get();
    List<Document> chunks = tokenTextSplitter.apply(documents);
    chunks.forEach(document -> {
        document.getMetadata().put("description", "java-env-build");
    });
    pgVectorStore.add(chunks);
}
```

### 检索

1. 在向量数据库中进行问题的相似度查询，得到结果
2. 将检索结果和问题一起拼装到prompt中，然后通过chatmodel发送

```java
@Test
public void test_chat_with_db() {
    String question = "如何进行微信鉴权？";
    List<Document> documents = pgVectorStore.similaritySearch(
        SearchRequest.builder()
        .query(question)
        .topK(5)
        .similarityThreshold(0.5)
        .filterExpression("description == 'java-env-build'")
        .build()
    );
    log.info("=======检索到的相关文档=======");
    documents.forEach(document -> {
        System.out.println("· " + document.getText());
    });
    String context = documents.stream()
        .map(Document::getText)
        .collect(Collectors.joining("\n\n"));
    String ragPromptText = """
        请根据以下参考资料回答问题，资料中没有的内容请如实说明。

        参考资料：
        %s

        问题：%s
        """.formatted(context, question);

        Prompt prompt = new Prompt(List.of(
            new SystemMessage("你是一个知识库助手，请用中文简洁回答。"),
            new UserMessage(ragPromptText)
        ));

    ChatResponse chatResponse = openAiChatModel.call(prompt);
    // Step5: 提取结果
    String answer = chatResponse.getResult().getOutput().getText();
    log.info("\n=== 模型回答 ===");
    log.info(answer);

    // 额外：查看 token 用量
    Usage usage = chatResponse.getMetadata().getUsage();
    log.info("\n输入tokens: " + usage.getPromptTokens());
    log.info("输出tokens: " + usage.getCompletionTokens());
}
```

如果使用chatclient，则使用Advisors类来封装这个过程：

```java
@Test
public void test_chat_client_rag() {
    String question = "如何进行微信鉴权？";
    String content = openAiChatClient.prompt()
        .user(question)
        .advisors(QuestionAnswerAdvisor.builder(pgVectorStore)
                  .searchRequest(
                      SearchRequest.builder()
                      .topK(5)
                      .similarityThreshold(0.5)
                      .filterExpression("description == 'java-env-build'")
                      .build()
                  )
                  .build())
        .call()
        .content();
    log.info("advisors rag content: {}", content);
}
```

## MCP流程

模型上下文协议（MCP）是一种使AI模型能够以结构化方式与外部工具和资源交互的协议（这个外部工具可能是本地进程，可能是服务器或其他遵循协议的程序）。

说人话就是，有一个服务，对外暴露接口或api，把这些服务提供的api信息交给对话模型，模型的输出（json格式字符串）会自动调用工具（在json中表明要调用），然后客户端会调用对应的服务api，收到调用的结果后再发给对话模型去分析

### 基本流程

spring-ai将流程封装到了ChatClient的调用内部（也就是call()和stream()具体实现做的事），只提供几个类给开发人员调用，不易于理解流程，这里使用deepseek官网中的python进行阐述：

```python
from openai import OpenAI

def send_messages(messages):
    response = client.chat.completions.create(
        model="deepseek-chat",
        messages=messages,
        tools=tools
    )
    return response.choices[0].message

client = OpenAI(
    api_key="<your api key>",
    base_url="https://api.deepseek.com",
)
# tools数组，和对话消息一起发送给对话模型
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get weather of a location, the user should supply a location first.",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    }
                },
                "required": ["location"]
            },
        }
    },
]

messages = [{"role": "user", "content": "How's the weather in Hangzhou, Zhejiang?"}]
message = send_messages(messages)
print(f"User>\t {messages[0]['content']}")

tool = message.tool_calls[0]
messages.append(message)

messages.append({"role": "tool", "tool_call_id": tool.id, "content": "24℃"})
message = send_messages(messages)
print(f"Model>\t {message.content}")
```

1. 准备好tools（待调用的api说明），这里的python代码直接给出，平常需要由对话客户端主动向MCP服务器进行请求获取tools信息

2. 将tools和问题一起发送给对话模型

3. 查看返回结果：

   ```json
   ===response===
   {
     "id": "2beffab1-837e-4031-b137-d1d49b6f7800",
     "choices": [
       {   
         "finish_reason": "tool_calls",
         "index": 0,
         "logprobs": null,
         "message": {
           "content": "I'll check the weather in Hangzhou, Zhejiang for you.",
           "refusal": null,
           "role": "assistant",
           "annotations": null,
           "audio": null,
           "function_call": null,
           "tool_calls": [
             {
               "id": "call_00_oUNn7rFGaMXqENDHP3vC2ZRC",
               "function": {
                 "arguments": "{\"location\": \"Hangzhou, Zhejiang\"}",
                 "name": "get_weather"
               },
               "type": "function",
               "index": 0
             }
           ]
         }
       }
     ],
     "created": 1772699828,
     "model": "deepseek-chat",
     "object": "chat.completion",
     "service_tier": null,
     "system_fingerprint": "fp_eaab8d114b_prod0820_fp8_kvcache",
     "usage": {
       "completion_tokens": 62,
       "prompt_tokens": 333,
       "total_tokens": 395,
       "completion_tokens_details": null,
       "prompt_tokens_details": {
         "audio_tokens": null,
         "cached_tokens": 320
       },
       "prompt_cache_hit_tokens": 320,
       "prompt_cache_miss_tokens": 13
     }
   }
   ```

   主要查看"finish_reason"，这里表明是"tool_calls"，也就是模型在分析之后决定要进行api调用了，然后是里面的"tool_calls"数组，里面表明了要调用的api信息

4. 由客户端（主程序）去调用这个api，将得到的信息拼接，并重新发给对话模型，得到最终结果：

   ```json
   
   User>	 How's the weather in Hangzhou, Zhejiang?
   ===response===
   {
     "id": "12f0a8df-cb1b-4391-917f-1b190914ab6f",
     "choices": [
       {
         "finish_reason": "stop",
         "index": 0,
         "logprobs": null,
         "message": {
           "content": "The current weather in Hangzhou, Zhejiang is 24°C (about 75°F). It's a pleasant, mild temperature - perfect for outdoor activities!",
           "refusal": null,
           "role": "assistant",
           "annotations": null,
           "audio": null,
           "function_call": null,
           "tool_calls": null
         }
       }
     ],
     "created": 1772699833,
     "model": "deepseek-chat",
     "object": "chat.completion",
     "service_tier": null,
     "system_fingerprint": "fp_eaab8d114b_prod0820_fp8_kvcache",
     "usage": {
       "completion_tokens": 34,
       "prompt_tokens": 414,
       "total_tokens": 448,
       "completion_tokens_details": null,
       "prompt_tokens_details": {
         "audio_tokens": null,
         "cached_tokens": 384
       },
       "prompt_cache_hit_tokens": 384,
       "prompt_cache_miss_tokens": 30
     }
   }
   Model>	 The current weather in Hangzhou, Zhejiang is 24°C (about 75°F). It's a pleasant, mild temperature - perfect for outdoor activities!
   ```

   可以看到模型返回的结果，"finish_reason"已经变为了"stop"，代表模型回复完了此次对话

### spring-ai封装

POM引入：

1. 客户端：

   ```xml
   <dependency>
       <groupId>org.springframework.ai</groupId>
       <artifactId>spring-ai-starter-mcp-client</artifactId>
   </dependency>
   ```

2. 服务端：

   ```xml
   <dependency>
       <groupId>org.springframework.ai</groupId>
       <artifactId>spring-ai-starter-mcp-server</artifactId>
   </dependency>
   ```

#### 客户端MCP

配置文件：新增mcp项，可以使用外部文件的方式引入mcp server，也可以直接写在yaml中，这里直接写在yaml中

```yaml
spring:
    mcp:
      client:
        stdio:
          # servers-configuration: classpath:/config/mcp-servers-config.json
            connections:
              test-server:
                command: npx.cmd
                args:
                  - "-y"
                  - "@modelcontextprotocol/server-filesystem"
                  - "C:\\Users\\86183\\Desktop\\mcp-test"           # 授权访问的目录
        enabled: true
```



前面提到，在spring-ai中，ChatModel的调用已经封装了工具的调用，因此开发者只需要提供工具，这涉及到以下类：

1. McpSyncClient：与MCP Server对接的客户端对象，负责获取工具信息，调用工具等（理解有这个东西就好，一般工具在载入后会直接到SyncMcpToolCallbackProvider里）
2. SyncMcpToolCallbackProvider：字面意思，调用工具提供方，也就是解析响应，通过mcpclient拿到工具列表，并连接client和每一个toolcallback
3. ToolCallback[]：基本上可以理解是python流程中的tools数组，不过数组中的每一个tool都可以请求对应的mcp server api（通过McpSyncClient）

```java
@Resource(name = "openAiChatClient")
private ChatClient openAiChatClient;

@Resource(name = "openAiChatModel")
private ChatModel openAiChatModel;

@Resource
private List<McpSyncClient> mcpSyncClients;

private ToolCallback[] getMcpTools() {
    return mcpSyncClients.stream()
        .map(SyncMcpToolCallbackProvider::new)
        .flatMap(p -> Arrays.stream(p.getToolCallbacks()))
        .toArray(ToolCallback[]::new);
}
@Test
public void test_dynamic_mcp_tools() {
    String answer = openAiChatClient.prompt()
        .user("列出允许访问的目录并读取Desktop\\mcp-test\\mcp.txt 的内容")
        .toolCallbacks(List.of(getMcpTools()))      // ← 本次请求附加工具
        .call()
        .content();

    System.out.println("回答：" + answer);
}
```

#### 服务端MCP

配置文件：在日志配置文件中不能让日志文件作为

```yaml
spring:
  application:
    name: mcp-server-computer

  ai:
    mcp:
      server:
        name: ${spring.application.name}
        version: 1.0.0
        
  main:
    banner-mode: off
    web-application-type: none

  server:
    servlet:
      encoding:
        charset: UTF-8
        force: true
        enabled: true

logging:
  config: classpath:logback-spring.xml
```

流程：

1. 在spring能够扫描的组件里的方法上加上注解@Tool(description = "xxx")（代表是一个可调用的工具）

   ```java
   @Service
   public class ComputerService {
   
       @Tool(description = "获取电脑配置")
       public ComputerFunctionResponse queryConfig(ComputerFunctionRequest request) {
           log.info("获取电脑配置信息 {}", request.getComputer());
           // 获取系统属性
           Properties properties = System.getProperties();
   
           // 操作系统名称
           String osName = properties.getProperty("os.name");
           // 操作系统版本
           String osVersion = properties.getProperty("os.version");
           // 操作系统架构
           String osArch = properties.getProperty("os.arch");
           // 用户的账户名称
           String userName = properties.getProperty("user.name");
           // 用户的主目录
           String userHome = properties.getProperty("user.home");
           // 用户的当前工作目录
           String userDir = properties.getProperty("user.dir");
           // Java 运行时环境版本
           String javaVersion = properties.getProperty("java.version");
   
           String osInfo = "";
           // 根据操作系统执行特定的命令来获取更多信息
           if (osName.toLowerCase().contains("win")) {
               // Windows特定的代码
               osInfo = getWindowsSpecificInfo();
           } else if (osName.toLowerCase().contains("mac")) {
               // macOS特定的代码
               osInfo = getMacSpecificInfo();
           } else if (osName.toLowerCase().contains("nix") || osName.toLowerCase().contains("nux")) {
               // Linux特定的代码
               osInfo = getLinuxSpecificInfo();
           }
   
           ComputerFunctionResponse response = new ComputerFunctionResponse();
           response.setOsName(osName);
           response.setOsVersion(osVersion);
           response.setOsArch(osArch);
           response.setUserName(userName);
           response.setUserHome(userHome);
           response.setUserDir(userDir);
           response.setJavaVersion(javaVersion);
           response.setOsInfo(osInfo);
   
           return response;
       }
   }
   ```

2. 注册（向外部提供这些工具的基本信息），在配置类中声明Bean

   ```java
   @Bean
   public ToolCallbackProvider computerTools(ComputerService computerService) {
       return MethodToolCallbackProvider.builder()
           .toolObjects(computerService)
           .build();
   }
   ```



## Agent

