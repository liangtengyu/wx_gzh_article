springboot自动配置:

理解3个问题:

1.达到什么条件
2.创建什么bean
3.创建的bean配置是什么

> 以EmbeddedWebServerFactoryCustomizerAutoConfiguration为例子

```java
@Configuration(proxyBeanMethods = false)  
@ConditionalOnWebApplication    				//此处满足条件1  达到什么条件
@EnableConfigurationProperties(ServerProperties.class)   // 此处满足条件3 创建bean的配置是什么
public class EmbeddedWebServerFactoryCustomizerAutoConfiguration {

	/**
	 * Nested configuration if Tomcat is being used.
	 */
	@Configuration(proxyBeanMethods = false)	//此处满足条件2 创建什么bean 
	@ConditionalOnClass({ Tomcat.class, UpgradeProtocol.class })//此处满足条件1  达到什么条件
	public static class TomcatWebServerFactoryCustomizerConfiguration {

		@Bean
		public TomcatWebServerFactoryCustomizer tomcatWebServerFactoryCustomizer(Environment environment,
				ServerProperties serverProperties) {
			return new TomcatWebServerFactoryCustomizer(environment, serverProperties);
		}

	}
```

> 简单的例子
> 无条件创建一个bean 只要满足问题2和3
>
> 2.创建什么bean
> 3.创建的bean配置是什么

```java
@Configuration  //创建什么bean 
public class DemoConfiguration {

    @Bean  //创建的bean的配置是什么
    public void object() {
        return new Obejct();
    }

}
```

如此得出结论  

> 1 `@ConditionOnClass`注解的类要满足条件才会创建bean  ,可以解决需要什么条件的问题, 当然也可以不设置条件
> 2 `@Configuration` 注解的类,可以解决创建什么bean的问题

那最后一个问题呢 其实在serverProperties中, 看代码:

```java
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {

	/**
	 * Server HTTP port.
	 */
	private Integer port;

	/**
	 * Network address to which the server should bind.
	 */
	private InetAddress address;
  
  //..........省略
```

>3 ` @ConfigurationProperties` 注解的类,可以解决**创建的 Bean 的属性？**问题

目前:

我们已经比较清晰的理解 Spring Boot 是怎么解决我们上面提出的三个问题，但是这样还是无法实现自动配置。例如说，我们引入的 `spring-boot-starter-web` 等依赖，Spring Boot 是怎么知道要扫码哪些配置类的。