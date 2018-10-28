# SpringBoot 大杂烩


Spring Boot简化了基于Spring的应用开发，你只需要"run"就能创建一个独立的，产品级别的Spring应用。 我们为Spring平台及第三方库提供开箱即用的设置，这样你就可以有条不紊地开始。多数Spring Boot应用只需要很少的Spring配置。

你可以使用Spring Boot创建Java应用，并使用java -jar启动它或采用传统的war部署方式。我们也提供了一个运行"spring脚本"的命令行工具。

我们主要的目标是：

* 为所有Spring开发提供一个从根本上更快，且随处可得的入门体验。
* **开箱即用**，但通过不采用默认设置可以快速摆脱这种方式。
* 提供一系列大型项目常用的非功能性特征，比如：内嵌服务器，安全，指标，健康检测，外部化配置。
* 绝对没有代码生成，也不需要XML配置。

## SpringBoot开箱即用原理

`@EnableAutoConfiguration` 注解允许通过扫描 classpath 中的组件并注册匹配各种条件的 bean 来自动配置 Spring ApplicationContext。

 `@EnableAutoConfiguration`注解，**并暗地里定义了一个基础的包路径，Spring Boot会在该包路径来搜索类**

`@SpringBootApplication`注解继承了`@EnableAutoConfiguration`，所以一般情况下可认为，Springboot默认扫描规则是：自动扫描启动器类的同包或者其子包的下的注解。

SpringBoot 在 spring-boot-autoconfigure- {version} .jar 中提供了各种 AutoConfiguration 类，它们负责注册各种组件。SpringBoot启动后会处理处理AutoConfiguration类，但是并不是每个都需要执行，这些AutoConfiguration类都有`@ConditionalOnClass`标签，即判断特定类存在的时候才加载当前配置类，比如`EhCacheAutoConfiguration`类只有当`EhCacheCacheManager`存在的时候才会被加载，进而创建`EhCacheCacheManager`的`bean`等。

