# OpenV2X 部署文档

## 1. 部署架构

### 1.1 All-in-One 架构

![](/docs/images/openv2x-deployment-aio.drawio.svg)

### 1.2 多节点部署架构

![](/docs/images/openv2x-deployment-mutlinodes.drawio.svg)

未来：

1. 主站点部分可以缩容，只保留 omega 和 dandelion（仅云端模块）
2. 视频流和点云图应该可以走反向代理，而不是直接让客户端浏览器访问

## 2. All-in-One 部署

### 2.1 基本环境

硬件：4Core / 8G / 100G

OS：CentOS7-2009

```console
[root@v2x-release ~]# cat /etc/redhat-release
CentOS Linux release 7.9.2009 (Core)
```

注：Ubuntu 22.04 也经过测试，但本文仅以 CentOS 7 2009 为例说明

### 2.2 kernel 升级

```shell
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install -y https://www.elrepo.org/elrepo-release-7.0-4.el7.elrepo.noarch.rpm
yum -y --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum -y --disablerepo=\* --enablerepo=elrepo-kernel install kernel-lt.x86_64

awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg

grub2-set-default 0
# 这里的 0 要根据实际情况来填写

reboot
```

确认 Kernel 已经升级到 5.4

```console
[root@v2x-release ~]# uname -a
Linux v2x-release 5.4.203-1.el7.elrepo.x86_64 #1 SMP Fri Jul 1 09:00:33 EDT 2022 x86_64 x86_64 x86_64 GNU/Linux
```

## 2.3 docker 升级

```shell
yum remove -y docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
systemctl enable docker --now

echo "alias docker-compose='docker compose'" >> /etc/profile && source /etc/profile

docker version
docker-compose version
```

## 2.4 安装 curl

```shell
yum install curl
```

## 2.5 下载安装包

```shell
rm -rf openv2x-aio-master.tar.gz && wget https://openv2x.oss-ap-southeast-1.aliyuncs.com/deploy/master/openv2x-aio-master.tar.gz
rm -rf src && tar zxvf openv2x-aio-master.tar.gz
cd src

# yum install unzip -y
# rm -rf master.zip && wget https://gitee.com/open-v2x/docs/repository/archive/master.zip
# rm -rf docs-master && unzip master.zip
# cd docs-master/src/
```

## 2.6 一键部署服务

```shell
# 这里的外部 IP 地址要确保客户端可以访问，该 IP 用于访问边缘路口上的 Portal 和各类 Web 服务。
export OPENV2X_EXTERNAL_IP=100.100.100.100
# 这里的中心 IP 地址要确保客户端可以访问，该 IP 用于访问中心云端的 Portal 页面。 
export OPENV2X_CENTER_IP=100.100.100.100
export OPENV2X_IS_CENTER=true
export OPENV2X_REDIS_ROOT=password
export OPENV2X_MARIADB_ROOT=password
export OPENV2X_MARIADB_DANDELION=password
export OPENV2X_EMQX_ROOT=password

export OPENV2X_REGION=cn

export OPENV2X_ENABLE_DEMO_LIDAR=true
export OPENV2X_ENABLE_GPU=false
export OPENV2X_ENABLE_DEMO_CAMERA=true
export OPENV2X_ENDPOINT_HTTP_FLV=http://<HippoCampus-GPU-服务器-IP>:10101/live
export OPENV2X_ENDPOINT_LIDAR=ws://${OPENV2X_EXTERNAL_IP}:8000/ws/127.0.0.1

bash ./install.sh
```

安装效果如下：

```console
[root@v2x-demo src]# bash ./install.sh 

...

  openv2x has been installed successfully!
                                       ________           
  ____  ______    ____    ____ ___  __\_____  \ ___  ___ 
 /  _ \ \____ \ _/ __ \  /    \  \/ / /  ____/ \  \/  / 
(  <_> )|  |_> >\  ___/ |   |  \   / /       \  >    <  
 \____/ |   __/  \___  >|___|  / \_/  \_______ \/__/\_ \ 
        |__|         \/      \/               \/      \/ 
    repository: https://github.com/open-v2x
    portal: https://openv2x.org

  OpenV2X Central Omega Portal: http://100.100.100.100:2288

  username: admin
  password: dandelion
```

上述提示中包含了 Central Omega Portal 的访问路径，以及用户名密码。此时可以从客户端，通过 Chrome 浏览器（其它浏览器未测试）访问试用。

欢迎试用～，参考：[快速入门](v2x-quick-start.md)。

