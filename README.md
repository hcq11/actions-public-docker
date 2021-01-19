# actions-public-docker
golang app CI workflow demo using github action

对于 Go 语言项目的CI过程，推荐使用多阶段容器构建方式进行构建.

既可以解决 Go 程序的跨平台问题，又可以实现发布镜像的最小化。

首先，通过提供Dockerfile的方式完成本地化构建。
```dockerfile
FROM golang:1.14-alpine AS builder
# 按需安装依赖包
# RUN  apk --update --no-cache add gcc libc-dev ca-certificates  
# 设置Go编译参数

ARG VERSION
ARG COMMIT
ARG BUILDTIME
WORKDIR /app
COPY . .
RUN GOOS=linux go build -o main -ldflags "-X github.com/x-mod/build.version=${VERSION} -X github.com/x-mod/build.commit=${COMMIT} -X github.com/x-mod/build.date=${BUILDTIME}"

# 第二阶段
FROM  alpine
# 安装必要的工具包
RUN  apk --update --no-cache add tzdata ca-certificates \
    && cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
COPY --from=builder /app/main /usr/local/bin
ENTRYPOINT [ "main" ]
````
再通过 Makefile命令模拟CI触发：
```shell script

image:
 docker build --build-arg VERSION=${GITTAG} --build-arg COMMIT=${COMMIT} --build-arg BUILDTIME=${BUILD_TIME} -t ${DOCKER_USER}/${PROJECT}:latest .

```
完整的样例，完成上面代码后，其实整个CI过程就已经本地化了。就可以通过

$: make image
验证构建过程。验证通过后，我只需要将触发过程迁移到 Github Actions 上即可。

现在就样例项目写一个 Github Actions 构建模板:
```dockerfile
# This is a basic workflow to help you get started with Actions for Golang application

name: CI

# Controls when the action will run. Triggers the workflow on push or pull request
# events but only for the master branch
on:
  push:
    branches:
      - master
    tags:
      - "v*"

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - uses: actions/checkout@v2

      - name: Define variables
        run: |
          echo ::set-env name=PROJECT::$(echo "actions-public-docker")
          echo ::set-env name=VERSION::$(git describe --tags)
          echo ::set-env name=COMMIT::$(git rev-parse HEAD)
          echo ::set-env name=BUILDTIME::$(date +%FT%T%z)      - name: Login to docker hub
        uses: actions-hub/docker/login@master
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}

      - name: docker build
        run: docker build --build-arg VERSION=${VERSION} --build-arg COMMIT=${COMMIT} --build-arg BUILDTIME=${BUILDTIME} -t ${DOCKER_USERNAME}/${PROJECT}:${IMAGE_TAG} .
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}

      - name: Push to docker hub
        uses: actions-hub/docker@master
        with:
          args: push ${DOCKER_USERNAME}/${PROJECT}:${IMAGE_TAG}
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}

```
这个过程分为 4 个步骤：

准备环境变量
登录 Docker Hub
构建镜像
发布镜像到 Docker Hub
在第 3 步中，如果不是因为需要传递参数，可以直接使用make image命令。

这样一个构建模板，其实整个过程是不依赖与特定开发语言的，因为项目的构建过程已经完全本地化了，即通过本地Dockerfile实现。所以，这个模板可以适用范围是很广的，完全与开发语言无关。

Github Actions 变量设置
关于 Github Actions 的基础概念，我想阮一峰的这篇文章GitHub Actions 入门教程讲的很清楚了，我就不在赘述了。

这里单独将 Github Actions 环境变量的设置列出来说一下。

因为，当我们拿到一个 CI 模板之后，首先遇到的就是如何修改参数或是变量值。

在样例项目中，整个 CI 的 JOB 均是运行在同一台 ubuntu-latest 的虚拟机上。所以可以通过设置系统的环境变量来设置相关参数。

如样例中的第一步：

echo ::set-env name=BUILDTIME::$(date +%FT%T%z)
通过这个命令来设置环境变量。

这个命令的发现过程还挺有意思的。因为需要使用 Docker 发布，所以就找到了 actions-hub/docker 操作。

发现其中的参数 ${IMAGE_TAG} 并非 GitHub Actions 默认提供，所以就看了一下actions-hub/docker/login的实现代码。

其中就有这样一段代码，

echo ::set-env name=IMAGE_TAG::${IMAGE_TAG}
echo ::set-env name=IMAGE_NAME::${IMAGE_NAME}
所以，也就依葫芦画瓢借来做自己的环境变量定义使用了。

除了自定义的环境变量以外，Github Actions 本身提供默认变量可以参考该文档using environment variables.

对于一些密钥类的设置，则可以通过在项目中增加对应密钥变量的方式进行设置。官方参考文档:creating-and-storing-encrypted-secrets.

## 样例模板中

- name: Login to docker hub
        uses: actions-hub/docker/login@master
        env:
          DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
          DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
变量值DOCKER_USERNAME, DOCKER_USERNAME就来源于密钥设置。这里要注意的是，在 step 中通过 env 定义的变量只能在该 step 中使用，不可以全局使用。

有了以上相关变量设置的知识，一些基本的 GitHub Actions 问题都可以迎刃而解了。

# 小结
样例模板实现了每次master主分支推送以及tag推送触发容器镜像的构建，需要的同学可以 clone 该样例项目自行测试。