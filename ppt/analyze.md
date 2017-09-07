# 分析

随着基础设施市场规模下降，大家开始更关注云平台上线并运行之后会发生什么。

业界的焦点开始将云向上移，这开始变成一件严肃的事情。

![](images/up.png)

有一个思路是，开发人员照常创建应用程序，然后通过Istio运行其部署，Istio负责设置所有的部分。一旦完成，您只需简单地查看Grafana仪表板就能知道应用程序的状况。

在过去几年中，我们一直致力于使基础设施建成运行，但业界现在已经达到了一个转折点，默认基础设施是一件简单的事情; 现在关注的是结果。
在Draft和Istio这两个案例中，（我们看到的是）使开发更容易，使公司能够更快地得到这些结果，但除此之外，（我们还能看到）关于分解。

都将开发人员解放出来，他们只需要处理自己正在进行的开发，而无需操心那些与此没有直接关系的架构上的琐碎事情。这样，开发人员可以更轻松地创建应用程序，这些应用程序不仅仅是云原生的，而且还具有更多的云独立性，这为混合云应用打开了大门，并进一步强调了需要做什么，而不仅仅是如何做。


Kubernetes changed how we deploy applications. Istio is going to change how we connect, manage, and secure them.


## 想法

正如应用程序不应该编写自己的TCP堆栈一样，他们也不应该管理自己的负载平衡逻辑或其自己的服务发现管理，或他们自己的重试和超时逻辑。

### IBM

https://developer.ibm.com/dwblog/2017/istio/

我们亲眼目睹了我们大型企业客户走向云端的这一趋势。

从Blog中得到的信息,IBM加入istio的原因和动机:

We have personally witnessed this trend with our large enterprise clients as they move to the cloud. As microservices scale dynamically, problems such as service discovery, load balancing and failure recovery become increasingly important to solve uniformly. The individual development teams manage and make changes to their microservices independently, making it difficult to keep all of the pieces working together as a single unified application. Often, we see customers build custom solutions to these challenges that are unable to scale even outside of their own teams.
