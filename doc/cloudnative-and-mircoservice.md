# 云原生和微服务

## 注意 ⚠️

- *斜体表示引用*
- **未经允许，禁止转载**

## 学习本课程前，你应该具备如下知识：

- Linux 系统的基本配置和命令
- Web 开发经验
- Docker 命令
- K8S 基本配置和 kubectl 命令
- 对存储和网络有基本了解更佳

## 课程目录

| Date | Time | Title | Content |
| ---- | ---- | ----- | ------- |
| 第 1 天 | 上午 | [1. 云原生](#1-云原生) | [1.1 什么是云原生](#11-什么是云原生) |
| | | | [1.2 容器化和持续交付](#12-容器化和持续交付) |
| | 下午 | | [1.3 微服务和 API 设计](#13-微服务和-api-设计) |
| | | | [1.4 有状态服务](#14-有状态服务) |
| | | | [1.5 WebSocket](#15-websocket) |
| 第 2 天 | 上午 | [2. 微服务](#2-微服务) | [2.1 微服务应该有多微](#21-微服务应该有多微) |
| | | | [2.2 服务注册和服务发现](#22-服务注册和服务发现) |
| | 下午 | | [2.3 云安全](#23-云安全) |
| | | | [2.4 事件](#24-事件溯源和-cqrs) |
| 第 3 天 | 上午 | [3. istio](#3-istio) | [3.1 微服务框架](#31-微服务框架)
| | | | [3.2 环境搭建](#32-环境搭建) |
| | | | [3.3 流量监控](#33-流量监控-调用链跟踪) |
| | 下午 | | [3.4 灰度发布](#34-灰度发布) |
| | | | [3.5 流量治理](#35-流量治理) |
| | | | [3.6 服务保护](#36-服务保护) |

## 1. 云原生

[返回目录](#课程目录)

### 1.1 什么是云原生

[返回目录](#课程目录)

### 1.1.1 云的基本原则

云原生的 3 原则

- **简单**
- **易于自动化**
- **易于发布**

与之对应的解决方案：**API + 微服务 + 容器化**

[云原生的十二要素法则](https://12factor.net)

1. 用同一份受版本控制的基准代码，进行多处部署
1. 显式声明依赖关系
1. 配置信息依赖应用环境存储（即配置管理），与代码分开
1. 后端服务视作附加资源（Web Service / DB / MQ）
1. 严格分离构建和运行步骤
1. 以一个或多个无状态进程运行应用
1. 通过端口绑定提供服务
1. 通过进程模型进行服务扩缩容
1. 确保快速启动和优雅中止来最大化健壮性
1. 尽可能保持开发、预发布、线上环境一致
1. 日志处理成事件流（方便后续统一处理）
1. 用一次性进程运行后台管理任务

### 1.1.2 如何判断你的技术栈和代码实现是否云原生？

1. 所做的任何事（编辑和编译工具、语言和框架）是否简单？
    - IDE 是否可选？强制性是不足取的，比如代码必须用特定的 IDE 才能生成或者编译。
    - 能否通过命令行（即能否自动化）构建和部署？
    - 团队新成员能否快速理解代码？
1. 是否测试驱动开发（本质上即是否有信心交付）？
    - 测试是 trade off 的艺术，编程人员只需要为他们认为不自信的代码编写单元测试
    - 单元测试只对程序员有意义，代码覆盖率是单元测试的客观指标，太低不合适，一般认为需要覆盖 60% 以上
    - 测试工程师需要为所有的 API 测试和功能测试、界面测试负责，尽可能自动化，测试工程师不考虑代码覆盖率，而是考虑测试用例覆盖率
    - 一个 bug 被研发自己发现，和被测试人员发现，和被现场使用者发现，cost 不可同日而语
1. 尽早发布、频繁发布
    - 代码应该少量提交，多次提交，每次提交应该涵盖一个完整的小功能或者 bug 修复
    - 尽早发布、频繁发布才能让产品研发不畏惧发布，减少发布风险
    - 只有足够的自动化测试，才能实现尽早发布、频繁发布
1. 一切能自动化的，都应自动化
    - 任何每天要做的事，都应该被自动化
    - 流程中任何时常重复的部分，如果不能被按钮或者脚本代替，那么就属于过于复杂、脆弱的设计
1. 一切事物都是服务，包括应用
    - 一切服务都是微服务，micro 的前缀完全没必要
    - 单体为什么不好？
        - 每次变更都需要全量发布整个应用程序，维护困难
        - 启动和停止速度慢
        - 依赖关系耦合紧密

        因此，反之，如果应用恰好没有这三个问题，那么单体就是最好的设计，不需要硬拆微服务做过度设计。

### 1.2 容器化和持续交付

[返回目录](#课程目录)

#### 1.2.1 应用容器化

##### 1.2.1.0 容器技术栈

![](/image/k8s-cri-tools.png)

##### 1.2.1.1 Docker 和 Containerd

Containerd 是从 Docker 中分离出来的一个项目，是一个工业级标准的容器运行时，它强调简单性、健壮性和可移植性。

![](/image/k8s-cri-docker.png)

![](/image/k8s-cri-containerd.png)

![](/image/docker-and-kubernetes-use-containerd-2000-opt.png)

实验：[应用容器化和 Docker 基本操作](kubernetes-best-practices.md#213-docker-部署和基本使用)

![](/image/k8s-containerd-tools.png)

实验：[Containerd 相关工具：nerdctl](kubernetes-best-practices.md#2223-nerdctl)

##### 1.2.1.2 K8S

K8S 的操作要记得参考：<https://kubernetes.io/>

[K8S 有哪些组件](https://kubernetes.io/zh/docs/concepts/architecture/#)？api-server、kube-scheduler、kube-controller、etcd、coredns、kubelete、kubeproxy

组件结构图

![](/image/k8s-architecture.png)

实验：[K8S 基本操作和应用部署到 K8S](kubernetes-best-practices.md#3111-单节点集群部署)

#### 1.2.2 弹性容器实例（ECI）

弹性容器实例的功能项包括：

- 同时兼容原生容器和虚拟机的使用体验
- 比虚拟机轻量的资源分配能力，以方便资源快速申请、弹性
- 类似虚拟机的使用体验，可登陆，可任意安装组件
- 有固定 IP 地址，胖容器从创建到删除，IP 地址保持不变
- 可以通过 SSH 远程登陆系统。
- 严格的资源隔离，如 CPU、内存等

实现是一般是提供一个 Pod，Pod 中的容器可以是 OCI VM（比如 Kata），或者单纯容器。

哪些云厂商提供 ECI？

- [博云胖容器](https://mp.weixin.qq.com/s?__biz=MzIzNzA5NzM3Ng==&mid=2651860132&idx=1&sn=cb0eac52be444c162fb505ebbdef6c0f&chksm=f329546bc45edd7d4789aa85ae5edb8a9f8d8313dd1ca273946ef019e0997b39665d29e24c8c&scene=21#wechat_redirect)
- [Kata 和它的朋友们](kubernetes-best-practices.md#24-kata-和它的朋友们)
- [阿里云 ECI](https://eci.console.aliyun.com/#/eci/cn-shanghai/list)：[从 Docker Hub 拉取镜像创建实例](https://help.aliyun.com/document_detail/119093.html)

#### 1.2.3 弹性 K8S 服务（EKS）

K8S 作为一个整体的资源（类同 VM / 存储 / VPC）对外提供。公有云，私有云已广泛提供 EKS。比如：[阿里云容器服务 ACK](https://cs.console.aliyun.com/#/k8s/cluster/list)

#### 1.2.4 CI/CD

发布令人畏惧（深夜里所有 IT 人员一起在电话里讨论，各种失败情况的回退预案，发布后仍战战兢兢如同劫后余生，不知道工作日时是否有问题会突然冒出来），因为对应用程序是否能如预期一样运行没有信心。只有时刻发布才能不畏惧发布，才能不加班、不熬夜、不四处救火，“善战者无赫赫之功”。

托管的 CI/CD 包括：github workflow，阿里云 devops 云效

自建的 CI/CD 包括：jenkins，drain，drone，建木，tekton

Jenkins 是最使用最广的自动化任务管理框架。它有以下缺点：

- Jenkins 配置文件难以通过版本控制进行管理。这一点 OpenStack Zuul 致力于解决。
- Jenkins 基于 Java，需要资源比较多。
- Jenkins 集中式 CI/CD 的代表，所有 CI/CD 逻辑在 Jenkins 侧统一存储和管理。项目代码与 CI/CD 代码分离。

云原生建议项目代码与 CI/CD 代码放在一起，一同进版本控制，这样 CI/CD 自动化框架本身就可以是无状态的，非常适合扩展，也因此具备良好的鲁棒性。

参考：[Github](https://github.com/open-v2x/roadmocker/tree/master/.github/workflows) 或 [Gitee](https://gitee.com/open-v2x/roadmocker/tree/master/.github/workflows)

### 1.3 微服务和 API 设计

[返回目录](#课程目录)

#### 1.3.1 微服务设计

##### 1.3.1.1 微服务的定义和兴起

参考：<http://blog.wuwenxiang.net/Design-Pattern>，微服务遵循设计模式 6 原则中的**单一责任原则（SRP）**，即一个服务只负责一个功能。

在 [1.1.2 章节中](#112-如何判断你的技术栈和代码实现是否云原生)，提到：一切服务都是微服务，micro 的前缀完全没必要。

由上可知，微服务是一个宽泛的计算机编程哲学理念，核心就是 SRP。

面向服务编程（SOA）的理念兴起于亚马逊 Amazon，参考 [程序员的呐喊](https://book.douban.com/subject/25884108/)：

Amazon 的 SOA 法则：

- 从今天起，所有的团队都要以服务接口的方式，提供数据和各种功能。
- 团队之间必须通过接口来通信。
- 不允许任何其他形式的互操作：不允许直接链接，不允许直接读其他团队的数据，不允许共享内存，不允许任何形式的后门。唯一许可的通信方式，就是通过网络调用服务。
- 具体的实现技术不做规定，HTTP、Corba、PubSub、自定义协议皆可。
- 所有的服务接口，必须从一开始就以可以公开作为设计导向，没有例外。这就是说，在设计接口的时候，就默认这个接口可以对外部人员开放，没有讨价还价的余地。
- 不遵守上面规定，就开除。

SOA 的事件教训：

1. SOA架构的错误定位，非常麻烦。

    一个请求可能要经过20次服务器调用，才能找到问题的真正所在。通常，单单是问题的定位就要花费15分钟到几个小时，除非搭建大量的外围监控和报警措施。

2. 同事也是潜在的 DOS 攻击者。

    公司内部某个小组，会突然对你的服务发起大量请求。除非每个服务都设有严格的用量和限量措施，否则根本无法保证可用性。

3. 监控和质量保障（QA）是两回事。

    监控一个服务的时候，可能会得到“一切正常”的回复。但是很有可能，整个服务唯一还正常工作的部分，就是这个回应“一切正常”的模块。只有完整地调用服务，才能确定服务是正常的。

    这意味着，真正监控一个服务，必须做到对所有的服务和数据进行完整的语意检查，否则是看不出问题的。如果做到了这一点，本质上就是在做自动化 QA 了。

4. 必须有服务发现机制。

    面对成百上千的服务时，没有服务发现机制是不可想象的。这又离不开服务注册机制，而它本身也是一个服务。亚马逊有一套统一的服务注册机制，可以通过编程的方式找到所有服务，包括一个服务有哪些 API，目前是不是运行正常，在什么位置等。

5. 必须有沙箱用来调试

    如果代码中调用了他人服务，查找问题的难度要高很多，除非有统一的方式在沙箱里运行所有服务，否则几乎不可能进行任何调试。

6. 不能信任任何人

    团队采用服务的方式进行合作以后，基本上就不能信任其他团队了，正如不能信任第三方工程师一样。

##### 1.3.1.1 典型的微服务框架：OpenStack

可以认为，迄今最成功的微服务和面向服务应用厂商，就是亚马逊。OpenStack 从创始至今，就一直在追随亚马逊（本质上所有的云厂商都在追随亚马逊的云计算服务神迹），从产品设计到架构实现。

基本的设计原则：

- 整个 OpenStack 拆分成若干模块，各模块中包含若干服务
- 模块中必须有一个 API 通信服务，模块之间通过改 API 服务彼此通信
- 模块内通过消息队列通信
- 服务必须能横向扩展

![](/image/openstack-map-v20221001.jpeg)

![](/image/openstack-arch-design.png)

#### 1.3.2 Restful API

Restful API 设计：[Github](https://github.com/wu-wenxiang/training-python-public/blob/master/doc/autotest.md#41-rest-api-%E6%8E%A5%E5%8F%A3%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5) 或 [Gitee](https://gitee.com/wu-wen-xiang/training-python/blob/master/doc/autotest.md#41-rest-api-%E6%8E%A5%E5%8F%A3%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5)

Restful Demo:

- Flask：[Github](https://github.com/wu-wenxiang/rest_api_demo) 或 [Gitee](https://gitee.com/wu-wen-xiang/rest_api_demo)
- Pecan：[Github](https://github.com/wu-wenxiang/restful-api-demo) 或 [Gitee](https://gitee.com/wu-wen-xiang/restful-api-demo)
- Django：[Github](https://github.com/wu-wenxiang/Training-Django-Public/tree/master/09-RestAPI) 或 [Gitee](https://gitee.com/wu-wen-xiang/training-django/tree/master/09-RestAPI)
- FastAPI：[Github](https://github.com/wu-wenxiang/fastapi-demo) 或 [Gitee](https://gitee.com/wu-wen-xiang/fastapi-demo)

#### 1.3.3 API 测试

参考自动化测试：[Github](https://github.com/wu-wenxiang/training-python-public/blob/master/doc/autotest.md) 或 [Gitee](https://gitee.com/wu-wen-xiang/training-python/blob/master/doc/autotest.md)

- 单元测试：unittest / pytest
- 接口测试：requests & json / gabbi / tarven

### 1.4 有状态服务

#### 1.4.1 RDS 服务

[返回目录](#课程目录)

参考阿里云 RDS 服务

### 1.5 WebSocket

[返回目录](#课程目录)

#### 1.5.1 消息队列直接提供 WebSocket

参考 OpenV2X Roadmocker，[Github](https://github.com/open-v2x/roadmocker) 或 [Gitee](https://gitee.com/open-v2x/roadmocker)

#### 1.5.2 应用服务提供 WebSocket

参考 FastAPI：[WebSocket](https://fastapi.tiangolo.com/zh/advanced/websockets/)

## 2. 微服务

[返回目录](#课程目录)

### 2.1 微服务应该有多微

[返回目录](#课程目录)

### 2.2 服务注册和服务发现

[返回目录](#课程目录)

#### 2.2.1 服务发现类型

#### 2.2.2 K8S 服务发现

#### 2.2.3 API Gateway vs Middleware

基于 Nginx 的 API Gateway 应用：[Skyline](https://opendev.org/openstack/skyline-apiserver/src/branch/master/skyline_apiserver/templates/nginx.conf.j2)

基于 [OpenResty](https://openresty.org/cn/) 的 Kong

![](/image/microservice-apigateway-kong.png)

另一种解决思路：Middleware，比如 [OpenStack Keystone](https://opendev.org/openstack/keystonemiddleware)

- [Middleware Architecture](https://docs.openstack.org/keystonemiddleware/latest/middlewarearchitecture.html)
- [Audit middleware](https://docs.openstack.org/keystonemiddleware/latest/audit.html)

#### 2.2.4 微前端

qiankun：<https://qiankun.umijs.org/zh>

### 2.3 云安全

[返回目录](#课程目录)

### 2.4 事件溯源和 CQRS

[返回目录](#课程目录)

## 3. istio

### 3.1 微服务框架

微服务的三个阶段：

1. 拆解单体服务，自建微服务框架
1. 使用 SpringCloud / Dubbo 等侵入式微服务框架
1. 使用服务网格，Sidecar 服务框架

Istio 解决了 SpringCloud 等同行的如下遗留问题：

1. 多语言技术栈不统一：C++、Java、PHP、Go。Spring Cloud 无法提出非 Java 语言的微服务治理。
1. 服务治理周期长：微服务治理框架与业务耦合，上线周期长，策略调整周期长。
1. 产品能力弱：Spring Cloud 缺乏平台化和产品化的能力，可视化能力弱。

企业什么情况适合使用 SpringCloud？

1. 企业的开源语言主要是 Java
1. 治理策略变更不频繁（没有动态服务治理强需求），通过上线可以解决问题
1. 更新升级不频繁（微服务框架升级与业务升级不相互耦合）
1. 无过多高级治理功能需求（不需要混沌调试，智能调参，容器间网络可编程）
1. 业务规模不是非常大（核心业务不一定是微服务框架）

反之，应该选择 K8S 或 K8S + istio

### 3.2 环境搭建

[返回目录](#课程目录)

参考：<https://istio.io/latest/zh/docs/setup/getting-started/>

Demo：istio 环境搭建

Demo：istio 升级

Demo：bookinfo demo

### 3.3 流量监控-调用链跟踪

[返回目录](#课程目录)

### 3.4	灰度发布

[返回目录](#课程目录)

### 3.5 流量治理

[返回目录](#课程目录)

### 3.6 服务保护

[返回目录](#课程目录)
