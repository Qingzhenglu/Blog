---
title: 微服务场景
published: 2025-01-07
description: 微服务架构下遇到的问题及解决方案
image: ''
tags: [SpringBoot, SpringCloud]
category: Spring
draft: false
---

## 1. 创建订单时重复调用，导致结果不一样

解决方案：实现接口的幂等性（接口可重复调用，在调用多次的情况下，接口得到的结果是一致的）

- 分布式锁

  1. 创建一个工具类来管理分布式锁：

  ```java
  @Component
  public class DistributedLock {
      @Autowired
      private StringRedisTemplate redisTemplate;
  
      public boolean acquireLock(String lockKey, String lockValue, int expireTime) {
          Boolean success = redisTemplate.opsForValue().setIfAbsent(lockKey, lockValue, expireTime, TimeUnit.SECONDS);
          return success != null && success;
      }
  
      public void releaseLock(String lockKey, String lockValue) {
          String currentValue = redisTemplate.opsForValue().get(lockKey);
          if (currentValue != null && currentValue.equals(lockValue)) {
              redisTemplate.delete(lockKey);
          }
      }
  }
  ```

  2. 创建一个REST控制器来处理订单创建请求：

  ```java
  @RestController("/orders")
  public class OrderController {
      @Autowired
      private OrderService orderService;
  
      @PostMapping("/create")
      public String createOrder(@RequestHeader("X-Request-ID") String orderId, @RequestBody OrderRequest orderRequest) {
          return orderService.createOrder(orderId, orderRequest);
      }
  }
  ```

  3. 使用分布式锁来确保幂等性：

  ```java
  @Service
  public class OrderService {
      @Autowired
      private StringRedisTemplate redisTemplate;
  
      @Autowired
      private DistributedLock distributedLock;
  
      public String createOrder(String orderId, OrderRequest orderRequest) {
          // 尝试获取分布式锁
          String lockKey = "lock:" + orderId;
          String lockValue = UUID.randomUUID().toString();
          int expireTime = 10; // 锁的过期时间，单位秒
  
          if (distributedLock.acquireLock(lockKey, lockValue, expireTime)) {
              try {
                  // 检查订单ID是否已经处理过
                  if (redisTemplate.hasKey(orderId)) {
                      // 如果已经处理过，直接返回之前的结果
                      return "OrderID:" + orderId + " has already been processed.";
                  }
  
                  // 处理订单创建逻辑
                  // 这里假设订单创建成功并返回订单详情
                  String orderDetails = "Order created successfully with details: " + orderRequest.toString();
  
                  // 记录已经处理过的订单ID
                  redisTemplate.opsForValue().set(orderId, orderDetails);
  
                  return orderDetails;
              } finally {
                  // 释放锁
                  distributedLock.releaseLock(lockKey, lockValue);
              }
          } else {
              // 获取锁失败，返回错误信息
              return "Failed to acquire lock for orderID:" + orderId;
          }
      }
  }
  ```

  > 对于更新订单服务，可以通过一个版本号机制，每次更新数据前校验版本号，更新数据同时自增版本号，这样的方式（乐观锁），来确保更新订单服务的幂等性。
