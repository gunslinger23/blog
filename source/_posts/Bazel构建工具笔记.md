---
title: Bazel构建工具笔记
date: 2020-06-12 01:36:55
tags:
  - Bazel
  - CI
categories:
  - 技术
---

# 废话

最近在学习微服务，第一次接触到谷歌的MonoRepo大仓库模式，他们用自己写的Bazel构建工具来管理整个仓库代码的构建。

多文件模块构建本身就非常复杂，所有代码塞进一个仓库里，可以说是极其疯狂。但在其中学到了许多关于自动化CI/CD的思想。

# 背景项目

近期重写的NCS（NewPage Core System）使用MonoRepo仓库+微服务。其中也遇到了许多坑，特别是国内特殊的网络环境，让本身用来加快仓库构建的工具变得不那么快🐌

# 笔记

## GitLab CI 正确设置 Bazel 缓存目录

Bazel提供了Remote Cache功能，但是这个是用来缓存构建产物，可本身工具需要各种依赖是不会存储在Remote Cache中。

直接上配置

```shell
# .bazelrc
# Cache
build --experimental_repository_cache_hardlinks
startup --output_user_root=./.cache/bazel
```

通过参数`--output_user_root`，我们可以设置每个WorkSpace都输出目录。我们可以从中看得出，每个WorkSpace都可以设置独立的输出目录以保证单机上每个项目的依赖整洁有序。

那么我们就可以在CI配置中设置缓存相应的目录了

```yaml
# .gitlab-ci.yml
variables:
  GIT_CLEAN_FLAGS: -ffdx -e .cache/

cache:
  key: bazel_cache
  paths:
    - .cache/
```

## WorkSpace 依赖下载问题

在使用官方或其他开发者提供构建规则时，下载相关依赖会特别的慢🐌。那么我们可以在`WORKSPACE`文件内优先将相关资源下载链接设置好，来覆盖原来规则中资源的下载地址。

你可以通过自己下载压缩包放在某个Web，或者是反代理源网站。当然你土豪的话也可以使用梯子代理来加快下载速度⚡️。

### 覆盖下载例子

```python
# WORKSPACE
# Download the rules_docker repository at release v0.14.3
http_archive(
    name = "io_bazel_rules_docker",
    sha256 = "6287241e033d247e9da5ff705dd6ef526bac39ae82f3d17de1b69f8cb313f9cd",
    strip_prefix = "rules_docker-0.14.3",
    urls = ["https://new.download.site/rules_docker-v0.14.3.tar.gz"],
)
```

Bazel通过`http_archive` `git_repository`等函数中的`name`参数进行路由，例如我们使用该资源可以通过`@io_bazel_rules_docker`来访问其中的文件。按么我们只要保证名字对应，就能成功覆盖那些下载不到或者慢的资源了👌

> 注意：规则的相关依赖需要自己从相关仓库中找

## Docker 镜像自动化上传

如果你也是懒人（像我那样😂），那么你可以自己基于`rules_docker`写一个自动化脚本～

```python
# build/docker.bzl
load("@io_bazel_rules_docker//go:image.bzl", "go_image")
load("@io_bazel_rules_docker//container:container.bzl", "container_image", "container_push")

def push_docker(name, app_name, embed):
    # golang 程序自动打包镜像
    # embed 为 go_library 对象
    go_image(
        name = "app",
        embed = embed,
    )
    container_image(
        name = "image",
        base = ":app",
        stamp = True,
    )
    container_push(
        name = name,
        image = ":image",
        format = "Docker",
        registry = "gcr.io",
        repository = "my-project/" + app_name,
        tag = "dev",
    )
```

```python
# BUILD.bazel
load("@io_bazel_rules_go//go:def.bzl", "go_binary", "go_library")
load("//build:docker.bzl", "push_docker")

go_library(
    name = "go_default_library",
    srcs = ["main.go"],
    importpath = "package/cmd",
    visibility = ["//visibility:private"],
    deps = [
        "//path:some_library",
    ],
)

go_binary(
    name = "cmd",
    embed = [":go_default_library"],
    visibility = ["//visibility:public"],
)

push_docker(
    name = "push",
    app_name = "my-image",
    embed = [":go_default_library"],
)

```

那么我们只需要执行一下指令即可自动构建镜像并上传啦

```shell
# image tag: grc.io/my-project/my-image:dev
bazel run //path:push
```

> 交叉编译需要添加编译目标平台参数`--platforms`，不同语言规则有不同的写法，这里就不一一叙述了。

# 结尾

这里推荐各位有兴趣的小伙伴们去看看 [kubernetes](https://github.com/kubernetes/kubernetes) 的官方仓库，看他们是如何使用Bazel来管理仓库的构建+部署，而且里面也有许多他们编写的相关Hack脚本来辅助使用Bazel。

🙏感谢观看

