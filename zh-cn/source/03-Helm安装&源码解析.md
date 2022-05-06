# Helm安装源码解析

## 编写Helm安装chart包方法
Helm客户端提供了通过命令方式安装chart，但有些时候我们希望通过代码自动完成安装chart过程，而不是登录到目标集群主机执行helm命令安装chart包，
这时我们需要自己开发安装chart包的接口。以下将介绍如何开发一个Helm安装chart的接口

### 添加依赖
首先我们需要添加helm依赖，对于如果有自己特殊需求需要修改helm源码的，可以将helm源码放到自己的仓库中。我在github上维护了一个自己helm客户端，
源至官方最新代码
```
require (
	github.com/aimjianzhang/helm v0.0.0-20220411123222-8d7d0eade5d5
)
```


     




