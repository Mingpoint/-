### 自定义spring boot start

1、spring boot 自定义starts

2、如何编写自动配置

```java
@Configuration //指定配置了
@ConditionalOn //排他条件
@ConfigurationProperties //结合xxProperties 绑定相关配置
@EnableConfigurationProperties //启用@ConfigurationProperties 绑定的配置
```

3、自动配置类要能加载，需要将启动的自动配置类，配置在META-INF/srping.factories

```xml
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\org.springframework.boot.autoconfigure.BackgroundPreinitializer
```

4、模式

启动器只用来依赖引导

专门写一个自动配置模块

启动器依赖自动配置；别人只需要引用启动器（start）

xxx-spring-boot-start

