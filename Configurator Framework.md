# Configurator Framework



## phase上下文规范

### phase定义

>​	事件处理分阶段旨在解决configurator转换函数中对一些对其他服务端（k8s, configstore, secretstore等）的资源请求，统一由`appcontroller`提供的框架进行处理。在概念上，一个phase的handler做的是资源转换的幂等操作，如果业务逻辑有产生reject，框架可以进行机制上的重试。逻辑上不同phase的产生可以由以下场景表征：
>
>1. 一个资源监听事件，如果在处理逻辑中涉及到多个资源和外部服务的交互，例如`apiserver`、`configstore`、`secretstore`，就需要分不同的phase，如果涉及的资源交互比较多，考虑一个阶段产生多个跨服务资源交互请求。
>2. 代码逻辑的多phase，即一个phase本质上不产生任何外部服务资源的交互请求，但是这种逻辑的划分会新增执行上的复杂度，因此不推荐这样的划分。
>
>   phase分为两个类型: root phase与normal phase，在`configurator` CR中定义的某个资源事件的第一个phase为root phase，它的入参仅为框架从`apiserver`监听到的k8s资源对象，注意Update事件的入参为最新的对象，不包括旧的对象。而其他的phase均为normal phase，入参包括root phase输入得到的资源对象，以及上一个phase的跨服务资源交互请求的结果，以及上一个phase透传的一些参数。这些结果和参数都组织成map结构, key由上一个phase的输出定义。
>
>​	phase的handler函数可以支持对资源的批量处理(batch processing)，因此phase的入参可以是批量的资源对象，开启框架的这个特性，需要在`configurator` CR中将`isBatch`设置为`true`（默认为false, 框架默认传给handler一个资源对象）。
>
>​	对于批量处理的场景，且每一个资源都会涉及到一个跨服务资源交互请求，可以在phase输出的交互请求中指定`groupBy`字段，下一个phase的输入中会在`requestResults`中产生这个`groupBy`字段的值相同的key，key对应的value为批处理资源请求的array。



Todo: batch模式的request可以配置onError drop，如果一个batch的请求其中有一个失败，不需要全局phase级别的重试，可以返回失败，由下一个phase感知

### 输入

- handler输入对象

| Attribute Name  | Type                                    | Description                                 | From                  |
| --------------- | --------------------------------------- | ------------------------------------------- | --------------------- |
| resourceObject  | KubernetesObject / KubernetesObjectList | 监听事件对应的资源对象                      | `ApiServer`           |
| requestResults  | Map                                     | 上一个phase的输出中定义的资源跨服务交互请求 | 框架执行请求后注入    |
| passThroughArgs | Map                                     | 上一个phase的输出中定义的需要透传的参数     | 用户在phase输出中指定 |

- `requestResults对应的Map`对象

> 对于root phase，该值为空。

| Key Name          | Value Types | Description                                                  | Range                                |
| ----------------- | ----------- | ------------------------------------------------------------ | ------------------------------------ |
| requestSink       | enum/string | 表示该请求的目标服务端类型                                   | kubernetes, configstore, secretstore |
| isDeclarative     | boolean     | 表示该请求是否为声明式请求(即不需要关注服务侧返回)           | /                                    |
| queryObjects      | rray        | 服务侧返回的返回值列表                                       | /                                    |
| createdTimestamp  | string      | 框架创建该请求的时间戳                                       | /                                    |
| finishedTimestamp | tring       | 框架执行完请求的时间戳                                       | /                                    |
| error             | Error       | 该请求若执行失败，且请求策略为`catchError`，框架将Error对象注入给该key。 | /                                    |

```json
{
  resourceObject: KubernetesObject,  // KubernetesObjectList: cr中定义的事件通知对应的资源类型(single/batch)
  // 跨服务资源请求结果
  // key: 每一个phase需要进行的跨服务资源交互请求名
  // value: 不同类型请求的返回结果
  // 对于root phase来说，该值默认为undefined
  requestResults: {
    // user defined key
  	"user_request_first": {
      requestSink: 'kubernetes'
      ...
    },
    // user defined key
    "user_request_second": {
      requestSink: 'configstore'
      ...
    },
    // user defined key
    "user_request_third": {
      requestSink: 'secretstore'
      ...
    },
    // user defined key, identified by groupBy key from the last phase output
    "user_grouped_requests" : [{}, {}, {}],
	},
  // 上一个phase可以往相邻的phase透传某些参数，该key用于保存这些参数
  passThroughArgs: {
    
  }
}
```



