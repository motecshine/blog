---
title: 创建一个简单的k8s应用
date: 2018-06-28 14:23:42
tags: k8s 
---

# 创建K8S应用

## 前言

从创建Docker Container开始一步一步给大家讲述如何创建自己的K8S应用.看懂这篇操作手册你可能需要
了解:
  1. Docker 命令的使用
  2. kubectl 命令的使用
  3. Yaml的使用
<!-- more -->
## 概念

##### Docker

* Docker Registry是什么？
  - 他是存储docker镜像的仓库, 如果你熟悉git，那么上手这个也很快, 你需要了解到的几个命令:
  - docker push (上传镜像)
  - docker pull (拉取镜像)

> 常用的仓储有 https://index.docker.io

* Docker Container 与 Docker Image 之间的关系？

    - 所有的Docker Container 都是以 Docker Image为蓝本创建的
    - Docker Image 可以理解为一个还没装到你电脑上的Win7系统镜像，Docker Container是已经装到你的电脑上的系统.

* 如何基于Dockerfile创建一个Docker Container？
```
# Docker Hub 拉取ubuntu 基础镜像，并以它为蓝本创建自己的镜像
FROM ubuntu
# 维护者
MAINTAINER docker_user docker_user@email.com
# 更新ubuntu系统
RUN echo "deb http://archive.ubuntu.com/ubuntu/ raring main universe" >> /etc/apt/sources.list
# 安装Nginx
RUN apt-get update && apt-get install -y nginx
# 为了保证容器与nginx生命周期一致，所有的程序不建议用background的方式运行
RUN echo "\ndaemon off;" >> /etc/nginx/nginx.conf
# Docker Container 运行时需要执行的命令
CMD /usr/sbin/nginx
```


###### K8S

* POD 是什么？
  - 是K8S里最小可运行的单元，一个POD里至少需要一个Docker Container。

* Service 是什么？
  > Container需要对外服务，以Nginx为例子，来描述Service创建过程

    1. Container 暴露一个端口映射到POD端口上(K8S会分给POD一个内部的Cluster IP)
    2. POD 暴露出相应的端口。
    3. Service 通过K8S的Label标签功能,发现POD，以及POD暴露出来的端口
* Ingress 是什么
  - Ingress是一个互联网入口，可以看做一个简单的Nginx，Ingress通过KUBE-PROXY将外部访问流量引导至Service上

## 创建流程

> main.go

``` golang
package main

import (
    "net/http"
)

func SayHello(w http.ResponseWriter, req *http.Request) {
    w.Write([]byte("Hello World"))
}

func main() {
    http.HandleFunc("/hello", SayHello)
    http.ListenAndServe(":8001", nil)

}
```

##### 创建基准Docker Image

```bash
# 以BusyBox做为基准镜像创建我们自己的Docker Image
FROM busybox
# 设置工作目录
WORKDIR /go/src/app
# 把当前目录的二进制文件放到Docker Images
COPY . .
# 纠正时间
RUN  cp -r -f Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone 
# 运行时执行的命令
CMD ["/go/src/app/helloworld"]
```

##### 创建POD
> deployment.yaml

```yaml
# 使用k8s哪一个版本的API
apiVersion: extensions/v1beta1 
# 以Deployment方式创建POD， 这里有很多类型以后有机会再讲
kind: Deployment
metadata:
  # 创建的POD名字K8S内部会hash这个值
  name: helloworld
  # 指定在哪个namespace下创建
  namespace: frm
spec:
  # 部署副本数量
  replicas: 3
  # 保留历史版本的副本数量的上限值
  revisionHistoryLimit: 5
  template:
    metadata:
      labels:
        # Service 通过这个值讲 SVC与POD关联上
        app: helloworld
        version: production
    spec:
      containers:
      # 创建的ContainerName
      - name: helloworld
        # docker image 地址
        image: helloworld:v1.3
        # 触发滚动更新的规则
        imagePullPolicy: IfNotPresent
        # 资源限制
        resources:
          limits:
            cpu: 80m
            memory: 80Mi
          requests:
            cpu: 20m
            memory: 20Mi 
        # container 暴露出来的端口
        ports:
        - name: helloworld-port
          containerPort: 80
        # 挂载目录  
        volumeMounts:
        - mountPath: /go/src/app/logs
          name: log  
     # 指定该POD卷挂载到宿主机目录
      volumes:
      - name: log
        hostPath:
          path: /home/data/logs/helloworld # 宿主及目录
          type: Directory

```

##### 创建Service

> service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld-srv
  namespace: frm
  labels:
    # Service 的Label, Ingress通过这个来关联Service
    app: helloworld-srv
    version: production
spec:
  type: ClusterIP
  selector:
    # 选择一个POD Label
    app: helloworld
    version: production
  # 选择POD暴露的端口  
  ports:
    - name: http
      port: 80
```

##### 创建Ingress

> ingress.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: helloworld-ingress
  namespace: frm
  labels:
    app: helloworld-ingress
spec:
  rules:
  ### 指定域名
  - host: helloworld.test
    http:
      paths:
      - path: /
        backend:
          # 选择绑定哪个service
          serviceName: helloworld-srv
          servicePort: 80
```

##### 创建命令

> kubectl create -f deployment.yaml service.yaml ingress.yaml

## 流程

> User Request -> Ingress Port-> Service -> Pod -> Container

## 问题排查

1. 构建Docker Image后先自己 docker run 一下来确认构建是否是成功的!

2. 创建失败 ，查看POD创建状态

- `kubectl describe pods POD_NAME -n NAMESPACE`

3. 创建失败 ，查看POD LOG
- `kubectl log POD_NAME -n NAMESPACE`

4. POD启动成功但是无法访问
- `kubectl get svc,ingresss,pod -n NAMESPACE`

5. 查看service的状态
- `kubectl exec -it POD_NAME -n NAMESPACE -c CONTAINER_NAME`，进入有问题的容器看看
