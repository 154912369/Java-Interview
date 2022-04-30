## Spring
### 什么是Spring
Spring是个包含一系列功能的合集，如快速开发的Spring Boot，支持微服务的Spring Cloud，支持认证与鉴权的Spring Security，Web框架Spring MVC。IOC与AOP依然是核心。

### Spring MVC流程
过滤器是在tomcat那一层处理的,过滤器的doFilter函数内部执行所有DispatcherServlet的过程。
1. 发送请求，进入过滤器。过滤器中doFilter函数会将请求交给jetty（或者tomact）。
    doFilter函数完成下面的2-5步骤。
2.jetty（或者tomact）将请求交给DispatcherServlet。
     DispatcherServlet继承自javax.servlet.Servlet是处理web的函数。
3.DispatcherServlet拦截器拿到交给HandlerMapping
    交给HandlerMapping的函数是doDispatch，目前看到七种HandlerMapping，普通的get请求用的是RequestMappingHandlerMapping。
4. 依次调用配置的拦截器，最后找到配置好的业务代码Handler并执行业务方法
    拦截器和业务代码都是保存在HandlerExecutionChain，HandlerExecutionChain是HandlerMapping给出的,都是spring中的概念。
5. 包装成ModelAndView返回给ViewResolver解析器渲染页面
6.过滤器处理完毕。

### 解决循环依赖
无参数构造器、字段注入

### Bean的创造周期
#### 函数
- doCreateBean:
    - createBeanInstance：根据工厂方法或者构造方法等创造对象。
    - addSingletonFactory:添加到singletonFactories(这是第三级缓存)
    - populateBean：填充属性,就是填充@Autowire里面。
    - initializeBean:初始化成为实例
        - invokeAwareMethods:调用bean继承的Aware接口相关方法。
        - applyBeanPostProcessorsBeforeInitialization:调用postProcessBeforeInitialization方法。
        - invokeInitMethods:用接口InitializingBean中的afterPropertiesSet方法。
        - applyBeanPostProcessorsBeforeInitialization：调用postProcessAfterInitialization方法,aop就是这里进行织入的,利用的是AbstractAutoProxyCreator类。
#### 过程描述
* 1.SpringApplication中含有一个ApplicationContext，然后ApplicationContext含有一个beanfactory,beanfactory里面的doGetBean创建bean。
* 2.createBeanInstance中根据工厂方法或者构造方法等创造实例。
* 3.populateBean中填充属性,就是填充@Autowire里面。
* 4.initializeBean中实现Aware接口的内容,例如BeanFactoryAware（获取beanfactory）,EnvironmentAware（获取环境信息）。
* 5.initializeBean中的applyBeanPostProcessorsBeforeInitialization调用所有BeanPostProcessor继承类中postProcessBeforeInitialization方法。
* 6.initializeBean中的invokeInitMethods中调用接口InitializingBean中的afterPropertiesSet方法。
* 7.initializeBean中的applyBeanPostProcessorsBeforeInitialization调用所有BeanPostProcessor继承类中postProcessAfterInitialization方法。aop就是这里进行织入的,利用的是AbstractAutoProxyCreator类。

### Bean的生命周期

* 1. Spring对Bean进行实例化
* 2. Spring将值和Bean的引用注入进Bean对应的属性中
* 3. 容器通过Aware接口把容器信息注入Bean
* 4. BeanPostProcessor。进行进一步的构造，会在InitialzationBean前后执行对应方法，当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理
* 5. InitializingBean。这一阶段也可以在bean正式构造完成前增加我们自定义的逻辑，但它与前置处理不同，由于该函数并不会把当前bean对象传进来，因此在这一步没办法处理对象本身，只能增加一些额外的逻辑。
* 6. DisposableBean。Bean将一直驻留在应用上下文中给应用使用，直到应用上下文被销毁，如果Bean实现了接口，Spring将调用它的destory方法

### Bean的作用域