参见：[spring boot实战(第十三篇)自动配置原理分析 - CSDN博客](http://blog.csdn.net/liaokailin/article/details/49559951)

`EhCacheAutoConfiguration`的代码如下：

```java
package org.springframework.boot.autoconfigure.cache;

import net.sf.ehcache.Cache;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.cache.CacheCondition;
import org.springframework.boot.autoconfigure.cache.CacheProperties;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ResourceCondition;
import org.springframework.cache.CacheManager;
import org.springframework.cache.ehcache.EhCacheCacheManager;
import org.springframework.cache.ehcache.EhCacheManagerUtils;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Conditional;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;

@Configuration
@ConditionalOnClass({Cache.class, EhCacheCacheManager.class}) //这里其实就是当pom中引入了spring-boot-starter-cache、ehcache时，才加载本配置
@ConditionalOnMissingBean({CacheManager.class}) //这里的意思是如果开发者自己定义了CacheManager，那么当前的配置类也就不加载了
@Conditional({CacheCondition.class, EhCacheCacheConfiguration.ConfigAvailableCondition.class}) //这里主要是去获取配置参数
class EhCacheCacheConfiguration {
    @Autowired
    private CacheProperties cacheProperties;

    EhCacheCacheConfiguration() {
    }

    @Bean
    public EhCacheCacheManager cacheManager(net.sf.ehcache.CacheManager ehCacheCacheManager) {
        return new EhCacheCacheManager(ehCacheCacheManager);
    }

    @Bean
    @ConditionalOnMissingBean
    public net.sf.ehcache.CacheManager ehCacheCacheManager() {
        Resource location = this.cacheProperties.resolveConfigLocation(this.cacheProperties.getEhcache().getConfig());
        return location != null?EhCacheManagerUtils.buildCacheManager(location):EhCacheManagerUtils.buildCacheManager();
    }

    static class ConfigAvailableCondition extends ResourceCondition {
        ConfigAvailableCondition() {
            super("EhCache", "spring.cache.ehcache", "config", new String[]{"classpath:/ehcache.xml"});
        }
    }
}
```

SpringBoot的`AutoConfiguration`参见：

[Spring Boot自动配置 - JavaChen Blog](http://blog.javachen.com/2016/02/19/spring-boot-auto-configuration.html)

## SpringBoot的Maven配置

基本的maven依赖引入：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter</artifactId>
	</dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
  
  	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
</dependencies>
```

其中 spring-boot-starter 包含 spring-core 以及 autoconfigure，spring-boot-starter-web 中包含了 tomcat.

---

## SpringBoot 属性配置文件

属性默认配置文件名：application.properties

```java
application.properties:
com.didispace.blog.name=程序猿DD
com.didispace.blog.title=Spring Boot教程

//文件中
@Component
public class BlogProperties {

    @Value("${com.didispace.blog.name}")
    private String name;
    @Value("${com.didispace.blog.title}")
    private String title;

    // 省略getter和setter

}

//单元测试
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(Application.class)
public class ApplicationTests {

	@Autowired
	private BlogProperties blogProperties;


	@Test
	public void getHello() throws Exception {
		Assert.assertEquals(blogProperties.getName(), "程序猿DD");
		Assert.assertEquals(blogProperties.getTitle(), "Spring Boot教程");
	}

}
```

不同配置参数之间也能相互引用，如

```yaml
com.proj.name=yourname
com.proj.title=yourtitle
com.proj.desc=${com.proj.name}...
```

配置参数中支持使用随机生成

```yaml
# 随机字符串
com.didispace.blog.value=${random.value}
# 随机int
com.didispace.blog.number=${random.int}
# 随机long
com.didispace.blog.bignumber=${random.long}
# 10以内的随机数
com.didispace.blog.test1=${random.int(10)}
# 10-20的随机数
com.didispace.blog.test2=${random.int[10,20]}
```

可以采用命令行参数方式，覆盖初始配置参数，如：

```sh
java -jar xxx.jar --server.port=888
```

覆盖配置文件中的 server.port 参数

如果需要禁止命令行参数覆盖，则 `SpringApplication.setAddCommandLineProperties(false)` 

多环境配置：

在application.properties 文件中多为通用配置，如果针对不同的环境（开发环境，生产环境等）需要自定义某些配置，可以采用多个配置文件，如：

* application-dev.properties
* application-test.properties
* application-prod.properties

那么如何引入上述不同环境配置文件中的 一个呢？在application.properties中加入：

```yaml
spring.profiles.active=***
eg. spring.profiles.active=test
```

当然也可以在命令行中指定：java -jar xxx.jar --spring.profiles.active=test

---

## 静态资源访问

在web应用开发过程中，如果前端代码嵌入在同一个应用中（未采用前后端分离），会大量用到js,css,html等资源

Spring Boot默认提供静态资源目录位置需置于classpath下，目录名需符合如下规则：

- /static
- /public
- /resources
- /META-INF/resources

举例：我们可以在`src/main/resources/`目录下创建`static`，在该位置放置一个图片文件。启动程序后，尝试访问`http://localhost:8080/D.jpg`。如能显示图片，配置成功。

Spring Boot提供了默认配置的模板引擎主要有以下几种：

- Thymeleaf
- FreeMarker
- Velocity
- Groovy
- Mustache

当你使用上述模板引擎中的任何一个，它们默认的模板配置路径为：`src/main/resources/templates`

在application.properties中配置模板属性：

```yaml
# Enable template caching.
spring.thymeleaf.cache=true 
# Check that the templates location exists.
spring.thymeleaf.check-template-location=true 
# Content-Type value.
spring.thymeleaf.content-type=text/html 
# Enable MVC Thymeleaf view resolution.
spring.thymeleaf.enabled=true 
# Template encoding.
spring.thymeleaf.encoding=UTF-8 
# Comma-separated list of view names that should be excluded from resolution.
spring.thymeleaf.excluded-view-names= 
# Template mode to be applied to templates. See also StandardTemplateModeHandlers.
spring.thymeleaf.mode=HTML5 
# Prefix that gets prepended to view names when building a URL.
spring.thymeleaf.prefix=classpath:/templates/ 
# Suffix that gets appended to view names when building a URL.
spring.thymeleaf.suffix=.html  
# Order of the template resolver in the chain. 
spring.thymeleaf.template-resolver-order= 
# Comma-separated list of view names that can be resolved.
spring.thymeleaf.view-names= 
```

在Spring Boot中使用Thymeleaf，只需要引入下面依赖，并在默认的模板路径src/main/resources/templates下编写模板文件即可完成。

参照：

http://blog.didispace.com/springbootweb/

---

## 关于在IDEA 中运行Java应用的 classpath

首先在打开的项目窗口打开File->Project Structure...，得到如下图所示的项目结构：

​    ![img](http://img.blog.csdn.net/20170222205237556?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU2t5ZUJlRnJlZW1hbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​    在最上面一栏可以看到这里是Source标签中的详细信息，在右边可以看到项目里面目录的类型，有Source Folders、Resource Folders等等，这里指的是Source Folders表示的都是代码源文件目录，生成的class文件会输出到 target->classess 文件夹中，但是里面的源文件不会复制到target->classes文件夹中，Test Source Folders表示的都是测试代码源文件目录，生成的class文件同样会输出到 target->classess 文件夹中，并且里面的源文件不会复制到target->classes文件夹中，**而Recource Folders 表示的都是资源文件目录，这些目录里面的文件会在代码编译运行被直接复制到 target->classess 文件夹中**。可以这么讲，target->classes 即为classpath，任何我们需要在classpath前缀中获取的资源都必须在target->classes文件夹中找到，否则将出现 java.io.FileNotFoundException 的错误信息。



## Swagger2构建强大的RESTful API文档

Maven依赖

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.2.2</version>
</dependency>
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.2.2</version>
</dependency>
```

创建Swagger2配置类

```java
@Configuration
@EnableSwagger2
public class Swagger2 {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.didispace.web"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Spring Boot中使用Swagger2构建RESTful APIs")
                .description("更多Spring Boot相关文章请关注：http://blog.didispace.com/")
                .termsOfServiceUrl("http://blog.didispace.com/")
                .contact("程序猿DD")
                .version("1.0")
                .build();
    }

}
```

添加文档内容：

* @ApiOperation注解来给API增加说明通
* @ApiImplicitParams、@ApiImplicitParam注解来给参数增加说明。

```java

@RestController
@RequestMapping(value="/users")     // 通过这里配置使下面的映射都在/users下，可去除
public class UserController {

    static Map<Long, User> users = Collections.synchronizedMap(new HashMap<Long, User>());

    @ApiOperation(value="获取用户列表", notes="")
    @RequestMapping(value={""}, method=RequestMethod.GET)
    public List<User> getUserList() {
        List<User> r = new ArrayList<User>(users.values());
        return r;
    }

    @ApiOperation(value="创建用户", notes="根据User对象创建用户")
    @ApiImplicitParam(name = "user", value = "用户详细实体user", required = true, dataType = "User")
    @RequestMapping(value="", method=RequestMethod.POST)
    public String postUser(@RequestBody User user) {
        users.put(user.getId(), user);
        return "success";
    }

    @ApiOperation(value="获取用户详细信息", notes="根据url的id来获取用户详细信息")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long")
    @RequestMapping(value="/{id}", method=RequestMethod.GET)
    public User getUser(@PathVariable Long id) {
        return users.get(id);
    }

    @ApiOperation(value="更新用户详细信息", notes="根据url的id来指定更新对象，并根据传过来的user信息来更新用户详细信息")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long"),
            @ApiImplicitParam(name = "user", value = "用户详细实体user", required = true, dataType = "User")
    })
    @RequestMapping(value="/{id}", method=RequestMethod.PUT)
    public String putUser(@PathVariable Long id, @RequestBody User user) {
        User u = users.get(id);
        u.setName(user.getName());
        u.setAge(user.getAge());
        users.put(id, u);
        return "success";
    }

    @ApiOperation(value="删除用户", notes="根据url的id来指定删除对象")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true, dataType = "Long")
    @RequestMapping(value="/{id}", method=RequestMethod.DELETE)
    public String deleteUser(@PathVariable Long id) {
        users.remove(id);
        return "success";
    }

}

