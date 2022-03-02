## Operator Pattern

### Operator是什么



### Operator是如何工作的



  

### Operator开发模式



### Operator举例



### Operator开发模式



#### `List&Watch`机制

#### `Informer`机制

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

### Operator开发框架

#### Client-go

#### KubeBuilder

#### OperatorSDK

#### KUDO

### `CloudSOP`平台应用场景举例

