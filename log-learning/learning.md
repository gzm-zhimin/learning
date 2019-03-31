### 前言
Log4j是面向Java的记录应用日志的工具包，由日志界大佬Ceki Gülcü创造于1996年并随后捐献给了Apache，`Log4j 1.*`在2015年停止了更新，并升级到变化较大的`Log4j 2`。这个最早的Java日志框架以其灵活简单的特点赢得了众多开发者的青睐，整个源代码包的大小不足0.5M!  

本文学习的Log4j版本为1.2.16，文章内容不涉及Log4j配置、Log4j如何使用等等问题，只研究源代码

### 整体架构
interface定义行为，查看一个框架的源代码，首先得好好理解它的接口！Log4j的核心接口/类有以下几个：
- LoggingEvent，日志事件是对用户希望记录的消息、异常、级别等信息的封装，将流转于日志框架的各个部分；
- Layout，决定了日志最终的展示样式&布局，核心方法就一个:
  ```
  abstract public String format(LoggingEvent event);
  ```
- Filter，LoggerEvent的过滤器，还包含了指向下一个过滤器的引用，核心方法也只有一个：
  ```
  abstract public int decide(LoggingEvent event);
  ```
  返回值定义了三个：-1(DENY,日志事件立即被丢弃不会流转到其他地方),1(ACCEPT，日志事件立即被记录),0(NEUTRAL,需要根据其他过滤器的结果来决定是否要记录或丢弃)
- Appender，定义了输出日志事件的策略，核心方法有以下3种（1个+2类）
  ```
  // 第1个：输出日志事件
  public void doAppend(LoggingEvent event);
  // 第2类：Filter相关，getter&clear省略；可以看出Appender有一个许多Filter组成的链表结构的过滤链
  public void addFilter(Filter newFilter);
  // 第3类：Layout相关，getter省略
  public void setLayout(Layout layout);
  ```
- Level，定义了系统能识别的日志所有级别
- Category&Logger，log4j最核心的类，定义了大多数日志记录操作，包括debug&info&warn&error&fatal等等；Category在2003年已经被废弃
- LoggerRepository, 管理Logger实例的容器，用于创建Logger实例、维护Logger实例之间的关系

下面再给出这些核心接口/类之间的关系图：
![log4j架构图](https://github.com/gzm-zhimin/learning/blob/master/log-learning/log4j%E6%9E%B6%E6%9E%84.png)

#### 说明：
1. AppenderSkeleton是所有其他Appender实现的父类，该类主要定义了执行append日志事件前的Filter链检查；
2. Hierarchy是主要的LoggerRepositiory的实现，该类主要定义了如何以name索引Logger，并且维护了诸多Logger实例之间的父子层级关系；
3. Logger的name的命名习惯参考包名，例如LoggerA的name为a.b.c，LoggerB的name为a.b，那么LoggerA是LoggerB的儿子；
4. AsyncAppender是一种异步处理LoggingEvent的Appender实现，内置了一个Dispatcher；AsyncAppender&Dispatcher是一个典型的生产者消费者模型！
5. WriterAppender是一种依赖java.io.Writer输出日志的Appender，输出前会首先调用Layout的format方法，然后再把结果交给Writer。

### log.info()的过程
![log.info()执行过程](https://github.com/gzm-zhimin/learning/blob/master/log-learning/log4j%E8%BE%93%E5%87%BA%E8%BF%87%E7%A8%8B%E5%BA%8F%E5%88%97%E5%9B%BE.png)
#### 说明：
1. Logger有一个指向最直接父Logger的引用；RootLogger只有一个，默认是所有其他Logger的父亲；当某个Logger处理LoggingEvent时，还会把该LoggingEvent传播给其父Logger，或者说**子Logger会继承父Logger所有的Appender**，可以通过指定additive=false禁用这种特性！


### 加载&初始化的过程
1. 使用log4j前，一般要先配置好log4j.xml配置文件；log4j初始化的时候于是就会根据该文件动态解析出对应的logger、appender、filter、layout、level等等信息。解析的入口在LoggerManager类的静态块中，具体执行解析的类主要是`org.apache.log4j.xml.DOMConfigurator`，具体的就不细讲了...

2. log4j还设置了FileWatchDog，用于跟踪log4j.xml文件的变化并更新到内存中的诸多组件对象

### 总结
1. logger、appender、filter、layout、loggerRepository等组件的抽象与封装良好，使得自定义扩展、理解维护都变得简单！
2. 性能问题：增删appender&调用append日志的操作有同步锁，ConsoleAppender没有缓冲区性能较低。
3. 其他问题看完Log4j 2以后再来补充...

