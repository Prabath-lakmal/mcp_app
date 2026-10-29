# Comprehensive Guide to Spring AI Concepts

Spring AI is an application framework designed to simplify the development of applications that incorporate Artificial Intelligence (AI) functionality without unnecessary complexity. Inspired by Spring design principles like portability, modularity, and unified interfaces (similar to `JdbcTemplate` or `RestTemplate`), Spring AI allows Java developers to integrate Artificial Intelligence models and services into existing Spring applications seamlessly.

---

## Table of Contents
1. [Core Philosophy & Architecture](#1-core-philosophy--architecture)
2. [Model APIs (`ChatModel`, `EmbeddingModel`, `ImageModel`)](#2-model-apis)
3. [Prompts & Prompt Templates](#3-prompts--prompt-templates)
4. [ChatClient API](#4-chatclient-api)
5. [Structured Output Converters](#5-structured-output-converters)
6. [Function Calling / Tool Integration](#6-function-calling--tool-integration)
7. [Vector Databases & VectorStore API](#7-vector-databases--vectorstore-api)
8. [Retrieval-Augmented Generation (RAG) & Document Readers](#8-retrieval-augmented-generation-rag--document-readers)
9. [Advisors API & Chat Memory](#9-advisors-api--chat-memory)
10. [End-to-End Code Example](#10-end-to-end-code-example)

---

## 1. Core Philosophy & Architecture

Spring AI provides a **portable abstraction layer** across various AI model providers (OpenAI, Azure OpenAI, Anthropic Claude, Ollama, AWS Bedrock, Google Vertex AI, HuggingFace, etc.). 

Key benefits of Spring AI's design:
- **Provider Portability:** Switch from OpenAI to Ollama (local LLM) or Azure OpenAI by simply changing configurations and starter dependencies—without rewriting domain logic.
- **Spring Idiomatic Design:** Leverages Dependency Injection (`@Autowired`), Spring Boot Auto-configuration (`application.yml`), and Fluent Builder Patterns.
- **Enterprise Ready:** Native support for vector stores, structured JSON mapping, function calling, chat memory management, and evaluation.

---

## 2. Model APIs

Model APIs represent the fundamental abstraction for interacting with AI models. Spring AI categorizes models into standard types based on their modal capabilities.

### Key Model Interfaces:
- `ChatModel`: Generates text or conversational responses.
- `StreamingChatModel`: Supports reactive, token-by-token streaming responses using Project Reactor (`Flux`).
- `EmbeddingModel`: Converts text strings into numeric vector arrays (`float[]`).
- `ImageModel`: Generates images from textual descriptions.
- `AudioTranscriptionModel` & `TextToSpeechModel`: Handles speech processing.

### Example: Using `ChatModel` directly
```java
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class SimpleAiService {

    private final ChatModel chatModel;

    @Autowired
    public SimpleAiService(ChatModel chatModel) {
        this.chatModel = chatModel;
    }

    public String askQuestion(String question) {
        // Wrap query in a Prompt object
        Prompt prompt = new Prompt(question);
        ChatResponse response = chatModel.call(prompt);
        
        // Extract textual response content
        return response.getResult().getOutput().getContent();
    }
}
```

---

## 3. Prompts & Prompt Templates

A **Prompt** in Spring AI is an encapsulated input sent to an AI model. It consists of structured messages representing different roles in a conversation:

1. **`SystemMessage`**: Provides context, persona instructions, or rules for the AI.
2. **`UserMessage`**: Represents input submitted by the human user.
3. **`AssistantMessage`**: Represents previous or current responses produced by the AI model.

### Prompt Templates
`PromptTemplate` allows dynamic parameter substitution into prompt strings using templates with placeholder tags (e.g., `{topic}`).

### Example: Prompt Templating with Roles
```java
import org.springframework.ai.chat.messages.Message;
import org.springframework.ai.chat.messages.SystemMessage;
import org.springframework.ai.chat.messages.UserMessage;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import java.util.List;
import java.util.Map;

public class PromptExample {

    public Prompt createCustomPrompt(String userTopic, String style) {
        // System message setting behavior/persona
        SystemMessage systemMessage = new SystemMessage("You are an expert technical writer.");

        // Dynamic template for user input
        PromptTemplate userTemplate = new PromptTemplate("Explain {topic} in a {style} tone with practical examples.");
        Message userMessage = userTemplate.createMessage(Map.of("topic", userTopic, "style", style));

        // Combine into a multi-message Prompt
        return new Prompt(List.of(systemMessage, userMessage));
    }
}
```

---

## 4. ChatClient API

`ChatClient` is the recommended high-level, fluent API for interacting with LLMs in Spring AI. It simplifies prompt construction, advisor registration, default options, and output parsing.

### Advantages of `ChatClient`:
- Chainable fluent builder interface (`prompt().user().call().content()`).
- Inlined prompt templates.
- Automatic object mapping / entities support.
- Simple integration with streaming (`stream()`) and memory advisors.

### Example: Controller using `ChatClient`
```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
public class RecommendationController {

    private final ChatClient chatClient;

    // ChatClient.Builder is auto-configured by Spring Boot
    public RecommendationController(ChatClient.Builder builder) {
        this.chatClient = builder
                .defaultSystem("You are a friendly travel recommendation assistant.")
                .build();
    }

    // Synchronous call
    @GetMapping("/recommend")
    public String getRecommendation(@RequestParam String city) {
        return chatClient.prompt()
                .user(u -> u.text("What are 3 must-visit spots in {city}?").param("city", city))
                .call()
                .content();
    }

    // Reactive / Streaming call
    @GetMapping("/recommend/stream")
    public Flux<String> streamRecommendation(@RequestParam String city) {
        return chatClient.prompt()
                .user(u -> u.text("Describe the cultural history of {city} in detail.").param("city", city))
                .stream()
                .content();
    }
}
```

---

## 5. Structured Output Converters

LLMs output unstructured raw text by default. Spring AI provides `StructuredOutputConverter` (and `BeanOutputConverter`) to convert raw LLM responses directly into strongly typed Java objects/records.

Under the hood, Spring AI injects JSON Schema instructions into the prompt demanding formatted output and deserializes the response using Jackson.

### Example: Mapping Response to Java Record
```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;
import java.util.List;

// Strongly typed Java record
public record MovieRecommendation(
        String title,
        int releaseYear,
        String genre,
        List<String> mainCast
) {}

@Service
public class MovieService {

    private final ChatClient chatClient;

    public MovieService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public MovieRecommendation getMovieInfo(String movieName) {
        return chatClient.prompt()
                .user(u -> u.text("Provide details for the movie: {movie}").param("movie", movieName))
                .call()
                .entity(MovieRecommendation.class); // Automatically requests JSON schema & parses
    }
}
```

---

## 6. Function Calling / Tool Integration

Function Calling allows LLMs to interact with custom local Java code or external APIs (e.g., live weather APIs, database lookups, payment processing).

### How it works:
1. Define a Java `Function<Request, Response>` bean.
2. Annotate it with `@Description` to tell the model *when* and *why* to use it.
3. Register the function name with the model request.
4. When processing, the LLM determines if function execution is required, returns parameters to execute the Java function locally, and receives the result back to formulate its final answer.

### Example: Tool / Function Integration
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Description;
import java.util.function.Function;

@Configuration
public class ToolConfig {

    public record WeatherRequest(String city) {}
    public record WeatherResponse(String city, String temperature, String condition) {}

    @Bean
    @Description("Get current weather report for a given city")
    public Function<WeatherRequest, WeatherResponse> currentWeatherFunction() {
        return request -> {
            // Simulated live API call or database query
            if (request.city().equalsIgnoreCase("Tokyo")) {
                return new WeatherResponse("Tokyo", "22°C", "Sunny");
            }
            return new WeatherResponse(request.city(), "18°C", "Partly Cloudy");
        };
    }
}
```

### Invoking the Tool via `ChatClient`
```java
@Service
public class WeatherAiService {

    private final ChatClient chatClient;

    public WeatherAiService(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    public String askAboutWeather(String promptText) {
        return chatClient.prompt()
                .user(promptText)
                .functions("currentWeatherFunction") // Links the @Bean by name
                .call()
                .content();
    }
}
```

---

## 7. Vector Databases & VectorStore API

In AI applications, unstructured text is converted into high-dimensional numerical vectors (embeddings) to enable **semantic search**.

Spring AI's `VectorStore` interface provides unified data operations across different vector store engines (PGvector, Pinecone, Qdrant, Milvus, Redis, Weaviate, Chroma, etc.).

### Key Capabilities:
- `add(List<Document> documents)`: Encodes text via `EmbeddingModel` and saves vector embeddings.
- `similaritySearch(SearchRequest request)`: Retrieves top-K documents most semantically similar to a query.

### Example: Adding and Searching Vector Data
```java
import org.springframework.ai.document.Document;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;

@Service
public class KnowledgeBaseService {

    private final VectorStore vectorStore;

    public KnowledgeBaseService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void storeDocuments() {
        List<Document> docs = List.of(
                new Document("Spring Boot simplifies Java enterprise application setup.", Map.of("category", "java")),
                new Document("Spring AI offers seamless integration with vector stores and LLMs.", Map.of("category", "ai"))
        );
        
        // Converts text to embeddings & persists to database
        vectorStore.add(docs);
    }

    public List<Document> searchKnowledge(String query) {
        return vectorStore.similaritySearch(
                SearchRequest.query(query)
                        .withTopK(2)
                        .withSimilarityThreshold(0.7)
        );
    }
}
```

---

## 8. Retrieval-Augmented Generation (RAG) & Document Readers

**RAG (Retrieval-Augmented Generation)** addresses LLM knowledge cutoff dates and hallucination by retrieving private domain documents relevant to a user query and prepending them as context into the prompt.

### Document Pipeline (ETL in Spring AI):
1. **Reader:** Loads documents from sources (PDF Reader, Tika, Text/JSON Reader).
2. **Transformer / Splitter:** Splits long documents into smaller chunks (`TokenTextSplitter`).
3. **Writer:** Converts chunks to embeddings and saves into `VectorStore`.

### RAG Flow Architecture Diagram:
```
User Query ---> Search VectorStore ---> Extract Relevant Chunks
                                              |
                                              v
Final Answer <--- Send Context + Query <--- Formulate Augmented Prompt
```

### Example: Ingesting Documents for RAG
```java
import org.springframework.ai.document.Document;
import org.springframework.ai.reader.TextReader;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

@Service
public class DocumentIngestionService {

    private final VectorStore vectorStore;

    public DocumentIngestionService(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    public void ingestResource(Resource resource) {
        // 1. Read document
        TextReader reader = new TextReader(resource);
        var rawDocs = reader.get();

        // 2. Transform / Split into chunks suitable for LLM context windows
        TokenTextSplitter splitter = new TokenTextSplitter(500, 100, 5, 10000, true);
        var chunkedDocs = splitter.apply(rawDocs);

        // 3. Write to VectorStore
        vectorStore.accept(chunkedDocs);
    }
}
```

---

## 9. Advisors API & Chat Memory

Advisors act as interceptors / middleware in Spring AI execution flow. They modify prompts before sending them to the model or transform responses afterwards.

Common use cases:
- **`MessageChatMemoryAdvisor`**: Maintains conversational history across turns.
- **`QuestionAnswerAdvisor`**: Automates RAG by fetching contextual context from a `VectorStore` seamlessly before executing prompt.

### Example: RAG and Conversational Memory Advisor
```java
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.chat.memory.InMemoryChatMemory;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/chat")
public class RAGChatController {

    private final ChatClient chatClient;

    public RAGChatController(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
                .defaultAdvisors(
                        // Enables conversational memory
                        new MessageChatMemoryAdvisor(new InMemoryChatMemory()),
                        // Enables automatic RAG document retrieval
                        new QuestionAnswerAdvisor(vectorStore)
                )
                .build();
    }

    @PostMapping
    public String chat(@RequestParam String conversationId, @RequestBody String userMessage) {
        return chatClient.prompt()
                .user(userMessage)
                .advisors(a -> a.param(MessageChatMemoryAdvisor.CHAT_MEMORY_CONVERSATION_ID_KEY, conversationId))
                .call()
                .content();
    }
}
```

---

## 10. End-to-End Code Example

Here is a complete Spring Boot service combining ChatClient, Vector Storage RAG, and Structured Output.

```java
package com.example.ai;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.VectorStore;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class CustomerSupportAiService {

    private final ChatClient chatClient;

    public record SupportAnalysis(
            String summary,
            String sentiment,
            List<String> recommendedActions
    ) {}

    public CustomerSupportAiService(ChatClient.Builder builder, VectorStore vectorStore) {
        this.chatClient = builder
                .defaultSystem("You are an automated support analysis assistant.")
                .defaultAdvisors(new QuestionAnswerAdvisor(vectorStore, SearchRequest.defaults()))
                .build();
    }

    public SupportAnalysis analyzeTicket(String ticketDescription) {
        return chatClient.prompt()
                .user(u -> u.text("Analyze customer ticket and recommend actions based on knowledge base: {ticket}")
                           .param("ticket", ticketDescription))
                .call()
                .entity(SupportAnalysis.class);
    }
}
```

---

## Summary of Spring AI Core Components

| Component | Responsibility |
| :--- | :--- |
| **`ChatModel` / `StreamingChatModel`** | Low-level model invocation abstractions. |
| **`ChatClient`** | Fluent API for building prompts, chaining advisors, and getting responses. |
| **`Prompt` & `PromptTemplate`** | Structured message formatting and dynamic variable injection. |
| **`StructuredOutputConverter`** | Maps raw model responses to Java objects/records. |
| **`FunctionCallback` / `@Bean`** | Enables LLMs to execute custom Java functions/tools. |
| **`VectorStore`** | Storage and similarity searching of embedded text data. |
| **`QuestionAnswerAdvisor`** | Automates Retrieval-Augmented Generation (RAG). |
| **`MessageChatMemoryAdvisor`** | Persists conversation history across multi-turn interactions. |
