# Spring Framework Core

IoC Container, Events, Resources, i18n, Validation, Data Binding, Type Conversion, SpEL, AOP.

# Spring设计理念

当学习一个框架时，重要的是不仅要知道它做什么，还要知道它遵循什么原则。下面是Spring框架的指导原则：

- 为每个层次提供选择。Spring允许您尽可能延迟设计决策。例如，您可以通过配置切换持久性提供程序，而无需更改代码。对于许多其他基础设施以及与第三方api的集成，也是如此。
- 容纳不同的观点。Spring崇尚灵活性，对于应该如何做事情并不固执己见。它支持各种不同角度的应用程序需求。
- 保持强大的向后兼容性。Spring的发展经过了精心的管理，在版本之间很少有破坏性的变化。Spring支持精心选择的JDK版本和第三方库，以促进依赖于Spring的应用程序和库的维护。
- 关心API设计。Spring团队投入了大量的思想和时间来制作直观的api，这些api可以跨多个版本和多年使用。
- 为代码质量设定高标准。Spring框架非常强调有意义的、当前的和准确的javadoc。它是极少数可以宣称代码结构干净、包之间没有循环依赖关系的项目之一。

# Spring IoC容器

## Spring IoC容器和bean的介绍

​	IoC（控制反转）也被成为依赖注入（DI：dependency injection）。它是一个过程，对象仅通过构造函数参数、工厂方法的参数、在对象实例被构造或从工厂方法返回后在其上设置的属性来定义它们的依赖项(即它们使用的其他对象)。容器在创建bean时注入这些依赖项。这个过程基本上是bean本身的反向(因此得名，控制反转)，它通过直接构造类或类似服务定位器模式的机制来控制其依赖项的实例化或位置。

​	***org.springframework.beans*** 和 ***org.springframework.context***  包是Spring框架IoC容器的基础。BeanFactory接口提供了能够管理任何类型对象的高级配置机制。ApplicationContext是BeanFactory的子接口。它添加了：

- 更容易与Spring's AOP特性集成；
- 消息资源处理(用于国际化)；
- 事件发布；
- 应用程序层特定的上下文，如web应用程序中使用的WebApplicationContext；

简而言之，BeanFactory提供了配置框架和基本功能，而ApplicationContext添加了更多特定于企业级的功能。ApplicationContext是BeanFactory的一个完整超集，在本章Spring's IoC容器的描述中专门使用。有关使用BeanFactory而不是ApplicationContext的更多信息，请参见BeanFactory的API文档。

在Spring中，构成应用程序主干并由Spring IoC容器管理的对象称为bean。bean是Spring IoC容器实例化、组装和管理的对象。否则，bean只是应用程序中许多对象中的一个。bean及其之间的依赖关系反映在容器使用的配置元数据中。

## 容器概述

​	***org.springframework.context.ApplicationContext*** 接口表示Spring IoC容器，并负责实例化、配置和组装bean。容器通过读取配置元数据获得关于要实例化、配置和组装哪些对象的指令。配置元数据用XML、Java注释或Java代码表示。它允许您表达组成应用程序的对象以及这些对象之间丰富的相互依赖关系。

​	Spring提供了ApplicationContext接口的几个实现。在独立应用程序中，通常创建***ClassPathXmlApplicationContext*** 或 ***FileSystemXmlApplicationContext*** 的实例。虽然XML是定义配置元数据的传统格式，但是您可以通过提供少量XML配置以声明方式启用对这些附加元数据格式的支持，来指示容器使用Java注释或代码作为元数据格式。

​	在大多数应用程序场景中，不需要显式的用户代码来实例化Spring IoC容器的一个或多个实例。例如，在web应用程序场景中，应用程序的web. XML文件中简单的8行样板web描述符XML通常就足够了(请参阅web应用程序的简便ApplicationContext实例化)。

​	如果你使用 Spring Tools for Eclipse(一个eclipse支持的开发环境)，只需单击几下鼠标或按键，就可以轻松创建这个样板配置。

