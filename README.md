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

&emsp;&emsp;一般的诸如配置类和条件装配规则自不必说，在这个类中，还有一个静态内部类

![MVCStatic1](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/MVCStatic1.png)

&emsp;&emsp;可以看到，这个静态内部类有一个EnableConfigurationProperties注解，这个注解开启了里面加载的类配置绑定功能并把其注册到ioc容器中。我们分别进入WebMvcProperties和ResourceProperties类中查看。

```java

@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {}

@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties {)
```

&emsp;&emsp;可以看到这两个类分别与配置文件绑定且前缀为spring.mvc和spring.resources。这里暂且不做过多解释，不过前面对于静态资源访问在配置文件中的配置联系起来了，可以看到为什么前面是mvc以及resources了。

####1.配置类只有一个有参构造器

而这个WebMvcAutoConfigurationAdapter这个静态内部类里面只有一个有参构造。

```java

public WebMvcAutoConfigurationAdapter(ResourceProperties resourceProperties, WebMvcProperties mvcProperties,
				ListableBeanFactory beanFactory, ObjectProvider<HttpMessageConverters> messageConvertersProvider,
				ObjectProvider<ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider,
				ObjectProvider<DispatcherServletPath> dispatcherServletPath,
				ObjectProvider<ServletRegistrationBean<?>> servletRegistrations) {
			this.resourceProperties = resourceProperties;
			this.mvcProperties = mvcProperties;
			this.beanFactory = beanFactory;
			this.messageConvertersProvider = messageConvertersProvider;
			this.resourceHandlerRegistrationCustomizer = resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
			this.dispatcherServletPath = dispatcherServletPath;
			this.servletRegistrations = servletRegistrations;
		}
````

在这个构造器中，所有的参数都是直接从容器中获取的,因为这个Mvc自动配置类是采用了轻量级的配置，也就是前面configuration注解里的proxyBeanMethods为false

####2.资源处理默认规则
&emsp;&emsp;接下来看看资源处理的默认规则，同样在MVC这个自动配置类里面，往下找到addREsourceHandlers这个方法。

```java

		@Override
		public void addResourceHandlers(ResourceHandlerRegistry registry) {
			if (!this.resourceProperties.isAddMappings()) {
				logger.debug("Default resource handling disabled");
				return;
			}
			Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
			CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
			if (!registry.hasMappingForPattern("/webjars/**")) {
				customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
						.addResourceLocations("classpath:/META-INF/resources/webjars/")
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
			String staticPathPattern = this.mvcProperties.getStaticPathPattern();
			if (!registry.hasMappingForPattern(staticPathPattern)) {
				customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
						.addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
		}
```

首先第一个条件判断语句

```java

if (!this.resourceProperties.isAddMappings()) {
	logger.debug("Default resource handling disabled");
	return;
	}
```
这里进入这个isAddMappings方法

```java

	public boolean isAddMappings() {
		return this.addMappings;
	}
```

继续往里走

![resourceproperties](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/resourceproperties.png)

&emsp;可以看到ResourceProperties和addMapiings这个属性进行绑定,且默认为true，而从最开始的条件判断语句输出的debug语句可以分析得知，这个属性表示是否开启静态资源规则。因此，假使需要禁用静态资源规则，则在配置里面将其改为true即可

```java

spring:
	resources:
	  add-mappings: false
```

第一次条件判断结束后，这里会有一个缓存策略

```java

Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
```

同样这个cache属性也是在ResourceProperties中的，我们可以在配置中设置缓存时间，单位为s，这里不过多赘述

```java

spring:
	resources:
	  cache: 10000   #设置10000s
```

&emsp;第二个条件判断是webjars的映射规则，有兴趣的朋友可以去百度，方法原理同接下来要讲的静态资源路径规则类似。

首先，获取到静态资源默认访问前缀

```java

String staticPathPattern = this.mvcProperties.getStaticPathPattern();
```

同样不停的往里走，最后最后可以看到在WebMvcProperties里面有一个staticPathPattern

![staticPathPattern1](https://github.com/lijiasheng12333/image/blob/main/picture/springboot/MVC/staticPathPattern1.png)

这里就解释了前面为什么修改静态资源访问前缀是static-path-pattern了

&emsp;&emsp;接下来的条件判断

```java

if (!registry.hasMappingForPattern(staticPathPattern)) {
				customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
						.addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
						.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
			}
```

这里面有一条语句是添加资源地址

```java

addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
```

注意这里有一个是this.resourceProperties.getStaticLocations()
往里走可以看到

最后获取到的是ResourceProperties的staticLocations属性

```java

private String[] staticLocations = CLASSPATH_RESOURCE_LOCATIONS;

private static final String[] CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/",
			"classpath:/resources/", "classpath:/static/", "classpath:/public/" };
```

从这里也就解释了为什么默认静态资源访问地址是在META-INF/resources/",resources/,static/,public/这些文件夹下了。同样也解释了在配置文件中修改为什么是修改名字为static-locations这个属性了。

总结起来就是，在默认情况下，在访问/**时，会去上述的默认文件夹去寻找静态资源。