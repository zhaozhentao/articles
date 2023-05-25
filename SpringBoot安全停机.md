### 背景

最近做一个项目，需要实现服务节点的优雅下线功能。所谓优雅下线，可以简单概括为停止接收新的请求，并在处理完现有请求后关闭 (核心思想)。就像 Nginx 对 quit 信号的处理一样。

因为其他项目已经实现过这个功能，可能只需要搬一下代码过来就 OK。

### 不要轻易相信别人的代码

不要轻易相信别人的代码(包括本文)，在复制代码之前，不妨先看一下原来的实现。通过代码可以看到，安全停机功能通过一个 http 接口触发，实现的功能大概如下。

1. 从注册中心 nacos 中注销服务
2. 检查当前是否还有未结束的请求，如果有未结束的请求则不关闭 Springboot 程序 (这里的代码比较复杂)

### 疑问

我们知道，一个运行中的 SpringBoot 程序，除了与注册中心建立的连接、集群内节点之间的请求 (通常是集群内部的 feign 调用) 需要在停机时候做安全销毁，其实还有很多的资源诸如数据库连接、消息队列连接、Redis 连接、正在执行的异步任务、定时任务（可能正在执行）这些同样也是需要做安全销毁的。但从目前实现的代码中，似乎没有找到相关的处理 ？

另外一个不合理的现象是，即便是这个安全停机的接口调用完后，Java 进程并没有结束 (既然是停机进程就要彻底关闭) ... 所以我认为这其实并没有实现真正意义的安全停机。

### 了解原理少写代码

出于一个原则，如果实现一个功能需要写大量代码的时候，应该要意识到这可能不是一个好的解决办法，所以是时候重新思考一下，应该如何实现 SpringBoot 程序的安全停机，更普适地说就是如何正确关闭一个进程。

> 网络上也有不少文章介绍通过 spring-boot-starter-actuator 的 shutdown 端点实现停机，但是同样面临一个问题 spring-boot-starter-actuator 并不会知道开发者会在应用运行过程申请什么资源，所以并不存在一个一劳永逸的接口可以直接实现安全停机。

其实，操作系统在设计之初就包含了一个用于进程间通讯的机制 - - - "信号" 。我们平时经常会使用 ctrl + c 去停止正在执行的程序其实就是向程序发送 SIGINT 信号，当一个进程收到 SIGINT 信号时，就应该要在清理完各种资源后结束进程（资源包括打开的文件，建立的各种连接等等），下面是一个简单的例子。

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>

void handler(int sign_no) {
    printf("收到信号 %d bye~\n", sign_no);
    exit(0);
}

int main() {
    printf("等待信号...\n");

    signal(SIGINT, handler);

    while (1) {
        pause();
    }

    return 0;
}
```

> 相较于 SIGINT 这种应用程序可感知的信号，也有像 SIGKILL 这种应用程序不可感知的信号 (无法注册信号处理函数)，这种信号用于进程无法正常退出时 (收到 SIGINT 后忽略或因其他原因没有退出)，操作系统强制关闭进程，我们平时就是用 kill -9 pid 向进程发送 SIGKILL 信号。

所以，操作系统一开始就为我们提供了关闭进程的标准入口，就像那些带有图形界面的应用，每个应用都天然自带图中左上角的关闭按钮，也是操作系统提供的规范，我们没有必要在应用内部再额外的去创建退出按钮。对于我们的例子，自己实现的停机接口就相当于下图中的退出按钮。

![file](http://coding.bozhen.live/upload/2023_05_24/17_11_34_image.png)

另外，通常我们写的一些多线程程序，都会在主线程中等待其他所有的线程结束，所以进程退出的另一个条件就是没有其他存活的线程(非守护线程)。通过 debug 工具可以看到，在调用完前面的安全停机接口后，依然有活动的非守护线程。

### Bean 的生命周期

知道应该如何退出一个进程后，那么在 SpringBoot 这个框架下要如何实现上面的这一过程呢？其实想想也知道，任何一个成熟的框架比如 Spring ，都应该早就处理好这些最基本的事情，我们只需要遵守 Spring 的规范在正确的时机实现资源回收即可。

查看一下 SpringBoot 源码不难发现，Spring 在启动阶段也会注册和上面程序类似的处理程序，具体过程

1. SpringApplication.run()
2. SpringApplication.refreshContext()
3. SpringApplicationShutdownHook.registerApplicationContext()
4. SpringApplicationShutdownHook.addRuntimeShutdownHookIfNecessary()
5. SpringApplicationShutdownHook.addRuntimeShutdownHook()

SpringApplicationShutdownHook 实现的 run 接口代码如下。

```java
@Override
public void run() {
	Set<ConfigurableApplicationContext> contexts;
	Set<ConfigurableApplicationContext> closedContexts;
	Set<Runnable> actions;
	synchronized (SpringApplicationShutdownHook.class) {
		this.inProgress = true;
		contexts = new LinkedHashSet<>(this.contexts);
		closedContexts = new LinkedHashSet<>(this.closedContexts);
		actions = new LinkedHashSet<>(this.handlers.getActions());
	}
	contexts.forEach(this::closeAndWait);
	closedContexts.forEach(this::closeAndWait);
	actions.forEach(Runnable::run);
}
```

不难猜测，上面的代码就是要实现资源的回收，对于我们的应用来说，我们只需要利用好 Spring 提供的生命周期，实现相关的回调，等待 Spring 通知我们执行触发资源回收则可。在项目中我选用的是 @PreDestroy。

> 相较于前端的框架，生命周期这一概念在后端框架的存在感可以说是弱很多，写过 Android 或 iOS 应用的一定不会忘记 onCreate、onDestroy 和 viewWillAppear、viewWillDisappear，因为每个页面都有可能被频繁的创建或销毁，而后端的 Bean 则不存在这种情况，更关注是依赖注入。

### 实现

想清楚原理后，可以着手实现一下相关代码，首先要考虑的是哪些资源是需要回收的。一个经验之谈，我们引入的那些 xx-starter 组件，都是已经适配好 SpringBoot 生命周期的，不需要我们关注，比如连接注册中心用到的 spring-cloud-starter-alibaba-nacos-discovery。而那些没有适配 Spring 生命周期的比如 org.redisson:redisson 这种需要使用长连接的组件，或者我们自己创建的各种线程池，则需要我们在特定的生命周期去回收这些资源。

检验是否我们的程序是否已经处理完资源回收的方法也很简单(至少是关闭了所有需要被关闭的线程)，只需要向正在运行中的 SpringBoot 程序发送 SIGINT 或 SIGTERM (如果程序在前台运行，直接 ctrl + c 即可，如果是后台运行的程序，只需要使用命令 kill pid，注意不是 kill -9)，如果进程能够关闭，说明所有该关闭的线程都关闭了。

如果发送终止信号后，进程没有关闭，则可以使用一些 debug 工具比如 jconsole 或 jstack 等，查看一下存活的线程，根据线程名找找对应的资源即可。

![file](http://coding.bozhen.live/upload/2023_05_25/15_01_20_image.png)

经过排查，项目中使用到的 Redis、Rocketmq 生产者和消费者、OssClient 和一个手动创建的 ThreadPoolTaskExecutor 需要被关闭，这里列举两处修改。

Redisson 的关闭

```diff
index f55bfa2..e3765d8 100644
--- a/src/main/java/com/ake/ape/common/infrastructure/lock/redis/RedisLockConfiguration.java
+++ b/src/main/java/com/ake/ape/common/infrastructure/lock/redis/RedisLockConfiguration.java
@@ -2,6 +2,7 @@ package com.ake.ape.common.infrastructure.lock.redis;

 import com.ake.ape.common.domain.contracts.lock.LockFactory;

