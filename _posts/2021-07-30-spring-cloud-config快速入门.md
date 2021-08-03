---
title: "spring cloud config快速入门"
categories:
  - Spring Cloud
header:
  image: /assets/images/spring-cloud/spring-cloud-config-header.jpg
  caption: "官网: [**spring-cloud-config**](https://spring.io/projects/spring-cloud-config)"  
tags:
  - Spring Cloud Config
# link: https://github.com
cover_image: /assets/images/spring-cloud/spring-cloud-config1.jpeg

gallery1:
  - url: https://spring.io/projects/spring-cloud-config
    image_path: /assets/images/spring-cloud/spring-cloud-config1.jpeg
    alt: "Black and grays with a hint of green"
  - url: https://spring.io/projects/spring-cloud-config
    image_path: /assets/images/spring-cloud/spring-cloud-config3.jpg
    alt: "Fog in the trees"
---

`Spring Cloud Config` 跨多个应用程序和环境，用于存储和提供分布式配置的 `clinet/server`方案。
理想情况下，此配置存储在 `Git` 版本控制下进行版本控制，并且可以在应用程序运行时进行修改。
虽然它非常适合`Spring applications`所有支持的配置文件格式以及 `Environment`、`PropertySource` 或 `@Value` 等结构的 Spring 应用程序，但它也可以用于任何编程语言运行的任何环境使用。

在这篇文章中，我们将重点介绍如何设置 `Git` 支持的配置服务器、在简单的 `REST` 应用程序服务器中使用它以及设置包括加密属性值在内的安全环境的示例。

{% include gallery id="gallery1" caption="spring cloud config 架构" %}

{% include toc %}

## 项目依赖

1. 首先创建两个新的 `Maven` 项目。`server` 项目依赖于 `spring-cloud-config-server` 模块，以及 `spring-boot-starter-security` 和 `spring-boot-starter-web` 启动包.

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

2. 对于`client`项目，我们将只需要 `spring-cloud-starter-config` 和 `spring-boot-starter-web` 模块.
  
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## `server`端配置

`server`应用程序的主要部分是一个配置类——更具体地说是一个`@SpringBootApplication`——它通过自动配置注释`@EnableConfigServer` 引入所有需要的设置：

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServer {
    
    public static void main(String[] arguments) {
        SpringApplication.run(ConfigServer.class, arguments);
    }
}
```

配置服务器端口和一个提供版本控制的 `Git-url`。

**TIP:** 如果您计划使用指向同一个配置存储库的多个配置服务器实例，您可以将服务器配置为将您的存储库克隆到本地临时文件夹中。但是请注意具有双因素身份验证的私有存储库，它们很难处理！在这种情况下，在本地文件系统上克隆它们并使用副本会更容易。
{: .notice--info}

还有一些占位符变量和搜索模式用于配置可用的存储库 `url`.但这超出了我们文章的范围。官方文档是一个很好的起点。 我们还需要在 `application.properties` 中为 `Basic-Authentication` 设置用户名和密码，以避免每次都自动生成密码 应用程序重启：

```properties
server.port=8888
spring.cloud.config.server.git.uri=ssh://localhost/config-repo
spring.cloud.config.server.git.clone-on-start=true
spring.security.user.name=root
spring.security.user.password=s3cr3t

```

## 初始化 `配置仓库`

为了完成我们的服务器，我们必须在`server`  配置的 `spring.cloud.config.server.git.uri` 初始化一个 `Git repository`。 
配置文件的名称像普通的 `Spring` `application.properties` 一样组成，但不是`application`这个词。例如：

```bash
$> git init
$> echo 'user.role=Developer' > config-client-development.properties
$> echo 'user.role=User'      > config-client-production.properties
$> git add .
$> git commit -m 'Initial config-client properties'
```

## 查询配置

1. 启动`server` 端。
2. 使用以下api 查询 配置信息.
  
  ```txt
  /{application}/{profile}[/{label}]
  /{application}-{profile}.yml
  /{label}/{application}-{profile}.yml
  /{application}-{profile}.properties
  /{label}/{application}-{profile}.properties
  ```

其中 `{label}` 占位符指的是 `Git` 分支，`{application}` 指的是客户端的应用程序名称，而 `{profile}` 指的是客户端当前活动的应用程序配置文件。

比如， 我们可以通过以下方式检索在分支 `master` 下 `profile=development`的配置属性.

```bash
$> curl http://root:s3cr3t@localhost:8888/config-client/development/master
```

## `client`配置

这是一个非常简单的客户端应用程序，由一个带有一个 `GET` 方法的 `REST` 控制器组成。 用于获取服务器的配置必须放在名为 `bootstrap.application` 的`resource`文件中，因为该文件（顾名思义）将在应用程序启动时很早加载：

1. app:
  
```java
@SpringBootApplication
@RestController
public class ConfigClient {
    
