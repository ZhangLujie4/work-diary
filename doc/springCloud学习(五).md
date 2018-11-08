# Zuul的综合使用 （笔记，具体实现看代码）
服务网关和zuul 
api-gateway
稳定性，高可用，性能优化，并发控制，安全性，扩展性

服务网关
+ nginx(性能极高)+lua
+ kong
+ Tyk(go语言)
+ spring cloud zuul

路由+过滤器=zuul
核心：系列过滤器
+ 前置（Pre）
+ 路由（Route）
+ 后置（Post）
+ 错误（Error）

![zuul处理流程](/img/cloud_5_1.png)

**前置（pre）**
限流 鉴权 参数校验调整
**后置（post）**
统计 日志

# zuul限流
+ 防轰炸
+ 请求被转发前调用
+ 限流早于鉴权

zuul的高可用

spring-boot-starter-web 服务注册一定要加这个依赖

nginx+zuul混搭

@CrossOrigin => 放在方法/类 用于跨域

分布式session vs oauth2

zuul -> filter过滤器

ajax跨域完全讲解

A->B->C
雪崩效应

Hystrix 
**基于Netflix对应的hystrix** => **服务降级** <-1.优先核心服务 2.非核心弱可用或不可用 3.HystrixCommand
**熔断** **依赖隔离** **监控(Hystrix Dashboard)**

依赖隔离
线程池隔离

Hystrix自动实现依赖隔离
断路器 超时时间

circle breaker 断路器 及时切断故障的主逻辑调用
feign api-gateway 都有hystrix

容错：
1、重试（短暂）
2、断路器（封装在断路器对象中）

feign里面用hystrix：
+ yml
+ 包扫描
+ 降级类上面的注解

初次访问懒加载

actutor和jdbc依赖循环冲突，是版本自己的问题，把boot 2.0.1->2.0.2

# 服务追踪
Spring Cloud Sleuth->侦查
**zipkin**
starter-zipkin = sleuth-zipkin + starter-sleuth
运行一遍就能在可视化界面找到服务了

使用方法：
+ 引入依赖
+ 启动zipkin server
+ 配置参数

openTracing ?
分布式追踪系统
数据采集
数据存储
数据展示
调用链


annotation =事件=>
事件类型 cs cr ss sr
traceId 起点
spanId 下一层请求的跟踪id
parentId 上一次请求的id


# rancher

docker login 一定要在sudo下
docekr build -t 名字 





