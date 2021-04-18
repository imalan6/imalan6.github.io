# 一次慢查询sql导致的故障排查

最近项目后台管理系统出现问题，页面刷新没有数据，这里记录一下排查和解决的过程。

## 现象

1、后台页面没有数据，刷新也不起作用。

2、查看浏览器页面接口返回消息，后台接口报错500，初步定为应该后台接口出了问题。

3、检查后台服务，`hystrix`报`timeout`，相关服务超时。

![0.png](https://i.loli.net/2021/04/17/q7laURwVrXLeDKk.png)

## 问题分析

1、查看被调用的服务，没有报错，但是`log`没有继续打印，最后执行完几条`sql`后卡住了，初步估计是运行太慢导致的问题。

![1.png](https://i.loli.net/2021/04/17/s4G7XcNtSumQIvr.png)

2、服务没有问题，就得看看数据库是否出问题了，执行 `#show processlist` 查看`mysql`正在运行的`sql`线程。发现有部分`sql`语句卡住，状态一直是`Sending data`，没有变过。这是导致`mysql`运行慢的问题了

![2.png](https://i.loli.net/2021/04/17/nIwfuyYDBrKVLWk.png)

3、查看`mysql`慢查询日志，发现有部分`sql`执行效率太低，`query time`超过了10秒（相当慢的查询，数据库工程师背锅）。和上面卡住的`sql`语句一比对，发现正是这部分`sql`，那问题是找到了。

![3.png](https://i.loli.net/2021/04/17/jqD1xegC2mBMOny.png)

 

 ![4.png](https://i.loli.net/2021/04/17/fsDW96KFrX7UuOT.png)

## 解决

1、`explain`分析`sql`，查看是否使用索引；

2、优化`sql`，查询时间要小于`2s`；

3、加大`mysql`缓存