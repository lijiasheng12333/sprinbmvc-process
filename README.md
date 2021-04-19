# sprinbmvc-process
springmvc process in the springboot framework

在springboot中，springMVC可以进行自动配置，大多场景都无需我们自定义配置，官方是这样说的

![web简介](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/MVCconfig.png)

因此我们需要关注一些常用模块的使用，例如静态资源，请求参数等等
###静态资源访问
####静态资源目录以及访问前缀
首先看看官方解释

![静态资源](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/staticContent.png)

只要静态资源放在类路径下:called /static (or /public or /resources or /META-INF/resources

访问 ： 当前项目根路径/ + 静态资源名

原理： 静态映射/**。

请求进来，先去找Controller看能不能处理。不能处理的所有请求又都交给静态资源处理器。静态资源也找不到则响应404页面

对于访问前缀，springboot底层是默认无前缀:/**

当前项目 + static-path-pattern + 静态资源名 = 静态资源文件夹下找。

至于为什么是配置是这两个名字，后文会详细提及

那么，我们可以如何配置静态资源路径以及访问的前缀呢?

```java

spring:
  mvc:
    static-path-pattern: /res/**   

  resources:
    static-locations: [classpath:/test/]
```

![代码1](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/%E4%BB%A3%E7%A0%811.png)

注意配置文件除了在properties下，也可以在yaml下，最好在yaml文件下

###欢迎页支持
  欢迎页默认情况是index.html
  
  * 注意，需要设置欢迎页的情况下，如果需要直接访问静态资源中的欢迎页。静态资源路径可以进行配置修改，但是访问前缀不能修改，否则的话则需要通过处理器handler进行跳转访问，至于为什么同样后面会提到。

###静态资源配置原理
  * SpringBoot在启动时会默认加载一系列的AutoConfiguration类，也就是自动配置类。
  * 而对于MVC功能的配置类是WebMvcAutoConfiguration。

![autoconfigurationaddress](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/autoconfigurationaddress.png)

![mvcautoconfigurationaddress](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/mvcautoconfigurationaddress.png)

在这里我们可以清晰的看到

![MVC1](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/MVC1.png)

一般的诸如配置类和条件装配规则自不必说，在这个类中，还有一个静态内部类

![MVCStatic1](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/MVCStatic1.png)

可以看到，这个静态内部类有一个EnableConfigurationProperties注解，这个注解开启了里面加载的类配置绑定功能并把其注册到ioc容器中。我们分别进入WebMvcProperties和ResourceProperties类中查看。

```java

@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {}

@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties {)
```

可以看到这两个类分别与配置文件绑定且前缀为spring.mvc和spring.resources。这里暂且不做过多解释，不过前面对于静态资源访问在配置文件中的配置联系起来了，可以看到为什么前面是mvc以及resources了。

