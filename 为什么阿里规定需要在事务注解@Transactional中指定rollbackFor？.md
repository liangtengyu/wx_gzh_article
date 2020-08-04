# 为什么阿里规定需要在事务注解@Transactional中指定rollbackFor？

> @Transactional(rollbackFor=Exception.class)的使用

点击上方“方志朋”，选择“设为星标”

回复”666“获取新整理的面试文章

![](https://mmbiz.qpic.cn/mmbiz_jpg/ow6przZuPIENb0m5iawutIf90N2Ub3dcPuP2KXHJvaR1Fv2FnicTuOy3KcHuIEJbd9lUyOibeXqW8tEhoJGL98qOw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

_作者：Mint6_

_blog.csdn.net/Mint6/article/details/78363761_

> 阿里巴巴Java规范：方法【edit】需要在Transactional注解指定rollbackFor或者在方法中显示的rollback。

1.异常的分类
-------

**先来看看异常的分类**

**![](https://mmbiz.qpic.cn/mmbiz_jpg/eQPyBffYbucbmx7UOaLmmL52O0SOqsDxJoOpNOYgrLNKicDfkQKbBQCvyapnjvq5kJz2OvIn75VuKfIk02Mj38w/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**

error是一定会回滚的

这里Exception是异常，他又分为运行时异常RuntimeException和非运行时异常

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

*   可查的异常（checked exceptions）:Exception下除了RuntimeException外的异常
    
*   不可查的异常（unchecked exceptions）:RuntimeException及其子类和错误（Error）
    

如果不对运行时异常进行处理，那么出现运行时异常之后，要么是线程中止，要么是主程序终止。  如果不想终止，则必须捕获所有的运行时异常，决不让这个处理线程退出。

队列里面出现异常数据了，正常的处理应该是把异常数据舍弃，然后记录日志。不应该由于异常数据而影响下面对正常数据的处理。

非运行时异常是RuntimeException以外的异常，类型上都属于Exception类及其子类。如IOException、SQLException等以及用户自定义的Exception异常。扩展：[Java项目构建基础：统一结果，统一异常，统一日志](http://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247491875&idx=1&sn=f924372666bfb0ef372f8e14a7623913&chksm=ebd5de0fdca25719cf285eca6413b067f9bbf7143c024a40c1941a88101cf68c52fbc8c28865&scene=21#wechat_redirect)

对于这种异常，JAVA编译器强制要求我们必需对出现的这些异常进行catch并处理，否则程序就不能编译通过。所以，面对这种异常不管我们是否愿意，只能自己去写一大堆catch块去处理可能的异常。

2.@Transactional 的写法
--------------------

开始主题，@Transactional如果只这样写，

**Spring框架的事务基础架构代码将默认地 只 在抛出运行时和unchecked exceptions时才标识事务回滚。**

> 也就是说，当抛出个RuntimeException 或其子类例的实例时。（Errors 也一样 - 默认地 - 标识事务回滚。）从事务方法中抛出的Checked exceptions将 不 被标识进行事务回滚。

*   让checked例外也回滚：在整个方法前加上 @Transactional(rollbackFor=Exception.class)
    
*   让unchecked例外不回滚：@Transactional(notRollbackFor=RunTimeException.class)
    
*   不需要事务管理的(只查询的)方法：@Transactional(propagation=Propagation.NOT\_SUPPORTED)
    

> 注意： 如果异常被try｛｝catch｛｝了，事务就不回滚了，如果想让事务回滚必须再往外抛try｛｝catch｛throw Exception｝。

注意
--

Spring团队的建议是你在具体的类（或类的方法）上使用 @Transactional 注解，而不要使用在类所要实现的任何接口上。

你当然可以在接口上使用 @Transactional 注解，但是这将只能当你设置了基于接口的代理时它才生效。

因为**注解是不能继承的**，这就意味着如果你正在使用基于类的代理时，那么事务的设置将不能被基于类的代理所识别，而且对象也将不会被事务代理所包装（将被确认为严重的）。因此，请接受Spring团队的建议并且在具体的类上使用 @Transactional 注解。

@Transactional 注解标识的方法，处理过程尽量的简单。尤其是带锁的事务方法，能不放在事务里面的最好不要放在事务里面。可以将常规的数据库查询操作放在事务前面进行，而事务内进行增、删、改、加锁查询等操作。

> 注：rollbackFor 可以指定能够触发事务回滚的异常类型。Spring默认抛出了未检查unchecked异常（继承自 RuntimeException 的异常）或者 Error才回滚事务；其他异常不会触发回滚事务。

> 如果在事务中抛出其他类型的异常，但却期望 Spring 能够回滚事务，就需要指定 rollbackFor属性。

**热门内容：**

*   [一套简单通用的Java后台管理系统，拿来即用，非常方便（附项目地址）](http://mp.weixin.qq.com/s?__biz=MzAxNjk4ODE4OQ==&mid=2247491200&idx=1&sn=0f967d56ec5564060fd864733645efca&chksm=9bed3ff2ac9ab6e43b28b58512a3f0133c5575bdc6c606794fca36ead1d4f4213e2a031c818f&scene=21#wechat_redirect)
    
*   [半吊子架构师，一来就想干掉RabbitMQ ...](http://mp.weixin.qq.com/s?__biz=MzAxNjk4ODE4OQ==&mid=2247490838&idx=2&sn=a6bb6dcaa0b0122480b5e69f272b6d3f&chksm=9bed3c64ac9ab572bf8c966a1c27c351e14c79a0c6c839e0b959de1da6ae0f2451ad3ac8400c&scene=21#wechat_redirect)  
    
*   [前、后端分离权限控制设计和实现思路](http://mp.weixin.qq.com/s?__biz=MzAxNjk4ODE4OQ==&mid=2247490825&idx=2&sn=58cc517cb2713c1134fae40588d8c79c&chksm=9bed3c7bac9ab56dcb6d69aeea165f9ab82de0a904b8dd6534e5f0315f3274ae585089f2cd04&scene=21#wechat_redirect)
    

![](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

最近面试BAT，整理一份面试资料《**Java面试BAT通关手册**》，覆盖了Java核心技术、JVM、Java并发、SSM、微服务、数据库、数据结构等等。

获取方式：点“**在看**”，关注公众号并回复 **666** 领取，更多内容陆续奉上。

**明天见(｡･ω･｡)ﾉ♡**


[Source](https://mp.weixin.qq.com/s/QpoqLS7hLSXK13sKhYWluA)