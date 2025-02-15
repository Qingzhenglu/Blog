---
title: spring
published: 2025-02-15
description: ''
image: ./cover2.jpg
tags: ["Spring"]
category: Spring
draft: false
---


### Spring依赖注入方式（5种）

- **构造器注入**：通过构造函数传递依赖项。Spring 推荐使用这种方式，因为它确保了对象在创建时就已经完全初始化。

  ```java
  @Component
  public class MyService {
      private final MyDependency myDependency;
      
      @Autowired
      public Myservice(MyDependency myDependency) {
          this.myDependency = myDependency;
      }
  }
  
  ```

- **Setter 注入**：通过 setter 方法设置依赖项。这种方式的好处是在有变更的情况下，可以重新注入。

  ```java
  @Component
  public class MyService {
      private MyDependency myDependency;
  
      @Autowired
      public void setMyDependency(MyDependency myDependency) {
          this.myDependency = myDependency;
      }
  }
  
  ```

- **字段注入**：直接在字段上使用 `@Autowired` 注解。这种方式虽然常见，但官方不推荐，因为它隐藏了类的依赖关系，且无法注入静态字段。

  ```java
  @Component
  public class MyService {
      @Autowired
      private MyDependency myDependency;
  }
  
  ```

- **方法注入**：通过方法参数注入依赖项，通常用于特定方法的依赖。

  ```java
  @Component
  public class MyService {
      public void performAction(@Autowired MyDependency myDependency) {
          myDependency.doSomething();
      }
  }
  
  ```

- **接口回调注入**：通过实现 Spring 定义的一些内建接口，例如 `BeanFactoryAware`，会进行 `BeanFactory` 的注入。这种方式不常用。

---

### Spring Bean作用域

- **singleton**：默认作用域，整个 Spring 容器中只有一个实例。
- **prototype**：每次获取 Bean 时都会创建一个新的实例。

- **request**：每个 HTTP 请求都会创建一个新的 Bean 实例，仅在 Spring Web 应用中有效。
- **session**：每个 HTTP 会话中会创建一个 Bean 实例，仅在 Spring Web 应用中有效。
- **application**：整个 `ServletContext` 生命周期中只有一个 Bean 实例，仅在 Spring Web 应用中有效。
- **websocket**：每个 `WebSocket`会话中会创建一个 Bean 实例，仅在 Spring Web 应用中有效。

---

### @Bean 和 @Component 的区别

@Bean 和 @Component 都是用于定义 Spring 容器中的 Bean 的注解，但它们的使用场景和方式有所不同：

#### **@Bean**

- **使用位置**：通常用于 Java 配置类的方法上。

- **用途**：用于显式声明一个 Bean 并将其添加到 Spring 容器中，适用于配置第三方库或复杂对象。

- **扫描机制**：不支持自动扫描，需要手动注册。

- **示例**：

  ```java
  @Configuration
  public class AppConfig {
      @Bean
      public DataSource dataSource() {
          DriverManagerDataSource dataSource = new DriverManagerDataSource();
          dataSource.setDriverClassName("com.mysql.jdbc.Driver");
          dataSource.setUrl("jdbc:mysql://localhost:3306/mydb");
          dataSource.setUsername("user");
          dataSource.setPassword("password");
          return dataSource;
      }
  }
  
  ```

#### **@Component**

- **使用位置**：用于类级别，将该类标记为 Spring 容器中的一个组件。

- **用途**：用于自动扫描和注入，适用于自定义服务、DAO 层、控制器等类的自动注册。

- **扫描机制**：支持自动扫描，通过 `@ComponentScan` 自动发现。

- **衍生注解**：

  - **@Service**：用于标识服务层的类
  - **@Repository**：用于标识数据访问层的类（DAO层）
  - **@Controller**：用于标识控制器类，通常用于Spring MVC中处理HTTP请求

- **示例**：

  ```java
  @Component
  public class UserService {
      public void createUser(String name) {
          System.out.println("Creating user: " + name);
      }
  }
  
  ```

总结来说，@Bean 更灵活，适合复杂初始化和手动配置，而 @Component 自动化更强，适合类的简单注册和自动发现。

---

### @Qualifier 注解的作用

@Qualifier 注解在 Spring 中的主要作用是用于在依赖注入时消除歧义。当一个类型有多个实现时，@Qualifier 注解可以指定需要注入哪一个具体的 Bean。

例如，假设有多个 Service 实现类，可以通过 @Qualifier 指定名称选择对应的实现 Bean：

```java
@Component
public class Client {
    private final Service service;

    @Autowired
    public Client(@Qualifier("serviceImpl1") Service service) {
        this.service = service;
    }

    public void doSomething() {
        service.serve();
    }
}

```

