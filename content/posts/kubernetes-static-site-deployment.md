---
title: "一次完整的 Kubernetes 静态网页部署实践"
date: "2026-04-01T12:00:00+08:00"
tags: ["Kubernetes", "Docker", "Nginx", "部署实践"]
author: "乐豆芽"
showToc: true
TocOpen: true
draft: false
hidemeta: false
comments: false
description: "从静态网页、Docker 镜像到 Kubernetes 部署，我完整走通了一次最小可用的上线流程。"
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
UseHugoToc: true
---

## 项目目标

这次我想做的不是“写一个网页”，而是把一条完整的部署链路走通：

- 写一个最小静态网页
- 用 Docker 打包成 nginx 镜像
- 把镜像推送到 Docker Hub
- 在云上的 Kubernetes 集群里部署
- 最后通过 SSH 隧道在本地浏览器访问

项目本身不复杂，但它把前端、容器、镜像仓库和 K8s 串在了一起。对我来说，这比单独学某一个命令更有价值。

## 我做了哪些事情

整个过程大致分成五步：

1. 编写欢迎页：HTML、CSS、JavaScript
2. 编写 Dockerfile，把静态资源打包进 nginx
3. 编写 `deployment.yaml` 和 `service.yaml`
4. 在本地 WSL 安装并验证 Docker 环境
5. 把镜像推送到 Docker Hub，再部署到华为云 Kubernetes

最后访问路径也打通了：

- 服务器本机验证：`curl http://127.0.0.1:30523`
- 本地浏览器访问：`http://127.0.0.1:8080`
- 使用 SSH 隧道：`ssh -N -L 8080:127.0.0.1:30523 huaweiyun`

## 核心工作流

### 1. 本地构建镜像

在 WSL 中完成镜像构建和推送：

```bash
cd /mnt/d/codex/static-k8s-demo
docker login -u <dockerhub-user>
docker build -t leoduya/static-web-demo:latest .
docker push leoduya/static-web-demo:latest
```

这里最关键的是把网页内容和 nginx 一起打进镜像，这样 Kubernetes 节点只需要拉镜像，不需要再手动拷贝网页文件。

### 2. 云端部署

在云主机上完成 Kubernetes 资源创建和滚动更新：

```bash
kubectl apply -f /home/ledy2025/deployment.yaml
kubectl apply -f /home/ledy2025/service.yaml
kubectl set image deployment/static-web-demo nginx=leoduya/static-web-demo:latest
kubectl rollout status deployment/static-web-demo
kubectl get pods -o wide
kubectl get svc -o wide
```

其中我最常用的几个检查命令是：

- `kubectl get pods -o wide`
- `kubectl get svc -o wide`
- `kubectl describe pod <pod名>`

前两个看整体状态，最后一个专门用来排查异常事件。

### 3. 本地访问验证

服务虽然部署在远端，但我没有直接暴露公网端口，而是先用 SSH 隧道做本地验证：

```bash
ssh -N -L 8080:127.0.0.1:30523 huaweiyun
```

然后直接在浏览器打开：

```text
http://127.0.0.1:8080
```

这种做法很适合调试早期版本，因为它简单、安全，而且不需要额外改防火墙或 ingress。

## 这次踩过的坑

### Docker Hub 登录名和仓库命名空间不是一回事

我一开始用邮箱登录 Docker Hub，没有意识到“登录账号”和“推送仓库前缀”是两件不同的事。  
最后真正能成功拉取的镜像名是：

```text
leoduya/static-web-demo:latest
```

### WSL 和云主机是两个独立环境

本地 Docker 登录成功，不代表云主机也有相同环境。  
WSL 和远端主机的：

- 登录状态
- 文件
- 代理配置
- 镜像缓存

都是彼此独立的。

### `kubectl apply` 成功不等于应用能跑

这是最容易误判的一点。  
资源对象创建成功，只说明 YAML 没有语法问题，不说明镜像能拉下来，也不说明容器已经起来。

真正要继续检查的是：

- Pod 是否进入 `Running`
- 是否出现 `ImagePullBackOff`
- Service 端口是否正确暴露

## 这次学到的东西

这个项目虽然很小，但我真正打通了一条完整的容器化部署链路：

- 写静态网页
- 打包 Docker 镜像
- 推送镜像仓库
- Kubernetes 部署
- Service 暴露服务
- 排查镜像拉取问题
- 用 SSH 隧道完成本地验证

对我来说，这比单独背 `kubectl` 命令更重要。因为只有把整条链路走通，才会知道问题到底会出现在“镜像、集群、网络还是本地访问”哪一层。

## 后续想继续优化的方向

- 补上 Ingress，而不是只用 NodePort
- 用 CI 自动构建和推送镜像
- 给部署文件加上更清晰的环境变量和版本管理
- 把这套最小流程整理成可重复复用的模板项目
