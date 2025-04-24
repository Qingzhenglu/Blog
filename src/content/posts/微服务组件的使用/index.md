---
title: 微服务相关组件的使用
published: 2025-01-07
description: 微服务组件的相关使用
image: ./cover.png
tags: [SpringBoot, SpringCloud]
category: Spring
draft: false
---
### 微服务拆分原则

- 不同的微服务，不要重复开发相同的业务

- 数据独立，不要访问其他微服务的数据库
- 将自己的业务暴露，以供其他微服务调用

## Nacos - 注册/发现服务 + 配置中心

1. ### 简介

   官网：<https://nacos.io/zh-cn/docs/v2/quickstart/quick-start.html>

   一个更易于构建云原生应用的动态服务发现，配置管理和服务管理的平台。

2. ### 安装

   - Docker 安装

     ``` shell
     // 拉取镜像
     docker pull nacos/nacos-server:v2.4.3
     // 创建容器
     docker run -d -p 8848:8848 -p 9848:9848 -e MODE=standalone --name nacos nacos/nacos-server:v2.4.3
     
     ```

3. ### 注册中心

   - 依赖引入

     ```xml
     <dependency>
         <groupId>com.alibaba.cloud</groupId>
         <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
     </dependency>
     
     ```

   - 在`application.yml`（推荐yml，配置多时比较整洁）或`application.properties`中进行配置

     ```properties
     spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
     
     #暂未用到配置中心功能，需要关闭配置检查
     #spring.cloud.nacos.config.import-check.enabled=false
     ```

   - 开启服务注册/发现功能

     ```java
     @EnableDiscoveryClient //核心注解
     @SpringBootApplication
     public class MyApplication {
     
      public static void main(String[] args) {
       SpringApplication.run(MyApplication.class, args);
      }
     
     }
     
     ```

     运行项目，访问：<http://localhost:8848/nacos> 可以看到服务已经注册。

4. ### 服务发现

   - `DiscoveryClient`是 Spring Cloud 提供的一个接口，用于服务发现功能。它可以与服务注册中心（如 Eureka、Nacos、Consul 等）进行交互，获取已注册的服务信息。

     ```java
     @Autowired
     DiscoveryClient discoveryClient;
     
     @Test
     void discoveryClientTest(){
         // 从注册中心获取服务列表
         for (String service : discoveryClient.getServices()) {
             System.out.println("service = " + service);
             //获取ip+port
             List<ServiceInstance> instances = discoveryClient.getInstances(service);
             for (ServiceInstance instance : instances) {
                 System.out.println("ip："+instance.getHost()+"；"+"port = " + instance.getPort());
             }
         }
     }
     
     ```

   - `NacosServiceDiscovery` 是 Spring Cloud Alibaba 提供的用于与 Nacos 进行服务发现交互的实现类。它实现了 Spring Cloud 的 `DiscoveryClient` 接口，使得应用程序可以从 Nacos 服务注册中心获取服务实例信息。

     ```java
     @Autowired
     NacosServiceDiscovery nacosServiceDiscovery;
     
     @Test
     void  nacosServiceDiscoveryTest() throws NacosException {
         for (String service : nacosServiceDiscovery.getServices()) {
             System.out.println("service = " + service);
             List<ServiceInstance> instances = nacosServiceDiscovery.getInstances(service);
             for (ServiceInstance instance : instances) {
                 System.out.println("ip："+instance.getHost()+"；"+"port = " + instance.getPort());
             }
         }
     }
     
     ```

5. ### 远程调用

   - `RestTemplate`，编程式REST客户端

     1. 配置`RestTemplate`

        ```java
        @Configuration
        public class Configuration {
        
            @Bean
            RestTemplate restTemplate() {
                return new RestTemplate();
            }
        }
        
        ```

   2. 调用

      ```java
        @Autowired
        RestTemplate restTemplate;
        
         @Test
         void testRestTemplate() {
           String forObject = restTemplate.getForObject("url", String.class);
           System.out.println(forObject);
           System.out.println("-----------------------------");
        
         }
        
      ```

   > 使用RestTemplate，必须精确指定地址和端口
   >
   > 存在的问题：可读性差，参数url难维护