### 输出

>Todo: configstore事务接口

- phase handler输出对象

| Attribute Name  | Attribute Type | Description                       | Required |
| --------------- | -------------- | --------------------------------- | -------- |
| requests        | Requests       | 跨服务资源交互请求对象            | Yes      |
| passThroughArgs | Map            | 下一个phase的输入中需要透传的参数 | No       |
|                 |                |                                   |          |

- `Requests结构定义

| Attribute Name     | Attribute Type         | Description                                                  | Required | Value Range                            |
| ------------------ | ---------------------- | ------------------------------------------------------------ | -------- | -------------------------------------- |
| executeStrategy    | enum/string            | 请求执行策略                                                 | No       | [serial, parallel] 默认parallel        |
| resourceRequests   | Array<resourceRequest> | 跨服务资源交互请求列表                                       | Yes      | /                                      |
| batchErrorStrategy | enum/string            | 含有`groupBy`字段的resourceRequest, 执行错误后的处理策略。dropError为若发生error则忽略该请求, catchError为若发生error则获取到该error并于下一个phase的输入中体现该error。 | No       | [dropError, catchError] 默认catchError |



![requests](https://tva1.sinaimg.cn/large/e6c9d24egy1h01i42yoe6j20zl0ehq4p.jpg)

- `resourceRequest`  父接口定义

| Attribute Name | Attribute Type | Description                                                  | Required |
| -------------- | -------------- | ------------------------------------------------------------ | -------- |
| requestName    | string         | 请求名、标识(exclusive)                                      | Yes      |
| isDeclarative  | boolean        | 请求是否为声明式(不需要校验服务端返回成功与否)               | Yes      |
| groupBy        | string         | 批量请求聚合的key name, 在下一个phase的输入中体现。用于批量的资源涉及到相同的query操作。 | No       |
| bootstrapDep   | boolean        | 该资源交互请求是否为kernel自举需要，用于自举阶段重编排对相关配置是否成功生成的判断，默认为false。 | No       |



- `kubernetesResourceRequest` 子接口定义

| Attribute Name | Attribute Type | Description                                                  | Required | Value Range                                 |
| -------------- | -------------- | ------------------------------------------------------------ | -------- | ------------------------------------------- |
| group          | string         | k8s资源组                                                    | Yes      | /                                           |
| version        | string         | k8s资源版本                                                  | Yes      | /                                           |
| plural         | string         | k8s资源复数名                                                | Yes      | /                                           |
| kind           | string         | k8s资源类型                                                  | Yes      | /                                           |
| namespace      | string         | k8s资源命名空间                                              | Yes      | /                                           |
| name           | string         | k8s资源名称                                                  | Yes      | /                                           |
| verb           | enum/ string   | 资源动词                                                     | Yes      | [get, list, patch, delete, create, replace] |
| fieldSelector  | string         | 字段选择器(用于list动词)                                     | No       | /                                           |
| labelSelector  | string         | 标签选择器(用于list动词)                                     | No       | /                                           |
| resourceBody   | object         | 资源结构体(用于patch, create, replace动词)                   | Yes      | /                                           |
| precheckExist  | boolean        | 创建资源前判断是否存在，默认为true，如果已存在不创建，且不抛出错误 | No       | /                                           |

- `configResourceRequest` 子接口定义

| Attribute Name | Attribute Type | Description      | Required | Value Range                      |
| -------------- | -------------- | ---------------- | -------- | -------------------------------- |
| configType     | enum / string  | 资源配置类型     | Yes      | [pod, node, namespace, workload] |
| configVerb     | enum / string  | 资源配置操作类型 | Yes      | [get, delete, update, create]    |
| resourceName   | string         | 资源名           | Yes      | /                                |
| resourceUid    | string         | 资源uid          | No       | /                                |
| namespace      | string         | 命名空间         | No       | /                                |
| value          | string         | 配置值           | No       | /                                |

- `configAppConfRequest`子接口定义

| Attribute Name | Attribute Type | Description         | Required | Value Range              |
| -------------- | -------------- | ------------------- | -------- | ------------------------ |
| verb           | enum  / string | appconf资源操作类型 | Yes      | [create, delete, update] |
| podName        | string         | 关联的pod名         | Yes      | /                        |
| key            | string         | appconf的key        | Yes      | /                        |
| value          | string         | appconf的value      | No       | /                        |

- `configBasicRequest` 子接口定义

| Attribute Name | Attribute Type | Description  | Required | Value Range        |
| -------------- | -------------- | ------------ | -------- | ------------------ |
| verb           | enum / string  | 配置操作类型 | Yes      | [get, put, delete] |
| key            | string         | 配置key      | Yes      | /                  |
| value          | string         | 配置value    | No       | /                  |
|                |                |              |          |                    |

- `configTwoLayersRequest`子接口定义

| Attribute Name | Attribute Type | Description                     | Required | Value Range                   |
| -------------- | -------------- | ------------------------------- | -------- | ----------------------------- |
| secondPath     | string         | 组成key结构的第二级的自定义内容 | Yes      | /                             |
| thirdPath      | string         | 组成key结构的第三级的自定义内容 | Yes      | /                             |
| value          | String         | 底层key对应的value              | No       | /                             |
| verb           | enum / string  | 二层结构操作类型                | Yes      | [get, create, update, delete] |

- `secretRequest` 子接口定义



| Attribute Name | Attribute Type | Description    | Required | Value Range           |
| -------------- | -------------- | -------------- | -------- | --------------------- |
| secretName     | string         | secret名       | Yes      | /                     |
| verb           | enum / string  | secret操作类型 | Yes      | [get, create, delete] |
|                |                |                |          |                       |
|                |                |                |          |                       |

### 设计实现约束

- Root phase中输入结构中，仅带`resourceObject`字段，代表从`apiserver`监听到的资源对象。
- 如果k8s资源的一个监听事件，仅需要一个phase即可处理完毕 , (例如phase的handler函数做简单的配置转换，生成configstore配置)，该phase output声明的所有跨服务资源交互请求，**框架侧默认如果其中一个请求产生错误，就要进行handler的重试。**
- 涉及到phase output的多个资源交互请求，phase的设计需要实现幂等履行，否则框架的重试机制容易产生side effect。因此涉及到如资源创建任务请求、配置入库请求建议放在独立的phase中执行。
- 对configstore的配置操作请求，因为配置本身涉及到多层结构，因此框架本身需要处理配置的多层依赖，请求中应该只感知最底层的配置，而不需要感知其依赖的上层配置结构。（配置中心结构设计）
- 创建资源/配置请求时必须先查询是否存在，如果存在则不创建，且不抛出错误。



## 错误处理与重试

​	

### `Error Interface`

```typescript
/**
 * RetryExceedLimitError代表重试超过最大限制
 */
