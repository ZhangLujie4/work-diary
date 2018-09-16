刚开始学springcloud的时候版本问题各种坑，然后弃坑了。

现在相对稳定的版本已经出来了，所以开始搞一搞。接下来是对**springboot(2.0.2.RELEASE)** **springcloud(Finchley.RELEASE)**学习的一些记录。

话不多说，先从最基础的eureka server开始:

服务注册中心应该是最基本的了，驾轻就熟(yml配置)：
启动类加上：
```@EnableEurekaServer```
配置文件：
```yaml
server:
  port: 8081
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false #不将eureka server当做服务注册上去
    fetch-registry: false #不返回注册的信息
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: false
```

有了注册中心，必然要先搞个eureka client注册试试看，贼方便：
首先在启动类加上：
```@EnableDiscoveryClient```
然后配置：
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8081/eureka/
spring:
  application:
    name: product
```

效果如下图：
![eureka client](https://github.com/ZhangLujie4/work-diary/blob/master/img/cloud_1_1.png)

除此之外，eureka的高可用也是需要用小本本记下来的知识点，这个版本高可用的实现和我之前学的有很大出入，不过官方文档写的挺好的，具体实现如下：

server1

```yaml
server:
  port: 8081
eureka:
  instance:
    hostname: peer1
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      #defaultZone: http://peer2:8082/eureka/,http://peer3:8083/eureka/
      defaultZone: http://peer2/eureka/,http://peer3/eureka/
```

server2

```yaml
server:
  port: 8082
eureka:
  instance:
    hostname: peer2
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://peer1/eureka/,http://peer3/eureka
```

server3

```yaml
server:
  port: 8083
eureka:
  instance:
    hostname: peer3
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://peer1/eureka/,http://peer2/eureka/
```

效果如下：

![高可用](https://github.com/ZhangLujie4/work-diary/blob/master/img/cloud_1_2.png)

当服务注册到一个server的时候相当于注册到了三个server，但是显示在server上的只有一个，当一个server挂掉的时候，就会跑到另外两个好的server，避免了服务也挂掉。默认的负载均衡方式是轮询。

