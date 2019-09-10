## .NET日志记录之——log4net划重点篇
### 1.概述
log4net是.Net下一个非常优秀的开源日志记录组件。log4net记录日志的功能非常强大。它可以将日志分不同的等级，以不同的格式，输出到不同的媒介。

### 2.Log4net的主要组成部分
#### 2.1 **Appenders**
Appenders用来定义日志的输出方式，即日志要写到那种介质上去。较常用的Log4net已经实现好了，直接在配置文件中调用即可，可参见上面配置文件例子；当然也可以自己写一个，需要从log4net.Appender.AppenderSkeleton类继承。它还可以通过配置Filters和Layout来实现日志的过滤和输出格式。

已经实现的输出方式有：
```
AdoNetAppender 将日志记录到数据库中。可以采用SQL和存储过程两种方式。

AnsiColorTerminalAppender 将日志高亮输出到ANSI终端。

AspNetTraceAppender  能用asp.net中Trace的方式查看记录的日志。

BufferingForwardingAppender 在输出到子Appenders之前先缓存日志事件。

ConsoleAppender 将日志输出到应用程序控制台。

EventLogAppender 将日志写到Windows Event Log。

FileAppender 将日志输出到文件。

ForwardingAppender 发送日志事件到子Appenders。

LocalSyslogAppender 将日志写到local syslog service (仅用于UNIX环境下)。

MemoryAppender 将日志存到内存缓冲区。

NetSendAppender 将日志输出到Windows Messenger service.这些日志信息将在用户终端的对话框中显示。

OutputDebugStringAppender 将日志输出到Debuger，如果程序没有Debuger，就输出到系统Debuger。如果系统Debuger也不可用，将忽略消息。

RemoteSyslogAppender 通过UDP网络协议将日志写到Remote syslog service。

RemotingAppender 通过.NET Remoting将日志写到远程接收端。

RollingFileAppender 将日志以回滚文件的形式写到文件中。

SmtpAppender 将日志写到邮件中。

SmtpPickupDirAppender 将消息以文件的方式放入一个目录中，像IIS SMTP agent这样的SMTP代理就可以阅读或发送它们。

TelnetAppender 客户端通过Telnet来接受日志事件。

TraceAppender 将日志写到.NET trace 系统。

UdpAppender 将日志以无连接UDP数据报的形式送到远程宿主或用UdpClient的形式广播。

```

#### **2.2 Filters**

使用过滤器可以过滤掉Appender输出的内容。过滤器通常有以下几种：
```
DenyAllFilter 阻止所有的日志事件被记录

LevelMatchFilter 只有指定等级的日志事件才被记录

LevelRangeFilter 日志等级在指定范围内的事件才被记录

LoggerMatchFilter 与Logger名称匹配，才记录

PropertyFilter 消息匹配指定的属性值时才被记录

StringMathFilter 消息匹配指定的字符串才被记录
```

#### **2.3 Layouts**

Layout用于控制Appender的输出格式，可以是线性的也可以是XML。

一个Appender只能有一个Layout。

最常用的Layout应该是经典格式的PatternLayout，其次是SimpleLayout，RawTimeStampLayout和ExceptionLayout。然后还有IRawLayout，XMLLayout等几个，使用较少。Layout可以自己实现，需要从log4net.Layout.LayoutSkeleton类继承，来输出一些特殊需要的格式，在后面扩展时就重新实现了一个Layout。
```
SimpleLayout 简单输出格式，只输出日志级别与消息内容。

RawTimeStampLayout 用来格式化时间，在向数据库输出时会用到。样式如“yyyy-MM-dd HH:mm:ss“

ExceptionLayout 需要给Logger的方法传入Exception对象作为参数才起作用，否则就什么也不输出。输出的时候会包含Message和Trace。

PatternLayout 使用最多的一个Layout，能输出的信息很多，使用方式可参见上面例子中的配置文件。PatterLayout的格式化字符串见文后附注8.1。
```

#### **2.4 Loggers**

Logger是直接和应用程序交互的组件。Logger只是产生日志，然后由它引用的Appender记录到指定的媒介，并由Layout控制输出格式。

Logger提供了多种方式来记录一个日志消息，也可以有多个Logger同时存在。每个实例化的Logger对象对被log4net作为命名实体（Named Entity）来维护。log4net使用继承体系，也就是说假如存在两个Logger，名字分别为a.b.c和a.b。那么a.b就是a.b.c的祖先。每个Logger都继承了它祖先的属性。所有的Logger都从Root继承,Root本身也是一个Logger。

日志的等级，它们由高到底分别为：
```
OFF &gt; FATAL &gt; ERROR &gt; WARN &gt; INFO &gt; DEBUG  &gt; ALL 
```

