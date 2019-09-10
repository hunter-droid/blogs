## .NET实现持续集成与自动化部署2-NuGet

### 前言
&nbsp;&nbsp;&nbsp;&nbsp;Nuget是一个.NET平台下的开源的项目，它是Visual Studio的扩展。在使用Visual Studio开发基于.NET Framework的应用时，Nuget能把在项目中添加、移除和更新引用的工作变得更加快捷方便。这是维基百科中的定义，实际上Nuget就是一个包管理器，类似于Java的Maven，可以帮助我们更方便的管理dll。<!--more-->

&nbsp;&nbsp;&nbsp;&nbsp;相信每个人都从官方的nuget服务器上下载过一些第三方组件。如:log4net、quartz.net等。实际上随着公司业务慢慢的拓展，项目也会越来越来多，很多项目会依赖其他项目DLL，比如一些底层的技术组件项目多，交叉引用多，这个时候对这些DLL的管理就至关重要。起初我们公司的方案是把这些公共的组件放到SVN的一个目录下，然后大家更新到本地，然后添加引用到项目里。这种方式管理起来较为复杂，而且必须要求所有项目人员的SVN更新路径必须是一致的。起初项目较少，项目之间没什么依赖，可重用的组件也不多，用起来没什么问题，但随着项目越来越多，可重用的组件也越来越多，引用越来越复杂，这个时候这些组件管理起来就很吃力了。<br>
&nbsp;&nbsp;&nbsp;&nbsp;以上问题并没有意识到，我是在做Jenkisn持续集成与自动化发布的时候发现Jenkins把SVN更新到自己的工作空间内时，并不能更新到这些依赖的组件(因为这些公共的组件不在项目的SVN工作目录内)，导致构建失败的时候，苦思良久才想到搭建我们公司自己的Nuget服务器，来管理这些组件的。等真正用起来之后才觉得，可能正规军都这么玩，我们之前那种方式只是野路子。

### 系列文章

[.NET实现持续集成与自动化部署1-Jenkins](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B21-Jenkins/)

[.NET实现持续集成与自动化部署2-NuGet](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B22-NuGet/)

[.NET实现持续集成与自动化部署3-测试环境到生产环境策略](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B23-%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83%E5%88%B0%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E7%AD%96%E7%95%A5/)

### 一、下载Nuget.Server
&nbsp;&nbsp;&nbsp;&nbsp;从官方Nuget服务器上搜索nuget.server,点击项目url中的github路径。从github中下载nuget.server的源码。
![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180625113552103-109706720.png)
下载并解压后的文件路径如下图所示:
![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180625113951128-1503563884.png)

### 二、搭建Nuget.Server
1. 打开项目文件NuGet.Server.sln,找到NuGet.Server,右键发布，选择文件系统(跟发布web程序一样，发布到IIS中)。
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180625114642621-262211103.png)
2. IIS新建站点MyNuGet
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180625114859263-2006416616.png)
    启动程序出现以下页面代表搭建成功
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180625115218641-1622168019.png)
3. 注意:若点击here出现404页面如下图所示:
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180625115706412-729145768.png)
    可以通过VS运行起来Nuget.Server项目，然后将bin目录替换IIS下的bin目录，即可解决。出现下图代表搭建成功
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627102545835-1004233709.png)
    打开VS的Nuget管理器，点击图中设置图标，新建我们自己的nuget服务器
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627103124004-1628086707.png)
    之后就可以连上我们自己搭建的服务器了
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627103239225-49848535.png)
### 三、自建NuGet基本使用
1. 下载NuGet命令行打包工具nuget.exe
    下载地址:https://www.nuget.org/downloads

2. 打包我们程序
* 方式1：通过类库文件csproj的方式打包
  首先打开我们程序的AssemblyInfo.cs文件修改程序集信息
  ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627104018738-2049830247.png)

使用nuget.exe打包程序集<br>
在.csproj文件目录下执行命令spec
```
nuget.exe spec //spec 在.csproj文件目录下执行命令
```
![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627104333535-1316984971.png)
此时会生成一个.nuspec文件，打开这个文件
修改其中的xml属性即可(注意此处一些信息最好和AssemblyInfo.cs中的程序集信息一致)
![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627105343352-1672213188.png)
修改完成后继续执行pack命令
![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627105438149-788832720.png)
这时将生成的.nupkg文件直接copy到nuget服务器IIS目录下的packages文件夹内即可
也可通过命令push推送至nuget服务器
```
nuget push *.nupkg -s http://127.0.0.1:8005 123456 //push 程序包路径 选项 地址 apikey
//apikey 可以在服务器webconfig中配置
```
完成后即可查看或使用我们发布的程序集
![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627113647970-1188544280.png)

![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627113704127-1201145466.png)

### 四、问题
1. 在刚开始使用的时候经常因为失误或者程序有问题从而导致需要重新发布nuget包，但是发现覆盖掉原来的之后，项目里更新下来的始终还是原来的程序。
    解决:慎重慎重再慎重打包，需要重新发布包的时候可以升级，不能覆盖。(当时认为这个东西只能升级不能覆盖)
2. 用了一段时间后，由于当时至提供了nuget管理包的技术方案，却没有相应的使用规范与制度，导致团队nuget包混乱，开发人员胡乱升级，胡乱引用nuget包，终于有一天造成大问题。因此需要制定一个完善的使用规范与制度，包括如何打包，如何发布，谁来打包，谁来发布，慎重打包、升级、专人管理等
3. 由于问题2引起的问题，因此决定重新整理nuget包(不破不立)，于是重新搭建了一个nuget服务器，重新规整虽有的程序集、组件、重新打包发布等，但是发现迁移到新的后，项目中下载下来的程序集还是原来的。(又遇到了问题1)。这次灵感一来发现问题解决方案。VS2017通过工具->选项->清除所有NuGet缓存 再重新下载包问题即可解决
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627114835612-1565198402.png)
    若没有VS2017或找不到清楚NuGet缓存选项，也可找到自己机器上nuget的缓存文件夹删除掉里面对应的内容也可以，一般是在C:\Users\Administrator\.nuget
    ![](https://images2018.cnblogs.com/blog/740814/201806/740814-20180627115244329-208915392.png)
    </font>