+import lombok.extern.slf4j.Slf4j;
 import org.redisson.Redisson;
 import org.redisson.api.RedissonClient;
 import org.redisson.config.Config;
@@ -10,8 +11,10 @@ import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
 import org.springframework.context.annotation.Bean;
 import org.springframework.context.annotation.Configuration;
 
+import javax.annotation.PreDestroy;
 import java.io.IOException;
 
+@Slf4j
 @Configuration
 @ConditionalOnProperty(value = "lock.type", havingValue = "redisson")
 public class RedisLockConfiguration {
@@ -19,10 +22,19 @@ public class RedisLockConfiguration {
     @Value("${lock.redisson.config}")
     private String redissonConfig;
 
+    private RedissonClient client;
+
     @Bean
     public RedissonClient redissonLock() throws IOException {
         Config config = Config.fromYAML(redissonConfig);
-        return Redisson.create(config);
+        client = Redisson.create(config);
+        return client;
+    }
+
+    @PreDestroy
+    public void preDestroy() {
+        log.info("executor.shutdown");
+        client.shutdown();
     }
     
     @Bean
```

Rocketmq 生产者和消费者的关闭

```diff
diff --git a/src/main/java/com/ake/ape/common/infrastructure/eventbus/rocketmq/RocketMqEventBusInternalConfiguration.java b/src/main/java/com/ake/ape/common/infrastructure/e
ventbus/rocketmq/RocketMqEventBusInternalConfiguration.java
index fc0ec1e..33c1f9c 100644
--- a/src/main/java/com/ake/ape/common/infrastructure/eventbus/rocketmq/RocketMqEventBusInternalConfiguration.java
+++ b/src/main/java/com/ake/ape/common/infrastructure/eventbus/rocketmq/RocketMqEventBusInternalConfiguration.java
@@ -16,6 +16,8 @@ import brave.Tracer;
 import lombok.RequiredArgsConstructor;
 import lombok.extern.slf4j.Slf4j;
 
+import javax.annotation.PreDestroy;
+
 @Configuration
 @Slf4j
 @RequiredArgsConstructor
@@ -45,6 +47,13 @@ public class RocketMqEventBusInternalConfiguration implements SmartInitializingS
         return eventBusExternal;
     }
 
+    @PreDestroy
+    public void preDestroy() {
+        log.info("executor.shutdown");
+        eventBusExternal.producer.shutdown();
+        eventBusExternal.consumer.shutdown();
+    }
+
     @Override
     public void afterSingletonsInstantiated() {
         if (eventBusExternal != null) {
```

> '巧合'的是所有的资源都提供了一个统一的销毁方法名 shutdown 。

### 成果

经过改造后，现在向启动中的程序发送 SIGINT 可以看到如下输出。

![file](http://coding.bozhen.live/upload/2023_05_25/17_00_09_image.png)

和前面的猜想一样，spring-cloud-starter-alibaba-nacos-discovery 这种 xx-starter，不需要我们干预就会自动 shutdown，关闭完那些需要我们手动关闭的资源后，Java 进程也销毁了。