    @Value("${user.role}")
    private String role;

    public static void main(String[] args) {
        SpringApplication.run(ConfigClient.class, args);
    }

    @GetMapping(
      value = "/whoami/{username}",  
      produces = MediaType.TEXT_PLAIN_VALUE)
    public String whoami(@PathVariable("username") String username) {
        return String.format("Hello! 
          You're %s and you'll become a(n) %s...\n", username, role);
    }
}
```

2. bootstrap.properties:

```properties
spring.application.name=config-client
spring.profiles.active=development
spring.cloud.config.uri=http://localhost:8888
spring.cloud.config.username=root
spring.cloud.config.password=s3cr3t
```

3. test:

```bash
$> curl http://localhost:8080/whoami/Mr_Pink
```
4. 结果'

```txt
Hello! You're Mr_Pink and you'll become a(n) Developer...
```

## 加密和解密

**必须**：要将“密码学“中强加密与 `Spring` 加密和解密功能一起使用，您需要在 JVM 中安装 `Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files`。这些可以从 Oracle 官网下载。
{: .notice--info}

由于配置服务器正在支持属性值的加密和解密，您可以使用公共存储库来存储诸如用户名和密码等敏感数据。加密值以字符串 {cipher} 为前缀。如果服务器配置为使用对称密钥或密钥对，则可以通过对路径`/encrypt`的 REST 调用生成加密信息。

当然解密的端点也是可用的。加解密这两个端点都接受包含应用程序名称及其当前`profile`配置文件的占位符的路径：`/*/{name}/{profile}`。这对于控制每​​个客户端的加密特别有用。但是，在它们变得有用之前，您必须配置一个加密密钥，我们将在下一节中进行。

**TIP**: 如果您使用 `curl` 调用 `en-/decryption` API，最好使用 `–data-urlencode` 选项（而不是 `–data/-d`），或者将 `Content-Type` 标头显式设置为 `text/plain`。这确保正确处理加密值中的特殊字符，如`+`。
{: .notice--info}

如果通过客户端获取配置时发现无法自动解密某个值，那么其keys将使用名称本身重命名，并以 `invalid`一词为前缀。这种情况应该避免发生，例如使用加密值作为密码。

**TIP:**: 在设置包含 `YAML` 文件的存储库时，您必须用单引号将加密和前缀值括起来！对于`Properties`，情况并非如此。

## CSRF

默认情况下，`Spring Security` 为发送到我们应用程序的所有请求启用 `CSRF` 保护。 因此，为了能够使用 `/encrypt` 和 `/decrypt` 端点，让我们为它们禁用 CSRF：

```java
@Configuration
public class SecurityConfiguration extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http.csrf()
          .ignoringAntMatchers("/encrypt/**")
          .ignoringAntMatchers("/decrypt/**");

        super.configure(http);
    }
}
```

## key管理

配置服务器默认启用以对称或非对称方式加密属性值。

- **对称加密**，您只需将 `application.properties` 中的属性 `encrypt.key` 设置为您选择的机密。或者，您可以传入环境变量 `ENCRYPT_KEY`。

- **非对称加密**， 您可以将 `encrypt.key` 设置为 PEM编码 的字符串值或 配置使用`keystore`。

因为我们的演示服务器需要一个高度安全的环境，所以我们选择了后一个选项并生成一个新的密钥库，包括一个 `RSA` 密钥对.

1. 使用 Java 密钥工具：

```bash
$> keytool -genkeypair -alias config-server-key \
       -keyalg RSA -keysize 4096 -sigalg SHA512withRSA \
       -dname 'CN=Config Server,OU=Spring Cloud,O=Baeldung' \
       -keypass my-k34-s3cr3t -keystore config-server.jks \
       -storepass my-s70r3-s3cr3t
```

2. 我们将创建的密钥库添加到我们服务器的 bootstrap.properties 并重新运行它：

```properties
encrypt.keyStore.location=classpath:/config-server.jks
encrypt.keyStore.password=my-s70r3-s3cr3t
encrypt.keyStore.alias=config-server-key
encrypt.keyStore.secret=my-k34-s3cr3t
```

3. 我们可以查询加密端点并将响应作为值添加到我们存储库中的配置中：

```bash
$> export PASSWORD=$(curl -X POST --data-urlencode d3v3L \
       http://root:s3cr3t@localhost:8888/encrypt)
$> echo "user.password={cipher}$PASSWORD" >> config-client-development.properties
$> git commit -am 'Added encrypted password'
$> curl -X POST http://root:s3cr3t@localhost:8888/refresh
```

4. 验证。 如果我们的设置工作正常，修改 `ConfigClient` 相关类并重新启动我们的客户端：
  
```java
@SpringBootApplication
@RestController
public class ConfigClient {

    ...
    
    @Value("${user.password}")
    private String password;

    ...
    public String whoami(@PathVariable("username") String username) {
        return String.format("Hello! 
          You're %s and you'll become a(n) %s, " +
          "but only if your password is '%s'!\n", 
          username, role, password);
    }
}
```

5. 结果. 如果我们的配置值被正确解密，对我们的客户端的最终查询为:

```bash
$> curl http://localhost:8080/whoami/Mr_Pink
Hello! You're Mr_Pink and you'll become a(n) Developer, \
  but only if your password is 'd3v3L'!

```

## 使用多个keys

如果您想使用多个密钥进行加密和解密，例如：为每个服务的应用程序使用一个专用密钥，您可以在 `{cipher}` 前缀和 `BASE64` 编码的属性值之间以 `{name:value}` 的形式添加另一个前缀. 

配置服务器理解如`{secret:my-crypto-secret}` 或 `{key:my-key-alias}`的前缀，, 几乎是开箱即用的。非对称加密 需要在 `application.properties` 中配置`keystore`。密钥库以所匹配密钥别名来搜索出来。例如：

```properties
user.password={cipher}{secret:my-499-s3cr3t}AgAMirj1DkQC0WjRv...
user.password={cipher}{key:config-client-key}AgAMirj1DkQC0WjRv...
```

**TIP**: 对于没有密钥库的场景，您必须实现一个 `TextEncryptorLocator` 类型的 `@Bean`，它处理查找并为每个密钥返回一个 `TextEncryptor-Object`。

## 加密配置

如果要禁用服务器端加密并在本地处理属性值的解密，可以将以下内容放入服务器的 `application.properties`：

```properites
spring.cloud.config.server.encrypt.enabled=false
```

## 总结

现在我们能够创建一个配置服务器来从 `Git` 存储库向客户端应用程序提供一组配置文件。您还可以使用此类服务​​器执行其他一些操作。 例如： 

- 以 `YAML` 或 `Properties` 格式而不是 `JSON `提供配置——也解决了占位符问题。 在非 Spring 环境中使用它时，这可能很有用，其中配置不直接映射到 `PropertySource`。
- 提供纯文本配置文件。 依次可选地使用已解析的占位符。例如，这对于提供依赖于环境的日志记录配置很有用。
- 将配置服务器嵌入到应用程序中，它从 `Git` 存储库配置自身，而不是作为独立的应用程序运行，为客户端提供服务。因此，必须设置一些引导程序属性和/或必须删除 `@EnableConfigServer` 注释，这取决于用例。
- 使配置服务器可用于 `Spring Netflix Eureka`服务发现并启用 配置客户端中的自动服务器发现。 如果服务器没有固定位置或在其位置移动，这将变得很重要。