```

完成上述代码添加上，启动Spring Boot程序，访问：`http://localhost:8080/swagger-ui.html`

---

## 统一异常处理

当访问某个url，处理过程中发生错误，需要固定跳转到特定的错误也面试后，就需要统一异常处理

```java
@RequestMapping("/hello")
public String hello() throws Exception {
    throw new Exception("发生错误");
}
```

这个时候，我们需要实现统一异常捕获与处理的逻辑：

```java
@ControllerAdvice
class GlobalExceptionHandler {

    public static final String DEFAULT_ERROR_VIEW = "error";

    @ExceptionHandler(value = Exception.class)
    public ModelAndView defaultErrorHandler(HttpServletRequest req, Exception e) throws Exception {
        ModelAndView mav = new ModelAndView();
        mav.addObject("exception", e);
        mav.addObject("url", req.getRequestURL());
        mav.setViewName(DEFAULT_ERROR_VIEW);
        return mav;
    }

}
```

通过使用`@ControllerAdvice`定义统一的异常处理类，而不是在每个Controller中逐个定义。`@ExceptionHandler`用来定义函数针对的异常类型，将需要展示的信息输出并在`error.html`中渲染

（error.html 可以是Thymeleaf模板引擎指定的/WEB-INF/templates下的模板文件）

当然，也可以加上`@ResponseBody` 返回json格式数据

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(value = MyException.class)
    @ResponseBody
    public ErrorInfo<String> jsonErrorHandler(HttpServletRequest req, MyException e) throws Exception {
        ErrorInfo<String> r = new ErrorInfo<>();
        r.setMessage(e.getMessage());
        r.setCode(ErrorInfo.ERROR);
        r.setData("Some Data");
        r.setUrl(req.getRequestURL().toString());
        return r;
    }
}
class ErrorInfo<T> {
    public static final Integer OK = 0;
    public static final Integer ERROR = 100;

    private Integer code;
    private String message;
    private String url;
    private T data;

