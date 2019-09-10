## .NET实现持续集成与自动化部署1-Jenkins

### 前言

&nbsp;&nbsp;&nbsp;&nbsp;相信每一位程序员都经历过深夜加班上线的痛苦！而作为一个加班上线如家常便饭的码农，更是深感其痛。由于我们所做的系统业务复杂，系统庞大，设计到多个系统之间的合作，而核心系统更是采用分布式系统架构，由于当时对系统划分的不合理等等原因导致每次发版都会设计到多个系统的发布，小的版本三五个，大的版本十几个甚至几十个系统的同时发布！而我们也没有相应的基础设施的支撑，发版方式更是最传统的，开发人员将发布包发给运维人员，由其讲各个发布包一个一个覆盖到生产环境。因此每次上线仅仅发版就需要2-3个小时。这种方式不仅仅耗时、耗力，更是由于人工操作经常导致一些丢、落的现象。<!--more-->而我们当时的测试也是采用纯手工的测试，发版完毕后一轮回归测试就需要3-4个小时(当时主要是手工测试)。之前也一直提倡持续集成、自动化的测试和运维，但迟迟没有推进落地。终于在一个加班到凌晨四点的夜晚后，我再也受不了。回家后躺在床上迟迟睡不着，心想这个自动化的发布能有多难，他们搞不了，老子自己搞，于是6点爬起来来到公司，正式开始了我的持续集成、自动化部署的研究与推进之路。

### 系列文章

[.NET实现持续集成与自动化部署1-Jenkins](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B21-Jenkins/)

[.NET实现持续集成与自动化部署2-NuGet](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B22-NuGet/)

[.NET实现持续集成与自动化部署3-测试环境到生产环境策略](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B23-%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83%E5%88%B0%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E7%AD%96%E7%95%A5/)

### 一、初识Jenkins

&nbsp;&nbsp;&nbsp;&nbsp;由于之前亦没有相关知识的积累，因此也是对如何实现也是一头雾水。于是只能找度娘，关键字"自动化发布"。搜索到很多工具和方法，但都是以Java平台居多，.net平台相关资料不多。其中以Jenkins介绍较多，微软也提供一套自动化部署的方式，也有一些其他持续集成工具可以实现自动化的发布，但最终还是选择了Jenkins。主要有以下几个原因：

* 代码开源、插件丰富完善、系统稳定
* 社区活跃，成功实践和网上资源较为丰富
* 安装配置简单
* web形式的可视化的管理页面

#### 1. Jenkins是什么

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Jenkins是一个开源软件项目，是基于Java开发的一种持续集成工具，用于监控持续重复的工作，旨在提供一个开放易用的软件平台，使软件的持续集成变成可能。

**  持续集成： **

&nbsp;&nbsp;&nbsp;&nbsp;持续集成是一种软件开发实践，即团队开发成员经常集成他们的工作，通过每个成员每天至少集成一次，也就意味着每天可能会发生多次集成。每次集成都通过自动化的构建（包括编译，发布，自动化测试）来验证，从而尽早地发现集成错误。

#### 2.Jenkins能干什么

&nbsp;&nbsp;&nbsp;&nbsp;众所周知，工业革命解放了人类的双手，使得人们避免了很多重复性的工作，而Jenkins能帮助开发测试运维人员解决很多重复性工作，我们可以将一些重复性的工作，写成脚本，如：代码提交，执行单元测试，程序的编译、构建、发布等封装成脚本，由Jenkins替我们定时或按需执行。事实上Jenkins的众多插件就是如此，究其根本就是执行一个或多个windows或linux命令来完成我们的需求。

#### 3.Jenkins的一个工作流程

&nbsp;&nbsp;&nbsp;&nbsp;通过对Jenkins的简单了解后，对完成自动化发布有了大致思路，如下图为Jenkins的一个工作流程

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-4/79027010.jpg)



思路已经有了，接下来就是针对此流程，一步一步简单实现.NET Web应用程序基于Jenkins的自动化部署。

### 二、Jenkins 安装

&nbsp;&nbsp;&nbsp;&nbsp;Jenkins有windows版本也有linux版本，由于我们项目都是基于.net freamwork进行开发，而jenkins构建需要编译.net程序,为了更方便的编译，因此选择安装windows版本。

#### 1.下载

&nbsp;&nbsp;&nbsp;&nbsp;可从Jenkins官网https://jenkins.io/download/下载windows安装包。

#### 2.安装

&nbsp;&nbsp;&nbsp;&nbsp;下载完成后，可按照提示进行安装即可。(windows下傻瓜式安装，注意Jenkins是java开发，因此需先安装对应jdk版本)

