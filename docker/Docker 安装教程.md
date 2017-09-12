# Docker 安装教程

## 准备工作

* 一台服务器，也可以是你工作用的笔记本。我用的是 阿里云服务器，还不错。
* 操作系统的话还是用 Linux，程序员必备。推荐 CentOS。

## 官方教程
[Docker 官网](https://www.docker.com/)

[Docker CE CentOS 官方安装教程](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)

## 简介

本教程介绍在阿里云服务器，CentOS 7.3 64位 环境下安装 Docker CE 版。

简单介绍一下 `Docker CE` 和 `Docker EE` 的区别：Docker CE (community-edition) 是社区版，免费，开源，Docker EE(enterprise-edition) 为企业版需付费，区别与定价如下图：

![](https://github.com/Txiaozhe/docs/tree/master/docker/docker-plan.png)

## 安装

### 环境要求

* 64 位 CentOS 7.x

### 清除旧版本（如果有的话）

```sh
$ sudo yum remove docker \
                  docker-common \
                  docker-selinux \
                  docker-engine
```

### 安装 Docker CE

官方给了几种安装方式，本教程只介绍使用代码库的方式进行安装。

#### 设置代码仓库

* 安装 `yum-utils` 工具包，`yum-utils` 提供了 `yum-config-manager`、`device-mapper-persistent-data` 和 `lvm2` 等工具。设置代码仓库的时候会用到：

  ```sh
  $ sudo yum install -y yum-utils \
    device-mapper-persistent-data \
    lvm2
  ```

* 设置稳定版的代码仓库：

  ```sh
  $ sudo yum-config-manager \
      --add-repo \
      https://download.docker.com/linux/centos/docker-ce.repo
  ```

* 可选：也可以手动设置 edge 和 test 版的仓库：

  ```sh
  $ sudo yum-config-manager --enable docker-ce-edge # 设置 edge
  ```

  ```sh
  $ sudo yum-config-manager --enable docker-ce-test # 设置 test
  ```

  当然也可以手动禁用：

  ```sh
  $ sudo yum-config-manager --disable docker-ce-edge
  ```

#### 安装 Docker CE

* 安装最新版本的 Docker CE：

  ```sh
  $ sudo yum install docker-ce # 默认安装最新版 Docker CE
  ```

  安装完后会在命令行显示一串秘钥，可以比对一下确认安装是否正确：

  ```sh
  060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35
  ```

  此时 Docker 已经安装完毕，但是还没有启动。

* 生产环境下往往需要安装特定版本的 Docker CE，使用以下命令可以列出所有可用的版本：

  ```sh
  $ yum list docker-ce.x86_64  --showduplicates | sort -r

  docker-ce.x86_64            17.06.ce-1.el7.centos             docker-ce-stable
  ```

  然后安装特定版本的 Docker CE：

  ```sh
  $ sudo yum install <FULLY-QUALIFIED-PACKAGE-NAME>
  ```

* 启动 Docker：

  ```sh
  $ sudo systemctl start docker
  ```

* 启动  `hello-world` 镜像进行测试

  ```sh
  $ sudo docker run hello-world
  ```

#### 升级 Docker CE

```sh
$ sudo yum makecache fast
```

到此为止使用仓库的方式安装 Docker CE 就完成了，祝您 Docker 旅途愉快！！！