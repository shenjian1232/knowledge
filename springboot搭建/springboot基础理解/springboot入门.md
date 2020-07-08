#spring boot
spring boot只是默认了很多框架的使用方式，整合了所有的框架。
#使用spring boot的好处
在以往搭建spring web项目时：

1.配置web.xml，配置spring和spring mvc

2.配置数据源配置连接、配置spring事务（从而实现ioc、aop）

3.配置加载配置文件的读取，开启注解

4.配置日志文件

5.配置完成后部署tomcat

*......*

使用spring boot将拜托这些配置。

#搭建spring boot web
##Maven 构建项目
1.访问 http://start.spring.io/

2.选择构建工具Maven Project、Java、Spring Boot版本。

3.点击Generate Project下载项目压缩包

4.解压后，使用Idea导入项目，解压后，使用 Idea 导入项目，File -> New -> Model from Existing Source.. -> 选择解压后的文件夹 -> OK，选择 Maven 一路 Next，OK done!
##Idea构建项目
1.选择 File -> New —> Project… 弹出新建项目的框

2.选择 Spring Initializr，Next 也会出现上述类似的配置界面，Idea 帮我们做了集成

3.填写相关内容后，点击 Next 选择依赖的包再点击 Next，最后确定信息无误点击 Finish。
##项目结构介绍


如上图所示，Spring Boot 的基础结构共三个文件:
**src/main/java** 程序开发以及主程序入口
**src/main/resources** 配置文件
**src/test/java** 测试程序
另外， Spring Boot 建议的目录结果如下：
root package 结构：**com.example.myproject**
```
com
  +- example
    +- myproject
      +- Application.java
      |
      +- model
      |  +- Customer.java
      |  +- CustomerRepository.java
      |
      +- service
      |  +- CustomerService.java
      |
      +- controller
      |  +- CustomerController.java
      |
```
1、Application.java 建议放到根目录下面,主要用于做一些框架配置

2、model 目录主要用于实体与数据访问层（Repository）

3、service 层主要是业务类代码

4、controller 负责页面访问控制
采用默认配置可以省去很多配置，当然也可以根据自己的喜欢来进行更改
最后，启动 Application main 方法，至此一个 Java 项目搭建好了！

##引入web模块
1.pom.xml中添加支持web的模块
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
pom.xml 文件中默认有两个模块：

**spring-boot-starter** ：核心模块，包括自动配置支持、日志和 YAML，如果引入了**spring-boot-starter**那么 **spring-boot-starter-web** web模块可以去掉此配置，因为 **spring-boot-starter-web** 自动依赖了 **spring-boot-starter**。
**spring-boot-starter-test** ：测试模块，包括 JUnit、Hamcrest、Mockito。

2.编写Controller内容：
```
@RestController
public class HelloWorldController {
    @RequestMapping("/hello")
    public String index() {
        return "Hello World";
    }
}
```
**@RestController** 的意思就是 Controller 里面的方法都以 json 格式输出，不用再写什么 jackjson 配置的了！
==
3.启动主程序，打开浏览器访问 http://localhost:8080/hello。

##如何做单元测试
打开的**src/test/**下的测试入口，编写简单的 http 请求来测试；使用 mockmvc 进行，利用**MockMvcResultHandlers.print()**打印出执行结果。
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class HelloTests {


    private MockMvc mvc;

    @Before
    public void setUp() throws Exception {
        mvc = MockMvcBuilders.standaloneSetup(new HelloWorldController()).build();
    }

    @Test
    public void getHello() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/hello").accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Hello World")));
    }

}
```
##开发环境的调试

热启动在正常开发项目中已经很常见了吧，虽然平时开发web项目过程中，改动项目启重启总是报错；但springBoot对调试支持很好，修改之后可以实时生效，需要添加以下的配置：
```
 <dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <fork>true</fork>
            </configuration>
        </plugin>
</plugins>
</build>
```
该模块在完整的打包环境下运行的时候会被禁用。如果你使用 java -jar启动应用或者用一个特定的 classloader 启动，它会认为这是一个“生产环境”。


