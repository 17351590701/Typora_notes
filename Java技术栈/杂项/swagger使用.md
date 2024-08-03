> [!TIP]
>
> SpringBoot3版本依赖于jakarta依赖包，但是Swagger依赖底层应用的javax依赖包
>
> spring2---SpringFox
>
> sprng3---SpringDoc

### Springboot3使用Swagger

导入依赖

```xml
<!--整合swaggerdoc-->
<dependency>
   <groupId>org.springdoc</groupId>
   <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
   <version>2.5.0</version>
</dependency>
```

yml配置文件（**可选**）

```yaml
springdoc:
  swagger-ui:
    path: /swagger-ui.html
```

swagger配置文件

```java
@Configuration
public class SwaggerConfig {
    @Bean
    public OpenAPI myOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("API接口文档")
                        .description("接口文档")
                        .version("v1.0.0"));
    }
} 
```

### 常用注解

| openapi       | 含义                                       |
| ------------- | ------------------------------------------ |
| @Tag          | 用在controller类上，描述此controller的信息 |
| @Operation    | 用在controller的方法里，描述此api的信息    |
| @Parameter    | 用在controller方法里的参数上，描述参数信息 |
| @Parameters   | 用在controller方法里的参数上               |
| @Schema       | 用于Entity，以及Entity的属性上             |
| @ApiResponse  | 用在controller方法的返回值上               |
| @ApiResponses | 用在controller方法的返回值上               |
| @Hidden       | 用在各种地方，用于隐藏其api                |

Swagger2->OpenApi

- `@Api` → `@Tag`
- `@ApiIgnore`→`@Parameter(hidden = true)`或`@Operation(hidden = true)`或`@Hidden`
- `@ApiImplicitParam` → `@Parameter`
- `@ApiImplicitParams` → `@Parameters`
- `@ApiModel` → `@Schema`
- `@ApiModelProperty(hidden = true)` → `@Schema(accessMode = READ_ONLY)`
- `@ApiModelProperty` → `@Schema`
- `@ApiOperation(value = "foo", notes = "bar")` → `@Operation(summary = "foo", description = "bar")`
- `@ApiParam` → `@Parameter`
- `@ApiResponse(code = 404, message = "foo")` → `@ApiResponse(responseCode = "404", description = "foo")`



访问地址：http://localhost:8080/swagger-ui.html



[^参考文献]: https://zhuanlan.zhihu.com/p/638887405



