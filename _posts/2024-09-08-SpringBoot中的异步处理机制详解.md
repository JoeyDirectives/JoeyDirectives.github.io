---
layout:       post
title:        "SpringBoot 中的异步处理机制详解"
author:       "Joey"
header-style: text
catalog:      true
tags:
    - Spring
---

在现代 Web 应用程序开发中，处理并发任务和提高应用程序的性能是一个重要的主题。Spring Boot 提供了强大的异步处理机制，允许我们通过简单的配置实现多线程任务处理，避免阻塞主线程，从而提升应用的响应速度和吞吐量。本文将深入探讨 Spring Boot 中的异步处理机制，介绍其原理、配置方法以及实际使用场景。 

## **1、为什么需要异步处理？**

在 Web 应用程序中，某些任务可能需要花费较长的时间，比如调用外部服务、执行文件 I/O 操作或处理复杂的计算逻辑。如果这些任务在主线程（即请求处理线程）中执行，会导致响应变慢，影响用户体验。通过异步处理，我们可以将这些耗时任务交给后台线程执行，释放主线程，尽早返回响应。 

## **2、Spring Boot 异步处理的基本原理**

Spring Boot 中的异步处理基于 Spring Framework 的 @Async 注解，它利用线程池来异步执行任务。通过简单地在方法上加上 @Async，Spring 会自动在后台线程中执行该方法，而不会阻塞主线程。 Spring 的异步支持核心依赖以下两个组件： 

* @EnableAsync：启用异步处理的注解。    
* @Async：标注需要异步执行的方法。

**`@EnableAsync` 注解**

- **作用：** 开启 Spring 的异步处理能力。它通常被添加在 Spring 配置类上，用于告诉 Spring 框架启用异步执行。
- 实现原理：
  - `@EnableAsync` 注解内部使用了一个 `AsyncConfigurationSelector` 类，该类会根据条件选择合适的异步配置类。
  - 默认情况下，如果没有自定义的异步配置类，Spring Boot 会使用 `SimpleAsyncConfiguration`，它会自动配置一个 `ThreadPoolTaskExecutor` 线程池。
  - `ThreadPoolTaskExecutor` 负责管理线程的创建和调度，用于执行异步任务。

**`@Async` 注解**

- **作用：** 标记一个方法为异步方法。当调用被 `@Async` 注解标记的方法时，Spring 会将该方法提交给线程池异步执行，而不是在当前线程中同步执行。
- 实现原理：
  - Spring 使用 AOP（面向切面编程）来实现 `@Async` 注解的功能。
  - 当 Spring 容器启动时，会扫描所有 Bean，查找带有 `@Async` 注解的方法。
  - 对于找到的 `@Async` 方法，Spring 会为其创建一个代理对象。
  - 当调用代理对象的方法时，代理对象会将方法调用委托给线程池中的一个线程异步执行。

### **2.1.SpringBoot 中使用默认线程池**

在`org.springframework.boot.autoconfigure.task`包的`TaskExecutionAutoConfiguration.java`是SpringBoot默认的任务执行自动配置类。
从`@EnableConfigurationProperties(TaskExecutionProperties.class)`可以知道开启了属性绑定到`TaskExecutionProperties.java`的实体类上。

进入到`TaskExecutionProperties.java`类中，看到属性绑定以`spring.task.execution`为前缀。默认线程池的核心线程数`coreSize=8`，最大线程数`maxSize = Integer.MAX_VALUE`，以及任务等待队列`queueCapacity = Integer.MAX_VALUE`
因为`Integer.MAX_VALUE`的值为2147483647(2的31次方-1)，所以默认情况下，一般任务队列就可能把内存给堆满了。我们真正使用的时候，还需要对异步任务的执行线程池做一些基础配置，以防止出现内存溢出导致服务不可用的问题。

## **3、Spring Boot 异步处理的配置**

### 3.1. 启用异步支持

在使用异步功能之前，我们需要在 Spring Boot 应用的启动类或配置类中使用 @EnableAsync 注解启用异步支持：

