# Spring用到的设计模式

Spring框架中广泛运用了多种设计模式，以实现其强大的功能和灵活性。

以下是一些Spring框架中常见的设计模式：

### 1. 工厂模式（Factory Pattern）

- • **用途**：Spring通过BeanFactory和ApplicationContext等接口创建并管理对象实例。这种方式将对象的创建与使用解耦，使得程序更加灵活和可扩展。
- • **实现**：BeanFactory是一个工厂接口，提供了获取Bean对象的方法（如getBean(String name)）。ApplicationContext是BeanFactory的子接口，提供了更多高级功能，如国际化、事件传播和自动Bean注入等。

### 2. 单例模式（Singleton Pattern）

- • **用途**：Spring框架中的Bean默认是单例的，即在容器中只有一个实例。这种单例模式的设计有助于节省资源并提高性能。
- • **实现**：当Spring容器启动时，会为每个作用域为singleton的Bean创建并维护一个单实例对象，这些对象会被存储在一个缓存中，从而确保每次注入时都是同一个实例。

### 3. 代理模式（Proxy Pattern）

- • **用途**：Spring的AOP（面向切面编程）功能大量使用了代理模式。AOP通过在目标方法执行前后添加额外的行为（如日志、事务管理等），而这些额外的行为是通过代理对象来实现的。
- • **实现**：Spring提供了两种代理方式：JDK动态代理和CGLIB代理。如果目标类实现了一个接口，Spring默认使用JDK动态代理；如果没有实现接口，则使用CGLIB来生成子类代理。

### 4. 模板方法模式（Template Method Pattern）

- • **用途**：用于定义一个操作的算法骨架，将一些步骤推迟到子类中去实现。Spring框架在JDBC、Hibernate、JPA、事务管理等模块中都使用了模板方法模式。
- • **实现**：例如，JdbcTemplate类使用模板方法模式来封装与数据库交互的步骤。开发者只需提供SQL语句和参数，而不必关心资源获取、异常处理和资源释放等细节。

### 5. 观察者模式（Observer Pattern）

- • **用途**：Spring框架中的事件驱动机制使用了观察者模式。通过观察者模式，Spring框架实现了事件的发布和订阅机制，使得组件之间可以更加灵活地进行通信和协作。
- • **实现**：ApplicationEventPublisher是一个事件发布者，ApplicationListener是一个事件监听器。Spring容器允许多个监听器订阅和监听特定类型的事件，当事件发生时，所有订阅的监听器都会收到通知。

### 6. 策略模式（Strategy Pattern）

- • **用途**：用于定义一系列算法，将每一个算法封装起来，并让它们可以相互替换。策略模式让算法独立于使用它的客户而变化，提高了代码的可拓展性，降低了耦合度。
- • **实现**：Spring框架在很多地方使用策略模式，例如在事务管理中使用不同的事务管理策略（如JDBC、JTA），在视图解析器（ViewResolver）中使用不同的视图解析策略（如JSP、Thymeleaf）。

### 7. 适配器模式（Adapter Pattern）

- • **用途**：用于将一个接口转换为客户希望的另一个接口。Spring中的适配器模式在AOP和MVC框架中都有体现。
- • **实现**：在Spring MVC中，HandlerAdapter用于将不同的处理器（Handler）适配为统一的接口。例如，HttpRequestHandlerAdapter和SimpleControllerHandlerAdapter分别适配HttpRequestHandler和Controller，使得Spring MVC可以支持多种不同类型的控制器。

### 8. 装饰器模式（Decorator Pattern）

- • **用途**：用于动态地为对象添加行为而不改变其结构。Spring使用装饰器模式来增强Bean的功能。
- • **实现**：在Spring AOP中，代理对象（代理类）就是对目标对象的增强（装饰），可以动态地为目标对象添加新的行为（如方法拦截、日志记录、事务管理等）。

### 9. 依赖注入和控制反转（Dependency Injection and Inversion of Control）

- • **用途**：依赖注入（DI）模式用于将对象的依赖关系从内部转移到外部，使得对象更加解耦并且易于测试。控制反转（IoC）则是指将对象的创建和依赖关系的管理交给Spring容器，从而实现了对象的解耦。
- • **实现**：Spring通过XML配置、注解或Java配置类等方式，自动为Bean提供它们所需的依赖项。依赖注入主要有两种方式：构造函数注入和setter方法注入。