    // 省略getter和setter
}
```

---

## Spring-data-jpa 数据层

在实际开发过程中，对数据库的操作无非就“增删改查”。就最为普遍的单表操作而言，除了表和字段不同外，语句都是类似的，开发人员需要写大量类似而枯燥的语句来完成业务逻辑。

由于模板Dao的实现，使得这些具体实体的Dao层已经变的非常“薄”，有一些具体实体的Dao实现可能完全就是对模板Dao的简单代理，并且往往这样的实现类可能会出现在很多实体上。Spring-data-jpa的出现正可以让这样一个已经很“薄”的数据访问层变成只是一层接口的编写方式

Spring-data-jpa的作用就在于：

* 复用一些基础的数据访问逻辑，主需要定义继承自`JpaRepository<T,ID>`的接口就能完成数据访问，注意是::泛型::，前一个代表数据模型的类型，后一个为ID的类型
* 按照约定，通过函数名自动解析出数据访问操作

mave配置：

```xml
<dependency
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

application.properties配置：

```yaml
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=root
spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.jpa.properties.hibernate.hbm2ddl.auto=create-drop
```

由于配置了`hibernate.hbm2ddl.auto`，在应用启动的时候框架会**自动去数据库中创建对应的表**

```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
        entityManagerFactoryRef="entityManagerFactoryPrimary",
        transactionManagerRef="transactionManagerPrimary",
        basePackages= { "com.didispace.domain.p" }) //设置Repository所在位置
public class PrimaryConfig {

    @Autowired @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;

    @Primary
    @Bean(name = "entityManagerPrimary")
    public EntityManager entityManager(EntityManagerFactoryBuilder builder) {
        return entityManagerFactoryPrimary(builder).getObject().createEntityManager();
    }

    @Primary
    @Bean(name = "entityManagerFactoryPrimary")
    public LocalContainerEntityManagerFactoryBean entityManagerFactoryPrimary (EntityManagerFactoryBuilder builder) {
        return builder
                .dataSource(primaryDataSource)
                .properties(getVendorProperties(primaryDataSource))
                .packages("com.didispace.domain.p") //设置实体类所在位置
                .persistenceUnit("primaryPersistenceUnit")
                .build();
    }

    @Autowired
    private JpaProperties jpaProperties;

    private Map<String, String> getVendorProperties(DataSource dataSource) {
        return jpaProperties.getHibernateProperties(dataSource);
    }

    @Primary
    @Bean(name = "transactionManagerPrimary")
    public PlatformTransactionManager transactionManagerPrimary(EntityManagerFactoryBuilder builder) {
        return new JpaTransactionManager(entityManagerFactoryPrimary(builder).getObject());
    }

}
```

数据模型：

```java
@Entity
public class User {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Integer age;

    // 省略构造函数

    // 省略getter和setter

}
```

数据访问接口：

```java
public interface UserRepository extends JpaRepository<User, Long> {

    User findByName(String name);

    User findByNameAndAge(String name, Integer age);

    @Query("from User u where u.name=:name")
    User findUser(@Param("name") String name);

}
```

在上例中，我们可以看到下面两个函数：

- `User findByName(String name)`
- `User findByNameAndAge(String name, Integer age)`

它们分别实现了按 name 查询 User 实体和按 name 和 age 查询 User 实体，可以看到我们这里没有任何类 SQL语句就完成了两个条件查询方法。这就是 Spring-data-jpa 的一大特性：**通过解析方法名创建查询**

除了通过解析方法名来创建查询外，它也提供通过使用 @Query 注解来创建查询，您只需要编写 JPQL 语句，并通过类似 “:name” 来映射 @Param 指定的参数

### 多数据库配置：

```java

@Configuration
public class DataSourceConfig {

    @Bean(name = "primaryDataSource")
    @Qualifier("primaryDataSource")
    @ConfigurationProperties(prefix="spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "secondaryDataSource")
    @Qualifier("secondaryDataSource")
    @Primary
    @ConfigurationProperties(prefix="spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }

}
```

```yaml
spring.datasource.primary.url=jdbc:mysql://localhost:3306/test1
spring.datasource.primary.username=root
spring.datasource.primary.password=root
spring.datasource.primary.driver-class-name=com.mysql.jdbc.Driver

spring.datasource.secondary.url=jdbc:mysql://localhost:3306/test2
spring.datasource.secondary.username=root
spring.datasource.secondary.password=root
spring.datasource.secondary.driver-class-name=com.mysql.jdbc.Driver
```

后续：
可结合JdbcTemplate，也可结合SpringDataJpa

参照：

