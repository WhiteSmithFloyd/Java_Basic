# Essential Backlog [link](http://www.importnew.com/22290.html)

## LogBack、Slf4j和Log4j之间的关系
Slf4j是The Simple Logging Facade for Java的简称，是一个简单日志门面抽象框架，它本身只提供了日志Facade API和一个简单的日志类实现，一般常配合Log4j，LogBack，java.util.logging使用。Slf4j作为应用层的Log接入时，程序可以根据实际应用场景动态调整底层的日志实现框架(Log4j/LogBack/JdkLog…)。

LogBack和Log4j都是开源日记工具库，LogBack是Log4j的改良版本，比Log4j拥有更多的特性，同时也带来很大性能提升。详细数据可参照下面地址：[Reasons to prefer logback over log4j](https://logback.qos.ch/reasonsToSwitch.html)。

LogBack官方建议配合Slf4j使用，这样可以灵活地替换底层日志框架。

> TIPS：为了优化log4j，以及更大性能的提升，Apache基金会已经着手开发了log4j 2.0, 其中也借鉴和吸收了logback的一些先进特性，目前log4j2还处于beta阶段。

## LogBack的结构
LogBack被分为3个组件，logback-core, logback-classic 和 logback-access。
+ logback-core提供了LogBack的核心功能，是另外两个组件的基础
+ logback-classic则实现了Slf4j的API，所以当想配合Slf4j使用时，需要将logback-classic加入classpath。
+ logback-access是为了集成Servlet环境而准备的，可提供HTTP-access的日志接口。


## 配置详解
### 根节点<configuration>包含的属性
scan：当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true.
  
scanPeriod：设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟.
  
debug：当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

```xml
<configuration scan="true" scanPeriod="60 second" debug="false">  
      <!-- 其他配置省略-->  
</configuration>
```

### 根节点<configuration>的子节点
LogBack的配置大概包括3部分：appender, logger和root。

设置上下文名称<contextName>
每个logger都关联到logger上下文，默认上下文名称为“default”。但可以使用<contextName>设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改。
  
```xml
<configuration scan="true" scanPeriod="60 second" debug="false">  
      <contextName>myAppName</contextName>  
      <!-- 其他配置省略-->  
</configuration>
```

设置变量 <property>
用来定义变量值的标签，<property> 有两个属性，name和value；其中name的值是变量的名称，value的值时变量定义的值。通过<property>定义的值会被插入到logger上下文中。定义变量后，可以使“${}”来使用变量。
  
例如使用<property>定义上下文名称，然后在<contentName>设置logger上下文时使用
```xml
<configuration scan="true" scanPeriod="60 second" debug="false">  
      <property name="APP_Name" value="myAppName" />   
      <contextName>${APP_Name}</contextName>  
      <!-- 其他配置省略-->  
</configuration>
```

获取时间戳字符串 <timestamp>
两个属性 key:标识此<timestamp> 的名字；datePattern：设置将当前时间（解析配置文件的时间）转换为字符串的模式，遵循Java.txt.SimpleDateFormat的格式。

例如将解析配置文件的时间作为上下文名称：
```xml
<configuration scan="true" scanPeriod="60 second" debug="false">  
      <timestamp key="bySecond" datePattern="yyyyMMdd'T'HHmmss"/>   
      <contextName>${bySecond}</contextName>  
      <!-- 其他配置省略-->  
</configuration>
```












