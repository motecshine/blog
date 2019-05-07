---
title: k8s内部负载均衡原理
date: 2018-07-14 12:30:04
tags: kubernetes k8s ingress service kube-proxy
---
# 前言

> 个人理解有限，如有错误，请及时指正。

前前后后学习`kubernetes`已经有三个月了，一直想写一遍关于`kubernetes`内部实现的一系列文章来作为这三个月的总结，个人觉得`kubernetes`背后的架构理念以及技术会成为中大型公司架构的未来。我推荐可以先阅读下Google的[Large-scale cluster management at Google with Borg](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/43438.pdf)技术文献，它是实现`kubernetes`的基石。
<!-- more -->
## 准备

在阐述原理之前我们需要先了解下`kubernetes`关于内部负载均衡的几个基础概念以及组件。

### 概念

##### Pod

![alt](http://testvplpic170904.ufile.ucloud.com.cn/pod_infrastructure.png)

1.`Pod`是`Kubernetes`创建或部署的最小/最简单的基本单位。

2.如图所示，`Pod`的基础架构是由一个根容器`Pause Container`和多个业务`Container`组成的。

3.根容器的`IP`就是`Pod IP`，是由`kubernetes`从`etcd`中取出相应的网段分配的, `Container IP`是由`docker`分配的，同样这些`IP`相对应的`IP`网段是被存放在`etcd`里。

4.业务`Container`暴露出来端口并且映射到相应的根容器`Pause Container`端口，映射出来的端口叫做`endpoint`。

5.业务`Container`的生命周期就是`POD`的生命周期，任何一个与之相关联的`Container`死亡，`POD`也应该随之消失

##### Service

1.`Service` 是定义一系列Pod以及访问这些Pod的策略的一层抽象。`Service`通过`Label`找到`Pod`组。因为`Service`是抽象的，所以在图表里通常看不到它们的存在，这也就让这一概念更难以理解。

2.`Kubernetes`也会分给`Service`一个内部的`Cluster IP`，`Service`通过`Label`查询到相应的`Pod`组, 如果你的`Pod`是对外服务的那么还应该有一组`endpoint`，需要将`endpoint`绑到`Service`上，这样一个微服务就形成了。

##### Kubernetes CNI

CNI（Container Network Interface）是用于配置Linux容器的网络接口的规范和库组成，同时还包含了一些插件。CNI仅关心容器创建时的网络分配，和当容器被删除时释放网络资源。

##### Ingress

1.俗称边缘节点，假如你的`Service`是对外服务的，那么需要将`Cluster IP`暴露为对外服务，这时候就需要将`Ingress`与`Service`的`Cluster IP`与端口绑定起来对外服务。这样看来其实`Ingress`就是将外部流量引入到`Kubernetes`内部，这也是这篇文章重要要将的。

2.实现Ingress的开源组件有`Traefik`和`Nginx-Ingress`, 前者方便部署，后者部署复杂但是性能和灵活性更好。

### 组件

### Kube-Proxy

1.`Kube-Proxy`是被内置在`Kubernetes`的插件。
2.当`Service`与`Pod` `Endpoint`变化时，`Kube-Proxy`将会改变宿主机`iptables`, 然后配合`Flannel`或者`Calico`将流量引入`Service`.

### Etcd

1.`Etcd`是一个简单的`Key-Value`存储工具。
2.`Etcd`实现了`Raft`协议，这个协议主要解决`分布式强一致性`的问题，与之相似的有`Paxos`, `Raft`比`Paxos`要容易实现。
3.`Etcd`用来存储`Kubernetes`的一些网络配置和其他的需要强一致性的配置，以供其他组件使用。
4.如果你想要深入了解`Raft`, 不放先看看[raft相关资料](https://github.com/motecshine/simple-raft)

### Flannel

1.`Flannel`是`CoreOS`团队针对`Kubernetes`设计的一个覆盖网络`Overlay Network`工具，其目的在于帮助每一个使用`Kuberentes`的`CoreOS`主机拥有一个完整的子网。
2.主要解决`POD`与`Service`,`跨节点`相互通讯的。

### Traefik

1.`Traefik`是一个使得部署微服务更容易的现代HTTP反向代理、负载。
2.`Traefik`不仅仅是对`Kubernetes`服务的，除了`Kubernetes`他还有很多的`Providers`，如`Zookeeper`,`Docker Swarm`, `Etcd`等等

## Traefik工作原理

授人以鱼不如授人以渔，我想通过我看源码的思路来抛砖引玉，给大家一个启发。

##### 思考

在我要深度了解一个组件的时候通常会做下面几件事情

- 组件扮演的角色

- 手动编译一个版本

- 根据语言特性来了解组件初始化流程

- 看单元测试，了解函数具体干什么的

- 手动触发一个流程，在关键步骤处记录日志，单步调试

###### Traefik初始化流程

1.在`github.com/containous/traefik/cmd/traefik`下由一个名为`traefik.go`的文件是该组件的入口。`main()`方法里有这样一段代码

```go
// 加载 Traefik全局配置
traefikConfiguration := cmd.NewTraefikConfiguration()
// 加载providers的配置
traefikPointersConfiguration := cmd.NewTraefikDefaultPointersConfiguration()

...

// 加载store的配置
storeConfigCmd :=storeconfig.NewCmd(traefikConfiguration, traefikPointersConfiguration)

// 获取命令行参数
f := flaeg.New(traefikCmd, os.Args[1:])
// 解析参数
f.AddParser(reflect.TypeOf(configuration.EntryPoints{}), &configuration.EntryPoints{})
...

// 初始化Traefik
s := staert.NewStaert(traefikCmd)
// 加载配置文件
toml := staert.NewTomlSource("traefik", []string{traefikConfiguration.ConfigFile, "/etc/traefik/", "$HOME/.traefik/", "."})
...
// 启动服务
if err := s.Run(); err != nil {
    fmtlog.Printf("Error running traefik: %s\n", err)
    os.Exit(1)
}

os.Exit(0)
```

上面就是组件初始化流程，当我们看完初始化流程的时候应该会想到下面几个问题：

- 当我们手动或者自动伸缩`Pods`时，`Traefik`是怎么知道的？

    假设你已经知道`Kubernets`是一个`C/S`架构，所有的组件都要通过`kube-apiserver`来了解其他节点或者组件的运行状态。

    当然`Traefik`也不例外，他是通过`Kubernetes`开源的`Client-Go`SDK来完成与`kube-apiserver`交互的。

    我们来找找源码:
    > `github.com/containous/traefik/provider/kubernetes`是关于`Kubernetes`的源码。我们看看到底干了啥。

    ```go
    // client.go
    type Client interface {
        // 检测Namespaces下的所有变动
        WatchAll(namespaces Namespaces, stopCh <-chan struct{}) (<-chan interface{}, error)
        // 获取边缘节点
        GetIngresses() []*extensionsv1beta1.Ingress
        // 获取Service
        GetService(namespace, name string) (*corev1.Service, bool, error)
        // 获取秘钥
        GetSecret(namespace, name string) (*corev1.Secret, bool, error)
        // 获取Endpoint
        GetEndpoints(namespace, name string) (*corev1.Endpoints, bool, error)
        // 更新Ingress状态
        UpdateIngressStatus(namespace, name, ip, hostname string) error
    }
    ```
    显而易见，这里通过订阅`kube-apiserver`，来实时的知道`Service`的变化，从而实时更新`Traefik`。
    我们再来看看具体实现

    ```go
    // kubernetes.go
    func (p *Provider) Provide(configurationChan chan<- types.ConfigMessage, pool *safe.Pool, constraints types.Constraints) error {
    ...
    // 初始化一个kubernets client
    k8sClient, err := p.newK8sClient(p.LabelSelector)
    if err != nil {
        return err
    }
    ....
    // routines 连接池，这里的routines实现的很优雅，有心的同学看下
    pool.Go(func(stop chan bool) {
        operation := func() error {
            for {
                stopWatch := make(chan struct{}, 1)
                defer close(stopWatch)
                // 监视和更新namespaces下的所有变动
                eventsChan, err := k8sClient.WatchAll(p.Namespaces, stopWatch)
                ....
                for {
                        select {
                        case <-stop:
                            return nil
                        case event := <-eventsChan:
                            // 从kubernestes 那边接收到的事件
                            log.Debugf("Received Kubernetes event kind %T", event)
                            // 加载默认template配置
                            templateObjects, err := p.loadIngresses(k8sClient)
                            ...
                            // 对比最后一次的和这次的配置有什么不同
                            if reflect.DeepEqual(p.lastConfiguration.Get(), templateObjects) {
                                // 相同的话，滤过
                                log.Debugf("Skipping Kubernetes event kind %T", event)
                            } else {
                                // 否则更新配置
                                p.lastConfiguration.Set(templateObjects)
                                configurationChan <- types.ConfigMessage{
                                    ProviderName:  "kubernetes",
                                    Configuration: p.loadConfig(*templateObjects),
                                }
                            }
                        }
                }
        }
    }
    ```
    `Kubernets`返回给`Traefik`的数据结构大致是这样的:

    ```json
    {"service":{"pod_name":{"domain":"ClusterIP"}}}
    ```
    看过上述的代码分析应该就对Traefik有一个大致的了解了。

## Kube-Poxy工作原理

`Kube-Proxy`与`Traefik`实现原理很像，都是通过与`kube-apiserver`的交互来完成实时更新`iptables`的，这里就不细说了，以后会有一篇文章专门讲
`kube-dns`, `kube-proxy`, `Service`的。

## 组件协同与负载均衡

简单描述流程，然后思考问题，最后考虑是否需要深入了解(取决于个人兴趣)

### 组件协同

用户通过访问`Traefik`提供的L7层端口, `Traefik`会转发流量到`Cluster IP`，`Flannel`会将用户的请求准确的转发到相应的`Node`节点的`Service`上。(ps:  `Flannel`初始化的时候宿主机会建立一个叫`flannel0`【这里的数字取决于你的Node节点数】的虚拟网卡）

### 负载均衡

上文讲述了`kube-proxy`是通过`iptables`来配合`flannel`完成一次用户请求的。

具体的流程我们只要看一个`service`的`iptables rules`就知道了。

```iptables
// 只截取了一小段，假设我们起了两个Pods
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
// 流量跳转至 KUBE-SVC-ILP7Z622KEQYQKOB
-A KUBE-SERVICES -d 10.111.182.127/32 -p tcp -m comment --comment "pks/car-info-srv:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-ILP7Z622KEQYQKOB
// 50%的几率跳转至KUBE-SEP-GDPUTEQG2YTU7YON
-A KUBE-SVC-ILP7Z622KEQYQKOB -m comment --comment "pks/car-info-srv:http" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-GDPUTEQG2YTU7YON

// 流量转发至真正的Service Cluster IP
-A KUBE-SEP-GDPUTEQG2YTU7YON -s 10.244.1.57/32 -m comment --comment "pks/car-info-srv:http" -j KUBE-MARK-MASQ
-A KUBE-SEP-GDPUTEQG2YTU7YON -p tcp -m comment --comment "pks/car-info-srv:http" -m tcp -j DNAT --to-destination 10.244.1.57:80
```

可以很明显的看出来，`kubernetes`内部的负载均衡是通过`iptables`的`probability`特性来做到的，这里就会有一个问题，当`Pod`副本数量过多时，`iptables`的表将会变得很大，这时会有性能问题。

### 总结

- `Traefik` 通过默认的负载均衡(wrr)直接将流量通过`Flannel`送进`POD`.
- `kube-proxy` 在没有 `ipvs`的情况下, 会通过`iptables`转发做负载均衡.

## 结尾

通过这篇文章我们简单的了解到内部负载均衡的机制，但是任然不够深入，你也可用通过这篇文章查漏补缺，觉得有什么错误的地方欢迎及时指正，我的邮箱`shinemotec@gmail.com`。下一篇将会讲`Kubernetes`的`HPA`工作原理。