在这个例子中，@Qualifier("serviceImpl1") 指定了要注入的具体实现类 `serviceImpl1`。

此外，@Qualifier 可以与 @Primary 一起使用，覆盖 @Primary 的默认行为。例如：

```java
@Component
@Primary
public class DefaultService implements Service {
    public void serve() {
        System.out.println("Default Service");
    }
}

@Component
@Qualifier("specificService")
public class SpecificService implements Service {
    public void serve() {
        System.out.println("Specific Service");
    }
}

@Component
public class Client {
    private final Service service;

    @Autowired
    public Client(@Qualifier("specificService") Service service) {
        this.service = service;
    }

    public void doSomething() {
        service.serve();
    }
}

```

即使 `DefaultService` 被标记为 @Primary，但由于 `@Qualifier("specificService")`，所以最终注入的仍然是 `SpecificService`。

---

### @Value 的作用

在 Spring 框架中，`@Value` 注解用于将外部化的配置值注入到 Spring 管理的 Bean 中。通过 `@Value` 注解，可以将属性文件、环境变量、系统属性等外部资源中的值注入到 Spring Bean 的字段、方法参数或构造函数参数中。

#### 使用场景

1. **配置文件注入**：将属性文件中的值注入到 Bean 中。
2. **系统属性和环境变量**：将系统属性或环境变量的值注入到 Bean 中。
3. **默认值设置**：在属性不可用时，提供默认值。

#### 举例说明

Properties文件

```properties
app.name=MyApp
app.version=1.0.0

```

@Value注解注入属性

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

@Component
public class AppConfig {

    @Value("${app.name}")
    private String appName;

    @Value("${app.version}")
    private String appVersion;

    public String getAppName() {
        return appName;
    }

    public String getAppVersion() {
        return appVersion;
    }
}