* singleton：单例模式，Spring IoC容器中只会存在一个共享的Bean实例，无论有多少个Bean引用它，始终指向同一对象。
* prototype：原型模式，每次通过Spring容器获取prototype定义的bean时，容器都将创建一个新的Bean实例，每个Bean实例都有自己的属性和状态。
* request：在一次Http请求中，容器会返回该Bean的同一实例。而对不同的Http请求则会产生新的Bean，而且该bean仅在当前Http Request内有效。
* session：在一次Http Session中，容器会返回该Bean的同一实例。而对不同的Session请求则会创建新的实例，该bean实例仅在当前Session内有效。
* global Session：在一个全局的Http Session中，容器会返回该Bean的同一个实例，仅在使用portlet context时有效。

### 三级缓存
DefaultSingletonBeanRegistry(包含三级缓存的类，以及put的函数)<-FactoryBeanRegistrySupport<-AbstractBeanFactory<- AbstractAutowireCapableBeanFactory<-DefaultListableBeanFactory;
* 第一级：Map<String, Object> singletonObjects,存放完成初始化的bean。存放的代码在addSingleton(String beanName, Object singletonObject) 。
* 第二级：Map<String, ObjectFactory<?>> earlySingletonObjects,存放完成实例化的且会循环引用的bean。循环引用的时候，从里面拿未初始化的实例。
* 第三级：Map<String, ObjectFactory<?>> singletonFactories ，存放创造bean的工厂。存放的代码在 addSingletonFactory。

#### 例子
如果A，B循环依赖，则进入下面的过程：
* 1.开始docreateBean利用创建A。
* 2.docreateBean中会将createBeanInstance后未填充属性的A放在singletonFactories。
* 3.填充A的属性，发现B未创建，开始创建B。
* 4.docreateBean中会将createBeanInstance后未填充属性的B放在singletonFactories。
* 5.B发现循环依赖于A,将A从singletonFactories中移除，放到earlySingletonObjects，将A注入B中。
* 6.完成B的初始化，将B放入singletonObjects。
* 7.完成A的初始化，将A放入singletonObjects，将A从earlySingletonObjects中移除。

#### 具体实现
getSingleton有两种方法：
* 1.这里面会做出docreateBean的动作:Object getSingleton(String beanName, ObjectFactory<?> singletonFactory)
* 2.这里面不会做出docreateBean的动作:只是获取singletonFactories中的bean，然后将结果放到earlySingletonObjects中:Object getSingleton(String beanName, boolean allowEarlyReference)。

循环引用解除的方法:
* docreateBean->getSingleton(第一种)->docreateBean->getSingleton(第二种)，不存在循环了。





### IOC（DI）
* 控制反转

由 Spring IOC 容器来负责对象的生命周期和对象之间的关系。IoC 容器控制对象的创建，依赖对象的获取被反转了  
没有 IoC 的时候我们都是在自己对象中主动去创建被依赖的对象，这是正转。但是有了 IoC 后，所依赖的对象直接由 IoC 容器创建后注入到被注入的对象中，依赖的对象由原来的主动获取变成被动接受，所以是反转  

* 依赖注入

组件之间依赖关系由容器在运行期决定，由容器动态的将某个依赖关系注入到组件之中，提升组件重用的频率、灵活、可扩展  
通过依赖注入机制，我们只需要通过简单的配置，而无需任何代码就可指定目标需要的资源，完成自身的业务逻辑，而不需要关心具体的资源来自何处，由谁实现  

注入方式：构造器注入、setter 方法注入、接口方式注入

### Spring AOP
#### 介绍
面向切面的编程，是一种编程技术，是OOP（面向对象编程）的补充和完善。OOP的执行是一种从上往下的流程，并没有从左到右的关系。因此在OOP编程中，会有大量的重复代码。而AOP则是将这些与业务无关的重复代码抽取出来，然后再嵌入到业务代码当中。常见的应用有：权限管理、日志、事务管理等。

#### 实现方式
实现AOP的技术，主要分为两大类：一是采用动态代理技术，利用截取消息的方式，对该消息进行装饰，以取代原有对象行为的执行；二是采用静态织入的方式，引入特定的语法创建“方面”，从而使得编译器可以在编译期间织入有关“方面”的代码。Spring AOP实现用的是动态代理的方式。

#### Spring AOP使用的动态代理原理
jdk反射：通过反射机制生成代理类的字节码文件，调用具体方法前调用InvokeHandler来处理  
cglib工具：利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理

1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换


