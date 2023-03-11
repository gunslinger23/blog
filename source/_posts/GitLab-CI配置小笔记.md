---
title: GitLab CI配置小笔记
date: 2020-04-02 16:18:31
tags:
  - GitLab
  - CI
categories:
  - 技术
---

## 遇到问题

近期更新项目的时候遇到了问题

- 写Markdown文档，频繁提交触发了多个无用的构建
- 只有当仓库中存在Dockerfile才触发构建容器
- 覆盖引用文件中已定义Job中的环境变量

## 解决方案

在研究了GitLab CI的触发流水线相关语法后，发现rules关键字能满足我的要求

而only/except无法满足，并且提示后期将有可能**删除该语法糖**

> 值得注意的是在rules拥有子项的时候，默认不匹配的操作为never，即不触发

```yaml
rules:
  # 指定分支
  - if: $CI_COMMIT_BRANCH == "master"
    when: on_success
  #- when: never 这段是多余的
```

那么直接上脚本~

---

### 修改某文件不触发

```yaml
rules:
  # 仅针对push事件
  - if: $CI_PIPELINE_SOURCE == "push"
    changes:
    - "**.md"
    when: never #不触发
  # 其余仅之前流水线成功后开始
  - when: on_success
```

### 仅存在某文件时触发

```yaml
rules:
  # 指定分支
  - if: $CI_COMMIT_BRANCH == "master"
    exists:
    - Dockerfile
    when: on_success
```

### 覆盖Job配置

```yaml
include: xxx.yml
# 引用文件已定义的Job
job:
  # 对于其他配置也可以在主配置文件中覆盖
  variables:
    xxx: xxx
```

## 参考文档

rules：<https://docs.gitlab.com/ee/ci/yaml/README.html#rules>

预设环境变量：<https://docs.gitlab.com/ee/ci/variables/predefined_variables.html>

变量优先级：<https://docs.gitlab.com/ee/ci/variables/README.html#priority-of-environment-variables>