#### 3.配置

&nbsp;&nbsp;&nbsp;&nbsp;安装完成后会自动安装并启动一个windows服务，名为Jenkins，打开浏览器localhost:8080(Jenkins默认端口号为8080，如需修改可打开Jenkins安装目录找到Jenkins.xml修改其中端口，然后打开服务重启Jenkins服务即可)之后按照提示进行配置即可！配置完成后看到如下界面代表安装成功!

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-4/27633802.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;整个安装过程非常简单，基本上是傻瓜式按照提示操作即可，期间并未遇到问题，基本上10分钟左右就搞定了！接下来将介绍如何按照上述流程实现.NET下Jenkins的持续集成与自动化部署!

### 三、通过SVN获取源代码

#### 1.安装插件

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;根据我们的思路，首先要做的就是获取到我们的源代码。由于我们公司使用的源代码管理工具主要是SVN因此在这里主要介绍SVN的方式方法。根据度娘的指引，我们需要安装一个SVN的插件:Subversion Plug-in(如果：安装Jenkins时选择的安装推荐的插件，则Jenkins会直接给安装上这个插件，无需自己安装)。

#### 2.项目配置

&nbsp;&nbsp;&nbsp;&nbsp;安装插件后，选择新建一个自由风格的软件项目，起个名字，进入到项目配置后，找到源代码管理选项：

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-4/51220937.jpg)

主要有以下几个选项需要配置:

* Repository URL:要获取的SVN的路径，如:https://127.0.0.1:9666/svn/HS.Mall/SoureCode/Trunk/Test

* Credentials:配置SVN用户名和密码

* Ignore externals:是否忽略SVN外部引用(这个很重要，稍后会用到，关于SVN外部引用，可自行百度)

* Additional Credentials:当你的SVN版本库使用外部引用关联其它版本库是这个就很重要了

  Realm：填写SVN服务器的地址

  ```
   <https://127.0.0.1:9666> VisualSVN Server //(注意这个格式)
  ```

  Credentials:填写SVN用户名和密码信息

其它一些选项直接按照默认值就可以，关于每一项的详细介绍可以点击后面的小？号查看。

配置完成后点击保存后，构建该项目查看结果。若能够将源代码更新至Jenkins的工作空间内，则代表配置成功!

### 四、通过MSBuild编译应用程序

#### 1.安装插件与环境

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;编译.NET应用程序可通过微软提供的MSBuild工具，先安装插件:MSBuild。(注意:Jenkins服务器需安装MSBuild，建议在Jenkins上安装VS开发工具，可以在构建出问题的时候打开VS调试，省去很多不必要的麻烦)。

#### 2.全局配置

&nbsp;&nbsp;&nbsp;&nbsp;插件安装完毕后，进入系统管理->全局工具配置(ConfigureTools)找到MSBuild配置选项:

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-4/3872697.jpg)

* Name：自己起个名字
* Path to MSBuild:MSBuild.exe程序的物理路径

注意:此处MSBuild.exe必须与程序所使用freamwork版本相对应，此处我在这就遇到了一个大坑，一开始随便找个一个MSBuild工具，没想到根本编译不了C#6.0的语法。建议直接指向visual studio安装目录内的MSBuild.exe,可以避免很多问题。如VS2017在:Program Files (x86)\Microsoft Visual Studio\2017\Enterprise\MSBuild\15.0\Bin路径内。

#### 3.项目配置

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;打开我们之前创建的项目，找到构建选项->增加构建步骤->Build a Visual Studio project or solution using MSBuild

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-4/45403259.jpg)

* Name:选择全局MSBuild配置的名称

* MSBuild Build File:填写我们的要构建的项目.csproj文件，所相对工作的路径。如:/Test.csproj

* Command Line Arguments:MSBuild的参数如：/t:Rebuild /P:Configuration=Release /p:VisualStudioVersion=14.0 /p:DeployOnBuild=True;PublishProfile=Test.pubxml

  * /t:Rebuild 重新生成 

  - /p:Configuration=Release Release 生成模式 
  - /p:VisualStudioVersion=14.0 指定子工具集(VS2015为14.0，2017为15.0)，不设置会报错 
  - /p:DeployOnBuild=True;PublishProfile=Test.pubxml 使用 Test.pubxml 发布文件来发布项目 .pubxml文件可在VS发布时配置，位于Properties文件夹内。

  配置完成后，点击构建，查看控制台信息，如能构建成功，则代表我们的配置无误！

#### 4.遇到的问题