[Spring Boot多数据源配置与使用](https://blog.csdn.net/Winter_chen001/article/details/78508376)

[Spring Boot 整合 Mybatis 实现 Druid 多数据源详解](https://www.bysocket.com/?p=1712)

---

## SpringBoot 中使用 Redis

maven配置：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-redis</artifactId>
</dependency>
```

application.properties 配置：

```yaml
# REDIS (RedisProperties)
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=localhost
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
# 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
# 连接超时时间（毫秒）
spring.redis.timeout=0
```

SpringBoot中提供有默认的redisTemplate操作，比如针对string类型数据的：

```java
	@Autowired
	private StringRedisTemplate stringRedisTemplate;

	@Test
	public void test() throws Exception {

		// 保存字符串
		stringRedisTemplate.opsForValue().set("aaa", "111");
		Assert.assertEquals("111", stringRedisTemplate.opsForValue().get("aaa"));

    }
```

但是，SpringBoot并不直接支持针对对象的模板操作，需要自己创建，并且自定义序列化对象

序列化对象如下：

```java
public class RedisObjectSerializer implements RedisSerializer<Object> {

  private Converter<Object, byte[]> serializer = new SerializingConverter();
  private Converter<byte[], Object> deserializer = new DeserializingConverter();

  static final byte[] EMPTY_ARRAY = new byte[0];

  public Object deserialize(byte[] bytes) {
    if (isEmpty(bytes)) {
      return null;
    }

    try {
      return deserializer.convert(bytes);
    } catch (Exception ex) {
      throw new SerializationException("Cannot deserialize", ex);
    }
  }

  public byte[] serialize(Object object) {
    if (object == null) {
      return EMPTY_ARRAY;
    }

    try {
      return serializer.convert(object);
    } catch (Exception ex) {
      return EMPTY_ARRAY;
    }
  }

  private boolean isEmpty(byte[] data) {
    return (data == null || data.length == 0);
  }
}
```

自定义template：

```java
@Configuration
public class RedisConfig {

    @Bean
    JedisConnectionFactory jedisConnectionFactory() {
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, User> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, User> template = new RedisTemplate<String, User>();
        template.setConnectionFactory(jedisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new RedisObjectSerializer());
        return template;
    }
}
```

具体使用：

```java
@Autowired
	private RedisTemplate<String, User> redisTemplate;

	@Test
	public void test() throws Exception {

		// 保存对象
		User user = new User("超人", 20);
		redisTemplate.opsForValue().set(user.getUsername(), user);

		user = new User("蝙蝠侠", 30);
		redisTemplate.opsForValue().set(user.getUsername(), user);

		user = new User("蜘蛛侠", 40);
		redisTemplate.opsForValue().set(user.getUsername(), user);

		Assert.assertEquals(20, redisTemplate.opsForValue().get("超人").getAge().longValue());
		Assert.assertEquals(30, redisTemplate.opsForValue().get("蝙蝠侠").getAge().longValue());
		Assert.assertEquals(40, redisTemplate.opsForValue().get("蜘蛛侠").getAge().longValue());

	}
```


---

## SpringBoot 中使用MongoDB

maven 配置：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

application.properties 中配置：

```yaml
spring.data.mongodb.uri=mongodb://name:pass@localhost:27017/test
```


数据访问层：

```java

public interface UserRepository extends MongoRepository<User, Long> {

    User findByUsername(String username);

}
```


使用示例：

```java
	@Autowired
	private UserRepository userRepository;

	@Before
	public void setUp() {
		userRepository.deleteAll();
	}

	@Test
	public void test() throws Exception {

		// 创建三个User，并验证User总数
		userRepository.save(new User(1L, "didi", 30));
		userRepository.save(new User(2L, "mama", 40));
		userRepository.save(new User(3L, "kaka", 50));
		Assert.assertEquals(3, userRepository.findAll().size());

		// 删除一个User，再验证User总数
		User u = userRepository.findOne(1L);
		userRepository.delete(u);
		Assert.assertEquals(2, userRepository.findAll().size());

		// 删除一个User，再验证User总数
		u = userRepository.findByUsername("mama");
		userRepository.delete(u);
		Assert.assertEquals(1, userRepository.findAll().size());

	}
```


---

### SpringBoot 整合 MyBatis

与其他的对象关系映射框架不同，MyBatis并没有将Java对象与数据库表关联起来，而是**将Java方法与SQL语句关联**。MyBatis允许用户充分利用数据库的各种功能，例如存储过程、视图、各种复杂的查询以及某数据库的专有特性。如果要对遗留数据库、不规范的数据库进行操作，或者要完全控制SQL的执行，MyBatis是一个不错的选择。

MyBatis支持**声明式数据缓存（declarative data caching）**。当一条SQL语句被标记为“可缓存”后，首次执行它时从数据库获取的所有数据会被存储在一段高速缓存中，今后执行这条语句时就会从高速缓存中读取结果，而不是再次命中数据库。MyBatis提供了基于 Java HashMap 的默认缓存实现，以及用于与OSCache、Ehcache、Hazelcast和Memcached连接的默认连接器。MyBatis还提供API供其他缓存实现使用。

pom 依赖：

* 引入连接mysql的必要依赖mysql-connector-java
* 引入整合MyBatis的核心依赖mybatis-spring-boot-starter
* 不需要引入spring-boot-starter-jdbc依赖，是由于mybatis-spring-boot-starter中已经包含了此依赖

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>1.3.2.RELEASE</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>

<dependencies>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter</artifactId>
	</dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>

	<dependency>
		<groupId>org.mybatis.spring.boot</groupId>
		<artifactId>mybatis-spring-boot-starter</artifactId>
		<version>1.1.1</version>
	</dependency>

	<dependency>
		<groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
		<version>5.1.21</version>
	</dependency>

</dependencies>
```

在`application.properties`中配置mysql的连接配置:
```yaml
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

#### 使用MyBatis

* 在Mysql中创建User表，包含id(BIGINT)、name(INT)、age(VARCHAR)字段。同时，创建映射对象User

```java
public class User {

    private Long id;
    private String name;
    private Integer age;

    // 省略getter和setter

}
```

* 创建User映射的操作UserMapper，为了后续单元测试验证，实现插入和查询操作

```java
@Mapper
public interface UserMapper {

    @Select("SELECT * FROM USER WHERE NAME = #{name}")
    User findByName(@Param("name") String name);

    @Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
    int insert(@Param("name") String name, @Param("age") Integer age);

}
```

* 单元测试：

```java
	@Test
	@Rollback
	public void findByName() throws Exception {
		userMapper.insert("AAA", 20);
		User u = userMapper.findByName("AAA");
		Assert.assertEquals(20, u.getAge().intValue());
	}
```

#### 使用MyBatis注解配置详解

* 使用`@Param`
```java
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insert(@Param("name") String name, @Param("age") Integer age);
```
其中，@Param中定义的name对应了SQL中的{name}，age对应了SQL中的{age}。

* 使用Map
```java
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name,jdbcType=VARCHAR}, #{age,jdbcType=INTEGER})")
int insertByMap(Map<String, Object> map);

Map<String, Object> map = new HashMap<>();
map.put("name", "CCC");
map.put("age", 40);
userMapper.insertByMap(map);
```

* 使用对象

```java
@Insert("INSERT INTO USER(NAME, AGE) VALUES(#{name}, #{age})")
int insertByUser(User user);
```

* 增删改查

```java
public interface UserMapper {

    @Select("SELECT * FROM user WHERE name = #{name}")
    User findByName(@Param("name") String name);

    @Insert("INSERT INTO user(name, age) VALUES(#{name}, #{age})")
    int insert(@Param("name") String name, @Param("age") Integer age);

    @Update("UPDATE user SET age=#{age} WHERE name=#{name}")
    void update(User user);

    @Delete("DELETE FROM user WHERE id =#{id}")
    void delete(Long id);
}
```


* 返回结果的绑定

```java
@Results({
    @Result(property = "name", column = "name"),
    @Result(property = "age", column = "age")
})
@Select("SELECT name, age FROM user")
List<User> findAll();
```

`@Results` 代表的是返回多个对象，即集合。`@Result`中的property属性对应User对象中的成员名，column对应SELECT出的字段名。这里的返回对象类型不一定需要时数据模型类，也可以自定义的临时类，只要在`@Result`中指定好属性与列的对应关系就可以。

---

## SpringBoot的事务管理机制

在Spring Boot中，当我们使用了spring-boot-starter-jdbc或spring-boot-starter-data-jpa依赖的时候，框架会::自动默认::分别注入DataSourceTransactionManager或JpaTransactionManager。所以我们不需要任何额外配置就可以用`@Transactional`注解进行事务的使用。

事务的作用就是为了保证用户的每一个操作都是可靠的，事务中的每一步操作都必须成功执行，只要有发生异常就回退到事务开始未进行操作的状态。

打个比方，测试函数`test()`中需要插入多条数据，但是插入的name字段的长度有限制，当超出长度限制时，系统会报错

```java
// 创建10条记录
userRepository.save(new User("AAA", 10));
userRepository.save(new User("BBB", 20));
userRepository.save(new User("CCC", 30));
userRepository.save(new User("DDD", 40));
userRepository.save(new User("EEE", 50));
userRepository.save(new User("FFF", 60));
userRepository.save(new User("GGG", 70));
userRepository.save(new User("HHHHHHHHHH", 80));
userRepository.save(new User("III", 90));
userRepository.save(new User("JJJ", 100));
```

对于不加事务管理的操作，上述操作中，当执行到`userRepository.save(new User("HHHHHHHHHH", 80));`系统报错返回，但是前面的数据都已经完成插入操作，存在数据库当中

当我们为`test()`函数加上`@Transactional`标签时，定义当前函数需满足事务管理机制的要求，如：

```java
@Test
@Transactional
public void test() throws Exception {

    // 省略测试内容

}
```

这时候，当出现长度超限，系统报错的情况，前面AAA~GGG虽然已经插入到数据库中，也需要进行回滚。

这里介绍另外一个常用标签`@Rollback`，常用于测试，也就是在函数上加上此标签后，当测试单元执行结束后，将回到原始的数据状态。

而对于多数据源配置情况下，往往存在多个事务管理器，如果需要指定事务管理器的，可以采用如下操作：

```java
@Transactional(value="transactionManagerPrimary")
```

#### 事务隔离级别

隔离级别是指若干个并发的事务之间的隔离程度，与我们开发时候主要相关的场景包括：**脏读取、重复读、幻读**。

`org.springframework.transaction.annotation.Isolation`枚举类中定义了五个表示隔离级别的值：

```java
public enum Isolation {
    DEFAULT(-1),
    READ_UNCOMMITTED(1),
    READ_COMMITTED(2),
    REPEATABLE_READ(4),
    SERIALIZABLE(8);
}
```
* DEFAULT：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是：READ_COMMITTED。
* READ_UNCOMMITTED：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读和不可重复读，因此很少使用该隔离级别。
* READ_COMMITTED：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
* REPEATABLE_READ：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。即使在多次查询之间有新增的数据满足该查询，这些新增的记录也会被忽略。该级别可以防止脏读和不可重复读。
* SERIALIZABLE：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

指定方法：通过使用isolation属性设置，例如：

```java
@Transactional(isolation = Isolation.DEFAULT)
```

#### 事务的传播行为

所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。

`org.springframework.transaction.annotation.Propagation`枚举类中定义了6个表示传播行为的枚举值：

```java
public enum Propagation {
    REQUIRED(0),
    SUPPORTS(1),
    MANDATORY(2),
    REQUIRES_NEW(3),
    NOT_SUPPORTED(4),
    NEVER(5),
    NESTED(6);
}
```

* REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
* SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
* MANDATORY：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
* REQUIRES_NEW：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
* NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
* NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。
* NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于REQUIRED。

指定方法：通过使用propagation属性设置，例如：

```java
@Transactional(propagation = Propagation.REQUIRED)
```

___

## 缓存配置

pom依赖如下：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

在SpringBoot主类中通过`EnableCaching`开启缓存功能

```java
@SpringBootApplication
@EnableCaching
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}

}
```

在数据访问接口中，增加缓存配置注解：

```java
@CacheConfig(cacheNames = "users")
public interface UserRepository extends JpaRepository<User, Long> {