高于等级设定值方法（如何设置参见“配置文件详解”）都能写入日志， Off所有的写入方法都不写到日志里，ALL则相反。例如当我们设成Info时，logger.Debug就会被忽略而不写入文件，但是FATAL,ERROR,WARN,INFO会被写入，因为他们等级高于INFO。

在具体写日志时，一般可以这样理解日志等级：
```
FATAL（致命错误）：记录系统中出现的能使用系统完全失去功能，服务停止，系统崩溃等使系统无法继续运行下去的错误。例如，数据库无法连接，系统出现死循环。

ERROR（一般错误）：记录系统中出现的导致系统不稳定，部分功能出现混乱或部分功能失效一类的错误。例如，数据字段为空，数据操作不可完成，操作出现异常等。

WARN（警告）：记录系统中不影响系统继续运行，但不符合系统运行正常条件，有可能引起系统错误的信息。例如，记录内容为空，数据内容不正确等。

INFO（一般信息）：记录系统运行中应该让用户知道的基本信息。例如，服务开始运行，功能已经开户等。

DEBUG （调试信息）：记录系统用于调试的一切信息，内容或者是一些关键数据内容的输出。
```
Logger实现的ILog接口，ILog定义了5个方法（Debug,Inof,Warn,Error,Fatal）分别对不同的日志等级记录日志。这5个方法还有5个重载。以Debug为例说明一下，其它的和它差不多。

ILog中对Debug方法的定义如下：
```
void Debug(object message);

void Debug(object message, Exception ex);
```
还有一个布尔属性：
```
bool IsDebugEnabled { get; }
```
如果使用Debug(object message, Exception ex)，则无论Layout中是否定义了%exception，默认配置下日志都会输出Exception。包括Exception的Message和Trace。如果使用Debug(object message)，则日志是不会输出Exception。

最后还要说一个LogManager类，它用来管理所有的Logger。它的GetLogger静态方法，可以获得配置文件中相应的Logger：
```
log4net.ILog log = log4net.LogManager.GetLogger("logger-name");
```

#### **2.5 Object Renders**

它将告诉logger如何把一个对象转化为一个字符串记录到日志里。（ILog中定义的接口接收的参数是Object，而不是String。）

例如你想把Orange对象记录到日志中，但此时logger只会调用Orange默认的ToString方法而已。所以要定义一个OrangeRender类实现log4net.ObjectRender.IObjectRender接口，然后注册它（我们在本文中的扩展不使用这种方法，而是直接实现一个自定义的Layout）。这时logger就会知道如何把Orange记录到日志中了。

#### **2.6 Repository**

Repository主要用于日志对象组织结构的维护。

### 3.配置文件详解

#### **3.1 配置文件构成**

主要有两大部分，一是申明一个名为“log4net“的自定义配置节，如下所示：
```
&lt;configSections&gt;
    &lt;section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net" /&gt;
&lt;/configSections&gt;
&lt;log4net&gt;
    ...
    ...
    ...
&lt;/log4net&gt;
```
二是&lt;log4net&gt;节的具体配置，这是下面要重点说明的。
#### 3.1.1 &lt;log4net&gt;

所有的配置都要在&lt;log4net&gt;元素里定义。

支持的属性：

属性 | 详解 
------------- | -------------
 debug | 可选，取值是true或false，默认是false。设置为true，开启log4net的内部调试。 
 update | 可选，取值是Merge(合并)或Overwrite(覆盖)，默认值是Merge。设置为Overwrite，在提交配置的时候会重置已经配置过的库。 
 threshold | 可选，取值是repository（库）中注册的level，默认值是ALL。 

支持的子元素：

属性 | 详解 
- | :-:
appender | 0或多个
logger | 0或多个
renderer | 0或多个
root | 最多一个
param | 0或多个

#### 3.1.2 &lt;root&gt;

实际上就是一个根logger，所有其它logger都默认继承它，如果配置文件里没有显式定义，则框架使用根日志中定义的属性。root元素没有属性。

支持的子元素：

属性 | 详解 
- | :-:
appender-ref | 0个或多个，要引用的appender的名字。
level | 最多一个。 只有在这个级别或之上的事件才会被记录。
param | 0个或多个， 设置一些参数。

#### 3.1.3 &lt;logger&gt;
支持的属性：

属性 | 详解 
- | :-:
name | 必须的，logger的名称
additivity | 可选，取值是true或false，默认值是true。设置为false时将阻止父logger中的appender。

支持的子元素：

属性 | 详解 
- | :-:
appender-ref | 0个或多个，要引用的appender的名字。
level | 最多一个。 只有在这个级别或之上的事件才会被记录。
param | 0个或多个， 设置一些参数。

