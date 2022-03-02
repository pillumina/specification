## Operator Pattern

### Operator开发机制

#### `List&Watch`机制

​	controller和`apiServer`交互即为client端和server端的交互。而`k8s`组件之间的交互并没有依赖其他中间件，仅仅采用了`http`协议通信。

​	`List-Watch`是`k8s`统一的异步消息处理机制，它保证了消息的实时性、可靠性、顺序性，性能等，也是`k8s`架构的精髓。

![list-watch](https://static.oschina.net/uploads/img/201607/14120254_zvta.png)

>`List-Watch`机制是什么？

​	list即是调用资源的`list API`去罗列资源，基于`http`短连接实现；watch是调用资源的`watch API`监听资源变更事件，基于`http`长连接实现。以pod为例，它的list和watch的api如下：

>[List API](https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#list-all-namespaces-63)，返回值为 [PodList](https://v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/#podlist-v1-core)，即一组pod。
>
>>GET /api/v1/pods
>
>[Watch API](https://link.zhihu.com/?target=https%3A//v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/%23watch-list-all-namespaces-66)，往往带上`watch=true`，表示采用`HTTP 长连接`持续监听`pod 相关事件`，每当有事件来临，返回一个[WatchEvent](https://link.zhihu.com/?target=https%3A//v1-10.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.10/%23watchevent-v1-meta)：
>
>>GET /api/v1/watch/pods

​	其中watch机制是通过[Chunked transfer encoding(分块传输编码)实现，首次出现在`HTTP/1.1，引用维基如下:

>HTTP 分块传输编码允许服务器为动态生成的内容维持 HTTP 持久链接。通常，持久链接需要服务器在开始发送消息体前发送Content-Length消息头字段，但是对于动态生成的内容来说，在内容创建完之前是不可知的。使用分块传输编码，数据分解成一系列数据块，并以一个或多个块发送，这样服务器可以发送数据而不需要预先知道发送内容的总大小。

​	客户端调用watch API时，`ApiServer`在response的`http header`中设置`Transfer-Encoding`的值为`chunked`，采用分块传输编码，客户端接收到消息后，和服务端连接，并且等待下一个数据块，也就是资源的事件信息，举例如下:

```shell
$ curl -i http://{kube-api-server-ip}:8080/api/v1/watch/pods?watch=yes
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 02 Jan 2019 20:22:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

​	对于一个异步消息系统是否优秀的评价，那么就要有几个维度去评价消息机制本身：

- 可靠性

- 实时性

- 顺序性

- 性能

  >​	首先对于可靠性而言，list可以查询到当前资源以及对应的期望状态，客户端可以通过比较当前状态和实际状态进行动作触发。watch和`apiServer`维护长连接，接受资源的状态变更事件做出处理。如果仅调用watch，那么长连接中断，会导致消息丢失，所以需要list获取全量数据。因此我们可以理解list获取全量，watch获取增量，而仅仅通过轮询list接口，虽然也可以实现相同的资源同步效果，但是实时性和开销又是问题。
  >
  >​	实时性由watch推送资源变更事件来保证。
  >
  >​	消息的顺序性，在并发场景下，客户端可能在短时间内接收到同一个资源的多个事件。`k8s`本身关注的是最终一致性，它需要了解哪个是最近发生的事件，并且保证资源最终状态和最近发生的事件有一样的表达。因此`k8s`资源中都带有`ResourceVersion`字段，这个是递增的数字，当客户端并发处理同一个资源的事件时，它可以对比`resourceVersion`保证最终的状态和最新的事件所期望的状态一致。
  >
  >​	性能方面，仅使用周期轮询list接口会大幅度增加开销，增加`apiServer`的压力。而watch作为异步的事件通知机制，复用了一条长连接，保证了实时性的同时也保证了性能。

#### `Informer`机制

> ​	informer最初是`k8s`团队开发的`client-go`包提供的核心工具包。在`k8s`源码当中，如果组件需要`list/get`在集群中的特定资源对象，一般不会直接请求`k8s`的`api`，而是会调用informer中的`Lister()`方法（包含了get和list方法）。

​	其设计的痛点在于，如果controller很多，且逻辑中有很多`get/list`请求，对于`apiServer`的负载也是相当高的。因此它是一个包含以下特性的工具包：

- 快速返回`Get/List`请求的结果，减少对`ApiServer API`接口直接调用。

  >​	使用informer提供的`lister()`方法，在controller去`list/get`对象的时候，informer会查找缓存在本地内存的数据，而非新建一个连接。这种方式既能够更快返回结果，也能够减少对`api`的调用

- 依赖`k8s`的`list-wath API`。

  >​	informer只是调用`k8s`的`List`和`Watch`两种api。在informer初始化时，先调用`List`获取某种资源的全量对象，缓存在内存中。而后调用`Watch`去监听这个资源，维护这个缓存。此后informer不再调用任何其他的api。
  >
  >​	informer的设计中，有`resync`机制，去周期性`List`一遍资源的所有对象，从而保证缓存和集群资源一致性。例如其源码中的`listAndWatch`是被`wait.Util`周期性调用，但是为了减少服务端压力，一般这个period默认会非常长。

- 能够监听事件并且触发回调函数。

  >​	informer通过watch接口监听某个资源的事件。而且informer能够添加自定义的回调函数。controller只需要实现`OnAdd`、`OnUpdate`、`OnDelete`方法，注册给informer即可，informer会在对应事件接收到时，分发给各个controller的回调。

- 二级缓存。

  >​	二级缓存是informer的底层机制，分别是`DeltaFIFO`和`LocalStore`。前者用于处理存储`watch`接口返回的各类事件，后者只会被`Lister`的`get/list`接口访问到。

![informer](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/ZeroMind/Kubernetes/Informer/informer-ar.png)

#### 调谐循环

![arch](https://assets.openshift.com/hubfs/Imported_Blog_Media/rafop1.png)

​	Operator可以由多组监听不同资源的controller构成，而controller的业务逻辑核心是调谐循环。当资源的相关事件发生时，调谐循环开始。在调谐循环中，controller需要去检查目前的资源状态和期望的资源状态是否一致。

​	这种触发的设计逻辑即为水平触发(level-based triggering)，与此相对的是边沿触发(edge-based triggering)。这是来自于电子电路设计的概念。

>水平触发指的是，接收到一个事件（例如一个中断）后对状态本身作相应的反应；而边沿触发指的是，接收到一个事件对状态的变化本身作反应。
>
>[Edge Triggered v.s. Level Triggered interrups](http://venkateshabbarapu.blogspot.com/2013/03/edge-triggered-vs-level-triggered.html)

​	水平触发是较为低效的，因为这需要调谐循环去分析所有的状态，而与之相反的是对状态变化作出反应。但是这种调谐触发逻辑有一个好处，就是它对那种复杂且不可靠的环境来说（例如信号会丢失并且需要重新发送好多次）更加合适。

​	因此这种设计思路的区别会影响到我们如何去实现我们的controller代码。

​	下面是一个controller通用的调谐循环示意，当然这不能描述所有的场景：

![reconcile](https://assets.openshift.com/hubfs/Imported_Blog_Media/rafop3.png)

​	上述的步骤有：

	1. 获取到特定的CR实例。
 	2. 检查CR实例的合法性，因为我们不需要处理带有不合法值的CR。
 	3. 检查实例的初始化。如果一些实例的值没有被初始化，那下面的模块会处理它们。
 	4. 检查实例是否被删除。如果实例被删除了，且我们需要进行一些cleanup操作，那么下面的模块会处理它们。
 	5. 处理controller的业务逻辑。如果前面的步骤都过了，可以对这个CR实例执行调谐循环。

![overall-mech](https://user-images.githubusercontent.com/57335825/116524709-45c15f80-a90a-11eb-87a1-447527d07f7b.png)



![overall-informer](https://octetz.s3.us-east-2.amazonaws.com/k8s-controllers-vs-operators/client-go-flow.png)

### Operator开发工具与框架

#### Client-go（other kubernetes clients）

​	使用`kubernetes client`去开发operator是自由度最大的方式，最常见的`client-go`，其他语言比如java、js/ts、python都有自己语言的client。这些client提供了较为底层的`k8s APIs`访问，封装有限。

>Pros: 
>
>- 开发上对资源控制的自由度比较大
>
>Cons:
>
>- 不同语言的clients实现的特性有所不同，例如`client-go`最为成熟，而其他语言的成熟度相对较低，可能需要开发者去解决一些已知的问题。
>- 需要开发者去对`k8s api`、informer机制、调谐实现机制、code generator有比较深入的理解。
>
>举例：
>
>- [kube-controller-manager](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller)
>- [tidb operator](https://github.com/pingcap/tidb-operator)

#### KubeBuilder

![kubebuilder](https://ssup2.github.io/images/programming/Kubernetes_Kubebuilder/Controller_Manager_Architecture.PNG)

​	`KubeBuilder`是开发Operator的一个框架，开发者可以通过`CRD`的文件声明，由框架自动生成资源的结构体和`API Client`。它由`golang`实现，利用[controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)库作为基础构建controller。

​	当开发者使用这个框架开发controller，只需要声明`CRD`定义，且关注`Reconcile`函数的实现即可，通过监听特定的CR和其他资源触发相应的业务逻辑。

>Pros:
>
>- 框架提供了controller搭建的脚手架（资源定义、代码生成等）。
>- 框架提供了测试包，可以让用户与单机etcd和apiserver进行简单测试。
>- 框架提供makefile进行operator任务(build, test, run, code generation etc.)
>- ......
>
>Cons:
>
>- ......

#### OperatorSDK

![sdk](https://sdk.operatorframework.io/operator-capability-level.png)

​	和`kubeBuilder`一样使用了`controller-runtime`和`controller-tools，提供了更多controller脚手架和更高层次的框架封装。

>Pros:
>
>- 可以和helm或者ansible集成，开发者甚至可以不需要了解`golang`。
>- 集成了Operator LifeCycle Manager(OLM)，用于管理operator的在线升级等。构建`Day2 Operator`的基础。
>- 集成了e2e的测试框架，可以和真实的集群进行测试。
>- [Operator Best Practices](https://sdk.operatorframework.io/docs/best-practices/best-practices/)
>
>Cons:
>
>- ......

#### KUDO

![kudo](https://assets.d2iq.com/production/uploads/posts/attachments/screen-shot-2019-09-24-at-23211-pm-5d8a8b7235867.png)

​	Kubernetes Universal Declarative Operator (KUDO) 框架让开发者能用声明式的方式去构造operator，甚至不需要写go代码。此外，框架管理了operator的生命周期、运维等`DevOps`特性。

### `CloudSOP`平台应用场景举例

