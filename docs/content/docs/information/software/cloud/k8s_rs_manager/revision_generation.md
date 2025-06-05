---
weight: 19
title: "ResourceVersion & Generation"
draft: false
---

最近用`kubernetes` `client-go`实现一个监听 `deployments` 变化的功能，在如何判断 `kubernetes` 资源的变化有了疑问，查阅文档得知有两个与kubernetes资源对象相关的属性。

- ResourceVersion 
- Generation

### ResourceVersion

`resourceVersion`的维护利用了底层存储`etcd`的`Revision`机制。

#### Etcd Version

ETCD共四种version

- Revision
- ModRevision
- Version
- CreateRevision

|字段|作用范围|说明|
|-|-|-|
|Version|Key|单个Key的修改次数，单调递增|
|Revision|全局|Key在集群中的全局版本号，全局唯一|
|ModRevison|Key|Key 最后一次修改时的 Revision|
|CreateRevision|全局|Key 创建时的 Revision|

根据更新资源时是否带有`resourceVersion`分两种情况：

- 未带resourceVersion：无条件更新，获得etcd中最新的数据然后再此基础上更新
- 带有resourceVersion：和etcd中modRevision对比，不一样就提示版本冲突，说明数据已发生修改，当前要修改的版本已不是最新数据。

`Kubernetes`资源版本控制采用乐观锁，`apiserver`在写入`etcd`时作冲突检测。

### Generation

`Generation`初始值为1，随`Spec`内容的改变而自增。

如果`controller`更新资源失败，常见的做法是，重新拉取资源，如果资源`Generation`没有变化，将更新结果`Patch`到新的资源上再尝试更新。