6. ### 负载均衡

   1. 依赖导入

      ```xml
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
      </dependency>
      
      ```

   2. 直接调用实现负载均衡

      ```java
      @Autowired
      private LoadBalancerClient loadBalancerClient;
      
      private Object getObjectFromRemoteWithLoadBalance(Long id) {
          //根据注册的微服务名称选择服务实例（负载均衡默认算法:轮询）
          ServiceInstance choose = loadBalancerClient.choose("service-name");
      
          String url = "http://" + choose.getHost() + ":" + choose.getPort() + "/请求路径/" + id;
      
          return restTemplate.getForObject(url, Object.class);
      }
      ```

   3. 注解实现负载均衡

      ```java
      @Bean
      @LoadBalanced //基于注解的负载均衡
      public RestTemplate restTemplate() {
          return new RestTemplate();
      }
      ```

      ```java
      private Object getObjectFromRemoteWithLoadBalanceAnnoation(Long id) {
      
          //service-name 会被动态替换
          String url = "http://service-name/getInfo(请求路径)/" + id;
      
          return restTemplate.getForObject(url, Object.class);
      }
      ```

      > 实现负载均衡只需要传入`服务名称`，请求发起前会去注册中心确定微服务地址。

7. ### 配置中心

   #### 配置获取步骤 ：

   - 优先读取`bootstrap.yml`中的配置文件获取nacos地址
   - 读取nacos中的配置文件
   - 读取本地`application.yml`
   - 两者合并
   - 创建spring容器
   - 加载bean

     ```java
     @Component
     @ConfigurationProperties(prefix = "order") // 自动绑定配置，动态更新
     @Data
     public class OrderProperties {
     
         String timeout;
     
         String autoConfirm;
     
     }
     
     ```

     ```properties
     order.timeout=10min
     order.auto-confirm=7d
     ```

#### 微服务启动时会从nacos读取多个配置文件：

- `[spring.application.name]-[spring.profiles.active].yaml`,例如:`userservice-dev.yaml`
- `[spring.application.name].yaml`,例如:`userservice.yaml`
  无论profile如何变化，`[spring.application.name].yaml`这个文件一定会加载，因此多环境共享配置可以写入这个文件

#### 配置优先原则：

- 服务名-环境名.yaml > 服务名.yaml > 本地配置



## OpenFeign - 远程调用

1. ### 引入依赖，开启功能

   ```java
   @SpringBootApplication
   @EnableFeignClients
   public class Application {
   
       public static void main(String[] args) {
           SpringApplication.run(Application.class, args);
       }
   
   }
   ```

   > ```
   > @EnableFeignClients
   > @EnableFeignClients(basePackages = "com.example.clients")
   > ```

2. ### 使用

   ```java
   @FeignClient("stores")
   public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();
   
    @GetMapping("/stores")
    Page<Store> getStores(Pageable pageable);
   
    @PostMapping(value = "/stores/{storeId}", consumes = "application/json",
       params = "mode=upsert")
    Store update(@PathVariable("storeId") Long storeId, Store store);
   
    @DeleteMapping("/stores/{storeId}")
    void delete(@PathVariable Long storeId);
   }
   
   ```
   
   主要是基于SpringMVC的注解来声明远程调用的信息，比如：
   
   - 服务名称：userservice
   - 请求方式：GET
   - 请求路径：/user/{id}
   - 请求参数：Longid
   - 返回值类型：User

3. ### 自定义Feign的配置

   | 类型                  | 作用             | 说明                                                   |
   | --------------------- | ---------------- | ------------------------------------------------------ |
   | `feign.Logger.Level`  | 修改日志级别     | 包含四种不同的级别：NONE、BASIC、HEADERS、FULL         |
   | `feign.codec.Decoder` | 响应结果的解析器 | http远程调用的结果做解析，例如解析json字符串为java对象 |
   | `feign.codec.Encoder` | 请求参数编码     | 将请求参数编码，便于通过http请求发送                   |
   | `feign.Contract`      | 支持的注解格式   | 默认是SpringMVC的注解                                  |
   | `feign.Retryer`       | 失败重试机制     | 请求失败的重试机制，默认是没有，不过会使用Ribbon的重试 |

   一般配置日志级别即可。

