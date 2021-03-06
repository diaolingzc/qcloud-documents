## 函数运行时的容器模型
SCF 将在事件触发时代表您执行 SCF 函数，根据您的配置信息（如内存大小等）进行资源分配，并启动和管理容器（即函数的执行环境）。SCF 平台负责所有函数运行容器的创建、管理和删除操作，用户没有权限对其进行管理。

在容器启动时需要一些时间，这会使得每次调用函数时增加一些延迟。但是，通常仅在首次调用函数、更新函数、或长时间未调用时重新调用时会察觉到此延迟，因为平台为了尽量减少此启动延时，会尝试对后续调用重用容器，在调用函数后容器仍会存留一段时间，预期用于下次调用。在此段时间内的调用会直接重用该存留的容器。

容器重用机制的意义在于：

- 用户代码中位于[执行方法](https://cloud.tencent.com/document/product/583/9210#.E6.89.A7.E8.A1.8C.E6.96.B9.E6.B3.95)外部的任何声明保持已初始化的状态，再次调用函数时可以直接重用。例如，如果您的函数代码中建立了数据库连接，容器重用时可以直接使用原始连接。您可以在代码中添加逻辑，在创建新连接之前检查是否已存在连接。
- 每个容器在 `/tmp` 目录中提供部分磁盘空间。容器存留时该目录内容会保留，提供可用于多次调用的暂时性缓存。再次调用函数时有可能可以直接使用该磁盘内容，您可以添加额外的代码来检查缓存中是否有您存储的数据。

```
注意事项：
请勿在函数代码中假定始终重用容器，因为是否重用和单次实际调用相关，无法保证是创建新容器还是重用现有容器。
```
## 临时磁盘空间
SCF 函数在执行过程中，都拥有一块 512MB 的临时磁盘空间 `/tmp`，用户可以在执行代码中对该空间进行一些读写操作，但这部分数据可能 *不会* 在函数执行完成后保留。因此，如果您需要对执行过程中产生的数据进行持久化存储，请使用 COS 或 Redis/Memcached 等外部持久化存储。

## 调用类型
SCF 平台支持`同步`和`异步`两种调用方式来调用云函数。调用类型与函数本身的配置无关，只有在调用函数时才能控制调用类型。

- 同步调用函数将会在调用请求发出后持续等待函数的执行结果返回
- 异步调用将不会等待结果返回，只发出请求。

以下调用场景您可以自由定义函数的调用类型：

- 编写的应用程序调用 SCF 函数。如果您需要同步调用，请在 InvokeFunction 接口中传入参数 `invokeType=RequestResponse`；如果您需要异步调用则请传入参数`invokeType=Event`
- 手动调用 SCF 函数（使用 API 或 CLI）用于测试。调用时的参数区别同上。

但是，在您使用腾讯云其他云服务作为事件源时，云服务的调用类型是预定义的，用户无法在这种情况下自由指定调用类型。例如，COS 和定时器始终异步调用 SCF 函数。

## 用户限制

下表列出了内测期的资源限制。

|资源|默认限制|
|--|--|
|单地域函数个数|20|
|新建函数时支持同时创建的触发器个数|1|
|单函数COS触发器数量|2|
|单函数定时触发器数量|2|
|临时磁盘容量（/tmp）|512MB|
|最大执行时长|300 秒|
|控制台上传的代码包大小|5MB|
|最大并发容器数|20，用户可以[联系我们](https://cloud.tencent.com/document/product/583/9712)提高该限制|

## 函数并发量
函数的并发数量是指在任意指定时间对函数代码的执行数量。对于当前的 SCF 函数来说，每个发布的事件请求就会执行一次。因此，这些触发器发布的事件数（即请求量）会影响函数的并发数。您可以使用以下公式来估算并发的函数实例总数目。
```
每秒请求量 * 函数执行时间（按秒） 
```
例如，考虑一个处理 COS 事件的函数，假定函数平均用时 0.2 秒（即200毫秒），COS 每秒发布 300 个请求至函数。这样将同时生产 300*0.2=60 个函数实例。


## 并发限制

默认情况下，SCF 将会对每个函数的并发量限制为 20。您可以通过[联系我们](https://cloud.tencent.com/document/product/583/9712)来调高此数值。

如果调用导致函数的并发数目超过了默认限制，则该调用会被阻塞，SCF 将不会执行这次调用。根据函数的调用方式，受限制的调用的处理方式会有所不同：

- 同步调用：如果函数被同步调用时受到限制，将会直接返回 429 错误
- 异步调用：如果函数被异步调用时受到限制，SCF 将在一定的时间内以固定的频率自动重试受限制的事件
 
## 重试机制

如果您的函数因为超过最大并发数目、或遇到平台内部资源不足等内部限制导致失败。此时，如果您的函数被同步调用，将会直接返回错误（见上面并发执行限制内容）。如果您的函数被内部云服务使用异步方式调用，则该次调用将会自动进入一个重试队列，SCF 将自动重试调用。

## 执行环境和可用库

当前 SCF 的执行环境建立在以下基础上：

- 标准 CentOS 7.2 镜像
- Python 2.7 运行时环境

以下库和依赖包在 SCF 执行环境下直接可用，您可以直接使用 `import`语句导入它们：

- [Python标准库](http://usyiyi.cn/translate/python_278/library/index.html)
- [Qcloud COS SDK](https://cloud.tencent.com/document/product/436/6275)
- [Qcloud API SDK](https://cloud.tencent.com/document/developer-resource/494/7244)