    @Cacheable
    User findByName(String name);

}
```

### Cache注解详解

* `@CacheConfig`：主要用于配置该类中会用到的一些共用的缓存配置。在这里@CacheConfig(cacheNames = "users")：配置了该数据访问对象中返回的内容将存储于名为users的缓存对象中，我们也可以不使用该注解，直接通过@Cacheable自己配置缓存集的名字来定义。

* `@Cacheable`：配置了findByName函数的返回值将被加入缓存。同时在查询时，会先从缓存中获取，若不存在才再发起对数据库的访问。该注解主要有下面几个参数：

	* value、cacheNames：两个等同的参数（cacheNames为Spring 4新增，作为value的别名），用于指定缓存存储的集合名。由于Spring 4中新增了@CacheConfig，因此在Spring 3中原本必须有的value属性，也成为非必需项了
	* key：缓存对象存储在Map集合中的key值，非必需，缺省按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：@Cacheable(key = “井号p0")：使用函数第一个参数作为缓存的key值，更多关于SpEL表达式的详细内容可参考官方文档
	* condition：缓存对象的条件，非必需，也需使用SpEL表达式，只有满足表达式条件的内容才会被缓存，比如：@Cacheable(key = “井号p0", condition = “井号p0.length() < 3")，表示只有当第一个参数的长度小于3的时候才会被缓存，若做此配置上面的AAA用户就不会被缓存，读者可自行实验尝试。
	* unless：另外一个缓存条件参数，非必需，需使用SpEL表达式。它不同于condition参数的地方在于它的判断时机，该条件是在函数被调用之后才做判断的，所以它可以通过对result进行判断。
	* keyGenerator：用于指定key生成器，非必需。若需要指定一个自定义的key生成器，我们需要去实现org.springframework.cache.interceptor.KeyGenerator接口，并使用该参数来指定。需要注意的是：该参数与key是互斥的
	* cacheManager：用于指定使用哪个缓存管理器，非必需。只有当有多个时才需要使用
	* cacheResolver：用于指定使用那个缓存解析器，非必需。需通过org.springframework.cache.interceptor.CacheResolver接口来实现自己的缓存解析器，并用该参数指定。

除了这里用到的两个注解之外，还有下面几个核心注解：

* @CachePut：配置于函数上，能够根据参数定义条件来进行缓存，它与@Cacheable不同的是，它每次都会真是调用函数，所以主要用于数据新增和修改操作上。它的参数与@Cacheable类似，具体功能可参考上面对@Cacheable参数的解析
* @CacheEvict：配置于函数上，通常用在删除方法上，用来从缓存中移除相应数据。除了同@Cacheable一样的参数之外，它还有下面两个参数：
	* allEntries：非必需，默认为false。当为true时，会移除所有数据
	* beforeInvocation：非必需，默认为false，会在调用方法之后移除数据。当为true时，会在调用方法之前移除数据。

### 缓存自动配置

完成了上面的缓存实验之后，可能大家会问，那我们在Spring Boot中到底使用了什么缓存呢？

在Spring Boot中通过`@EnableCaching`注解自动化配置合适的缓存管理器（CacheManager），Spring Boot根据下面的顺序去侦测缓存提供者：

* Generic
* JCache (JSR-107)
* EhCache 2.x
* Hazelcast
* Infinispan
* Redis
* Guava
* Simple

除了按顺序侦测外，我们也可以通过配置属性`spring.cache.type`来强制指定。我们可以通过debug调试查看cacheManager对象的实例来判断当前使用了什么缓存。

### 使用EhCache作为缓存

在Spring Boot中开启EhCache非常简单，**只需要在工程中加入ehcache.xml配置文件并在pom.xml中增加ehcache依赖**，框架只要发现该文件，就会创建EhCache的缓存管理器

* 在src/main/resources目录下创建：ehcache.xml

```xml
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="ehcache.xsd">

    <cache name="users"
           maxEntriesLocalHeap="200"
           timeToLiveSeconds="600">
    </cache>

