---
title: Spring揭秘
tags: spring
grammar_cjkRuby: true
---

## Spring中的设计模式

不同的AopPorxy实现的实例化过程采用工厂模式（抽象工厂模式）进行封装 ——p185

策略模式——创建bean对象 p83

观察者模式——事件发布

## 第2章 IoC的基础概念

### 理念：让别人为你服务

Inversion of Control 控制反转

好莱坞原则“Don't call us, we will call you”

原来是需要什么对象自己去创建，现在需要什么对象IoC容器会送过来，更加轻松简洁

### 粘合剂

控制反转（Inversion of Control）是一种是面向对象编程中的一种设计原则，用来减低计算机代码之间的耦合度。其基本思想是：借助于“第三方”实现具有依赖关系的对象之间的解耦。

由于引进了中间位置的“第三方”，也就是IOC容器，使得A、B、C、D这4个对象没有了耦合关系，齿轮之间的传动全部依靠“第三方”了，全部对象的控制权全部上缴给“第三方”IOC容器，所以，IOC容器成了整个系统的关键核心，它起到了一种类似“粘合剂”的作用，把系统中的所有对象粘合在一起发挥作用，如果没有这个“粘合剂”，对象与对象之间会彼此失去联系，这就是有人把IOC容器比喻成“粘合剂”的由来。

### 好处

* 解耦
* 不会对业务对象构成很强的侵入性
* 对象具有更好的可测试性，可重用性，可扩展性
* 使用方便

### 结论

IoC是一种可以帮助我们解耦业务对象间依赖关系的对象绑定方式

## 第4章 IoC容器之BeanFactory

### BeanFactory与ApplicationContext

#### BeanFactory

基础类型IoC容器,提供完整的IoC服务支持。如果没有特殊指定,默认采用延迟初始化策略(lazy-load)。
只有当客户端对象需要访问容器中的某个受管对象的时候,才对 该受管对象进行初始化以及依赖注入操作。
所以,相对来说,容器启动初期速度较快,所需 要的资源有限。对于资源有限,并且功能要求不是很严格的场景,BeanFactory是比较合适的 IoC容器选择。

#### ApplicatonContext

ApplicationContext在BeanFactory的基础上构建,是相对比较高级的容器实现。
除了拥有BeanFactory的所有支持,ApplicationContext还提供了其他高级特性,比如事件发布、国际化信息支持等。
ApplicationContext所管理 的对象,在该类型容器启动之后,默认全部初始化并绑定完成。所以,相对于BeanFactory来说,ApplicationContext要求更多的系统资源,同时,因为在启动时就完成所有初始化,容器启动时间较之BeanFactory也会长一些。
在那些系统资源充足,并且要求更多功能的场景中,  ApplicationContext类型的容器是比较合适的选择。

### 实现

#### IoC容器功能实现两阶段

1. 容器启动阶段

容器启动伊始,首先会通过某种途径加载Configuration MetaData。除了代码方式比较直接,在大 部分情况下,容器需要依赖某些工具类(BeanDefinitionReader)对加载的Configuration MetaData进行解析和分析,并将分析后的信息编组为相应的BeanDefinition,最后把这些保存了bean定义必 要信息的BeanDefinition,注册到相应的BeanDefinitionRegistry,这样容器启动工作就完成了。

该阶段所做的工作可以认为是准备性的,重点更加侧重于对象管理信息的收集。当然, 一些验证性或者辅助性的工作也可以在这个阶段完成。

2. Bean实例化阶段

该阶段,容器会首先检查所请求的对象之前是否已经初始化。如果没有,则会根据注册的 BeanDefinition所提供的信息实例化被请求对象,并为其注入依赖。如果该对象实现了某些回调接口,也会根据回调接口的要求来装配它。当该对象装配完毕之后,容器会立即将其返回请求方使用。
如果说第一阶段只是根据图纸装配生产线的话,那么第二阶段就是使用装配好的生产线来生产具体的产品了。

#### 插手容器的启动 BeanFactoryPostProcessor

Spring提供了一种叫做BeanFactoryPostProcessor的容器扩展机制。该机制允许我们在容器实例化相应对象之前,对注册到容器的BeanDefinition所保存的信息做相应的修改。这就相当于在容 器实现的第一阶段最后加入一道工序,让我们对最终的BeanDefinition做一些额外的操作,比如修改其中bean定义的某些属性,为bean定义增加其他信息等。


#### Spring提供的BeanFactoryPostProcessor

**PropertyPlaceholderConfigurer**