#### 3.1.4 &lt;appender&gt;
定义日志的输出方式，只能作为 log4net 的子元素。name属性必须唯一，type属性必须指定。

支持的属性：

属性 | 详解 
- | :-:
name | 必须的，Appender对象的名称
type | 必须的，Appender对象的输出类型

支持的子元素：

属性 | 详解 
- | :-:
appender-ref | 0个或多个，允许此appender引用其他appender，并不是所以appender类型都支持。
filter | 0个或多个，定义此app使用的过滤器。
layout | 最多一个。定义appender使用的输出格式。
param | 0个或多个， 设置Appender类中对应的属性的值。

实际上&lt;appender&gt;所能包含的子元素远不止上面4个。

#### 3.1.5 &lt;layout&gt;

布局，只能作为&lt;appender&gt;的子元素。

支持的属性：

属性 | 详解 
- | :-:
type | 必须的，Layout的类型

支持的子元素：

属性 | 详解 
- | :-:
param | 0个或多个， 设置一些参数。

#### 3.1.6 &lt;filter&gt;
过滤器，只能作为&lt;appender&gt;的子元素。

支持的属性：

属性 | 详解 
- | :-:
type | 必须的，Filter的类型

支持的子元素：

属性 | 详解 
- | :-:
param | 0个或多个， 设置一些参数。

#### 3.1.7 &lt;param&gt;

&lt;param&gt;元素可以是任何元素的子元素。

支持的属性：

属性 | 详解 
- | :-:
name | 必须的，取值是父对象的参数名。
value | 可选的，value和type中，必须有一个属性被指定。value是一个能被转化为参数值的字符串。
type | 可选的，value和type中，必须有一个属性被指定。type是一个类型名，如果type不是在log4net程序集中定义的，就需要使用全名。

支持的子元素：

属性 | 详解 
- | :-:
param | 0个或多个， 设置一些参数。

### 4. 关联配置文件

log4net默认关联的是应用程序的配置文件App.config(BS程序是Web.config)，可以使用程序集自定义属性来进行设置。下面来介绍一下这个自定义属性：

log4net.Config.XmlConifguratorAttribute

XmlConfiguratorAttribute有3个属性：

ConfigFile： 配置文件的名字，文件路径相对于应用程序目录

(AppDomain.CurrentDomain.BaseDirectory)。ConfigFile属性不能和ConfigFileExtension属性一起使用。

ConfigFileExtension： 配置文件的扩展名，文件路径相对于应用程序的目录。ConfigFileExtension属性不能和ConfigFile属性一起使用。

Watch： 如果将Watch属性设置为true，就会监视配置文件。当配置文件发生变化的时候，就会重新加载。

如果ConfigFile和ConfigFileExtension都没有设置，则使用应用程序的配置文件App.config（Web.config）。

可以在项目的AssemblyInfo.cs文件里添加以下的语句：

```

//监视默认的配置文件，App.config 
[assembly: log4net.Config.XmlConfigurator(Watch = true)]

//使用配置文件log4net.config，不监视改变。注意log4net.config文件的目录，BS程序在站点目录//下，CS则在应用程序启动目录下，如调试时在/bin/Debug下，一般将文件属性的文件输出目录调为//始终复制即可

[assembly: log4net.Config.XmlConfigurator(ConfigFile = "log4net.config")]


//使用配置文件log4net.config，不监视改变

[assembly: log4net.Config.XmlConfigurator()]
```
 

也可以在Global.asax的Application_Start里或者是Program.cs中的Main方法中添加，注意这里一定是绝对路径，如下所示：
```
//这是在BS程序下，使用自定义的配置文件log4net.config，使用Server.MapPath("~") + //@"/log4net.config”来取得路径。/log4net.config为相对于站点的路径

// ConfigureAndWatch()相当于Configure(Watch = true)

log4net.Config.XmlConfigurator.ConfigureAndWatch(new System.IO.FileInfo(Server.MapPath("~") + @"/log4net.config"));
```
```
//这是在CS程序下，可以用以下方法获得：

string assemblyFilePath = Assembly.GetExecutingAssembly().Location;

string assemblyDirPath = Path.GetDirectoryName(assemblyFilePath);

string configFilePath = assemblyDirPath + " //log4net.config";

log4net.Config.XmlConfigurator.ConfigureAndWatch(new FileInfo(configFilePath));
```
或直接使用绝对路径：
```
//使用自定义的配置文件，直接绝对路径为：c:/log4net.config

log4net.Config.XmlConfigurator.Configure(new System.IO.FileInfo(@"c:/log4net.config"));
```
 
&lt;/font&gt;