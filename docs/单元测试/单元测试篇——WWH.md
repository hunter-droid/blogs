## 持续集成之单元测试篇——WWH(讲讲我们做单元测试的故事)

### 前言

- 临近上线的几天内非重大bug不敢进行发版修复，担心引起其它问题(摁下葫芦浮起瓢)
- 尽管我们如此小心，仍不能避免修改一些bug而引起更多的bug的现象
- 往往有些bug已经测试通过了但是又复现了
- 我们明明没有改动过的功能，却出了问题
- 有些很明显的bug往往在测试后期甚至到了线上才发现，而此时修复的代价极其之大。
- 测试时间与周期太长并且质量得不到保障
- 项目与服务越来越多，测试人员严重不足（后来甚至一个研发两个测试人员比）
- 上线的时候仅仅一轮回归测试就需要几个小时甚至更久
- 无休止的加班上线。。。

如果你对以上问题非常熟悉，那么我想你的团队和我们遇到了相同的问题。

WWH:Why,What,How为什么要做单元测试，什么事单元测试，如何做单元测试。

### 一、为什么我们要做单元测试

#### 1.1 问题滋生解决方案——自动化测试

&nbsp;&nbsp;&nbsp;&nbsp;一门技术或一个解决方案的诞生的诞生，不可能凭空去创造，往往是问题而催生出来的。在我的[.NET持续集成与自动化部署之路第一篇(半天搭建你的Jenkins持续集成与自动化部署系统)](https://www.cnblogs.com/hunternet/p/9590287.html)这篇文章中提到，我在做研发负责人的时候饱受深夜加班上线之苦，其中提到的两个大问题一个是部署问题，另一个就是测试问题。部署问题，我们引入了自动化的部署(后来我们做到了几分钟就可以上线)。我们要做持续集成，剩下的就是测试问题了。

![持续集成](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-10-7/73716618.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;回归测试成了我们的第一大问题。随着我们项目的规模与复杂度的提升，我们的回归测试变得越来越困难。由于我们的当时的测试全依赖手工测试，我们项目的迭代周期大概在一个月左右，而测试的时间就要花费一半多的时间。甚至版本上线以后做一遍回归测试就需要几个小时的时间。而且这种手工进行的功能性测试很容易有遗漏的地方，因此线上Bug层出不穷。一堆问题困扰着我们，我们不得不考虑进行自动化的测试。

&nbsp;&nbsp;&nbsp;&nbsp;自动化测试同样不是银弹，自动化测试虽然与手工测试相比有其优点，其测试效率高，资源利用率高(一般白天开发写用例，晚上自动化程序跑)，可以进行压力、负载、并发、重复等人力不可完成的测试任务，执行效率较快，执行可靠性较高，测试脚本可重复利用，bug及时发现.......但也有其不可避免的缺点，如:只适合回归测试，开发中的功能或者变更频繁的功能，由于变更频繁而不断更改测试脚本是不划算的，并且脚本的开发也需要高水平的测试人员和时间......总体来说，虽然自动化的测试可以解决一部分的问题，但也同样会带来另一些问题。到底应该不应该引入自动化的测试还需要结合自己公司的团队现状来综合考虑。

&nbsp;&nbsp;&nbsp;&nbsp;而我们的团队从短期来看引入自动化的测试其必然会带来一些问题，但长远来看其优点还是要大于其缺陷的，因此我们决定做自动化的测试，当然这具体是不是另一个火坑还需要时间来判定！

#### 1.2 认识自动化测试金字塔

  ![测试金字塔](http://hunter-image.oss-cn-beijing.aliyuncs.com/18-9-27/99236945.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;以上便是经典的自动化测试金字塔。

&nbsp;&nbsp;&nbsp;&nbsp;位于金字塔顶端的是探索性测试，探索性测试并没有具体的测试方法，通常是团队成员基于对系统的理解，以及基于现有测试无法覆盖的部分，做出系统性的验证，譬如：跨浏览器的测试,一些视觉效果的测试等。探索性测试由于这类功能变更比较频繁，而且全部实现自动化成本较高，因此小范围的自动化的测试还是有效的。而且其强调测试人员的主观能动性，也不太容易通过自动化的测试来实现，更多的是手工来完成。因此其成本最高，难度最大，反馈周期也是最慢的。

&nbsp;&nbsp;&nbsp;&nbsp;而在测试金字塔的底部是单元测试,单元测试是针对程序单元的检测，通常单元测试都能通过自动化的方式运行,单元测试的实现成本较低，运行效率较高，能够通过工具或脚本完全自动化的运行，此外，单元测试的反馈周期也是最快的，当单元测试失败后,能够很快发现，并且能够较容易的找到出错的地方并修正。重要的事单元测试一般由开发人员编写完成。(这一点很重要，因为在我这个二线小城市里，能够编写代码的测试人员实在是罕见！)

&nbsp;&nbsp;&nbsp;&nbsp;在金字塔的中间部分，自底向上还包括接口(契约)测试，集成测试，组件测试以及端到端测试等，这些测试侧重点不同，所使用的技术方法工具等也不相同。

&nbsp;&nbsp;&nbsp;&nbsp;总体而言，在测试金字塔中，从底部到顶部业务价值的比重逐渐增加，即越顶部的测试其业务价值越大，但其成本也越来越大，而越底部的测试其业务价值虽小，但其成本较低，反馈周期较短，效率也更高。

#### 1.3 从单元测试开始

&nbsp;&nbsp;&nbsp;&nbsp;我们要开始做自动化测试，但不可能一下子全都做(考虑我们的人力与技能也做不到)。因此必须有侧重点，考虑良久最终我们决定从单元测试开始。于是我在刚吃了自动化部署的螃蟹之后，不得不来吃自动化测试的第一个螃蟹。既然决定要做，那么我们就要先明白单元测试是什么？

### 二、单元测试是什么

#### 2.1 什么是单元测试。

&nbsp;&nbsp;&nbsp;&nbsp;我们先来看几个常见的对单元测试的定义。

&nbsp;&nbsp;&nbsp;&nbsp;用最简单的话说：单元测试就是针对一个工作单元设计的测试，这里的“工作单元”是指对一个工作方法的要求。

&nbsp;&nbsp;&nbsp;&nbsp;单元测试是开发者编写的一小段代码，用于检测被测代码的一个很小的、很明确的功能是否正确。通常而言，一个单元测试用于判断某个特定条件(或场景)下某个特定函数的行为。

例：

&nbsp;&nbsp;&nbsp;&nbsp;你可能把一个很大的值放入一个有序list中去，然后确认该值出现在list的尾部。或者，你可能会从字符串中删除匹配某种模式的字符，然后确认字符串确实不再包含这些字符了。

**执行单元测试，就是为了证明某段代码的行为和开发者所期望的一致！**

#### 2.2 什么不是单元测试

&nbsp;&nbsp;&nbsp;&nbsp;这里我们暂且先将其分为三种情况

##### 2.2.1 跨边界的测试

&nbsp;&nbsp;&nbsp;&nbsp;单元测试背后的思想是，仅测试这个方法中的内容，测试失败时不希望必须穿过基层代码、数据库表或者第三方产品的文档去寻找可能的答案！

&nbsp;&nbsp;&nbsp;&nbsp;当测试开始渗透到其他类、服务或系统时，此时测试便跨越了边界，失败时会很难找到缺陷的代码。

&nbsp;&nbsp;&nbsp;&nbsp;测试跨边界时还会产生另一个问题，当边界是一个共享资源时，如数据库。与团队的其他开发人员共享资源时，可能会污染他们的测试结果！

##### 2.2.2 不具有针对性的测试

&nbsp;&nbsp;&nbsp;&nbsp;如果发现所编写的测试对一件以上的事情进行了测试，就可能违反了“单一职责原则”。从单元测试的角度来看，这意味着这些测试是难以理解的非针对性测试。随着时间的推移，向类或方法种添加了更多的不恰当的功能后，这些测试可能会变的非常脆弱。诊断问题也将变得极具有挑战性。

&nbsp;&nbsp;&nbsp;&nbsp;如：StringUtility中计算一个特定字符在字符串中出现的次数，它没有说明这个字符在字符串中处于什么位置也没有说明除了这个字符出现多少次之外的其他任何信息，那么这些功能就应该由StringUtility类的其它方法提供！同样，StringUtility类也不应该处理数字、日期或复杂数据类型的功能！

2.2.3 不可预测的测试

&nbsp;&nbsp;&nbsp;&nbsp;单元测试应当是可预测的。在针对一组给定的输入参数调用一个类的方法时，其结果应当总是一致的。有时，这一原则可能看起来很难遵守。例如：正在编写一个日用品交易程序，黄金的价格可能上午九时是一个值，14时就会变成另一个值。

&nbsp;&nbsp;&nbsp;&nbsp;好的设计原则就是将不可预测的数据的功能抽象到一个可以在单元测试中模拟(Mock)的类或方法中(关于Mock请往下看)。 

### 三、如何去做单元测试

#### 3.1 单元测试框架
&nbsp;&nbsp;&nbsp;&nbsp;在单元测试框架出现之前，开发人员在创建可执行测试时饱受折磨。最初的做法是在应用程序中创建一个窗口，配有"测试控制工具(harness)"。它只是一个窗口，每个测试对应一个按钮。这些测试的结果要么是一个消息框，要么是直接在窗体本身给出某种显示结果。由于每个测试都需要一个按钮，所以这些窗口很快就会变得拥挤、不可管理。

  由于人们编写的大多数单元测试都有非常简单的模式：

* 执行一些简单的操作以建立测试。

* 执行测试。

* 验证结果。

* 必要时重设环境。

于是，单元测试框架应运而生(实际上就像我们的代码优化中提取公共方法形成组件)。

&nbsp;&nbsp;&nbsp;&nbsp;单元测试框架(如NUnit)希望能够提供这些功能。单元测试框架提供了一种统一的编程模型，可以将测试定义为一些简单的类，这些类中的方法可以调用希望测试的应用程序代码。开发人员不需要编写自己的测试控制工具；单元测试框架提供了测试运行程序(runner)，只需要单击按钮就可以执行所有测试。利用单元测试框架，可以很轻松地插入、设置和分解有关测试的功能。测试失败时，测试运行程序可以提供有关失败的信息，包含任何可供利用的异常信息和堆栈跟踪。

​    .Net平台常用的单元测试框架有：MSTesting、Nunit、Xunit等。

#### 3.2 简单示例(基于Nunit)

```c#
    /// <summary>
    /// 计算器类
    /// </summary>
    public class Calculator
    {
        /// <summary>
        /// 加法
        /// </summary>
        /// <param name="a"></param>
        /// <param name="b"></param>
        /// <returns></returns>
        public double Add(double a, double b)
        {
            return a + b;
        }

        /// <summary>
        /// 减法
        /// </summary>
        /// <param name="a"></param>
        /// <param name="b"></param>
        /// <returns></returns>
        public double Sub(double a, double b)
        {
            return a - b;
        }

        /// <summary>
        /// 乘法
        /// </summary>
        /// <param name="a"></param>
        /// <param name="b"></param>
        /// <returns></returns>
        public double Mutiply(double a, double b)
        {
            return a * b;
        }

        /// <summary>
        /// 除法
        /// </summary>
        /// <param name="a"></param>
        /// <param name="b"></param>
        /// <returns></returns>
        public double Divide(double a, double b)
        {
            return a / b;
        }
    }
```

```c#
    /// <summary>
    /// 针对计算加减乘除的简单的单元测试类
    /// </summary>
    [TestFixture]
    public class CalculatorTest
    {
        /// <summary>
        /// 计算器类对象
        /// </summary>
        public Calculator Calculator { get; set; }

        /// <summary>
        /// 参数1
        /// </summary>
        public double NumA { get; set; }

        /// <summary>
        /// 参数2
        /// </summary>
        public double NumB { get; set; }

        /// <summary>
        /// 初始化
        /// </summary>
        [SetUp]
        public void SetUp()
        {
            NumA = 10;
            NumB = 20;
            Calculator = new Calculator();
        }

        /// <summary>
        /// 测试加法
        /// </summary>
        [Test]
        public void TestAdd()
        {
            double result = Calculator.Add(NumA, NumB);
            Assert.AreEqual(result, 30);
        }

        /// <summary>
        /// 测试减法
        /// </summary>
        [Test]
        public void TestSub()
        {
            double result = Calculator.Sub(NumA, NumB);
            Assert.LessOrEqual(result, 0);
        }

        /// <summary>
        /// 测试乘法
        /// </summary>
        [Test]
        public void TestMutiply()
        {
            double result = Calculator.Mutiply(NumA, NumB);
            Assert.GreaterOrEqual(result, 200);
        }
        
        /// <summary>
        /// 测试除法
        /// </summary>
        [Test]
        public void TestDivide()
        {
            double result = Calculator.Divide(NumA, NumB);
            Assert.IsTrue(0.5 == result);
        }
    }
```

#### 3.3 如何做好单元测试

​    单元测试是非常有魔力的魔法，但是如果使用不恰当亦会浪费大量的时间在维护和调试上从而影响代码和整个项目。

好的单元测试应该具有以下品质:

•  自动化  

•  彻底的

•  可重复的

•  独立的

•  专业的

##### 3.3.1 测试哪些内容

​    一般来说有六个值得测试的具体方面，可以把这六个方面统称为Right-BICEP:

* Right----结果是否正确？
* B----是否所有的边界条件都是正确的？
* I----能否检查一下反向关联？C----能否用其它手段检查一下反向关联？
* E----是否可以强制产生错误条件？
* P----是否满足性能条件？

##### 3.3.2 CORRECT边界条件
​    代码中的许多Bug经常出现在边界条件附近，我们对于边界条件的测试该如何考虑？

* 一致性----值是否满足预期的格式
* 有序性----一组值是否满足预期的排序要求
* 区间性----值是否在一个合理的最大值最小值范围内
* 引用、耦合性----代码是否引用了一些不受代码本身直接控制的外部因素
* 存在性----值是否存在（例如：非Null，非零，存在于某个集合中）
* 基数性----是否恰好具有足够的值
* 时间性----所有事情是否都按照顺序发生的？是否在正确的时间、是否及时

##### 3.3.3 使用Mock对象

&nbsp;&nbsp;&nbsp;&nbsp;单元测试的目标是一次只验证一个方法或一个类，但是如果这个方法依赖一些其他难以操控的东西，比如网络、数据库等。这时我们就要使用mock对象，使得在运行unit test的时候使用的那些难以操控的东西实际上是我们mock的对象，而我们mock的对象则可以按照我们的意愿返回一些值用于测试。通俗来讲，Mock对象就是真实对象在我们调试期间的测试品。

Mock对象创建的步骤:    

1. 使用一个接口来描述这个对象。

2. 为产品代码实现这个接口。

3. 以测试为目的，在mock对象中实现这个接口。

Mock对象示例:

```c#
   /// <summary>
    ///账户操作类
    /// </summary>
    public class AccountService
    {
        /// <summary>
        /// 接口地址
        /// </summary>
        public string Url { get; set; }

        /// <summary>
        /// Http请求帮助类
        /// </summary>
        public IHttpHelper HttpHelper { get; set; }
        /// <summary>
        /// 构造函数
        /// </summary>
        /// <param name="httpHelper"></param>
        public AccountService(IHttpHelper httpHelper)
        {
            HttpHelper = httpHelper;
        }

        #region 支付
        /// <summary>
        /// 支付
        /// </summary>
        /// <param name="json">支付报文</param>
        /// <param name="tranAmt">金额</param>
        /// <returns></returns>
        public bool Pay(string json)
        {            
            var result = HttpHelper.Post(json, Url);
            if (result == "SUCCESS")//这是我们要测试的业务逻辑
            {
                return true;
            }
            return false;
        }
        #endregion

        #region 查询余额
        /// <summary>
        /// 查询余额
        /// </summary>
        /// <param name="account"></param>
        /// <returns></returns>
        public decimal? QueryAmt(string account)
        {
            var url = string.Format("{0}?account={1}", Url, account);

            var result = HttpHelper.Get(url);

            if (!string.IsNullOrEmpty(result))//这是我们要测试的业务逻辑
            {
                return decimal.Parse(result);
            }
            return null;
        }
        #endregion

    }
```

```c#
 	/// <summary>
    /// Http请求接口
    /// </summary>
    public interface IHttpHelper
    {
        string Post(string json, string url);

        string Get(string url);
    }
```

```c#
	/// <summary>
    /// HttpHelper
    /// </summary>
    public class HttpHelper:IHttpHelper
    {
        public string Post(string json, string url)
        {
            //假设这是真实的Http请求
            var result = string.Empty;
            return result;
        }

        public string Get(string url)
        {
            //假设这是真实的Http请求
            var result = string.Empty;
            return result;
        }

    }
```

```c#
    /// <summary>
    /// Mock的 HttpHelper
    /// </summary>
    public class MockHttpHelper:IHttpHelper
    {
        public string Post(string json, string url)
        {
            //这是Mock的Http请求
            var result = "SUCCESS";
            return result;
        }

        public string Get(string url)
        {
            //这是Mock的Http请求
            var result = "0.01";
            return result;
        }

   }
```



&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;如上，我们的AccountService的业务逻辑依赖于外部对象Http请求的返回值在真实的业务中我们给AccountService注入真实的HttpHelper类，而在单元测试中我们注入自己Mock的HttpHelper，我们可以根据不同的用例来模拟不同的Http请求的返回值来测试我们的AccountService的业务逻辑。

**注意:记住，我们要测试的是AccountService的业务逻辑:根据不同http的请求(或传入不同的参数)而返回不同的结果，一定要弄明白自己要测的是什么！而无关的外部对象内的逻辑我们并不关心，我们只需要让它给我们返回我们想要的值，来验证我们的业务逻辑即可**

&nbsp;&nbsp;&nbsp;&nbsp;关于Mock对象一般会使用Mock框架，关于Mock框架的使用，我们将在下一篇文章中介绍。.net 平台常用的Mock框架有Moq,PhinoMocks,FakeItEasy等。

#### 3.4 单元测试之代码覆盖率

​    在做单元测试时，代码覆盖率常常被拿来作为衡量测试好坏的指标，甚至，用代码覆盖率来考核测试任务完成情况，比如，代码覆盖率必须达到80％或90％。于是乎，测试人员费尽心思设计案例覆盖代码。因此我认为用代码覆盖率来衡量是不合适的，我们最根本的目的是为了提高我们回归测试的效率，项目的质量不是吗?    

### 结束语

&nbsp;&nbsp;&nbsp;&nbsp;本篇文章主要介绍了单元测试的WWH，分享了我们为什么要做单元测试并简单介绍了单元测试的概念以及如何去做单元测试。当然，千万不要天真的以为看了本篇文章就能做好单元测试，如果你的组织开始推进了单元测试，那么在推进的过程中相信仍然会遇到许多问题(就像我们遇到的，依赖外部对象问题，静态方法如何mock......)。如何更好的去做单元测试任重而道远。下一篇文章将针对我们具体实施推进单元测试中遇到的一些问题，来讨论如何更好的做单元测试。如:如何破除依赖，如何编写可靠可维护的测试，以及如何面向测试进行程序的设计等。

​    未完待续，敬请关注......

### 参考

《单元测试的艺术》

 我们做单元测试的经历