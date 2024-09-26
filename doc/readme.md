# mtr的uat集群监控部署流程，loki + grafana + prometheus 
集群环境：
> k8s：v1.30.3+rke2r1  
> docker：v27.1.1  
> sc: nfs-storage  
> 镜像来源：harbor.datastudio.mtr  
> 节点：
> - 178.128.12.2 pro2 sles15.
>   178.128.12.3 pro3
>   178.128.12.4 pro4

# 镜像包准备
1. 先在本地docker上准备部署过程需要的镜像包
```
docker pull grafana/grafana:10.3.3
docker pull grafana/loki:3.1.0
docker pull grafana/promtail:3.0.0
docker pull gcr.io/cadvisor/cadvisor:v0.37.0
docker pull quay.io/prometheus/node-exporter:v1.8.2
docker pull registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1
docker pull prom/prometheus:v2.53.1
```
2. 登录公司阿里云仓库，将本地镜像打tag并上传
```
#docker login --username=jenkins@5628939625451055 registry-intl.cn-hangzhou.aliyuncs.com -p "g{HlLbpA)yqRlQNig92HtnW#Q1G"
docker login --username=*** registry-intl.cn-hangzhou.aliyuncs.com -p "***"
docker tag xxx:v* registry-intl.cn-hangzhou.aliyuncs.com/smartocc/***:v*
docker push registry-intl.cn-hangzhou.aliyuncs.com/smartocc/***:v*
```

3. 通过跳板机进入mtr的uat集群环境，登录阿里云仓库，拉取镜像
```
docker login --username=*** registry-intl.cn-hangzhou.aliyuncs.com -p "***"
docker pull registry-intl.cn-hangzhou.aliyuncs.com/smartocc/***:v*
```

4. 打上uat上harbor的tag，并上传至harbor
```
docker tag  registry-intl.cn-hangzhou.aliyuncs.com/smartocc/***:v*  harbor.datastudio.mtr/library/***:v*
docker push harbor.datastudio.mtr/library/***:v*
```

5. 将后续部署用到的yaml文件中的镜像路径都替换为上述harbor路径


# 新建命名空间
如果实际部署环境采用其它ns名称，将后续步骤的yaml文件的namespace值都替换掉。
```shell
kubectl create ns monitor
```


# 部署grafana
1. 创建对象  
Deployment: grafana
> image: harbor.datastudio.mtr/library/grafana:10.3.3  
> containerPort：3000
> requests.cpu: 100m requests.memory: 200Mi
> limits.cpu: 1 limits.memory: 2Gi

Service：grafana
> nodeport：30200

PersistentVolumeClaim：pvc-monitor-grafana  

2. 执行文件：
```shell
kubectl apply -f grafana.yaml
```
3. 校验：登录http://\<node-ip\>:30200，node-ip为grafana所在节点ip，port对应nodeport。登录默认账密均为admin。


# 部署loki
1. 创建对象：
> ServiceAccount: loki  
> Role：loki  
> RoleBinding：loki  
> StatefulSet: loki  
> ConfigMap: loki  
> Service: loki  
> PersistentVolumeClaim: pvc-monitor-loki  

2. 配置：
> containerPort：3100
> nodeport：30201
> image: harbor.datastudio.mtr/library/lmk-images/loki:3.1.0

3. 执行文件：
> 可修改配置：svc的端口映射（修改port需要同样修改configmap的http_listen_port），pvc的容量大小（暂定50G，后续根据磁盘占用情况增加），configmap中的retention_period为保留日志范围（uat暂定半年）
```shell
kubectl apply -f loki-rbac.yaml
kubectl apply -f loki-configmap.yaml
kubectl apply -f loki-statefulset.yaml
```
校验：在grafana中添加loki数据源，ip填写为http://loki:3100，loki为service中的name，端口为service中的port,测试连接能否保存。


# 部署fluent-bit
> fluent-bit用于抓取日志并推送至loki。相比于promtial，fluent-bit更加轻量级，可使用较少资源完成日志采集。
> 使用helm部署，准备 fluent-bit-0.47.9.tgz 和 fluent-bit-values.yaml。镜像来源为：harbor.datastudio.mtr/library/fluent-bit-plugin-loki 。
> 需要修改配置在 fluent-bit-values.yaml 完成，参考注释，官方配置参考：https://grafana.com/docs/loki/latest/send-data/fluentbit/#configuration-options 
```shell
helm install fluent-bit fluent-bit-0.47.9.tgz -f fluent-bit-values.yaml -n monitor
```
校验：等pod成功运行后，在grafana随机查看任意namespace下pod的日志。


# 部署cadvisor
cadvisor用于收集、处理和导出容器的运行时信息，特别是针对 Docker 容器。它提供了对容器的 CPU、内存、磁盘 I/O、网络使用情况等资源使用情况的实时统计。
> Prometheus通过config配置可抓取cadvisor数据。 
> 在uat中节点的8080被占用，因此注释hostNetwork: true，避免端口占用。
> 确保部署环境存在路径 /var/run ，/var/lib/docker 并且有读写权限。
```shell
kubectl apply -f cadvisor.yaml
```
校验：访问http://192.168.208.2:8080，ip可替换为集群中任意节点ip，端口为cadvisor.yaml中的containerPort，默认为8080。


# 部署node-exporter
Node Exporter 主要负责收集和暴露类 UNIX 系统（Linux、BSD 等）的硬件和系统运行指标。
> 确保存在路径 /proc ，/dev，/sys，/rootfs，/etc/localtime ，并且有读写权限
```shell
kubectl apply -f node-exporter.yaml
```
校验：访问http://192.168.208.2:9100，ip可替换为任意节点ip，端口为cadvisor.yaml中的containerPort，默认为9100。


# 部署kube-state-metrics
kube-state-metrics 主要是收集和报告集群中例如 Deployment、DaemonSet、Pod 等各种资源的实时状态信息。
> 可修改配置：svc的nodePort，ClusterRole可增加rules以获取权限监控更多资源
```shell
kubectl apply -f kube-state-metrics.yaml
```
校验：访问http://\<node-ip\>:31666，node-ip替换为kube-sate-metrics所在节点的ip，端口为kube-state-metrics中service的nodePort，默认为31666。


# 部署prometheus
> 可修改配置：svc的端口映射，pvc大小，configmap的job配置与监控组件匹配，匹配方式参考官网配置
```shell
kubectl apply -f prometheus-rbac.yaml
kubectl apply -f prometheus-configmap.yaml
kubectl apply -f prometheus.yaml
```
校验：在grafana中添加Prometheus数据源，ip为http://prometheus.monitor.svc:9090，其中prometheus为prometheus.yaml中service的name，monitor为prometheus.yaml中deployment的namespace，9090为prometheus.yaml中service的port，保存并测试数据源。
校验数据源：登录http://\<node-ip\>:31090，node-ip替换Prometheus所在节点的ip，端口为service的nodePort，默认为31090。检查target是否包含cadvisor、node-exporter、kube-state-metrics的数据来源（通过label判断） 

# 在grafana中创建展板
导入展板文件，可从 https://grafana.com/grafana/dashboards/ 查找适合的展板