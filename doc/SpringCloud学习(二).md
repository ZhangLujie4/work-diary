# 注册到eureka server的应用相互调用接口

常用有两种方式：1.restTemplate 2.feign

## 1.RestTemplate
RestTemplate的三种使用方式，服务之间调用接口。

先照上一篇说的在server上注册两个服务：**product**和**order**

在**product**创建一个简单的接口：

```java
@RestController
public class ServerController {
    @GetMapping("/msg")
    public String msg() {
        return "sdfgshjfgshjgfhsdfgsdgfsdgfy";
    }
}
```

可以看看接口的输出：

![api](https://img-blog.csdn.net/20180904111219279?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMjM1MDU5/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

想要在**order**获取到**product**的接口数据：

### 方法一

直接使用restTemplate,url写死

```java
@Slf4j
@RestController
public class ClientController {

    @GetMapping("/getProductMsg")
    public String getProductMsg() {
        RestTemplate restTemplate = new RestTemplate();
        String response = restTemplate.getForObject("http://localhost:8084/msg", String.class);
        log.info("response={}", response);
        return response;
    }
}
```

返回结果是正确的，但缺点也很明显，这里的url固定写死了，你在用的时候可能不知道对方ip。

### 方法二

利用loadBalancerClient通过应用名获取url，然后再使用restTemple

```java
@Slf4j
@RestController
public class ClientController {
    
    @Autowired
    private LoadBalancerClient loadBalancerClient;
    
    @GetMapping("/getProductMsg")
    public String getProductMsg() 
        ServiceInstance serviceInstance = loadBalancerClient.choose("PRODUCT");
        String url = String.format("http://%s:%s", serviceInstance.getHost(), serviceInstance.getPort()) + "/msg";
        String response = (new RestTemplate()).getForObject(url, String.class);
        log.info("response={}", response);
        return response;
    }
}
```

### 方法三

将restTemplate作为一个bean配置上去

```java
/**
 * @author tori
 * 2018/8/6 下午6:24
 */

@Component
public class RestTemplateConfig {

    @Bean
    @LoadBalanced //这个是重点
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```
利用@LoadBalanced，可在restTemplate里面用应用的名字

```java
@Slf4j
@RestController
public class ClientController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/getProductMsg")
    public String getProductMsg() {
        String response = restTemplate.getForObject("http://PRODUCT/msg", String.class);
        log.info("response={}", response);
        return response;
    }
}
```
## 2.Feign

首先加入依赖：

```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
    <version>1.4.5.RELEASE</version>
</dependency>
```

然后在启动类加上：

```@EnableFeignClients```

创建一个类获取**product**服务

```java
/**
 * @author tori
 * 2018/8/9 下午2:20
 */
@FeignClient(name = "product") //服务名称
public interface ProductClient {

    @GetMapping("/msg") //接口
    String productMsg();

    @PostMapping("/product/listForOrder") //完整的接口(除ip和端口外)
    List<ProductInfo> listForOrder(@RequestBody List<String> productIdList);
}
```
**ps:**这里注意@RequestBody一定是和@PostMapping连用的

如果要在**order**应用内调用**product**接口获取到的数据，直接用：

```java
@Autowired
private ProductClient productClient;
```


直接用
productClient.productMsg()，productClient.listForOrder(Arrays.asList("1", "2"))
就可以取到。

看起来是不是超级方便。

**Feign**总结：

+ 声明式REST客户端(伪RPC)   [RPC远程过程调用协议]
+ 采用了基于接口的注解