</ehcache>
```

* 在pom中引入

```xml
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

**默认自动加载resources目录下面的ehcache.xml文件**

对于EhCache的配置文件也可以通过`application.properties`文件中使用`spring.cache.ehcache.config`属性来指定，比如：

```
spring.cache.ehcache.config=classpath:config/another-config.xml
```


## 日志处理

### 控制台输出

在Spring Boot中默认配置了`ERROR`、`WARN`和`INFO`级别的日志输出到控制台

切换到DEBUG级别的方法：

* 在运行命令后加入--debug标志，如：`$ java -jar myapp.jar --debug`
* 在`application.properties`中配置`debug=true`，该属性置为true的时候，核心Logger（包含嵌入式容器、hibernate、spring）会输出更多内容，但是你自己应用的日志并不会输出为DEBUG级别。

### 多彩输出

通过在application.properties 中设置 `spring.output.ansi.enabled`参数：

* NEVER：禁用ANSI-colored输出（默认项）
* DETECT：会检查终端是否支持ANSI，是的话就采用彩色输出（推荐项）
* ALWAYS：总是使用ANSI-colored格式输出，若终端不支持的时候，会有很多干扰信息，不推荐使用

### 文件输出

若要增加文件输出，需要在application.properties中配置logging.file或logging.path属性。