4. ### 性能优化

   - 连接池配置：

     引入`HttpClient`依赖:

     ```xml
     <!--httpCLient的依赖-->
     <dependency>
         <groupId>io.github.openfeign</groupId>
         <artifactId>feign-httpclient</artifactId>
     </dependency>
     ```

     配置feign:

     ```yml
     feign:
       httpclient:
         enabled: true # 支持HttpClient的开关
         max-connections: 200 # 最大连接数
         max-connections-per-route: 50 # 单个路径的最大连接数
     ```

## Sentinel - 流量保护

微服务中，服务间调用关系错综复杂，一个微服务往往依赖于多个其它微服务，因此会产生雪崩问题。

什么是雪崩问题？

- 微服务之间相互调用，因为调用链中的一个服务故障，引起整个链路都无法访问的情况。

**限流**是对服务的保护，避免因瞬间高并发流量而导致服务故障，进而避免雪崩。是一种**预防**措施。

**超时处理、线程隔离、降级熔断**是在部分服务故障时，将故障控制在一定范围，避免雪崩。是一种**补救**措施。

### 流控

需求：有查询订单和创建订单业务，两者都需要查询商品。针对从查询订单进入到查询商品的请求统计，并设置限流。

使用链路模式，是对不同来源的两个链路做监控。但是sentinel默认会给进入SpringMVC的所有请求设置同一个root资源，会导致链路模式失效。

我们需要关闭这种对SpringMVC的资源聚合，修改order-service服务的application.yml文件：

```yml
spring:
  cloud:
    sentinel:
      web-context-unify: false # 关闭context整合
```

流控模式有哪些？

- 直接：对当前资源限流

- 关联：高优先级资源触发阈值，对低优先级资源限流。

- 链路：阈值统计时，只统计从指定资源进入当前资源的请求，是对请求来源的限流

### 热点参数限流

部分商品是热点商品，例如秒杀商品，我们希望这部分商品的QPS限制与其它商品不一样，高一些。

流控效果有哪些？

- 快速失败：QPS超过阈值时，拒绝新的请求
- warm up： QPS超过阈值时，拒绝新的请求；QPS阈值是逐渐提升的，可以避免冷启动时高并发导致服务宕机。
- 排队等待：请求会进入队列，按照阈值允许的时间间隔依次执行请求；如果请求预期等待时长大于超时时间，直接拒绝

**线程隔离**之前讲到过：调用者在调用服务提供者时，给每个调用的请求分配独立线程池，出现故障时，最多消耗这个线程池内资源，避免把调用者的所有资源耗尽。

**熔断降级**：是在调用方这边加入断路器，统计对服务提供者的调用，如果调用的失败比例过高，则熔断该业务，不允许访问该服务的提供者了。

不管是线程隔离还是熔断降级，都是对**客户端**（调用方）的保护。需要在**调用方** 发起远程调用时做线程隔离、或者服务熔断。

而我们的微服务远程调用都是基于Feign来完成的，因此我们需要将Feign与Sentinel整合，在Feign里面实现线程隔离和服务熔断。

## FeignClient整合Sentinel

SpringCloud中，微服务调用都是通过Feign来实现的，因此做客户端保护必须整合Feign和Sentinel。

1. ### 修改配置，开启sentinel功能

   修改OrderService的application.yml文件，开启Feign的Sentinel功能：

   ```yaml
   feign:
     sentinel:
       enabled: true # 开启feign对sentinel的支持
   ```

   

2. ### 编写失败降级逻辑

   业务失败后，不能直接报错，而应该返回用户一个友好提示或者默认结果，这个就是失败降级逻辑。

   给FeignClient编写失败后的降级逻辑

   ①方式一：FallbackClass，无法对远程调用的异常做处理

   ②方式二：FallbackFactory，可以对远程调用的异常做处理，我们选择这种

## Gateway - 网关

### 路由

### 断言

### 过滤器

## Seata - 分布式事务

TODO
