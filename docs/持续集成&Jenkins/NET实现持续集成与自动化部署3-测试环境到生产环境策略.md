## .NET实现持续集成与自动化部署3-测试环境到生产环境策略

### 一、前言
&nbsp;&nbsp;&nbsp;&nbsp;前面我们已经初步实现了开发集成环境、测试环境的持续集成(自动化构建、自动化测试、自动化部署)。但生产环境自动化部署迟迟没有推进。其原因主要在以下几个方面:
* 尚未实现部署之前的自动化备份
* 尚未实现部署出现问题后的自动化回滚
* 由于之前采用FTP上传部署需要生产环境开放FTP端口存在安全性问题且FTP会因为各种的网速问题，导致站点瞬间挂掉

只要解决以上三个问题，我们就可以初步实现生产环境的自动化部署。

<!--more-->

### 系列文章

[.NET实现持续集成与自动化部署1-Jenkins](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B21-Jenkins/)

[.NET实现持续集成与自动化部署2-NuGet](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B22-NuGet/)

[.NET实现持续集成与自动化部署3-测试环境到生产环境策略](http://blog.loading.ink/2018/11/13/NET%E5%AE%9E%E7%8E%B0%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E4%B8%8E%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B23-%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83%E5%88%B0%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E7%AD%96%E7%95%A5/)

### 二、实现思路
1. 利用Jenkins分布式的特性，其中Jenkins服务器作为Master服务器，将生产环境(可以一台也可以多台服务器)作为Jenkins集群中的一台Slave服务器。
2. 测试环境应该模拟和生产环境的配置和编译版本保持是Release状态，且功能已经满足预期发布需求。
3. 通过文件复制插件，复制测试环境上的部署文件到生产环境上的jenkins工作空间。
4. 通过批处理处理不需要覆盖的文件或者临时要修改的配置等。
5. 利用rar备份生成环境上即将要覆盖的文件，注意命名上遵循一定规律：项目-文件夹-{BuildID}.bak.rar或日期-项目-文件夹-{BuildID}.bak.rar。
6. 利用批处理进行从jenkins工作空间上把文件复制到站点上，常用命令：xcopy。
7. 若生产环境程序出现问题，由项目经理和运维人员决定是紧急修复bug还是启用回滚，回滚则采用批处理命令将备份的文件压缩回生产环境站点下的目录内。

通过以上策略可以实现测试环境到生产环境的一键部署，实现了部署前的自动化备份，出现问题的自动化回滚，利用Jenkins Master-Slave特性解决了需要开放FTP端口的的问题，并且将先在测试站点测试好的文件，复制到正式站点上的一个缓冲区，进行预热配置，之后在本机进行文件替换，速度是相当的快，解决了FTP上传过程中网络问题导致站点挂掉的问题。

缺陷与问题:
1. 生产环境需作为Jenkins 集群中的一台服务器并承担一部分构建任务，但通过配置此问题可忽略不计
2. 生产环境需安装JDK并开启一个Java服务
3. 待发现

### 三、生产环境拓扑图
![image](http://img.heshang365.com/group1/M00/05/F3/wKgR6Vr-uJiAAWNJAAGzH04VEaI361.png)

### 四、所需Jenkins插件
1. Copy data to workspace plugin 插件
2. Copy Artifact Plugin
3. Node and Label parameter plugin 插件

### 五、实现步骤

1. 搭建slave

1.1 Jenkins系统管理-->管理节点-->新建节点
!![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-oHuAV0cyAAGa8kLPPek104.png)
![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-oPSAWrxrAAA084f8-fE141.png)

1.2 输入节点名称，next，配置如下图
![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-o8iAUs6GAAFSPdUxR4A459.png)

其中，有如下几点需要注意：

* 【# of executors】根据CPU的个数来填写数量

* 【远程工作目录】这个就是用来存放master到slave时，存放的临时目录，如slave的服务软件也会放在此，并且会以每个job名称来区分开

* 【用法】只需要选择【只允许运行绑定到这台机器的Job】这种模式下，Jenkins只会构建哪些分配到这台机器的Job。这允许一个节点专门保留给某种类型的Job。例如，在Jenkins上连续的执行测试，你可以设置执行者数量为1，那么同一时间就只会有一个构建，一个实行者不会阻止其它构建，其它构建会在另外的节点运行。通过这个配置生产环境就可以仅做自己的构建。

* 【启动方式】只需要选择【Launch agent via Java Web Start】，以服务的方式启动，应用最广且最好配置，其余的都太复杂，不建议使用。注意：2.x版本的默认没有这个选项，需要单独开启。其余的基本按照上面默认选择即可。

Launch agent via Java Web Start开启方式:

Jenkins-->系统管理-->Configure Global Security-->Agents-->修改为随机选取
![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-ol2AGhRPAAIRpLCLKyU487.png)
![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-otCAPdcXAAK5mBHFsQ4907.png)


![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-pOiAODw6AADdSdWn-5Y467.png)

1.3 点击保存后，master上已经配置好节点，那么接下来就是到节点的服务器上安装slave的服务：
点击右侧列表的节点服务器，此时节点并未连通。
![image](http://img.heshang365.com/group1/M00/05/EE/wKgR6Vr-pOiAODw6AADdSdWn-5Y467.png)
进入详情页面，会提示你如何安装服务：
![image](http://img.heshang365.com/group1/M00/05/EF/wKgR6Vr-pqyAYBBMAAD-3AmzeqA038.png)

**注意:由于Slave服务为Java服务，因此Slave服务器上需安装JDK**

当Slave服务器上出现以下服务时代表安装并连接成功
![image](http://img.heshang365.com/group1/M00/05/EF/wKgR6Vr-qQ2AIlCVAABvoYcszKc472.png)
此时回到Jenkins 服务器上查看状态已经连接上
![image](http://img.heshang365.com/group1/M00/05/EF/wKgR6Vr-qYGAMXQUAABmxXOUQsw465.png)

说明:这里只介绍基于现有需求的一种策略，关于Jenkins Master-Slave连接机制与原理不多做介绍，网上关于这方面的介绍也很多，大家可以自行搜索。

2. 创建生产环境自动化部署任务
    2.1 参数化配置选择Slave构建
    Jenkins 新建自由风格的软件项目
    ![image](http://img.heshang365.com/group1/M00/05/F0/wKgR6Vr-q_mAW2c7AAG4v5WRAgo962.png)
    参数化构建-->添加参数-->选择node
    ![image](http://img.heshang365.com/group1/M00/05/F0/wKgR6Vr-q_yAD07rAADt18hYFRc490.png)
    若没有此参数安装Node and Label parameter plugin 插件
    参数化配置可按下图进行，也可根据需要自行配置
    ![image](http://img.heshang365.com/group1/M00/05/F0/wKgR6Vr-rACAWacKAADN3fY9JCs868.png)

2.2 文本复制
文本复制可选择两个插件

Copy data to workspace plugin 插件 <br>
可以复制Jenkins Master服务器的文件到Slave工作空间内
缺点:不支持参数化
Copy Artifact Plugin 插件 <br>
可以实现Jenkins Slave-Slave Master-Slave之间的复制，可以将一个Job构建后的生成物复制到当前工作空间内
缺点:需再要复制的Job内内配置Archive the artifact

可以根据所需自行选择插件，这里为了能够参数化我们选择Copy Artifact Plugin插件
![image](http://img.heshang365.com/group1/M00/05/F1/wKgR6Vr-rquAaNfPAACQylX60q0644.png)
配置说明：
* Project name:要Copy的项目名称，这里可以使用参数化
* Which build:选择那一次构建后的产物，一般可以选择Latest successful build
* Stable build only:是否选择稳定的构建
* Artifacts to copy:要Copy的文件，可以进行规则匹配，如Test/**/*,即Test文件夹下所有文件
* Artifacts not to copy：根据规则排除某些文件
* Target directory：本地工作空间的那个文件夹内
* Parameter filters：这里没用到，用到的话，可以自己看说明

注意:这里需要前置Job配置<br>
在要复制的Job内增加构建后操作如下图:
![image](http://img.heshang365.com/group1/M00/05/F3/wKgR6Vr-tu2AOR3ZAABUEIVOPVg241.png)

2.3 自动化备份
填写备份的批处理，这里可以使用WindowsRAR的压缩命令，所以如果要用RAR的时候，确保机器上已经安装WindowsRAR。注意名称必须要有规则且每次构建不能重复，因此可以使用项目名称+BuildID或者日期+项目名称+BuildID
![image](http://img.heshang365.com/group1/M00/05/F1/wKgR6Vr-sHOAaY2wAABFNf6cB0U792.png)
```
//自动备份批处理命令
start c:\"Program Files"\winrar\rar.exe a -k -r -s -m1
-ag{HS.Shop.My-%BUILD_ID%.bak}  {要备份到的文件夹} {要备份的文件夹}
```

2.4 覆盖站点目录下的文件
备份完成后将Jenkins工作空间下的文件复制到站点目录下，此时必须保证发布包已经排除掉了不需要覆盖的文件，并且是稳定可用的版本。批处理命令可采用xcopy命令。关于xcopy命令的使用可以自行百度
![image](http://img.heshang365.com/group1/M00/05/F2/wKgR6Vr-sYSAfE-kAABAnVmx6ng732.png)
```
xcopy  {slave工作空间上的项目文件夹} {要复制到替换的文件夹}  /Y/E
```
到了这一步就完成了生产环境的自动化部署的任务配置。点击构建即可完成测试环境到生产环境的一键部署。此时若程序出现问题可以采用紧急修复或者自动化回滚。

3. 创建生产环境自动化回滚任务

3.1 同样新建一个自由风格的软件项目
这里可以配置两个构建参数<br>
1.回滚哪一个项目的哪一次构建<br>
2.回滚哪一台服务器的构建(可以多台)<br>
参数化配置可见下图
![image](http://img.heshang365.com/group1/M00/05/F2/wKgR6Vr-s1-ACsgMAAEgMDxv_aI903.png)

3.2 自动化回滚批处理
![image](http://img.heshang365.com/group1/M00/05/F3/wKgR6Vr-s-KALO4lAAC2T8Sgrxk323.png)
```
Setlocal enabledelayedexpansion
set "projectKey=ChioceBuild"
set "bakUrl=D:\HS.Shop.Bak\HS.Shop.My\"  //备份文件的路径
set url="%ChioceBuild%"                  //参数
set "rollbackUrl=D:\"
set "projectName="
set "buildID="
set url=%url::=/%
set url=%url:///=/%
set url=%url://=/%

for /f "tokens=1,2,3,4,5,6,7,8* delims=/" %%a in (%url%) do (
 set "projectName=%%g"
 set "buildID=%%h"
)
set projectName=!projectName:%projectKey%=!
set "fileName="

for %%a in (%bakUrl%%projectName%-%buildID%.bak.rar) do (
 set "fileName=%%a"
)
c:\"Program Files"\winrar\rar.exe x -ep2 -o+- %fileName% %rollbackUrl%

```
点击保存即可完成自动化回滚任务的建立，点击构建选择参数即可进行回滚。

### 六、结束语
&nbsp;&nbsp;&nbsp;&nbsp;Jenkins是一个持续集成工具，其功能非常强大，可以帮助我们做自动构建、自动测试、自动发布等等，它根据不同的需求实现各种各样的功能，它可以最大幅度的减少我们日常工作中重复性的工作。以上仅仅是我根据当下所需研究的一种使用策略，可能有漏洞，也可能存在问题，但如果不愿意尝试着去改进现有流程，去接受新的东西，那么我们永远不会进步。而我对其使用的了解也不过是九牛一毛，大家可以根据需求研究制定自己的使用策略。<br>
&nbsp;&nbsp;&nbsp;&nbsp;最后希望大家:即使搬砖，也要搬出艺术感!做一个有追求的搬砖者!
</font>