export class RetryExceedLimitError extends Error {
  
}

/**
 * Symbol代表着error在逻辑上可以被重试
 */ 
export const RetryableError = Symbol('RetryableError')

/**
 * 返回error是否可被重试
 */
export const isRetryableError = (error: Error) => RetryableError in error;

/**
 * ResourceGenericError为资源跨服务交互请求交互产生的错误
 */
export class ResourceGenericError extends Error {}

/**
 * ResourceNotFoundError表示query的资源在服务侧或者本地缓存中无法找到
 */
export class ResourceNotFoundError extends ResourceGenericError {
  [RetryableError] = true;
}

/**
 * ResourceConflictError表示对资源的mutation请求冲突
 */
export class ResourceConflictError extends ResourceGenericError {
  [RetryableError] = true;
}

/**
 * ResourceMutationError表示对资源的修改请求发生错误
 */
export class ResourceMutationError extends ResourceGenericError {
  [RetryableError] = true;
}

/**
 * ResourcePermissionError表示资源的访问权限受到服务端或者框架的限制
 */
export class ResourcePermissionError extends ResourceGenericError {
  constructor(resourceName: string) {
    super(`Resource ${resourceName} access permission is restricted`);
  }
}

/**
 * UnsupportedResourceOperation表示框架不支持对应的资源操作
 */
export class UnsupportedResourceOperation extends ResourceGenericError {
  constructor(resourceType: string, operationType: string){
    super(`Resource type ${resourceType} does not support ${operationType} operation`);
  }
}

/**
 * ApiServerUnavailableErorr表示apiserver服务侧无法访问的错误
 */
export class ApiServerUnavailableErorr extends ResourceGenericError {
  [RetryableError] = true;
}

/**
 * ConfigstoreUnavailableErorr表示configstore服务侧无法访问的错误
 */
export class ConfigstoreUnavailableErorr extends ResourceGenericError {
  [RetryableError] = true;
}

/**
 * SecretstoreUnavailableErorr表示secretstore服务侧无法访问的错误
 */
export class SecretstoreUnavailableErorr extends ResourceGenericError {
  [RetryableError] = true;
}
```









