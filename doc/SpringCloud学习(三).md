# SpringCloud配置中心以及动态刷新配置


SpringCloud中每一个应用都有对应的项目，项目都有各自的配置文件。统一管理的配置中心，能够像应用一样注册到eureka server上。
各项目可以通过拉取不同的配置完成项目的启动，通过动态刷新配置的方法，应用还可以在不重启的情况下更改配置环境。

<strong style="color:red">springboot 2.0.1.RELEASE</strong>
<strong style="color:red">springcloud Finchley.SR1</strong>

## 建立配置中心

1、在pom.xml中引入依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

2、在github上建一个项目——**config-repo**

3、在github的项目里建一个order-dev.yml文件
```yaml
spring:
  datasource:
    username: root
    driver-class-name: com.mysql.jdbc.Driver
    password: zhanglujie
    url: jdbc:mysql://127.0.0.1:3306/springcloud?useSSL=false&characterEncoding=utf-8
server:
  port: 8085
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8762/eureka/
```

4、在配置文件yml中

![配置](/img/cloud_3_1.png)

如果所建的git是私人仓库，还需要在配置中加上`password`和`username`

5、在启动类上

![启动类](/img/cloud_3_2.png)

然后将配置中心启动就可以注册到eureke server了

## 注册服务获取配置

1、在pom.xml中引入依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```
2、将application.yml重命名为bootstrap.xml

<strong>ps:</strong>因为将application.xml的数据库配置剪切到了配置中心，按照一般的配置文件加载顺序，在application.xml中无法获得mysql的配置信息，程序启动会报错，用了bootstrap.xml可以解决这个问题。

3、bootstrap.xml配置:

![bootstrap](/img/cloud_3_3.png)

## 问题与解决方法

按照上述的配置启动，你会发现应用根本运行不起来，这就尴尬了。

看到启动的控制台的信息：

![控制台信息](/img/cloud_3_4.png)

可以看到一个很奇怪的东西
```
http://localhost:8888/order/dev
```
后来查阅资料发现只要在应用中加入
```yml
spring:
    cloud:
        config:
            uri: http://localhost:8080
```
就能解决这个问题，但是这种解决方法治标不治本。因为你需要在这里指出具体的配置中心的ip和端口，很不方便。

其实，如果eureka server是开启在`默认端口8761`,那么即使你不加uri的配置，注册应用也能顺利启动。这就说明，当eureka client在启动时，如果没有配置好eureka.client.service-url.defaultZone, 应用会从默认的8761端口找config服务。很显然，我们注册中心开启在端口8762, 所以找不到配置文件, eureka client启动失败

解决方法也很简单, 只需要把配置中心order-dev.yml中有关服务注册的代码写到bootstrap.yml, 启动项目时，会通过注册的地址找到config项目获取配置。

## 刷新配置

1、首先在config server引入依赖：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

2、启动会发现这样的一些数据：
![mappings](/img/cloud_3_5.png)

我们可以看到有
**/{label}/{name}-{profiles}.yml**
这里的label代表git的分支, name代表应用的名称， profiles代表环境,如dev、test

3、在需要修改的部分加上注解 **@RefreshScope**

4、动态刷新
刷新git上的配置时，一般使用git工具，但是实际操作时，gitlab,github,gitee的webhook都不好用。:joy:

后来查阅了网上的资料，解决了配置文件自动更新，具体实现如下：

![](/img/cloud_3_6.png)

config server的启动类
```java
package com.zlj.config;

import lombok.extern.slf4j.Slf4j;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.DefaultHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

import java.io.IOException;

@Slf4j
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
@RestController
public class ConfigApplication {

    @Value("${zlj.webhook}")
    private String webhook;

    public static void main(String[] args) {
        SpringApplication.run(ConfigApplication.class, args);
    }

    @PostMapping("/refresh")
    public void refresh() {
        CloseableHttpClient httpclient = HttpClients.createDefault();
        String url = webhook + "/actuator/bus-refresh";
        log.info("url = {}", url);
        HttpPost httpPost = new HttpPost(url);
        try {
            httpclient.execute(httpPost);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```
只要调用这个自定义的接口就可以动态刷新配置了。












