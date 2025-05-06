---
layout: mypost
title: Springboot数据源配置
categories: [源码阅读]
---

### 自动配置

在使用 Springboot 时, 即使没有配置数据源, 也可以连接数据库进行操作, 这是基于 Springboot 的自动配置实现的. 其中涉及到数据源的类为 DataSourceAutoConfiguration. 该类会生成一个数据源并且注入到 Spring 容器中, 然后使用该数据源提供的连接就可以访问数据库了.

### DataSourceAutoConfiguration

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import(DataSourcePoolMetadataProvidersConfiguration.class)
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.EmbeddedDatabaseCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration {}

    @Configuration(proxyBeanMethods = false)
    @Conditional(org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
            DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class,
            DataSourceConfiguration.Generic.class, DataSourceJmxConfiguration.class })
    protected static class PooledDataSourceConfiguration {}

    static class PooledDataSourceCondition extends AnyNestedCondition {...}

    static class PooledDataSourceAvailableCondition extends SpringBootCondition {...}

    static class EmbeddedDatabaseCondition extends SpringBootCondition {...}
}
```

观察可以发现, DataSourceAutoConfiguration 提供了五个内部静态类:

- EmbeddedDatabaseConfiguration
- PooledDataSourceConfiguration
- PooledDataSourceCondition
- PooledDataSourceAvailableCondition
- EmbeddedDatabaseCondition

其中两个 XxxConfiguration 为使用了 `@Configuration` 注解的自动配置类, 其他三个为限制条件类.

#### EmbeddedDatabaseConfiguration

EmbeddedDatabaseConfiguration 直译为"嵌入式数据库配置", 是内嵌数据源的自动配置类. 该类中没有任何方法实现, 主要功能是通过 `@Import` 注解引入 EmbeddedDataSourceConfiguration 类来实现的.

下面是 EmbeddedDataSourceConfiguration 类的具体实现:

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(DataSourceProperties.class)
public class EmbeddedDataSourceConfiguration implements BeanClassLoaderAware {

	private ClassLoader classLoader;

	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.classLoader = classLoader;
	}

	@Bean(destroyMethod = "shutdown")
	public EmbeddedDatabase dataSource(DataSourceProperties properties) {
        // 添加内嵌数据源
		return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseConnection.get(this.classLoader).getType())
			.setName(properties.determineDatabaseName())
			.build();
	}
}
```

该类向容器中添加一个内嵌的数据源, 该数据源支持 H2、DERBY、HSQLDB 三种数据库:

```java
public enum EmbeddedDatabaseConnection {
    NONE((String)null),
    H2("jdbc:h2:mem:%s;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE"),
    DERBY("jdbc:derby:memory:%s;create=true"),
    HSQLDB("org.hsqldb.jdbcDriver", "jdbc:hsqldb:mem:%s");
    ...
}
```

除此之外, EmbeddedDatabaseConfiguration 添加内嵌数据源是有条件限制的, 该类上面还有一个 `@Conditional` 注解, 这个注解的作用可以理解为"如果满足某个条件, 就启用该配置, 否则忽略". 而"某个条件"指的就是上面提到的三个限制条件类之一的 EmbeddedDatabaseCondition.

EmbeddedDatabaseCondition 类的注释原文与译文如下, 简单来说就是有就用你的, 没有再用我的:

> Condition to detect when an embedded DataSource type can be used. If a pooled DataSource is available, it will always be preferred to an EmbeddedDatabase
>
> 用于检测是否可以使用嵌入式的数据源类型的条件判断. **如果有连接池类型的数据源可用, 则总是优先使用连接池数据源**, 而不是使用嵌入式数据库.

部分代码如下:

```java
    public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
        if (hasDataSourceUrlProperty(context)) {
            // 配置了 spring.datasource.url 不使用嵌入式数据源
        }
        if (anyMatches(context, metadata, this.pooledCondition)) {
            // 连接池可用, 说明有 HikariCP 这样的依赖, 不使用嵌入式数据源
        }

        // 上面两个条件都不满足, 再查找嵌入式数据库类型
        EmbeddedDatabaseType type = EmbeddedDatabaseConnection.get(context.getClassLoader()).getType();
        if (type == null) {
            // 没有嵌入式数据库
        }
        // 有, 并且返回
        return ...
    }
```

#### PooledDataSourceConfiguration

PooledDataSourceConfiguration 直译为"池化数据源配置", 该类也没有任何方法实现, 注解结构和 EmbeddedDatabaseConfiguration 差不多.

首先通过 `@Conditional` 注解配置条件, 条件为上面提到的三个限制类的 PooledDataSourceCondition.

PooledDataSourceCondition 的注释和代码如下:

> AnyNestedCondition that checks that either spring.datasource.type is set or DataSourceAutoConfiguration. PooledDataSourceAvailableCondition applies.
>
> 用户明确指定了 spring.datasource.type 或者系统找得到支持的连接池实现(如 JDBC、Hikari 等). 满足上述条件之一就启用连接池数据源配置.

```java
	static class PooledDataSourceCondition extends AnyNestedCondition {

		PooledDataSourceCondition() {
			super(ConfigurationPhase.PARSE_CONFIGURATION);
		}

		@ConditionalOnProperty(prefix = "spring.datasource", name = "type")
		static class ExplicitType {

		}

		@Conditional(PooledDataSourceAvailableCondition.class)
		static class PooledDataSourceAvailable {

		}
	}
```

其次主要功能也通过 `@Import` 注解引入, 引入的五个都是 DataSourceConfiguration 的内部类(DataSourceJmxConfiguration 除外, 它用于配置数据源的 JMX 监控功能), 功能都是向容器中添加指定数据源. 以 Hikari 为例, 源码如下:

```java
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}
	}
```

`@ConditionalOnClass(HikariDataSource.class` 表示必须在类路径中存在 HikariDataSource 类时 Hikari 才会被初始化. 而 HikariDataSource 类由 `spring-boot-starter-jdbc` 默认引入, 所以只要 `pom.xml` 中引入了该 starter, 那么 OnClass 这个条件就满足了.

`@ConditionalOnMissingBean(DataSource.class)` 表示容器中没有用户自定的数据源时, 该配置类才会初始化.

> `DataSource.class` 是 `javax.sql` 包下的一个接口, 它提供数据库连接对象的工厂, 里面有 get/setConnection, get/setLogWriter 等方法. 可以取代传统的 DriverManager 连接方式. 比如上面的 HikariDataSource 就实现了这个接口.

`@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource", matchIfMissing = true)` 表示 Springboot 配置文件中, 明确指定使用 Hikari 数据源, 或者不配置 spring.datasource.type 时, Hikari 才会被初始化.

Hikari 类通过 `@Bean` 的方式向容器注入 HikariDataSource 组件, 通过 `createDataSource()` 调用 `initializeDataSourceBuilder()` 方法创建 DataSourceBuilder 对象, 配置用户名、密码等. 创建后的 Builder 还有一个 `.type(type).build()` 步骤, 这就是上面提到不用配置 type 的原因. 最后设置连接池名称, 将数据源注入到容器中.

```java
protected static <T> T createDataSource(DataSourceProperties properties, Class<? extends DataSource> type) {
	return (T) properties.initializeDataSourceBuilder().type(type).build();
}
```

至此结束.
