---
layout: post
title: 详解springboot自动装配原理
categories: Springboot
description: springboot自动装配原理
keywords: springboot autoconfiguration
---

# springboot自动装配原理

   

## 引述 

&nbsp; &nbsp; 在SpringBoot项目中，无需各种配置文件，一个main方法，就能把项目启动起来，离不开一个重要的功能--自动装配，这也是springboot和springmvc最大的不同之处。 

&nbsp; &nbsp; 说到自动装配的原理，springboot启动类的注解@SpringBootApplication是一个组合注解，其中包含了三个重要注解：  

- @SpringBootConfiguration：这个注解的底层是一个@Configuration注解，被@Configuration注解修饰的类是一个IOC容器，支持JavaConfig的方式来进行配置；

- @ComponentScan：这个就是扫描注解的意思，默认扫描当前类所在的包及其子包下包含的注解，将@Controller/@Service/@Component/@Repository等注解加载到IOC容器中；

- @EnableAutoConfiguration：这个注解表明启动自动装配，里面包含连个比较重要的注解@AutoConfigurationPackage和@Import。 

&nbsp; ![](/images/posts/springboot/autoconfig1.png) 

这里重点分析@EnableAutoConfiguration，这也是自动装配的核心注解。




## @EnableAutoConfiguration注解原理
&nbsp; &nbsp; 点击进入 @EnableAutoConfiguration注解，可以看到如下图所示，重点是AutoConfigurationImportSelector这个类，@import注解会在刷新容器时解析，并将import的类实例化注册到spring容器。

&nbsp; ![](/images/posts/springboot/autoconfig2.png) 

&nbsp; &nbsp; AutoConfigurationImportSelecto实现了ImportSelector接口的selectImports方法，重点逻辑也在此方法中（如果要知道具体怎么实现的，需要看spring源码细节，在此不详细讨论），依次按照下图顺序点击进入核心方法： 

&nbsp; ![](/images/posts/springboot/autoconfig3.png)  
&nbsp; ![](/images/posts/springboot/autoconfig4.png)  
&nbsp; ![](/images/posts/springboot/autoconfig5.png)  
&nbsp; ![](/images/posts/springboot/autoconfig6.png)  
&nbsp; ![](/images/posts/springboot/autoconfig7.png)  

&nbsp;&nbsp; **重点来了**，classLoader.getResources(FACTORIES_RESOURCE_LOCATION)，这句代码很显然是加载某个资源文件夹下面的数据，这个路径是：

&nbsp; ![](/images/posts/springboot/autoconfig8.png)    
 
&nbsp; &nbsp; **@EnableAutoConfiguration就是利用SpringFactoriesLoader机制加载自动装配配置的，它的配置数据在META-INF/spring.factories中，我们打开spring-boot-autoconfigure jar中的该文件，发现EnableAutoConfiguration对应着N多XXXAutoConfiguration配置类，我们截取几个重要的配置类如下(已经删除了很多)**
 
&nbsp; ![](/images/posts/springboot/autoconfig9.png)    
&nbsp; &nbsp; 可以看到Spring Boot提供了N多XXXAutoConfiguration类，有Spring Framework的、Web的、redis的、JDBC的等等。这些类都会在容器启动时，通过自动装配注解，解析并注册到spring容器，这样我们只需要简单的在yml中配置一下就可以直接使用了。  

&nbsp; &nbsp;  此外，@EnableAutoConfiguration注解会在容器启动时，扫描所有jar包下的META-INF/spring.factories，具体有什么用，接着往下看。

## 基于spring.factories的扩展机制
&nbsp; &nbsp;  SpringFactoriesLoader机制加载自动装配也是spring集成第三方框架的一种方式，拿spring集成dubbo这个框架举例，我们在导入jar包后，只需要在yml文件中做如下配置即可  

&nbsp; ![](/images/posts/springboot/autoconfig10.png)     

原理是什么呢？ 

来到dubbo的源码包下，会发现在META-INF目录下也有一个spring.factories文件，截图如下： 

