# 记录uat环境部署suse manager的过程
宿主机环境
>系统：Suse Linux Enterprise Server 15 sp2（sles15-2）  
>ip： 128.178.12.5  
>域名：pro5  
>网络：离线，docker使用代理连接阿里云镜像仓库  

虚拟机环境：
>虚拟架构：xen（半虚拟）。kvm架构或者xen（全虚拟）需要cpu支持vx-t等内核虚拟化技术，uat环境不符合。  
>系统：Suse Linux Enterprise Micro 5.5（slem5.5）  
>ip: 192.168.122.233（pro5下的虚拟ip）  
>域名：suma.server.com  

# 安装docker
1. 准备docker-27.1.1.tgz  
2. 解压文件，将解压目录下的内容放到 /var/lib/ 下  
```
tar -zxvf docker-27.1.1.tgz
cp ./docker/* /usr/bin/
```
3. 创建service文件  
```
vim /etc/systemd/system/docker.service

### 文件开头
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
### 文件结尾

# 增加权限
chmod +x /etc/systemd/system/docker.service
systemctl daemon-reload 
# 启动docker
systemctl start docker
systemctl enable docker.service
```

4. 配置daemonset，从其他节点直接复制到文件目录下  
```
vim /etc/docker/daemon.json
systemctl daemon-reload 
```

5. 配置http-proxy，从其他节点复制  
```
mkdir -p /etc/systemd/system/docker.service.d
vim /etc/systemd/system/docker.service.d/http-proxy.conf
# 生效配置并重启docker
systemctl daemon-reload
systemctl restart docker
# 登录阿里云镜像仓库测试
docker login --username=jenkins@5628939625451055 registry-intl.cn-hangzhou.aliyuncs.com -p "g{HlLbpA)yqRlQNig92HtnW#Q1G"
```

# 传输镜像源到uat
需要准备以下三个大文件传输到pro5，传输方式为本地通过dockerfile构建docker镜像，并将文件放入容器内，pro5拉取镜像构建容器，并从容器内复制出文件：
> a. sles15 sp2的完整镜像（sle15-2.iso）  
> b. Suse Manager的磁盘启动盘镜像（suma.qcow2)  
> c. rmt导出的sles15-2的repos  

1. 以sles15-2的iso镜像文件为例，拉取一个基础镜像。
```
docker pull busybox:latest
```
2. 在本地环境构建基础镜像，并将镜像文件放到容器内。
```
### Dockerfile
# 使用官方的 BusyBox 镜像
FROM busybox
# 复制文件到容器
COPY sle15-2.iso /sle15-2.iso
# 启动容器时运行的命令
CMD ["sh", "-c", "while true; do sleep 30; done"]
###
```

2. 构建镜像
```
docker build -t busybox_sle15-2 .
```

3. 打tag并上传到阿里云
```
#docker login --username=jenkins@5628939625451055 registry-intl.cn-hangzhou.aliyuncs.com -p "g{HlLbpA)yqRlQNig92HtnW#Q1G"
docker tag busybox_sle15-2 registry-intl.cn-hangzhou.aliyuncs.com/smartocc/busybox_sle15-2:v1
docker push registry-intl.cn-hangzhou.aliyuncs.com/smartocc/busybox_sle15-2:v1
```

4. 进入uat环境的pro5，拉取镜像
```
#docker login --username=jenkins@5628939625451055 registry-intl.cn-hangzhou.aliyuncs.com -p "g{HlLbpA)yqRlQNig92HtnW#Q1G"
docker pull registry-intl.cn-hangzhou.aliyuncs.com/smartocc/busybox_sle15-2:v1
```

5. 创建容器，拉取文件，然后删除容器
```
docker create --name temp-container registry-intl.cn-hangzhou.aliyuncs.com/smartocc/busybox_sle15-2:v1
docker cp temp-container:/sle15-2.iso /home/
docker rm temp-container
# 根据需要选择是否删除镜像，节省空间
docker rmi registry-intl.cn-hangzhou.aliyuncs.com/smartocc/busybox_sle15-2:v1
```

# 设置zypper软件源并安装xen虚拟套件
1. 打开yast2 》 Software > Software repositories
2. 选择 add 》 Local ISO image...
3. 填写iso所在路径 /home/sle15-2.iso
4. 选择所需模块，一路保存。（Basesystem module 和 server application 为必须）
5. 执行zypper ref，即可从软件源中安装软件包。

# 安装xen虚拟模块
1. 打开yast2 》 Virtualization 》 Install Hypervisor and Tools 
2. 选中 xen server 和 xen tools，保存并安装。
3. 由于uat无法看到开机界面，因此设置默认开机选项。进入yast2 》 system 》 Boot loader。
4. 选择 Bootloader Options 》 Default Boot Section ，选择带Xen 的启动方式。
5. 重启。

# 安装虚拟机
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="67" protocol="tcp" to-port="67" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="67" protocol="udp" to-port="67" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="69" protocol="tcp" to-port="67" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="69" protocol="udp" to-port="67" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="80" protocol="tcp" to-port="80" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="443" protocol="tcp" to-port="443" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="4505" protocol="tcp" to-port="4505" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="4506" protocol="tcp" to-port="4506" to-addr="192.168.122.233"'
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" forward-port port="25151" protocol="tcp" to-port="25151" to-addr="192.168.122.233"'