PropertyPlaceholderConfigurer允许我们在XML配置文件中使用占位符(PlaceHolder), 并将这些占位符所代表的资源单独配置到简单的properties文件中来加载。

当BeanFactory在第一阶段加载完成所有配置信息时,BeanFac- tory中保存的对象的属性信息还只是以占位符的形式存在,如${jdbc.url}、${jdbc.driver}。当 PropertyPlaceholderConfigurer作为BeanFactoryPostProcessor被应用时,它会使用properties 配置文件中的配置信息来替换相应BeanDefinition中占位符所表示的属性值。这样,当进入容器实 现的第二阶段实例化bean时,bean定义中的属性值就是最终替换完成的了。

**PropertyOverrideConfigurer**

可以通过PropertyOverrideConfigurer对容器中配置的任何你想处理的bean定义的property信息进行覆盖替换。

**CustomerEditorConfigurer**

辅助性地将后期会用到的信息注册到容器,对BeanDefinition没有做任何变动。

XML所记载的,都是String类型,即容器从XML格式的文件中读取的都是字符串形式,最终应用程序却是由各种类型的对象所构成。要想完成这种由字符串到具体对象的转换(不管这个转换工作最终由谁来做),都需要这种转换规则相关的信息,而CustomEditorConfigurer就是帮助我们传达类似信息的。

Spring内部通过JavaBean的PropertyEditor来帮助进行String类型到其他类型的转换工作。只要 为每种对象类型提供一个PropertyEditor,就可以根据该对象类型取得与其相对应的PropertyEditor来做具体的类型转换。

#### bean的一生

**Bean的实例化过程**

![Bean的实例化过程][1]

容器在内部实现的时候,采用“策略模式(Strategy Pattern)”来决定采用何种方式初始化bean实例。 通常,可以通过反射或者CGLIB动态字节码生成来初始化相应的bean实例或者动态生成其子类。

org.springframework.beans.factory.support.InstantiationStrategy定义是实例化策略的抽象接口,其直接子类SimpleInstantiationStrategy实现了简单的对象实例化功能,可以通过反射来实例化对象实例,但不支持方法注入方式的对象实例化。CglibSubclassingInstantiation-Strategy继承了SimpleInstantiationStrategy的以反射方式实例化对象的功能,并且通过CGLIB 的动态字节码生成功能,该策略实现类可以动态生成某个类的子类,进而满足了**方法注入**所需的对象实例化需求。默认情况下,容器内部采用的是CglibSubclassingInstantiationStrategy。

```
<bean id="newsBean" class="..domain.FXNewsBean" singleton="false"> </bean>
<bean id="mockPersister" class="..impl.MockNewsPersister">
<lookup-method name="getNewsBean" bean="newsBean"/>
</bean>
```

通过<lookup-method>的name属性指定需要注入的方法名,bean属性指定需要注入的对象,当getNewsBean方法被调用的时候,容器可以每次返回一个新的类型的实例。

关于方法注入，pdf见69页,书见60页

**Aware接口**

当对象实例化完成并且相关属性以及依赖设置完成之后,Spring容器会检查当前对象实例是否实现了一系列的以Aware命名结尾的接口定义。如果是,则将这些Aware接口定义中规定的依赖注入给当前对象实例。

BeanFactory相关aware

* org.springframework.beans.factory.BeanNameAware
* org.springframework.beans.factory.BeanClassLoaderAware
* org.springframework.beans.factory.BeanFactoryAware

ApplicationContext相关Aware

* org.springframework.context.ResourceLoaderAware
* org.springframework.context.ApplicationEventPublisherAware
* org.springframework.context.MessageSourceAware
* org.springframework.context.ApplicationContextAware

**BeanPostProcessor**

通常比较常见的使用BeanPostProcessor的场景,是处理标记接口实现类,或者为当前对象提供代理实现。

ApplicationContext对应的那些Aware接口实际上就是通过Bean-PostProcessor的方式进行处理的。

替换当前对象实例或者字节码增强当前对象实例。Spring的AOP则更多地使用BeanPostProcessor来为对象生成相应的代理对象,如org.springframework.aop.framework. autoproxy.BeanNameAutoProxyCreator。

**InitializingBean和init-method**

```
public interface InitializingBean {
    void afterPropertiesSet() throws Exception;
}
```

在对象实例化过程调用过“BeanPostProcessor的前置处理”之后,会接着检测当前对象是否实现了InitializingBean接口,如果是,则会调用其afterProper- tiesSet()方法进一步调整对象实例的状态。