​	下图显示了Spring如何工作的高级视图。您的应用程序类与配置元数据相结合，这样，在创建并初始化ApplicationContext之后，您就有了一个完全配置和可执行的系统或应用程序。

![container magic](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/images/container-magic.png)



### 配置元数据

​	如上图所示，Spring IoC容器使用一种形式的配置元数据。此配置元数据表示您作为应用程序开发人员如何告诉Spring容器在应用程序中实例化、配置和组装对象。

​	配置元数据传统上以简单直观的XML格式提供，本章的大部分内容都使用它来传达Spring IoC容器的关键概念和特性。

```
基于xml的元数据并不是唯一允许的配置元数据形式。Spring IoC容器本身与实际编写配置元数据的格式完全解耦。现在，许多开发人员为他们的Spring应用程序选择基于java的配置。
```

有关在Spring容器中使用其他形式的元数据的信息，如下所示：

- 基于注释的配置：Spring 2.5引入了对基于注释的配置元数据的支持；
- 基于Java的配置：从Spring 3.0开始，Spring JavaConfig项目提供的许多特性成为了核心Spring框架的一部分。因此，您可以使用 Java 而不是XML文件来定义应用程序类的外部bean。要使用这些新特性，请参阅@Configuration、@Bean、@Import和@DependsOn注释。

​	Spring配置由容器必须管理的至少一个(通常是多个)bean定义组成。基于XML配置元数据需要在标签beans中添加bean标签来配置这些beans。Java配置通常在@Configuration类中使用@ bean注释的方法。