&nbsp; ![](/images/posts/springboot/autoconfig11.png)      

&nbsp; &nbsp;  这里也有一个对EnableAutoConfiguration的配置，这是spring自定义的SPI服务。通过实现自动装配注解，在容器启动，解析启动类@EnableAutoConfiguration注解时，所有jar包下META-INF/spring.factories目录配置为EnableAutoConfiguration的类，都会被解析并注册到spring容器。

&nbsp; &nbsp; **注册到spring容器后，下一个问题就是怎么自动读取我们yml文件的配置？**

&nbsp; &nbsp; 还是以dubbo为例，我们点击进入spring.factories配置为EnableAutoConfiguration的类--DubboAutoConfiguration，如图：

&nbsp; ![](/images/posts/springboot/autoconfig12.png)      

上图标识的就是开启获取文件属性注解，点击进入
 
&nbsp; ![](/images/posts/springboot/autoconfig13.png)      
&nbsp; &nbsp;  @ConfigurationProperties(DUBBO_PREFIX)这句注解就是把yml文件中，以dubbo为前缀的配置项，都映射到本类的实例化对象对应的属性中，同时在这里可以看到所有的可配置项。  
&nbsp; &nbsp;  也就是说，在自动装配阶段，通过spring.factories配置，会随着容器启动解析dubbo框架中的自动装配类；这个类中会包含读取yml的逻辑。


***总结***  
&nbsp; &nbsp; Spring Boot自动装配的原理并不是非常复杂，其实背后的主要原理就是条件注解。  
&nbsp; &nbsp; 当我们使用@EnableAutoConfiguration注解激活自动装配时，实质对应着很多XXXAutoConfiguration类在执行装配工作，这些XXXAutoConfiguration类是在spring-boot-autoconfigure jar中的META-INF/spring.factories文件中配置好的，@EnableAutoConfiguration通过SpringFactoriesLoader机制创建XXXAutoConfiguration这些bean。XXXAutoConfiguration的bean会依次执行并判断是否需要创建对应的bean注入到Spring容器中，在注册过程中会通过ConfigurationProperties注解，去读取yml文件中所需要的固定前缀的配置。  
&nbsp; &nbsp; 在每个XXXAutoConfiguration类中，都会利用多种类型的条件注解@ConditionOnXXX对当前的应用环境做判断，如应用程序是否为Web应用、classpath路径上是否包含对应的类、Spring容器中是否已经包含了对应类型的bean。如果判断条件都成立，XXXAutoConfiguration就会认为需要向Spring容器中注入这个bean，否则就忽略。 
  

## 补充：基于aop的扩展机制 

&nbsp; &nbsp; springboot除了使用上述的这种扩展方式来集成很多第三方框架以外，还可以利用aop的方式来集成框架，比如常见的事务框架，线程池，Eureka等，只需要在配置类上面加@EnableTransactionManagement，@EnableAsync，@EnableEurekaServer启动注解，然后在对应方法上加上@Transactional，@Async注解即可使用（注意，事务框架比较特殊，它依赖于Aspectj，使用@EnableTransactionManagement注一定要加上@EnableAspectJAutoProxy）。 

&nbsp; &nbsp; 这种集成方式基于aop的方式扩展，在容器启动时，对切入点创建动态代理。拿@EnableAsync注解举例；@EnableAsync使用了这种方式，注解源码如下：  
&nbsp; ![](/images/posts/springboot/autoconfig14.png)        

&nbsp; &nbsp; EnableAsync注解中导入了AsyncConfigurationSelector，AsyncConfigurationSelector通过条件来选择需要导入的配置类，继承AdviceModeImportSelector又实现了ImportSelector接口，接口重写selectImports方法进行事先条件判断PROXY或者ASPECTJ选择不同的配置类。   
&nbsp; ![](/images/posts/springboot/autoconfig15.png)      

参考链接：[https://blog.csdn.net/m0_37595954/article/details/102720626](https://blog.csdn.net/m0_37595954/article/details/102720626 "神奇的@Enable*注解")