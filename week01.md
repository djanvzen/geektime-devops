#
week01 笔记

## Linux 中 Namespace 的作用

在内核中实现容器之间的资源隔离

| NameSpace  | 功能 | 
| ---  | --- |
| MNT Namespace(mount) | 隔离挂载点及文件系统 | 
| PID Namespace(Process Identification) | 隔离进程 |
| IPC Namespace(Inter-Process Communication) | 隔离进程间通信 |
| UTS Namespace(UNIX Timesharing System) | 主机名的隔离 |
| Net Namespace(network) | 隔离网络 | 
| User Namespace(user) | 隔离用户 |
| Time Namespace | 提供时间隔离能力 
| Syslog Namespace | 提供syslog隔离能力 | 
| Control group (cgroup) Namespace |  提供进程所属的控制组的身份隔离 |


## docker的安装

### apt安装
```bash
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装Docker-CE
sudo apt-get -y update
# 查看apt中的docker-ce版本
apt-cache madison docker-ce
#  docker-ce | 5:20.10.18~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
#  docker-ce | 5:20.10.17~3-0~ubuntu-jammy | https://mirrors.aliyun.com/docker-ce/linux/ubuntu jammy/stable amd64 Packages
# version即为第二段（如5:20.10.18~3-0~ubuntu-jammy）
sudo apt-get -y install docker-ce=[VERSION]
```

### yum安装
```bash
# step 1: 安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
# Step 2: 添加软件源信息
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# Step 3
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
# Step 4: 更新并安装Docker-CE
sudo yum makecache fast
sudo yum -y install docker-ce
# Step 4: 开启Docker服务
sudo service docker start

# 注意：
# 官方软件源默认启用了最新的软件，您可以通过编辑软件源的方式获取各个版本的软件包。例如官方并没有将测试版本的软件源置为可用，您可以通过以下方式开启。同理可以开启各种测试版本等。
# vim /etc/yum.repos.d/docker-ce.repo
#   将[docker-ce-test]下方的enabled=0修改为enabled=1
#
# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# yum list docker-ce.x86_64 --showduplicates | sort -r
#   Loading mirror speeds from cached hostfile
#   Loaded plugins: branch, fastestmirror, langpacks
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable
#   docker-ce.x86_64            17.03.1.ce-1.el7.centos            @docker-ce-stable
#   docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable
#   Available Packages
# Step2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.0.ce.1-1.el7.centos)
# sudo yum -y install docker-ce-[VERSION]

```
如果需要直接下载rpm包可以使用以下命令
```bash
yum install docker-ce --downloaddir=./ --downloadonly
```

### 二进制安装

提前下载好docker-ce的tar包，使用以下脚本进行安装，并替换 `PACKAGE_NAME` 字段
```bash
#!/bin/bash
DIR=`pwd`
PACKAGE_NAME="docker-20.10.19.tgz"
DOCKER_FILE=${DIR}/${PACKAGE_NAME}
#read -p "请输入使用docker server的普通用户名称，默认为docker:" USERNAME
if test -z ${USERNAME};then
  USERNAME=docker
fi
centos_install_docker(){
  grep "Kernel" /etc/issue &> /dev/null
  if [ $? -eq 0 ];then
    /bin/echo  "当前系统是`cat /etc/redhat-release`,即将开始系统初始化、配置docker-compose与安装docker" && sleep 1
    systemctl stop firewalld && systemctl disable firewalld && echo "防火墙已关闭" && sleep 1
    systemctl stop NetworkManager && systemctl disable NetworkManager && echo "NetworkManager" && sleep 1
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux && setenforce  0 && echo "selinux 已关闭" && sleep 1
    \cp ${DIR}/limits.conf /etc/security/limits.conf 
    \cp ${DIR}/sysctl.conf /etc/sysctl.conf
    /bin/tar xvf ${DOCKER_FILE}
    \cp docker/*  /usr/bin
    mkdir /etc/docker && \cp daemon.json /etc/docker

    \cp containerd.service /lib/systemd/system/containerd.service
    \cp docker.service  /lib/systemd/system/docker.service
    \cp docker.socket /lib/systemd/system/docker.socket

    \cp ${DIR}/docker-compose-Linux-x86_64_1.28.6 /usr/bin/docker-compose
    
    groupadd docker && useradd docker -s /sbin/nologin -g docker
    id -u  ${USERNAME} &> /dev/null
    if [ $? -ne 0 ];then
      useradd ${USERNAME}
      usermod ${USERNAME} -G docker
    else 
      usermod ${USERNAME} -G docker
    fi
    install_success_info
  fi
}

ubuntu_install_docker(){
  grep "Ubuntu" /etc/issue &> /dev/null
  if [ $? -eq 0 ];then
    /bin/echo  "当前系统是`cat /etc/issue`,即将开始系统初始化、配置docker-compose与安装docker" && sleep 1
    \cp ${DIR}/limits.conf /etc/security/limits.conf
    \cp ${DIR}/sysctl.conf /etc/sysctl.conf
    
    /bin/tar xvf ${DOCKER_FILE}
    \cp docker/*  /usr/bin 
    mkdir /etc/docker && \cp daemon.json /etc/docker

    \cp containerd.service /lib/systemd/system/containerd.service
    \cp docker.service  /lib/systemd/system/docker.service
    \cp docker.socket /lib/systemd/system/docker.socket

    \cp ${DIR}/docker-compose-Linux-x86_64_1.28.6 /usr/bin/docker-compose

    groupadd docker && useradd docker -r -m -s /sbin/nologin -g docker
    id -u  ${USERNAME} &> /dev/null
    if [ $? -ne 0 ];then
      groupadd  -r  ${USERNAME}
      useradd -r -m -s /bin/bash -g ${USERNAME} ${USERNAME}
      usermod ${USERNAME} -G docker
    else
      usermod ${USERNAME} -G docker
    fi  
    install_success_info
  fi
}


install_success_info(){
    /bin/echo "正在启动docker server并设置为开机自启动!" 
    systemctl  enable containerd.service && systemctl  restart containerd.service
    systemctl  enable docker.service && systemctl  restart docker.service
    systemctl  enable docker.socket && systemctl  restart docker.socket
    sleep 0.5 && /bin/echo "docker server安装完成,欢迎进入docker世界!" && sleep 1
}

main(){
  centos_install_docker  
  ubuntu_install_docker
}

main
```