如果遇到问题，欢迎在 github 提交
issue：<https://github.com/open-v2x/docs/issues/new/choose>，参考：[提交注意事项](v2x_contribution-zh_CN.md)。

## 3. 多节点部署

### 3.1 基本环境

3 台服务器，一台作为中心站点，两台作为边缘站点

硬件：2Core / 4G / 100G

OS：ubuntu20.04

### 3.2 安装 docker 和 docker compose

确认每台服务器都安装好了 docker 和 docker compose

安装 docker

```shell
curl -sSL https://get.daocloud.io/docker | sh
```

安装 docker compose。 如果不想要 v2.2.2 版本，可以进入最新发行的版本地址：https://github.com/docker/compose/releases 查看版本号。

```shell
curl -L "https://github.com/docker/compose/releases/download/v2.2.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
docker version
docker-compose version
```

### 3.3 下载安装包

在每台服务器上下载安装包

```shell
rm -rf openv2x-aio-master.tar.gz && wget https://openv2x.oss-ap-southeast-1.aliyuncs.com/deploy/master/openv2x-aio-master.tar.gz
rm -rf src && tar zxvf openv2x-aio-master.tar.gz
cd src
```

## 3.4 部署服务

### 3.4.1 部署中心站点

在中心站点执行以下 shell

```shell
# 这里的外部 IP 地址要确保客户端可以访问，该 IP 用于访问边缘路口上的 Portal 和各类 Web 服务。
export OPENV2X_EXTERNAL_IP=100.100.100.100
# 这里的中心 IP 地址要确保客户端可以访问，该 IP 用于访问中心云端的 Portal 页面。
export OPENV2X_CENTER_IP=100.100.100.100
export OPENV2X_IS_CENTER=true
export OPENV2X_REDIS_ROOT=password
export OPENV2X_MARIADB_ROOT=password
export OPENV2X_MARIADB_DANDELION=password
export OPENV2X_EMQX_ROOT=password

export OPENV2X_REGION=cn

export OPENV2X_ENABLE_DEMO_LIDAR=true
export OPENV2X_ENABLE_GPU=false
export OPENV2X_ENABLE_DEMO_CAMERA=true
export OPENV2X_ENDPOINT_HTTP_FLV=http://<HippoCampus-GPU-服务器-IP>:10101/live
export OPENV2X_ENDPOINT_LIDAR=ws://${OPENV2X_EXTERNAL_IP}:8000/ws/127.0.0.1

bash ./install.sh
```

### 3.4.2 部署边缘站点

在每台边缘站点服务器上执行以下 shell

```shell
# 这里的外部 IP 地址要确保客户端可以访问，该 IP 用于访问边缘路口上的 Portal 和各类 Web 服务。
export OPENV2X_EXTERNAL_IP=100.100.100.100
# 这里的中心 IP 地址要确保客户端可以访问，该 IP 用于访问中心云端的 Portal 页面。
export OPENV2X_CENTER_IP=101.101.101.101
export OPENV2X_IS_CENTER=false
export OPENV2X_REDIS_ROOT=password
export OPENV2X_MARIADB_ROOT=password
export OPENV2X_MARIADB_DANDELION=password
export OPENV2X_EMQX_ROOT=password

export OPENV2X_REGION=cn

export OPENV2X_ENABLE_DEMO_LIDAR=true
export OPENV2X_ENABLE_GPU=false
export OPENV2X_ENABLE_DEMO_CAMERA=true
export OPENV2X_ENDPOINT_HTTP_FLV=http://<HippoCampus-GPU-服务器-IP>:10101/live
export OPENV2X_ENDPOINT_LIDAR=ws://${OPENV2X_EXTERNAL_IP}:8000/ws/127.0.0.1

bash ./install.sh
```

安装效果如下：

```console
[root@v2x-demo src]# bash ./install.sh 

...

  openv2x has been installed successfully!
                                       ________           
  ____  ______    ____    ____ ___  __\_____  \ ___  ___ 
 /  _ \ \____ \ _/ __ \  /    \  \/ / /  ____/ \  \/  / 
(  <_> )|  |_> >\  ___/ |   |  \   / /       \  >    <  
 \____/ |   __/  \___  >|___|  / \_/  \_______ \/__/\_ \ 
        |__|         \/      \/               \/      \/ 
    repository: https://github.com/open-v2x
    portal: https://openv2x.org

  OpenV2X Central Omega Portal: http://101.101.101.101:2288

  username: admin
  password: dandelion
```

上述提示中包含了 Central Omega Portal 的访问路径，以及用户名密码。此时可以从客户端，通过 Chrome 浏览器（其它浏览器未测试）访问试用。
