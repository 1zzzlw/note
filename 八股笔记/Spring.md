Spring 框架一般指 Spring Framework，由多个模块组合而成，核心特性是 IoC（控制反转）与 AOP（面向切面编程）。

Spring 为简化开发而生，让程序员只需要关注核心业务实现，尽可能不再关注非核心业务逻辑的代码（eg：事务管理、安全日志等）。

Spring 的 7 大模块：
1. Spring Core Container：整个 Spring 框架的基石，提供依赖注入（DI）与控制反转（IoC）功能，管理所有 Bean 的生命周期。
2. Spring AOP：提供面向切面编程支持，允许将横切关注点模块化（事务、日志、安全等），通过动态代理织入核心业务。
3. Spring Context：核心容器基础之上提供企业级服务，如国际化、事件传播、数据验证、定时调度等，是 BeanFactory 的增强版。
4. Spring Data Access：提供统一的数据管理方案，包括 JDBC 抽象、事务管理、ORM 框架集成，简化数据开发。
5. Spring Web：Web 开发的公共基础，提供文件上传、HTTP 请求处理功能，是 MVC 和 WebFlux 共享的基础模块。
6. Spring Web MVC：基于 Servlet API 的同步阻塞时 MVC 框架，采用经典的前端控制器（MVC）模式，适合传统 Web 应用的开发。
7. Spring WebFlux：全新的异步非阻塞式 MVC 框架，支持==🟠响应式流背压==，适合高并发、低延迟的实时应用。

Spring 框架的特点：
1. 

IoC（控制反转 Inversion of Control）：是面向对象编程中的一种设计思想，可以降低代码之间的耦合度，符合依赖倒置原则（DIP）。
控制反转的核心：==🔴将对象的创建权交出去，将对象与对象之间关系的管理权交出去，由第三方容器（Spring）来负责管理和维护。==
控制反转常见的实现方式：依赖注入（Dependence Injection，简称 DI）
依赖注入的实现又包含三种方式：
1. set 方法注入
2. 构造方法注入
3. 字段注入（直接给 Field 赋值：通过反射机制。）