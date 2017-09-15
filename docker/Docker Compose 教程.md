# Docker Compose 教程

## 准备工作

- 一台服务器，也可以是你工作用的笔记本。我用的是 阿里云服务器，还不错。
- 操作系统的话还是用 Linux，程序员必备。推荐 CentOS。

## 官方教程

[Docker 官网](https://www.docker.com/)

[Docker Compose Linux 安装教程](https://docs.docker.com/compose/install/#install-compose)

## 安装

### 下载最新版的 Docker Compose：

```sh
sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

### 给二进制文件赋予可执行权限：

```sh
sudo chmod +x /usr/local/bin/docker-compose
```

### 测试安装是否完成：

```sh
$ docker-compose --version
docker-compose version 1.16.1, build 1719ceb # 看到这个样子的输出则表示安装成功
```

### 升级

> 如果你是从 1.2 或更早版本升级的，那么在升级之后需要删除或迁移现有的容器。因为 1.3 以后版本的 Compose 是使用 Docker 标签来跟踪容器的，所以他们需要重新添加标签。

如果 Compose 检测到没有标签的容器，它将拒绝运行。如果还想继续使用该容器，你可以使用 1.5.x 版本的 Compose：

```sh
$ docker-compose migrate-to-labels
```

如果不继续使用，可以删除，然后创建新的：

```shell
docker rm -f -v myapp_web_1 myapp_db_1 ...
```

### 卸载

* 使用 `curl` 安装：

```sh
sudo rm /usr/local/bin/docker-compose
```

* 使用 `pip` 安装：

```sh
pip uninstall docker-compose
```

## 使用

### 第一步：安装程序

#### 定义应用程序依赖项

* 创建目录

```shell
$ mkdir composetest
$ cd composetest
```

*  创建一个 `app.py` 测试文件并添加如下代码：

```python
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

在这个例子中， `redis` 是 `redis` 容器的主机名，使用默认的 Redis 端口：6379.

* 创建 `requirements.txt` 并添加如下字段：

```shell
flask
redis
```

###  第二步：创建 Dockerfile 文件

> 新建一个 Docker 文件来构建 Docker 镜像，这个镜像包含了这个 Python 程序所要求的所有依赖，也包含 Python 本身。

在工程目录下创建名为 `Dockerfile` 的文件并写入以下代码：

```dock
FROM python:3.4-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

以上代码：

> * 以 Python 3.4 为基础镜像构建一个镜像
> * 在镜像的 `/code` 目录中添加 `.` 目录
> * 设置工作目录：`/code`
> * 安装 Python 依赖
> * 设置默认命令：`python app.py`

从 [Docker 用户指南](https://docs.docker.com/engine/tutorials/dockerimages/#building-an-image-from-a-dockerfile) 和 [Dockerfile 参考 ](https://docs.docker.com/engine/reference/builder/)查看更多 Dockerfile 相关内容。

### 第三步：在 Compose 文件中定义服务

创建名叫 `docker-compose.yml` 的文件并添加以下代码：

```yaml
version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
  redis:
    image: "redis:alpine"
```

这个Compose 文件定义了两个服务：`web` 和 `redis`。

web 服务：

> * 使用由 Dockerfile 创建的镜像
> * 将主机 `5000` 端口映射至 容器的 `5000` 端口

### 第四步：使用 Compose 构建并运行程序

* 在应用根目录下运行如下命令：

```shell
$ docker-compose up
```

Compose 将会自动拉取 Redis 镜像、构建一个应用镜像并开启服务，在这种情况下，代码在生成时被静态复制到镜像中。

* 在浏览器中输入 `http://0.0.0.0:5000/` 查看应用运行状态。

![](https://docs.docker.com/compose/images/quick-hello-world-1.png)

* 刷新页面

![](https://docs.docker.com/compose/images/quick-hello-world-2.png)

* 切换到另外一个终端窗口，运行 `docker image ls` 查看镜像列表。

  也可以通过 `docker inspect <tag or id>` 检查镜像。

* 运行 `docker-compose down` 命令或通过 `CTRL-C` 终止程序运行。

### 第五步：编辑 Compose 文件添加绑定挂载

编辑 `docker-compose.yml` 来为 web 服务添加绑定挂载：

```yaml
version: '3'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: "redis:alpine"
```

`volumes` 挂载工程目录到主机容器的 `/code` 目录。

### 第六步：使用 Compose 重新构建和运行应用

在项目目录下运行 `docker-compose up` 重新构建并运行应用。

### 第七步：升级应用

因为此时程序已经挂载到容器卷中，因此可以实时看到代码修改的效果而无需重构镜像。

按照如下代码修改 `app.py` 并保存：

```python
return 'Hello from Docker! I have been seen {} times.\n'.format(count)
```

 刷新网页

![](https://docs.docker.com/compose/images/quick-hello-world-3.png)

### 第八步：用其他命令进行实验

在后台运行服务：

```shell
$ docker-compose up -d
```

查看正在运行的服务

```shell
$ docker-compose ps
```

`docker-compose run` 用来运行服务中的命令，如：

```shell
$ docker-compose run web env
```

 查看帮助

```shell
$ docker-compose --help
```

停止运行服务

```shell
$ docker-compose stop
```

卸载挂载的卷

```shell
$ docker-compose down --volumes
```

以上就是 Docker Compose 的基本使用了。