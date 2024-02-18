##### Spring 和 Springboot区别

简单来说 Springboot是Spring的扩展版本, Spring 对api，ioc做了一些实现，解决了代码解耦的问题，springboot则是在spring基础上，把很多需要配置的选项，直接默认化，简化了配置项，更加方便了开发。这个也是springboot倡导的 约定大于配置。

##### Spring IOC  DI ？

IOC是控制反转，DI 是依赖注入，IOC是一种思想，DI是这种思想的实现手段。IOC思想的的本质就是 本身由 用户程序对Bean进行管理，但是现在反转使用框架来进行Bean的统一管理，好处

- 所有bean统一管理，降低了成本
- 不同对象直接解耦合，松散耦合

DI：即依赖注入：组件之间依赖关系由容器在运行期决定

对象之间的依赖关系 使用容器控制，并且注入。主要好处

- 提升了对象利用率
- 降低了代码复杂度，用户程序不再需要管理对象之间的依赖关系

IOC配置三种方式

- JAVA配置
- xml配置
- 注解配置

DI三种方式

- set方法
- 构造函数
- 注解

##### Spring AOP？

AOP是一种思想，面向切面编程。Spring把这种思想引入了框架，并且使用了预编译和运行时动态代理的两种技术手段来实现了该技术。

![](https://pdai.tech/images/spring/springframework/spring-framework-aop-4.png)

OOP面向对象编程，针对业务处理过程的实体及其属性和行为进行抽象封装，以获得更加清晰高效的逻辑单元划分。而AOP则是针对业务处理过程中的切面进行提取，它所面对的是处理过程的某个步骤或阶段，以获得逻辑过程的中各部分之间低耦合的隔离效果。这两种设计思想在目标上有着本质的差异。

![](https://pdai.tech/images/spring/springframework/spring-framework-aop-2.png)

![](https://pdai.tech/images/spring/springframework/spring-framework-aop-3.png)

##### Spring AOP和 AspectJ区别是什么 ？

aspectj 是java实现的一个AOP框架。也是实现AOP功能，主要实现的基于静态织入的方式。Spring AOP也是AOP实现的一种方式，主要是基于 动态织入，也就是运行时代码增强。动态增强还有比如 JDK动态代理，和 Cglib的动态代理。

##### Spring AOP实现原理？

AOP实现是基于IOC实现的，也就是在IOC初始化完毕的时候，最后有个扩展点，BeanPostProcessor 的后置方法里实现的。

1. 在调用bean的init方法之前执行before方法，核心就是遍历所有bean，针对@AspectJ注解的类，判断是否aop的类，然后获取切面方法，切面表达式，等等属性，最后封装成一系列的 advisor对象
2. 然后再bean的 after init方法以后，拿到所有对象，判断是否符合切面表达式，符合的对象就要创建对应的代理对象了
3. 通过配置注解 或者默认 使用jdk动态代理 或者 cglib 来实现 代理对象的创建
4. 在执行的时候，会调用到代理对象
5. 代理对象实现的逻辑，把所有符合条件的 增强方法按照顺序形成责任链来调用
6. 后续中间会插入 真实的bean 方法调用
7. 最后返回调用结果

##### JDK动态代理和Cglib动态代理区别？

jdk是通过接口实现来生成一个新的类，新的类和原始类是组合关系，然后加载到jvm里去做调用，底层是使用了反射来实现的。

cglib是使用了 asm字节码框架，生成新的字节码文件，继承原始类，重写目标方法，并且使用fastclass机制来优化调用，提高性能。

fastclass就是 cglib针对原有类的所有方法，方法名称和参数类型做了key索引，然后生成一个新的类，新方法，当有调用的时候就直接通过方法和参数类型 生成key，去获取指定方法直接调用，速度优于反射。

###### Java 反射效率低主要原因是：

1. Method#invoke 方法会对参数做封装和解封操作
2. 需要检查方法可见性
3. 需要校验参数
4. JIT 无法优化

##### Spring Bean实例化过程？

![CleanShot 2023-03-21 at 23.50.55@2x](/Users/jazz/Library/Application Support/CleanShot/media/media_Sec6Zz3iK5/CleanShot 2023-03-21 at 23.50.55@2x.png)

1. 实例化bean，通过反射的方式创建对象
2.  填充bean属性，依赖注入。populateBean()方法，循环依赖的问题（三级缓存）
3. 调用aware接口相关的方法，invokeAwareMethod（完成BeanName，BeanFactory，BeanClassloader对象属性的设置）。
4. 调用BeanProcessor的 前置方法，使用比较多的，ApplicationContextPostProcessor  设置ApplicationContext，Environment等
5. 调用initMethod方法，判断是否实现了 InitiallizingBean，有的话就调用 afterPropertiesSet方法。
6. 调用BeanPostProcessor 后置方法， aop功能就是在这里实现的，AbstractAutoProxyCreator
7. 获取到完整的对象，然后就是放到一个大map里，可以通过getBean的方式来获取了
8. 销毁流程：判断是否实现了 DisposableBean接口，2、调用自定义destroy方法

##### 循环依赖如何解决？

比如A依赖B，B依赖A，这个就是循环依赖

先说明对象的创建过程，实例化和初始化（属性填充）

1. 先创建A对象，初始化A对象后，A对象中b属性为空
2. 然后从容器中查找B对象，如果找到了直接赋值，那就不存在循环依赖的问题，找不到的话直接创建B对象
3. 实例化B对象，此时B对象中A对象也为空，需要填充属性a
4. 从容器里查找对象A，此时找不到，需要实例化A，此时形成依赖。

先来看下这三级缓存

```java
/** Cache of singleton objects: bean name --> bean instance */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
 
/** Cache of early singleton objects: bean name --> bean instance */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);

/** Cache of singleton factories: bean name --> ObjectFactory */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
```

- **第一层缓存（singletonObjects）**：单例对象缓存池，已经实例化并且属性赋值，这里的对象是**成熟对象**；
- **第二层缓存（earlySingletonObjects）**：单例对象缓存池，已经实例化但尚未属性赋值，这里的对象是**半成品对象**；
- **第三层缓存（singletonFactories）**: 单例工厂的缓存

在bean建立过程当中，有两处比较重要的匿名内部类实现了该接口。一处是Spring利用其建立bean的时候，另外一处就是:

```java
addSingletonFactory(beanName, new ObjectFactory<Object>() {
   @Override   public Object getObject() throws BeansException {
      return getEarlyBeanReference(beanName, mbd, bean);
   }});
```

此处就是解决循环依赖的关键，这段代码发生在createBeanInstance以后，也就是说单例对象此时已经被建立出来的。这个对象已经被生产出来了，虽然还不完美（尚未进行初始化的第二步和第三步），可是已经能被人认出来了（根据对象引用能定位到堆中的对象），因此Spring此时将这个对象提早曝光出来让你们认识，让你们使用。

整体循环依赖解决流程如下：

A对象依赖B对象，A实例化以后，就会直接暴露给singletonFactories Factory 其实也就是放入第三层缓存，此时进行初始化，发现A依赖B，则去getBean寻找B，发现找不到，此时则会直接创建B对象，创建完B对象则会进行B对象的初始化，发现B对象依赖A对象，进而尝试getBean（A），从一级缓存找不到，二级缓存也找不到，尝试三级缓存singletonFactories，因为A经过ObjectFactory将本身提早曝光了，此时B可以拿到未初始化完成的A对象，此时B对象就完成了初始化，放入了一级缓存中，而返回A对象，拿到B对象后也完成了初始化，进而放入到一级缓存中，因为B对象里有A的引用，此时A,B都完全完成了 对象的初始化流程。

顺便三级缓存存在的意义是 普通对象和代理对象是不可以 同时存在容器中的，所以当需要获取对象的时候 就要判断对象是否需要代理，然后从三级对象获取的过程中实现 对象的代理。



##### Spring为什么不能解决构造器的循环依赖？

构造器注入形成的循环依赖： 也就是beanB需要在beanA的构造函数中完成初始化，beanA也需要在beanB的构造函数中完成初始化，这种情况的结果就是两个bean都不能完成初始化，循环依赖难以解决。

Spring解决循环依赖主要是依赖三级缓存，但是的**在调用构造方法之前还未将其放入三级缓存之中**，因此后续的依赖调用构造方法的时候并不能从三级缓存中获取到依赖的Bean，因此不能解决。

##### Spring为什么不能解决prototype作用域循环依赖？

这种循环依赖同样无法解决，因为spring不会缓存‘prototype’作用域的bean，而spring中循环依赖的解决正是通过缓存来实现的。

##### Spring为什么不能解决多例的循环依赖？

多实例Bean是每次调用一次getBean都会执行一次构造方法并且给属性赋值，根本没有三级缓存，因此不能解决循环依赖。

##### 为什么一定要使用三级缓存来解决循环依赖

将缓存分为三级是否有必要，依据处理流程，二级缓存似乎也能很好解决循环依赖问题。查证之后发现许多人也有和我一样的疑问，但很难找到一个权威的解答，最终找到一篇博客给出了令我信服的说法：

使用三级缓存而非二级缓存并不是因为只有三级缓存才能解决循环引用问题，其实二级缓存同样也能很好解决循环引用问题。使用三级而非二级缓存并非出于IOC的考虑，而是出于AOP的考虑，即若使用二级缓存，在AOP情形下，注入到其他bean的，不是最终的代理对象，而是原始对象。

但是有网友来总结一个不错的理由：

并不是说二级缓存如果存在aop的话就无法将代理对象注入的问题，本质应该说是初始spring是没有解决循环引用问题的，设计原则是 bean 实例化、属性设置、初始化之后 再 生成aop对象，但是为了解决循环依赖但又尽量不打破这个设计原则的情况下，使用了存储了函数式接口的第三级缓存； 如果使用二级缓存的话，可以将aop的代理工作提前到 提前暴露实例的阶段执行； 也就是说所有的bean在创建过程中就先生成代理对象再初始化和其他工作； 但是这样的话，就和spring的aop的设计原则相驳，aop的实现需要与bean的正常生命周期的创建分离； 这样只有使用第三级缓存封装一个函数式接口对象到缓存中， 发生循环依赖时，触发代理类的生成

##### Spring Dispatcher Servlet初始化过程



##### DispatcherServlet 处理请求的过程



##### Spring中事务传播？

事务的传播指的是不同方法的嵌套调用中，事务应该如何 处理。是使用一个事务 还是另起事务，出现异常了 是回滚，还是提交，日常工作中使用比较多的 require  require_new nested

总共有7种事务传播方式。

核心逻辑就是

1. 判断内外方法是否是同一个事务
2. 是的话：异常统一在外部方法里处理
3. 不是的话，内部异常可以影响外部方法，但是外部方法无法影响内部。

##### Spring事务是如何回滚的?

核心逻辑就是 使用aop来实现的，会为 开启事务的类 生成代理对象，代理方法。顺便spring初始化的时候 还会根据事务配置初始化一个 TransactionInteceptor,实际的事务处理都是 调用 TransactionInteceptor来实现的。

TransactionInteceptor 实现事务的核心逻辑其实还是包装 数据库的事务，规则基本如下

1. 开启事务，就把数据库 connection的autoCommit 关闭了
2. 然后执行业务代码，sql之类的
3. 发生异常，需要回滚，最后代码绕了一圈也是 调用connection的 rollback
4. 业务正常执行，然后调用commit
5. 最后因为 spring把事务的一些信息都存在 线程的threadlocal里，所以最后还会有个清理动作。

##### Spring事务传播是如何实现的？

spring事务是使用数据库connection实现的事务，当创建事务的时候，会创建一个 事务信息对象 TransactionInfo，然后spring会把线程的threadlocal里存储 connection对象，key是datasource，如何遇到内嵌事务，创建新事务的时候，会把旧的 TransactionInfo 映射到新的TransactionInfo里，并且 threadlocal 绑定最新的datasource，当内部事务执行完毕以后，回到外部事务的时候，会尝试 恢复旧的TransactionInfo对象，这个对象里包括了 旧的数据源的引用，从而通过这种方式来实现了事务的内嵌 传播。