​	这些bean定义对应于组成应用程序的实际对象。通常是你定义的服务层对象、数据访问对象（DAOs）、Struts的Action实例的表示层对象、Hibernate的SessionFactories基础层对象、JMS的Queues等等。通常，不需要在容器中配置细粒度的域对象，因为创建和加载域对象通常是DAOs和业务逻辑的职责。但是，您可以使用Spring与AspectJ的集成来配置在IoC容器控制之外创建的对象。参见[用Spring使用AspectJ进行依赖注入域对象。](https://docs.spring.io/spring/docs/5.2.7.RELEASE/spring-framework-reference/core.html#aop-atconfigurable).

下面的示例展示了基于xml的配置元数据的基本结构：

```xml


<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">  
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>
```

- id属性是标识单个bean定义的字符串。
- class属性定义bean的类型，并使用全限定的类名。

id属性的值引用协作对象。此示例中没有显示用于引用协作对象的XML。

### 实例化一个容器

​	提供给ApplicationContext构造函数的位置路径或路径是资源字符串，允许容器从各种外部资源加载配置元数据，例如本地文件系统、Java类路径等等。

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

（Spring提供一种可以从URI语法中定义的位置读取 InputStream的机制，详情看 ***Resources*** 章节。）

下面的示例列出了服务层对象 services.xml 配置文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->
    <bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
        <property name="accountDao" ref="accountDao"/>
        <property name="itemDao" ref="itemDao"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->

</beans>
```

下面的示例列出了数据访问层对象 dao .xml 文件：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountDao"
        class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>
```

​	在前面的示例中，服务层由 PetStoreServiceImpl 类和类型为 JpaAccountDao 和 JpaItemDao(基于JPA对象关系映射标准)的两个数据访问对象组成。property name元素引用 JavaBean 属性的名称（自身bean的属性），ref 元素引用另一个bean定义的名称（另一个bean的id）。id和ref元素之间的链接表示了协作对象之间的依赖关系（有关配置对象依赖关系的详细信息，见 ***依赖*** 小节）。

#### 组合基于xml的配置元数据

​	让bean定义跨越多个XML文件可能很有用。通常，每个单独的XML配置文件表示体系结构中的一个逻辑层或模块。

​	可以使用应用程序上下文构造函数从所有这些XML片段加载bean定义。此构造函数接受多个资源位置，如上一节所示。另外，使用一个或多个 <import/.> 标签从其他文件中加载bean。下面的示例展示了如何做到这一点：

```xml
<beans>
    <import resource="services.xml"/>
    <import resource="resources/messageSource.xml"/>
    <import resource="/resources/themeSource.xml"/>

    <bean id="bean1" class="..."/>
    <bean id="bean2" class="..."/>
</beans>
```

​	在前面的示例中，外部bean定义是从三个文件加载的：service.xml、messageSource.xml和themeSource.xml。所有位置路径都相对于执行导入的定义文件，因此services.xml必须与执行导入的文件位于相同的目录或类路径位置，而messageSource.xml和themeSource.xml必须位于导入文件位置下方的资源位置。如上示例所示，斜杠是可以被省略的。然而，考虑到这些路径是相对的，最好不要使用斜线。根据Spring模式，被导入的文件的内容，包含最高级<beans/.>标签，必须是有效的XML bean定义。

（可以使用相对路径引用父目录中的文件，但不建议使用 “../” 相对路径表示。这样做会在当前应用程序之外的文件上创建一个依赖项。特别地，引用文件方式不推荐使用 **classpath:** URLs的方式                                                   （ 例如classpath: ../services.xml），这种方式在运行时解析过程会选择最近（”nearest“）的类路径根，然后查看它的父目录。因为类路径配置更改可能导致选择不同的、不正确的目录。

您总是可以使用全限定的资源位置，而不是相对路径：例如，**file:C:/config/services.xm**l 或**classpath:/config/services.xml** 。但是，请注意您将应用程序配置耦合到特定的绝对位置。通常最好为这样的绝对位置保留一个间接关系，例如，通过在运行时根据JVM系统属性解析的“${}”占位符。）

​	名称空间本身提供导入指令特性。除了普通bean定义之外，Spring提供的XML名称空间中还有其他配置特性，例如context和util名称空间。

#### Groovy Bean定义DSL

（DSL：领域特定语言）

​	作为外部化配置元数据的进一步示例，bean定义也可以用Spring‘s Groovy bean定义DSL表示，正如Grails框架所知的那样。通常地，这样的配置在“.groovy”文件的结构如下所示：

```groovy
beans {
    dataSource(BasicDataSource) {
        driverClassName = "org.hsqldb.jdbcDriver"
        url = "jdbc:hsqldb:mem:grailsDB"
        username = "sa"
        password = ""
        settings = [mynew:"setting"]
    }
    sessionFactory(SessionFactory) {
        dataSource = dataSource
    }
    myService(MyService) {
        nestedBean = { AnotherBean bean ->
            dataSource = dataSource
        }
    }
}
```

这种配置风格在很大程度上等同于XML bean定义，甚至支持Spring’s XML配置名称空间。它还允许通过importBeans指令导入XML bean定义文件。

### 使用容器

​	**ApplicationContext**是一个高级工厂的接口，该工厂能够维护不同bean及其依赖项的注册表。通过使用        方法T getBean(String name，class<T> requiredType)，可以获得bean的实例。

​	**ApplicationContext**允许您读取和访问bean定义，如下面的示例所示：

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

​	使用Groovy配置，bootstrapping看起来非常类似。它有一个不同的上下文实现类，支持groovy(但也理解XML bean定义)。下面的示例展示了Groovy配置：

```java
ApplicationContext context = new GenericGroovyApplicationContext("services.groovy", "daos.groovy");
```

​	最灵活的变形是结合使用reader代理的**GenericApplicationContext**，例如使用**XmlBeanDefinitionReader**读取XML文件的配置，如下面的示例所示：

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();
```

​	你还可以对Groovy文件使用**GroovyBeanDefinitionReader**读取groovy文件的配置，如下面的示例所示：

```java
GenericApplicationContext context = new GenericApplicationContext();
new GroovyBeanDefinitionReader(context).loadBeanDefinitions("services.groovy", "daos.groovy");
context.refresh();
```

​	你可以在相同的ApplicationContext上混合和匹配这些阅读器委托，从不同的配置源读取bean定义。如上就是使用同一个**GenericApplicationContext**去读取文件配置，将两个示例结合就可以达到混合的方式。

​	然后可以使用getBean获取bean的实例。ApplicationContext接口有一些用于获取bean的其他方法，但理想情况下，你的应用程序代码不应该使用它们。实际上，你的应用程序代码根本不应该调用getBean()方法，因此根本不依赖于Spring APIs。例如，Spring与web框架的集成为各种web框架组件（如控制器和 JSF-managed beans ）提供了依赖注入，允许您通过元数据声明对特定bean的依赖(如自动装配注释)。

## Bean概述

​	Spring IoC容器管理一个或多个bean。这些bean是使用你提供给容器的配置元数据创建的（例如，在XML文件中的<bean/.>标签定义的bean）。

​	在容器本身内，这些bean定义表示为BeanDefinition对象，其中包含（以及其他信息）以下元数据：

- 一个包限定的类名:通常是定义的bean的实际实现类。
- Bean行为配置元素，它声明Bean在容器中应该如何运行(作用域、生命周期回调，等等)。
- 对bean执行其工作所需的其他bean的引用。这些引用也称为协作者（collaborators）或依赖项。
- 在新创建的对象中设置的其他配置设置，例如池的大小限制或在管理连接池的bean中使用的连接数量。

该元数据转换为组成每个bean定义的一组属性。下表描述了这些属性：

| 属性                     | Explained in…                                    |
| ------------------------ | ------------------------------------------------ |
| Class                    | 见 ***实例化Bean*** 章节                         |
| Name                     | 见 ***命名bean*** 章节                           |
| Scope                    | 见 ***bean作用域*** 章节                         |
| Constructor arguments    | 见 ***依赖注入*** 小节                           |
| Properties               | 见 ***依赖注入*** 小节                           |
| Autowiring mode          | 见 ***自动装配协作者***  小节                    |
| Lazy initialization mode | 见 ***懒初始化Bean*** 小节                       |
| Initialization method    | 见 ***初始化回调函数*** （回调函数声明周期）小节 |
| Destruction method       | 见 ***析构回调函数*** （回调函数声明周期）小节   |

​	除了包含关于如何创建特定bean的信息的 **bean definitions** 之外，**ApplicationContext **实现还允许注册在容器外部(由用户)创建的现有对象。这是通过 **getBeanFactory()** 方法访问 **ApplicationContext** 的 BeanFactory 来完成的，该方法返回 BeanFactory 的实现 **DefaultListableBeanFactory**。**DefaultListableBeanFactory** 通过 **registerSingleton(..) **和 **registerBeanDefinition(..) **方法支持这种注册。但是，典型的应用程序仅使用通过常规bean定义元数据定义的bean。

（Bean元数据和手动提供的单例实例需要尽早注册，以便容器在自动装配和其他自省步骤中正确地对它们进行推理。虽然在某种程度上支持覆盖现有元数据和现有的单例实例，但在运行时注册新bean(与对工厂的实时访问同时进行)不受官方支持，可能会导致并发访问异常、bean容器中的不一致状态，或者两者都有。）

### Beans命名

​	每个bean都有一个或多个标识符。这些标识符在宿主bean的容器中必须是唯一的。bean通常只有一个标识符。但是，如果需要多个别名，那么额外的别名可以被视为别名。

​	在基于xml的配置元数据中，可以使用id属性、name属性或两者来指定bean标识符。id属性只允许指定一个id。按照惯例，这些名称是字母数字('myBean'、'someService'等)，但它们也可以包含特殊字符。如果希望为bean引入其他别名，还可以在name属性中指定它们，用逗号(，)、分号(;)或空格分隔。作为一个历史记录，在Spring  3.1之前的版本中，id属性被定义为xsd: id类型，它限制了可能的字符。在3.1中，它被定义为xsd:string类型。注意，bean  id唯一性仍然由容器强制执行，但XML解析器不再强制执行。

​	你不一定需要为bean提供名称或id。如果不显式地提供名称或id，容器将为该bean生成惟一的名称。但是，如果你希望通过使用ref元素或服务定位器样式查找通过名称引用该bean，则必须提供一个名称。不提供名称的动机与使用内部bean和自动装配协作者有关。