通过init-method,系统中业务对象的自定义初始化操作可以以任何方式命名,而不再受制于InitializingBean的afterPropertiesSet()。

**DisposableBean和destory-method**

适用对象：singleton类型的bean实例

## 第5章 IoC容器之ApplicationContext

ApplicationContext扩展了BeanFactory的功能，包括了容器启动后bean实例的自动初始化，容器内事件发布，国际化信息支持。

### 统一资源加载策略

#### Spring中的Resource

Spring框架内部使用org.springframework.core.io.Resource接口作为所有资源的抽象和访问接口

```
BeanFactory beanFactory = new XmlBeanFactory(newClassPathResource("..."));
```

实现类：

* ByteArrayResource
* ClassPathResource
* FileSystemResource
* UrlResource
* InputStreamResource

#### ResourceLoader

查找和定位资源

实现类：

* DefaultResourceLoader
* FileSystemResourceLoader

ResourcePatternResolver ——批量查找的ResourceLoader

![Resource和ResourceLoader类层次图][2]

#### ApplicationContext与ResourceLoader

![AbstractApplicationContext作为ResourceLoader和ResourcePatternResolver][3]

**扮演ResourceLoader的角色**

可以通过ApplicationContext来加载任何Spring支持的Resource类型。

**ResourcLoader类型的注入**

实现ResourceLoaderAware或者ApplicationContextAware接口，获取ResourceLoader

**Resource类型注入**

```
public class XMailer {
private Resource template;
}

<bean id="mailer" class="...XMailer">
    <property name="template" value="..resources.default_template.vm"/>
</bean>
```

### 容器内部事件发布

![Spring容器内部时间发布实现类图][4]

**ApplicationEvent**

* ContextClosedEvent:ApplicationContext容器在即将关闭的时候发布的事件类型。
* ContextRefreshedEvent:ApplicationContext容器在初始化或者刷新的时候发布的事件类型。
* RequestHandledEvent:Web请求处理后发布的事件,其有一子类ServletRequestHandledEvent提供特定于Java EE的Servlet相关事件。

**ApplicationListener**

**ApplicationContext ApplicationEventPublish**

ApplicationContext接口定义还继承了ApplicationEventPublisher接口,该接口提供了void publish-Event(ApplicationEventevent)方法定义。

## 第7章 一起来看AOP

软件系统中，日志记录、安全检查、事务管理等系统需求像一把刀横切到组织良好的各个业务功能模块之上。这些系统需求是系统中的横切关注点（cross-cutting concern）。使用传统方法，无法更好地以模块化形式对系统中的横切关注点进行封装。
Aspect之于AOP，相当于Class之于OOP。AOP仅是OOP方法的一种补足，当我们把以Class形式模块化的业务需求和以Aspect形式模块化的系统需求拼装到一起，整个系统就算完成了。

![加入各种系统需求后的系统模块关系示意图][5]

![AOP各个概念所处的场景][6]

## 第9章 Spring AOP一世

![PointCut局部族谱][7]

![Advice略图][8]

![Advisor分支][9]

![PointcutAdvisor及相关子类][10]

![IntroductionAdvisor类结构图][11]

![AopProxy相关结构图][12]

![ProxyFactory继承层次类图][13]

![ProxyFactory的兄弟][14]


  [1]: ./assets/life_of_bean.jpeg "life_of_bean.jpeg"
  [2]: ./assets/resource_loader.png "resource_loader.png"
  [3]: ./assets/abstractApplicationContext.png "abstractApplicationContext.png"
  [4]: ./assets/springevent.png "springevent.png"
  [5]: ./assets/aop.png "aop.png"
  [6]: ./assets/aop_comps.png "aop_comps.png"
  [7]: ./assets/Pointcut%E5%B1%80%E9%83%A8%E6%97%8F%E8%B0%B1.png "Pointcut局部族谱.png"
  [8]: ./assets/advice%E7%95%A5%E5%9B%BE.png "advice略图.png"
  [9]: ./assets/Advisor%E5%88%86%E6%94%AF.png "Advisor分支.png"
  [10]: ./assets/PointcutAdvisor.png "PointcutAdvisor.png"
  [11]: ./assets/IntroductionAdvisor%E7%B1%BB%E7%BB%93%E6%9E%84%E5%9B%BE.png "IntroductionAdvisor类结构图.png"
  [12]: ./assets/AopProxy.png "AopProxy.png"
  [13]: ./assets/ProxyFactory.png "ProxyFactory继承层次类图"
  [14]: ./assets/ProxyFactory%E5%85%84%E5%BC%9F.png "ProxyFactory兄弟.png"