* `logging.file`，设置文件，可以是绝对路径，也可以是相对路径。如：logging.file=my.log
* `logging.path`，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：logging.path=/var/log

日志文件会在10Mb大小的时候被截断，产生新的日志文件，默认级别为：ERROR、WARN、INFO

#### 级别控制

在Spring Boot中只需要在application.properties中进行配置完成日志记录的级别控制。

配置格式：`logging.level.*=LEVEL`

* logging.level：日志级别控制前缀，*为包名或Logger名
* LEVEL：选项TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFF
举例：

```
logging.level.com.didispace=DEBUG：com.didispace包下所有class以DEBUG级别输出
logging.level.root=WARN：root日志以WARN级别输出
```

### 自定义日志配置

由于日志服务一般都在ApplicationContext创建前就初始化了，它并不是必须通过Spring的配置文件控制。因此通过系统属性和传统的Spring Boot外部配置文件依然可以很好的支持日志控制和管理。

根据**不同的日志系统**，你可以按如下规则组织**配置文件名**，就能被正确加载：

* Logback：logback-spring.xml, logback-spring.groovy, logback.xml, logback.groovy
* Log4j：log4j-spring.properties, log4j-spring.xml, log4j.properties, log4j.xml
* Log4j2：log4j2-spring.xml, log4j2.xml
* JDK (Java Util Logging)：logging.properties

Spring Boot官方推荐优先使用带有`-spring`的文件名作为你的日志配置（如使用logback-spring.xml，而不是logback.xml）

### log4j 依赖

pom依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
    <exclusions>
        <exclusion> 
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j</artifactId>
</dependency>
```

在`src/main/resources`目录下加入`log4j.properties`配置文件，就可以开始对应用的日志进行配置使用

配置示例：

```xml
# 输出到控制台
# LOG4J配置
log4j.rootCategory=INFO, stdout

# 控制台输出
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L - %m%n

#输出到文件：
log4j.rootCategory=INFO, stdout, file

# root日志输出
log4j.appender.file=org.apache.log4j.DailyRollingFileAppender
log4j.appender.file.file=logs/all.log
log4j.appender.file.DatePattern='.'yyyy-MM-dd
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p %c{1}:%L - %m%n

# com.didispace包下的日志配置
log4j.category.com.didispace=DEBUG, file

```