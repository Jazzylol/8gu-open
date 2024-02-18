##### 简单介绍下mybatis架构

mybatis大致上可以分为三层

框架支撑：主要负责 xml解析，事务管理，连接池管理、缓存管理等

数据处理：数据处理主要负责  SQL执行过程中的参数解析，SQL解析，SQL执行以及后续的结果处理与映射。

接口层：接口层主要就是给上层提供了一系列的 增删改查的接口。

说白了 其实mybatis 是 在jdbc上包装了一层，解决了SQL拼接，连接池管理，结果解析等等比较繁琐的问题，并且对外提供了一系列简单的api的一个框架。

优点：

1. sql方便统一管理，也是更好的复用
2. 减少了 对接jdbc的复杂性
3. 可以很好的和主流框架集成

缺点：

1、 复杂sql，越到后期管理维护成本越高。



##### mybatis中事务是怎么处理的

mybatis里的事务实现是比较简单的，因为他核心的能力并不是事务，所以设计实现的都比较简单。主要就分为两类，一个是 使用jdbc来管理事务，另外个就是使用 程序容器 来管理事务，比如jboss，weblogic。

jdbc connection 里面其实是有事务的管理的，比如 commit，rollback等，mybatis就是简单包装了 jdbc connection来调用他的方法 commit而已。

另外一个就是 ManagedTransaction 就是把 mybatis的事务交给外部容器来管理，本身调用 mybaits的方法 不起到任何作用。

还有就是mybatis可以在配置文件里配置 事务管理的模式，不配置的话，现在更多是直接被 spring 去覆盖事务管理。



##### mybatis中一条sql大概是怎么执行的？

首先，mybatis启动的时候会根据配置文件解析出 mybatis 的config配置，这个配置里包含了，哪个DAO，哪个方法对应 哪个XML配置文件里的哪条SQL。

当一条语句执行的时候，mybatis会先创建一次会话，这个会话核心入口就是sqlSession，使用工厂设计模式，从sqlSessionFactory里创建，在创建sqlSession对象的时候其实还会获取config配置，以及获取数据库连接 connection对象，还会创建一个用户执行任务的 executor对象。

紧接着 sqlSession把  会从config中获取对应的mapper ，这时候 sqlSession就把拿到的mapper，connection等对象传递给 executor来执行对应的任务

executor拿到各种配置以及参数对象以后，会继续把任务委托给 类 statementHandler，后续就是格式化下sql，statementHandler后续就是把sql传递给jdbc 来直接调用了，然后返回jdbc的 resultset

mybatis拿到resultset 会把resultSet再序列化为 对应的结果对象，整个过程就是这样



##### mybatis里的插件机制是怎么实现的？

mybatis里主要提供了4中类型插件

executor：各种方法的拦截器，比如select update  事务的commit rollback等等

parameterHander：参数拦截器，可以修改参数解析的方式

statementHandler：主要是拦截 sql动态拼接的拦截器

resultHandler：主要拦截 结果集的拦截器

各种拦截器需要实现 Interceptor 接口，实现对应方法后，可以在配置文件中加载拦截器，加载完毕以后，mybatis会在启动的时候 实例化所有的拦截器，同时在执行sql过程中，比如创建executor，parameter 等等几个对象的时候，直接调用所有拦截器，其实就是把所有拦截器全部调用一遍，所以拦截器里要判断 对象类型。调用完以后，会覆盖框架原有的创建对象。mybatis 和 spring结合的打印sql日志就是这样搞的，包括很多分页插件也是这样搞的。



##### mybatis一级二级缓存是什么机制

mybatis提供了一级和二级缓存，主要就是为了尽量想减少db查询，降低db的压力。

一级缓存 主要是mybatis自带的，一次sqlSession的时候 创建executor的时候，内置了一个 hashMap，同一个session里，多次查询相同的sql的话，那就会直接走缓存，然后依次sqlsession结束，所有对象也都会回收，所以影响不大。判定相同sql，主要是 调用的方法，参数值，limit offset，最后sql要一样。

二级缓存主要是在 整个应用层面，需要在配置文件里主动配置，二级缓存级别是跟mapper绑定的，一方面需要开启二级缓存，还需要配置mapper缓存，还需要在具体的sql上配置 使用缓存。二级缓存主要是在sqlsession 创建 executor之前 先查询缓存，有数据就返回，没数据才走真正的查询。顺便二级缓存还可以接入外部的缓存实现。



##### # $会有什么区别

#{} 会解析为占位符 ？ 

${} 会当成字符串直接处理，不会解析为占位符，不过表名是必须使用$的，主要 #{}会解析为字符串。${ } 在预编译之前已经被变量替换了

${} 因为解析为字符串，所以存在sql注入的问题。在预编译之前，已经解析为字符串了。