&nbsp;&nbsp;&nbsp;&nbsp;原以为按照度娘的一系列解决方案能够很顺利的构建，可是在连续失败了几十次之后，才明白远远没有那么简单。期间主要遇到几个问题：

* MSBuild版本不对导致构建不了C#6.0的语法
* Jenkins 是讲版本库源代码更新到自己的工作空间内，再执行后续的构建工作。我们的程序很不规范，其中引用了许多不属于自己版本库的第三方依赖包，和一些自己开发的公共库，当时这些第三方包和公共库放在我们SVN的另一个版本库里进行管理，因此在构建的时候导致很多程序集找不到引用。

关于问题1:上面已经提过，只需要找到对应版本即可

而问题2:一开始找了很多资料也没有找到解决方案，后来还是从源代码管理上找到了方案。

方案1：

&nbsp;&nbsp;&nbsp;&nbsp;借鉴Nuget的思想，使用Nuget服务器管理我们自己开发的一些公共依赖库。关于Nuget管理依赖的文章在另一篇博客里。

方案2:

&nbsp;&nbsp;&nbsp;&nbsp;就是上面提到的SVN 外部引用，当时也是走投无路，于是疯狂翻译Jenkins的这些英文解释，在翻译到SVN插件的Ignore externals时，找到了这种方案，就是SVN可以设置外部引用，这样在更新版本库的时候就可以把依赖的版本库也更新下来，然后Jenkins SVN插件把这个Ignore externals选项去掉，然后在Additional Credentials选项里填上所依赖版本库的SVN配置，就能够把这些依赖也更新到SVN工作空间内。

&nbsp;&nbsp;&nbsp;&nbsp;以上两个问题解决后，基本没有遇到太难的问题。由此可见我们的源代码管理的科学、规范是多么的重要。

几十次的构建失败，一堆乱七八糟的引用是多么痛的领悟!

### 五、通过Ftp发布至应用服务器

&nbsp;&nbsp;&nbsp;&nbsp;构建成功后，Test.pubxml会指定发布的包的路径(最好是放到工作空间下)，按照思路，接下来就是要想办法把发布包Copy到应用服务器的根目录下。由于我们的应用服务器都是windows系统，因此不能像linux系统一样通过ssh远程Copy过去，当时能想到的就是使用Ftp直接上传到应用服务器。

#### 1.安装插件与环境

&nbsp;&nbsp;&nbsp;&nbsp;Jenkins 安装插件Publish Over FTP,应用服务器上需开启Ftp。

####　2.全局配置

&nbsp;&nbsp;&nbsp;&nbsp;系统管理->系统配置下找到Publish over FTP配置项

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-5/40236536.jpg)

* Name:起个名字，后面项目配置里会用的到
* HostName:Ftp主机名(端口号默认21，在高级里面可以改)
* Username:Ftp用户名
* Password:Ftp密码

#### 3.项目配置

&nbsp;&nbsp;&nbsp;&nbsp;打开我们之前建的项目，找到构建后操作->增加构建后操作步骤->Send build artifacts over FTP

![image](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-5/25517488.jpg)

* Name:选择全局配置里的
* Source files:选择你的发布包路径(这里是相对于工作空间的路径)
* Remote directory：放到远程的哪个路径里(这里是相对于Ftp根目录的路径)

配置完成后，点击保存，构建即可！

### 六、结束语

&nbsp;&nbsp;&nbsp;&nbsp;如上，就基本实现了我们的自动化发布的需求，这期间从早晨六点开始，差不多中午就完成了，当然也并不像上面介绍的那么简单，期间也遇到了许多问题，构建了大概一百多次，才最终成功了第一次。本文主要介绍实现自动化部署的一种基本的思路，当然还有很多方案可以实现我们的需求，甚至不仅仅局限于Jenkins。而这种方案其中也有许多细节的地方在文章中没有提到，如：如何实现自动化的Nunit单元测试，如何定时构建......，因为当时我在完成之后也给我的团队成员提供了一个非常详细的配置文档，并且培训了很多次，但事实证明，讲的越详细越会限制他们自己的主动思考与动手的能力。这也导致了后来我去做其他工作的时候，我们将近一年的时间还是停留在我这半天的研究结果的层面上，而生产环境更是迟迟没有使用。其实思路才是最重要的，有了思路我们就可以通过各种方式来解决我们的问题，还是建议大家注重解决问题的思路，多动手，自己实践，才能学得更透!关于.NET 平台下Jenkins实现持续集成与自动化部署的落地与实现的问题与讨论，可以在文章下留言。