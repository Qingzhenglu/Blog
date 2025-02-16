---
title: 微服务
published: 2025-01-07
description: 微服务学习笔记
image: ./cover.png
tags: [SpringBoot, SpringCloud]
category: Spring
draft: false
---
## Nacos - 注册/配置中心

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

   - `ConfigurationProperties`，自动绑定配置，动态更新

     ```java
     @Component
     @ConfigurationProperties(prefix = "order")
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

   

## Sentinel - 流量保护

TODO

## Gateway - 网关

TODO

## Seata - 分布式事务

TODO