```java 
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableAsync;

@SpringBootApplication
@EnableAsync
public class AsyncApplication {
    public static void main(String[] args) {
        SpringApplication.run(AsyncApplication.class, args);
    }
}
```

### 3.2. 创建异步方法

接下来，我们可以在服务层创建一个异步执行的方法。使用 @Async 注解标注的方法会在单独的线程中执行。 

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class AsyncService {

    @Async
    public void executeAsyncTask() {
        System.out.println("执行异步任务: " + Thread.currentThread().getName());
        // 模拟长时间任务
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("任务完成");
    }
}
```

在该例子中，`executeAsyncTask()` 方法被标记为异步执行，当它被调用时，将不会阻塞调用者线程。 

### 3.3. 调用异步方法

我们可以在控制器或服务中调用异步方法： 

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AsyncController {

    private final AsyncService asyncService;

    public AsyncController(AsyncService asyncService) {
        this.asyncService = asyncService;
    }

    @GetMapping("/async")
    public String triggerAsyncTask() {
        asyncService.executeAsyncTask();
        return "异步任务已触发";
    }
}
```

当用户访问 /async 端点时，AsyncService 的异步任务会被触发，并立即返回响应，主线程不会等待异步任务的完成。 

## **4、自定义线程池** 

默认情况下，Spring Boot 会使用一个简单的线程池来执行异步任务。但是在实际的生产环境中，我们通常需要对线程池进行更精细的配置。我们可以通过定义 Executor Bean 来自定义线程池。 

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(25);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Override
    public Executor getAsyncExecutor() {
        return asyncTaskExecutor();
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        // 异步任务的异常处理默认情况下只会打印日志信息，不会做任何额外的额处理,SimpleAsyncUncaughtExceptionHandler就是异步任务异常处理的默认实现，如果想要自定义异常处理，只需要AsyncConfigurer接口即可
        return (throwable, method, objects) -> {
            log.error("异步任务执行出现异常, message {}, emthod {}, params {}", throwable, method, objects);
        };
    }
}
```

在这个配置中，我们创建了一个线程池 Executor，指定了核心线程数、最大线程数、队列容量等参数。线程池可以帮助我们更好地管理资源，避免大量并发任务导致系统资源耗尽。 为了让 Spring 使用这个自定义的线程池，我们需要在异步方法上指定线程池名称：

 ```java
@Async("taskExecutor")
public void executeAsyncTask() {
    // 任务逻辑
}
 ```

## **5、异步方法的返回值**

除了 void 类型，异步方法还可以返回 `java.util.concurrent.Future` 或 `org.springframework.util.concurrent.ListenableFuture`，允许我们在异步任务完成后获取其结果。 例如，返回 Future 类型的异步方法：

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.Future;

@Async
public Future<String> asyncMethodWithReturn() {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return CompletableFuture.completedFuture("异步任务结果");
}
```

调用该方法时，我们可以通过 `Future.get()` 获取异步任务的结果：

```java
@GetMapping("/asyncResult")
public String getAsyncResult() throws Exception {
    Future<String> result = asyncService.asyncMethodWithReturn();
    return result.get();
}
```

需要注意的是，`Future.get()` 方法会阻塞调用线程，直到异步任务完成，因此在实际使用时我们应当尽量避免在主线程中直接调用 `get()`。 

**6、异步处理的典型应用场景**  

* 外部 API 调用：调用第三方服务时，响应时间可能较长，可以使用异步请求来避免阻塞主线程。     

* 文件上传/下载：处理大文件上传或下载时，异步任务可以提升用户体验，确保应用的其他功能不会因文件处理而变慢。     

* 消息队列：异步处理常用于消息队列中，比如接收和处理 Kafka、RabbitMQ 消息时，可以在后台异步处理消息，提升应用的并发能力。    

## **7、总结**

Spring Boot 的异步处理机制极大地简化了多线程开发，使得我们能够在无需管理复杂的线程逻辑的情况下，通过简单的注解实现异步任务执行。本文介绍了异步处理的基础配置、线程池的自定义以及常见应用场景。在实际应用中，异步处理可以有效提升应用的性能，改善用户体验，但同时也需要我们合理管理线程池，确保系统资源的高效利用。
