---
title: "Alibaba Sentinel介绍"
categories:
  - Spring Cloud
header:
  overlay_color: "#333"
  caption: "官网: [**alibaba-Sentinel**](https://sentinelguard.io/zh-cn/)"  
tags:
  - Spring Cloud Config
  - Sentinel
cover_image: /assets/images/spring-cloud/sentinel.png
---

顾名思义，`Sentinel` 是微服务的强大守护者。它提供**流量控制**、**并发限制**、**断路**和**自适应系统保护**等功能，以保证其可靠性。它是阿里巴巴集团积极维护的开源组件。此外，它正式成为 `Spring Cloud Circuit Breaker.` 的一部分。

在本文中，我们将了解 `Sentinel` 的一些主要功能。此外，我们将看到如何使用它的示例、它的注释支持和它的监控仪表板

<figure style="width: 700px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/spring-cloud/sentinel.png" alt="">
  <figcaption>Sentinel 角色</figcaption>
</figure>

{% include toc %}

## 特性

### 流量控制

`Sentinel` 控制随机请求传入的速度，以避免微服务过载。这可确保我们的服务不会因流量激增而中断。它支持多种流量整形策略。当每秒查询数 (QPS) 过高时，这些策略会自动将流量调整为适当的策略。

其中一些流量整形策略是：

- 直接拒绝模式--当每秒请求数超过设定的阈值时，它会自动拒绝进一步的请求；
- 慢启动预热模式--如果流量突然激增，此模式可确保请求计数逐渐增加，直到达到上限;

### 熔断和降级

当一个服务同步调用另一个服务时，另一个服务可能会因某种原因关闭。在这种情况下，线程会被阻塞，因为它们一直在等待其他服务响应。这可能会导致资源耗尽，并且调用方服务也将无法处理进一步的请求。**这称为级联效应，可以摧毁我们的整个微服务架构**。

为了防止这种情况，断路器出现在架构中。它将立即阻止对其他服务的所有后续调用。超时时间过后，会传递一些请求。如果它们成功，则断路器恢复正常流程。否则，超时时间将重新开始。

**`Sentinel` 使用最大并发限制原理来实现熔断**。它通过限制并发线程数来减少不稳定资源的影响。

`Sentinel` 还会降级不稳定的资源。当资源的响应时间太长时，对资源的所有调用都会在指定的时间窗口内被拒绝。这可以防止调用变得非常缓慢，从而导致级联效应的情况。

### 自适应系统保护

**`Sentinel` 会在系统负载过高时保护我们的服务器** 。它使用 `load1`（系统负载）作为指标来启动流量控制。请求将在以下情况下被阻止：

- 当前系统负载（`load1`）> 阈值（`highestSystemLoad`）；
- 当前并发请求（线程数）> 估算容量（最小响应时间 * 最大 QPS）

## 使用

### 1. 依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.8.0</version>
</dependency>
```

### 2. 定义资源

使用 `Sentinel API` 在 `try-catch` 块中定义具有相应业务逻辑的资源：

```java
try (Entry entry = SphU.entry("HelloWorld")) {
    // Our business logic here.
    System.out.println("hello world");
} catch (BlockException e) {
    // Handle rejected request.
}

```

### 3. 定义流量控制规则

这些规则控制流向我们的资源，例如阈值计数或控制行为——例如，直接拒绝或缓慢启动。我们使用 `FlowRuleManager.loadRules()` 来配置流规则：

```java
List<FlowRule> flowRules = new ArrayList<>();
FlowRule flowRule = new FlowRule();
flowRule.setResource(RESOURCE_NAME);
flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
flowRule.setCount(1);
flowRules.add(flowRule);
FlowRuleManager.loadRules(flowRules);
```

> 此规则定义了我们的资源`RESOURCE_NAME`每秒最多可以响应一个请求。

### 4. 定义降级规则

使用降级规则，我们可以配置断路器的阈值 `请求计数`、`恢复超时`和其他设置。 我们使用 `DegradeRuleManager.loadRules(`) 配置降级规则：

```java
List<DegradeRule> rules = new ArrayList<DegradeRule>();
DegradeRule rule = new DegradeRule();
rule.setResource(RESOURCE_NAME);
rule.setCount(10);
rule.setTimeWindow(10);
rules.add(rule);
DegradeRuleManager.loadRules(rules);
```

> 此规则指定，当我们的资源 `RESOURCE_NAME` 无法满足 10 个请求（阈值计数）时，请求链路将中断。对资源的所有后续请求都将被 `Sentinel` 阻止 10 秒（时间窗口）。

### 5. 定义系统保护规则

使用系统保护规则，我们可以配置和确保自适应系统保护（`load1 阈值`、`平均响应时间`、`并发线程数`）。我们使用 `SystemRuleManager.loadRules()` 方法配置系统规则：

```java
List<SystemRule> rules = new ArrayList<>();
SystemRule rule = new SystemRule();
rule.setHighestSystemLoad(10);
rules.add(rule);
SystemRuleManager.loadRules(rules);
```

> 此规则指定，对于我们的系统，最高系统负载为每秒 10 个请求。如果当前负载超过此阈值，则所有进一步的请求都将被阻止。

## 注解支持

`Sentinel` 还为定义资源提供了面向方面的注释支持。

1. 添加依赖
  
```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-annotation-aspectj</artifactId>
    <version>1.8.0</version>
</dependency>
```

2. 将 `sentinel aspect ` 注入为bean;

```java
@Configuration
public class SentinelAspectConfiguration {

    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
}
```

3. `@SentinelResource` 表示资源定义。它有 `value` 之类的属性，它定义了资源名称。属性`fallback`是回退方法名称。当断路器断开时，这种回退方法定义了我们程序的替代流程。让我们使用 `@SentinelResource` 注释定义资源：

```java
@SentinelResource(value = "resource_name", fallback = "doFallback")
public String doSomething(long i) {
    return "Hello " + i;
}

public String doFallback(long i, Throwable t) {
    // Return fallback value.
    return "fallback";
}
```

> 这定义了名为 `resource_name` 的资源以及回退方法。

## 监控仪表盘

`Sentinel` 还提供了一个监控仪表板。有了这个，我们可以监控客户端并动态配置规则。我们可以实时查看我们定义的资源的传入流量。

### 1. 启动仪表盘服务

首先，我们需要下载 [Sentinel Dashboard jar](https://github.com/alibaba/Sentinel/releases) 。然后，我们可以使用以下命令启动仪表板：

```bash
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar
```

**TIP**： 一旦仪表板应用程序启动，我们就可以按照下一节中的步骤连接我们的应用程序。
{: .notice--info}

### 2. 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>1.8.0</version>
</dependency>
```

### 链接服务到仪表盘

启动应用程序时，我们需要添加仪表板IP地址：

```bash
-Dcsp.sentinel.dashboard.server=consoleIp:port
```

> 现在，无论何时调用资源，仪表板都会从我们的应用程序接收心跳：

<figure style="width: 700px">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/images/spring-cloud/sentinel_dashboard.png" alt="">
  <figcaption>sentinel dashboard</figcaption>
</figure>

**TIP**: 我们还可以使用仪表板动态地操纵流、降级和系统规则。
{: .notice--info}