```

---

### Spring 中的 @ModelAttribute注解的作用

在 Spring 框架中，`@ModelAttribute` 注解有两个主要作用：

1. **在控制器方法参数中使用**：将请求参数绑定到方法参数上，并将其添加到模型中，以便在视图中使用。

   ```java
   @Controller
   public class MyController {
       @ModelAttribute
       public void addAttributes(Model model) {
           model.addAttribute("msg", "Welcome to the site!");
       }
   
       @RequestMapping("/greeting")
       public String greeting(@ModelAttribute("user") User user) {
           // 处理逻辑
           return "greeting";
       }
   }
   
   ```

   在这个例子中，`@ModelAttribute("user")` 将请求参数绑定到 `User` 对象，并将其添加到模型中，以便在视图中使用。

2. **在方法级别使用**：预处理模型数据，在请求处理方法执行之前将数据添加到模型中。

   ```java
   @Controller
   public class MyController {
       @ModelAttribute("user")
       public User getUser() {
           return new User("John", "Doe");
       }
   
       @RequestMapping("/profile")
       public String profile() {
           // 处理逻辑
           return "profile";
       }
   }
   
   ```

   在这个例子中，`@ModelAttribute("user")` 方法将在每个请求处理方法执行之前被调用，并将返回的 `User` 对象添加到模型中。

---

### 循环依赖

循环依赖（Circular Dependency）是指两个或多个模块、类、组件之间相互依赖，形成一个闭环。简而言之，模块A依赖于模块B，而模块B又依赖于模块A，这会导致依赖链的循环，无法确定加载或初始化的顺序。

#### 解决方案

Spring 通过三级缓存机制来解决循环依赖问题：

1. **一级缓存**：存储所有创建完毕的单例Bean（完整的Bean）。
2. **二级缓存**：存储所有仅完成实例化，但未进行属性注入和初始化的Bean。
3. **三级缓存**：存储能建立这个Bean的一个工厂，通过工厂能获取这个Bean，延迟化Bean的生成，工厂生成的Bean会塞入二级缓存。

通过这种机制，Spring 可以在 Bean 初始化过程中提前暴露一个创建中的 Bean，从而解决循环依赖问题。

在Spring中，只有同时满足以下两点才能解决循环依赖的问题：

1. 以来的Bean必须都是单例
2. 依赖注入的方式，必须**不全是**构造器注入，且`beanName`字母序在前的不能是构造器注入。

> Spring容器 根据`BeanName`字母序来创建Bean。

#### Spring解决循环依赖全流程

1. 首先，获取Bean时会通过Bean Name先去一级缓存查找完整的Bean，如果找到直接返回，否则进行下一步；
2. 看对应的Bean是否在创建中，如果不在直接返回找不到（null），如果在，则会去二级缓存查找Bean，如果找到就返回，否则进行第三步；
3. 在三级缓存中根据Bean Name查找Bean对应的工厂，如果存在工厂则通过工厂创建Bean，并且放置到二级缓存中；
4. 如果三个缓存都没有找到，则返回null。

举例，A依赖B，B依赖A，构成循环依赖。

从上可知，如果查询发现Bean正在创建中，然后再调用`createBean`来创建Bean,而实际创建是调用方法`doCreateBean`。

`doCreateBean`这个方法会执行三个步骤：

1. 实例化
2. 属性注入
3. 初始化

在实例化Bean后，会往三级缓存中塞入一个工厂，通过调用这个工厂的`getObject`方法就可以得到这个Bean。

注意，Spring并不知道是否会有循环依赖发生，也不管，反正往三级缓存中塞入这个工厂。这就是提前暴露。

然后开始执行属性注入，A发现需要注入B，所以去`getBean(B)`，此时又走一遍上面的逻辑，到了B的属性注入，此时B调用`getBean(A)`，这是一级缓存找不到，但是发现A正在创建，于是在二级缓存里找，没找到，再去三级缓存里找，找到了。

通过再三级缓存里暴露的工厂得到A，然后将这个工厂从三级缓存里删除，并将A加入到二级缓存。

B属性注入成功后，调用`initialzeBean`进行初始化，最后返回，将B加入到一级缓存。

然后回到A的属性注入，从一级缓存里找到B注入，然后执行初始化，将A从二级缓存里删除，并加入到一级缓存里。

> 重点：在对象实例化后，都会在三级缓存里加入一个工厂，提前暴露还未完整的Bean，破坏了循环依赖的条件。
>
> 二级缓存虽然可以解决缓存依赖的问题，但在涉及到动态代理AOP时，直接使用二级缓存不做任何处理会导致我们拿到的Bean时未代理的原始对象。如果二级缓存内放的都是代理对象，则违反了Bean的生命周期。（正常代理的对象的生成是在被代理对象初始化后调用生成的）

---

### Spring MVC

Spring MVC 基于经典的MVC模式（Model-View-Controller），将Spring MVC将请求处理流程分为三层：模型层，视图层，控制层，它提供了一种松耦合的方式将用户请求，业务逻辑和视图渲染分离开。

核心：`DispatcherServlet`，即前端控制器。通过注解，配置等方式，将HTTP请求映射到控制器方法，然后由控制器处理请求逻辑并将数据返回给视图层进行渲染。

#### 工作流程

1. 客户端发起HTTP请求
2. 请求被发送到`DispatcherServlet`（前端控制器）
3. `DispatcherServlet`根据`HandleMapping`（处理映射器）将 `url`请求映射到对应的`Controller`上
4. 控制器接受请求并执行相应的业务逻辑，通过`@RequestMapping`和`@Controller`注解定义映射的请求方法
5. 控制器将返回的数据封装到模型对象中去，然后返回对应的视图
6. `DispatcherServlet`会调用视图解析器，来解析返回的视图信息
7. 根据解析好的视图信息，将数据模型渲染到浏览器，并返回客户端

---

### Spring IOC

Spring IOC 全称为Inversion Of Controller，也就是控制反转。

它的核心思想是把对象的管理权限交给Spring容器进行管理。应用程序如果需要使用某个对象的实例，可以直接从容器中获取，这样做降低了程序中对象与对象的耦合性，使程序的整个体系结构更加灵活。

 Spring提供了很多方式声明一个Bean，例如在XML配置文件中通过`<bean>`的标签或者通过`@Service`注解或者通过`@Configuration`里的`@Bean`注解去声明等等。

Spring在启动的时候会解析这些Bean，并将其保存到容器中。

IOC的工作流程大致可以分为两个阶段：

1. IOC容器的初始化阶段：将声明的Bean通过解析和加载后生成`BeanDefinition`，并将其注册到IOC容器中，这些`BeanDefinition`会被保存到一个Map里，完成初始化
2. 通过反射对没有设置`lazy-init`（懒加载）的单例Bean进行初始化，然后要完成Bean的依赖注入。我们通常用`@Autowired`注解或通过`BeanFactory.getBean()`从IOC容器中获取指定Bean的实例，对于设置了`lazy-init`以及非单例Bean 的实例化，是在每一次获取Bean对象的时候调用Bean的初始化方法来完成实例的。

---

### Spring 中有两个相同id的Bean会报错吗？

如果使用XML配置来声明Bean，在同一个xml文件不能存在相同id的Bean，在启动时会报错（发生在容器解析xml文件，检验Bean 的id的唯一性的时候）；

但是在不同的xml文件中可以存在，但是IOC容器在加载Bean的时候默认会把多个相同id的bean进行覆盖 ，在Spring Boot3.x后发生了改变：通过`@Configuration`里的`@Bean`注解去声明两个或多个相同名字的Bean，只会声明第一个，之后的就不再注册。
