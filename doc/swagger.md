### Swagger配置

在写完大量接口后，我们需要对接口进行测试，常见的工具有postman，insomnia等。

最方便快捷的方法是程序自动生成可测试api接口，并显示每个接口对应的参数类型等。

这里介绍一款RESTFUL接口的文档在线自动生成+功能测试功能软件——swagger。

首先引入maven依赖：

```java
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger2</artifactId>
    <version>2.7.0</version>
</dependency>

<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-swagger-ui</artifactId>
    <version>2.7.0</version>
</dependency>
```

然后写一个配置文件：

```java
/**
 * @author tori
 * @description
 * @date 2018/8/26 下午9:33
 */

@Configuration
@EnableSwagger2
public class SwaggerConfigurer {

    @Bean //这个千万别忘了
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("com.tori.swagger.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {

        return new ApiInfoBuilder()
                .title("Spring Boot中使用swagger2")
                .description("rest api 文档构建")
                .termsOfServiceUrl("http://localhost:8080/swag")
                .contact(new Contact("tori", "", "949xxxx75@qq.com"))
                .license("license")
                .licenseUrl("http://licenseUrl")
                .version("V1.0.0")
                .build();

    }
}
```

然后用swagger的一些常用注解描述接口就好了，非常简单。

对于日常基本的接口测试，swagger往往表现优秀。