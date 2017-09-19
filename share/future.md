# 未来

前面我们介绍了 istio 的基本情况, 还有istio的架构和主要组件. 相信大家对 istio 应该有了一个初步的认识.

需要提醒的是, istio是一个今年5月才发布 0.1.0 版本的新鲜出炉的开源项目, 目前该项目也才发布到 0.1.6 正式版本和 0.2.2 pre release 版本. 很多地方还不完善，希望大家可以理解，有点类似于最早期阶段的Kubernetes.

在接下来的时间, 我们将简单介绍一下 istio 后面的一些开发计划和发展预期.

## 运行环境支持

Istio 目前只支持 Kubernetes , 这是令人比较遗憾的一点. 不过 istio 给出的解释是 istio 未来会支持在各种环境中运行，只是目前在 0.1/0.2 这样的初始阶段暂时专注于Kubernetes，但很快会支持其他环境。

注意: Kubernetes平台，除了原生Kubernetes, 还有诸如 IBM Bluemix Container Service 和 RedHat OpenShift 这样的商业平台。 以及 google 自家的 Google Container Engine。这是自家的东西, 而且现在 k8s/istio/gRPC 都已经被划归到 google cloud platform 部门, 自然会优先支持.

另外isito所说的其他环境指的是:

- mesos: 这个估计是大多人非k8s的docker使用者最关心的了, 暂时从github上的代码中未见到有开工迹象, 但是istio的文档和官方声明都明显说会支持, 估计还是希望很大的.
- cloud foundry: 这个东东我们国内除了私有云外玩的不多, istio对它的支持似乎已经启动. 比如我看到代码中已经有了consul这个服务注册的支持, 从issue讨论上看到是说为上cloud foundry做准备, 因此cloud foundry没有k8s那样的原生服务注册机制.
- VM: 这块没有看到介绍, 但是有看到istio的讨论中提到会支持容器和非容器的混合(hybrid)支持

值得特别之处的是, 目前我还没有看到 istio 有对 docker 家的 swarm 有支持的计划或者讨论, 目前我找到的任何 istio 的资料中都不存在 swarm 这个东东. 我只能不负责任的解释说: 有人的地方就有江湖, 有人的地方就有恩怨.

## 路线图

按照istio的说法, 他们计划每3个月发布一次新版本, 我们看一下目前得到的一些信息:

- 0.1 版本2017年5月发布,只支持Kubernetes
- 0.2 即将发布,当前是0.2.1 pre-release, 也只支持Kubernetes
- 0.3 roadmap上说要支持k8s之外的平台, "Support for Istio meshes without Kubernetes.", 但是具体哪些特性会放在0.3中,还在讨论中.
- 1.0 版本预计今年年底发布

注: 1.0 版本的发布时间官方没有明确给出, 我只是看到官网资料里面有信息透露如下:

> "we invite the community to join us in shaping the project as we work toward a 1.0 release later this year."

按照上面给的信息,简单推算, 应该是9月发 0.2, 然后12月发 0.3, 但是这就已经是年底了, 所以不排除 1.0 推迟发布的可能, 或者 0.3 直接当成 1.0 发布.

## 社区支持

虽然istio初出江湖, 乳臭未干, 但是凭借google和IBM的金字招牌, 还有istio前卫而实际的设计理念, 目前已经有很多公司在开始提供对istio的支持或者集成, 这是istio官方页面有记载的:

- Red Hat: Openshift and OpenShift Application Runtimes
- Pivotal: Cloud Foundry
- Weaveworks: Weave Cloud and Weave Net 2.0
- Tigera: Project Calico Network Policy Engine
- Datawire: Ambassador project

然后一些其他外围支持, 从代码中看到的:

- eureka
- consul
- etcd v3: 这个还在争执中,作为etcd的拥护者, 我宣布对此保持密切关注.

## 存在的问题

istio毕竟目前才是0.2.2 pre release版本，毕竟才出来四个月，因此还是存在大量的问题，集中表现为：

1. 只支持k8s，而且要求k8s 1.7.4+，因为使用到k8s的 CustomResourceDefinitions
2. 性能较低，从目前的测试情况看，0.1版本很糟糕，0.2版本有改善
3. 很多功能尚未完成

给大家的建议：可以密切关注istio的动向，提前做好技术储备。但是，最起码在年底的1.0版本出来之前，别急着上生产环境。

## 最后的话

感谢大家在今天参与这次的 istio 分享, 由于时间有限, 很多细节无法在今天给大家尽情展开. 如果大家对 istio 感兴趣, 可以之后自行浏览 istio 的官方网站, 我也预期会在之后陆陆续续的给出istio相关的文章和分享.

最后推荐给大家两个资料：

1. istio的中文资料: [istio.doczh.cn](https://istio.doczh.cn/), 这是目前我正在翻译的istio官方文档, 进度大概30%左右, 其中最适合入门了解的 concept 部分已经翻译完成.作为目前市面上唯一的一份istio中文资料，推荐给大家阅读。
2. Service Mesh的微信技术群，由于服务网格的内容实在太新，不容易找到人交流，所以如果对这个技术有兴趣打算深入研究的，欢迎加入。请联系我的微信id(aoxiaojian80)加群。

谢谢大家, 下次再会!


