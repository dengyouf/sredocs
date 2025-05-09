## 监测类型

### startup Probe

判断Pod内容器是否启动完成


### liveness Probe

判断 Pods 是否存活

### readiness Probe

容器所在 Pod 上报还未就绪的信息，并且不接受通过 Kubernetes Service 的流量。


## 监测机制

### Exec Action

### TcpSocket Action

### HTTPGet Action