## Docker 数据卷

### 数据管理

- Lower Dir：image镜像层(镜像本身，只读)
- Upper Dir：容器的上层(读写)，容器运行中产生的文件都会放到此处
- Merged Dir：容器的文件系统，使用Union FS（联合文件系统）将lowerdir和upper Dir：合并给容器使用。
- Work Dir：容器在 宿主机的工作目录

### 数据的持久化
容器删除时会删除读写层的数据（stop时不会），所以需要对需要保存下来的数据进行持久化。
- 使用数据卷
 ```bash
 # docker run -d -v <宿主机目录1>:<容器目录1>[:ro]  [-v <宿主机目录2>:<容器目录2>[:ro]] -p <宿主机端口>:<容器端口> 镜像
docker run -d --name web2 -v /data/testapp:/usr/share/nginx/html/testapp:ro -p 81:80 nginx
 ```
- 使用数据卷容器（基本不使用）
	1. 创建数据卷
	
	```bash
	docker run -d --name volume-server -v data/testapp:/usr/share/nginx/html/testapp -v /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro registry.cn-hangzhou.aliyuncs.com/zhangshijie/pause:3.8
	```

  2. 创建继承的容器
	```bash
	docker run -d --name web1 --volumes-from volume-server -p80:80 nginx:1.20.2
	```

	3. 数据卷的特点
	- client会继承卷server挂载和挂载权限
	- 停止卷server对新建容器及已创建的容器没有影响
	- 删除卷server，不影响已经运行的容器，但是不能新建容器


## Docker 的 bridge 和 container 模式网络

### 网络模式

- null模式 
 是最简单的模式，也就是没有网络，但允许其他的网络插件来自定义网络连接

- bridge 模式
桥接模式，docker的默认网络模式，可以不显式指定网络模式，容器和宿主机再通过虚拟网卡接入这个网桥（即 docker0）。

	 ```bash
	 docker run -d -p 80:80 [--net=bridge] nginx:1.23.2-alpine  
	```
 	可以使用 `brctl` 查看网桥的对应关系
	```bash
	apt install bridge-utils 
	brctl show
	```
 
- host 模式
直接使用宿主机网络，不需要指定端口映射,相当于去掉了容器的网络隔离（其他隔离依然保留），所有的容器会共享宿主机的 IP 地址和网卡。这种模式没有中间层，性能最好，host 模式需要在 docker run 时使用 `--net=host` 参数

- container模式
使用参数 `--net=container` 指定，创建容器时指定另一已存在的容器的网络进行共享net namespace
	1. 创建bridge模式的容器
	```bash
	docker run -d --name ngx1 -p 80:80 b99
	```
	2. 创建container模式的容器
	```bash
	# --net=<需要共享的容器名>
	docker run -d --name php --net=ngx1 php:7.4.30-fpm-alpine
	```

### namespace
ubuntu 22.04 在 `/var/run/docker/netns/` 下
```bash
ln -s /var/run/docker/netns/* /var/run/netns/
ip netns list
ip netns exec 6359049d9b95 ip a

```