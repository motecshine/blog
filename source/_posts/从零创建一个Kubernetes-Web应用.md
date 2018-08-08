---
title: 从零创建一个 Kubernetes Web 应用
date: 2018-08-06 17:15:46
tags: kubernetes 容器应用
---
# 前言

上一篇文章简单的介绍了`Kubernetes`内部的负载均衡原理，有朋友在群里反映不要一上来就将原理，想了想也是，那我就从如何创建一个`PHP Web`应用入手，带大家进入`Kubernetes`的世界。

## 基础

### 环境

- CentOS 7.5 (Kernel 3.10)
- Minikube (Kubernetes 1.10.0)

### 对你的要求

我假设你已经掌握了下面的基础技能:

- Docker && 会写Dockerfile
- 如何Google
- 拥有一个DockerHub账号
- 手动编译过LNMP或者LAMP

# 构建基础镜像

![alt](http://testvplpic170904.ufile.ucloud.com.cn/Container.png)

上图描述了我们需要创建的`Containers`，其中`Pause Container`是`Kubernetes`自带的所以我们不用关心，但是十分重要，未来将会有一篇文章来描述`Pause Container`到底干什么的。
其实基础镜像一般用官方现成的就行了，但是在学习过程中建议还是手动编译一下，了解下官方默认配置有哪些坑。`Dockerfile`代码我会放到`GitHub`上, 因为在这里展示实在是太长了。

## 创建Nginx镜像

`Nginx`: [Nginx For K8S GitHub Repo](https://github.com/motecshine/nginx1.12-for-k8s)

### 编译Nginx镜像

```Shell
    docker build . -t motecshine/nginx1.12-for-k8s:v0.1.0
    docker push motecshine/nginx1.12-for-k8s:v0.1.0
```

## 创建PHP-FPM镜像

`FPM`: [FPM For K8S GitHub Repo](https://github.com/motecshine/php71-for-k8s)

### 编译FPM镜像

```Shell
    docker build . -t motecshine/php71-for-k8s:v0.1.0
    docker push motecshine/php71-for-k8s:v0.1.0
```

> 注意事项: `Dockerfile` `CMD` 需要关闭`Nginx` 和 `FPM`的`daemon`特性，具体看我REPO的`Dockerfile`， 这样是为了保证`Container`生命周期与`POD`生命周期一致。

# 构建业务镜像

我们将基于上述镜像来创建我们的业务镜像.

## 创建Code镜像

我们基于`Laravel`来创建镜像。

`Code`: [Code For K8S GitHub Repo](https://github.com/motecshine/code-for-k8s)

### 编译Code镜像

```Shell
    docker build . -t motecshine/code-for-k8s:v0.1.1
    docker push motecshine/code-for-k8s:v0.1.1
```

## 创建Nginx镜像

`laravel-nginx-for-k8s`: [Laravel For K8S GitHub Repo](https://github.com/motecshine/laravel-nginx-for-k8s)

### 编译Nginx镜像

```Shell
    docker build . -t  motecshine/laravel-nginx-for-k8s:v0.1.1
    docker push  motecshine/laravel-nginx-for-k8s:v0.1.1
```

## 创建PHP-FPM镜像

`laravel-fpm-for-k8s`: [Laravel-FPM For K8S GitHub Repo](https://github.com/motecshine/laravel-fpm-for-k8s)

### 编译FPM镜像

```Shell
    docker build . -t  motecshine/laravel-fpm-for-k8s:v0.1.0
    docker push  motecshine/laravel-fpm-for-k8s:v0.1.0
```

# 构建Kubernetes应用

![整体架构](http://testvplpic170904.ufile.ucloud.com.cn/k8s.png)

整体架构如上图所示

## 构建最小化运行单元(Pod）

![alt](http://testvplpic170904.ufile.ucloud.com.cn/deployment.png)

### 创建Deployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: laravel
  namespace: default
spec:
  replicas: 1 # 期待副本数量
  template:
    metadata:
      labels:
        app: laravel # app label
        version: testing
    spec:
      containers:
      - name: code
        image: motecshine/code-for-k8s:v0.1.1
        volumeMounts: # 挂载目录
        - mountPath: /data2
          name: code
      - name: fpm
        image: motecshine/laravel-fpm-for-k8s:v0.1.0
        imagePullPolicy: IfNotPresent
        resources: # 资源限制
           limits:
             cpu: 350m
             memory: 350Mi
           requests:
             cpu: 50m
             memory: 50Mi
        ports:
        - name: fpm
          containerPort: 9000
        volumeMounts:
        - mountPath: /data/code # 挂载code
          name: code
        - mountPath: /var/log # 挂载日志
          name: log  
      - name: laravel-nginx
        image: motecshine/laravel-nginx-for-k8s:v0.1.0
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            cpu: 350m
            memory: 350Mi
          requests:
            cpu: 50m
            memory: 50Mi
        ports:
        - name: laravel-nginx
          containerPort: 80 # 暴露Endpoint
        volumeMounts:
        - mountPath: /data/code
          name: code
        - mountPath: /var/log
          name: log  
      volumes:
      - name: code
        emptyDir: {}
      - name: log
        hostPath:
          path: /var/log
          type: Directory
```

## 构建Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
  namespace: default
  labels:
    app: laravel-service
    version: testing-service
spec:
  type: ClusterIP
  selector:
    app: laravel
    version: testing
  ports:
    - name: http
      port: 80
```

## 构建Ingress

```yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: laravel-ingress
  namespace: default
  labels:
    app: laravel-ingress
spec:
  rules:
  - host: laravel.test
    http:
      paths:
      - path: /
        backend:
          serviceName: laravel-service
          servicePort: 80
```

## 安装Minikube

```Shell
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo cp minikube /usr/local/bin/ && rm minikube
```

## 安装Traefik

我们使用开源的`Ingress`组件安装[参考这里](https://docs.traefik.io/user-guide/kubernetes/)

## 启动Web应用

[上面的配置文件在这里](https://github.com/motecshine/laravel-k8s-config.git)

```Shell
git clone git@github.com:motecshine/laravel-k8s-config.git
cd laravel-k8s-config && kubectl create -f .
```

### 效果

![效果](http://testvplpic170904.ufile.ucloud.com.cn/effect.png)

# 结语

简单的介绍了如何创建一个Web应用，这仅仅是个开始，`Kubernetes`背后是一个庞大的生态环境, `CI，CD，ELK(EFK), APM`，让我们一点点揭开它神秘的面纱。

这里挂载日志到`Host Path` 会有并发写入的问题, 下一篇将`Kubenetes`基于`EFK`日志收集平台，并且给出这个问题的解决方案。