---
title: Apple Silicon 个人开发体验
date: 2023-07-30 22:50:00
tags:
  - MacOS
  - Apple Silicon
  - 开发环境
  - 虚拟
categories:
  - 技术
---

## 前言

最近没事闲着蛋疼，入手了一台 M2 Max 版本的 Macbook Pro。折腾之路从此开始～

工作是 Windows 桌面应用/后端开发，之前那台 Intel 版本用的折腾了半天最终使用 Parallel Desktop 虚拟机来跑 Windows 环境。而这次换成了全新的 ARM 架构后，也是对 ARM 架构开发环境的一次尝试。

Apple 在对 x86 架构程序转移到 ARM 架构上给出了一个方案，那就是 `Rosetta2` 转译环境。而后面微软也紧跟着在 Windows 11 上推出了自己的 x86 转译环境，两者都均能在 ARM 架构环境中运行 x86 架构程序，但也不是完美运行，有时候可能会抽风。

以下有关速度的对比，是和 i7 9代标压的 Macbook Pro 对比的，速度上的形容只能说是大概描述，也没必要太纠结多几秒的问题。

## 开发环境

### Windows

在 Mac 上进行 Windows 开发就是属于脑抽的行为（当然我就是。。。

试了这么多虚拟机，最后还是觉得花钱的 Parallel Desktop 最省心。而在 M2 上则必须安装 Windows 11 ARM 版本。如果想折腾用旧系统例如 Windows 10，那只能通过 QEUM 来模拟了，速度也是像蜗牛一样（真的很慢，不要折腾了）

虚拟机配置：ARM上使用了 6 核心 24G 内存的设置（内存多就是爽 🤣

我在 Windows 上主要的开发工具就是 VS 了，在 22 年，微软在 insider 版本推出 ARM 架构的 VS 2022。也在今年推送到正式版了。

目前看来 .NET 环境以及全面支持 ARM 架构（包括编译器、调试器），如果目标架构是 x86 的，仅程序和调试器会运行 x86 上，其他仍在 ARM 架构上，速度我觉得是非常可以的。

> 开发环境: .NET Framework 4.0 winForm / .NET Core 3.0

最好用的JB 家的 Reshaper 插件也已经支持了 ARM 版本的 VS 2022。对于不支持 ARM 的或者是老版本的插件，只能安装旧版的 x86 VS 来使用，速度上我感觉比原来的 Intel 版本要慢点。

这里要吐槽一下 GitHub Copilot 的 VS 插件，写着支持 ARM64，却装不上。。。用了点歪门邪道方法才装上。

> https://stackoverflow.com/a/76737023

基本上没有什么 BUG，除了非 ARM 原生程序速度上会慢一点，你完全可以相信 M2 的性能。

---

### Linux

在 Mac 上跑 Linux 就舒服多了，虚拟化上我选择了 Lima。Linux 上我用的最多的就是 Docker 了，我用 Colima 来进行管理 Docker。我只能说非常好用

但是 Docker 大部分镜像是 x86 架构的，公司的镜像也是 x86。这就有效率问题了，不过好在 Apple 推出了为 Linux ARM 虚拟机提供 Rosetta2 功能。

> https://developer.apple.com/documentation/virtualization/running_intel_binaries_in_linux_vms_with_rosetta

我们只需要输入以下命令就可以启用 Rosetta2 的虚拟机了

```bash
colima start --arch aarch64 --vm-type=vz --vz-rosetta
```

这里简单解释一下环境

- 虚拟机架构：ARM 
- 系统运行架构：Linux（aarch64）
- Docker运行架构：ARM

所以如果是正常的 `docker run` 镜像是会使用 ARM 架构的镜像

如果我们需要运行 x86 的镜像，则需要加上参数 `--platform linux/amd64`

这样我们就能在 ARM 架构优雅快速的跑起 x86 的镜像了

> 不过要注意 Apple Silicon 在 Rosetta2 上不支持 `AVX、AVX2、AVX512` 指令集，如果程序使用了这些指令集，那么就会报错无法运行（建议自己编译一份吧

---

### MacOS

没啥好说的，几乎全部都支持 ARM 架构了，速度上不比 Intel 差。

例如在 Node 开发，编译速度完全碾压 x86，`npm run dev` 几乎等都不用等，有点离谱。

## 整体开发体验

由于大部分程序都原生支持 ARM 或是 ARM 虚拟化，Windows ARM 上几乎也是全 ARM 原生程序，开发过程功耗非常低。

Windows 编译的时候可能来到 30w 左右，一天下来风扇也是维持在 1000 RPM 左右，非常安静。我只能比 Intel 好太多了（Intel 你坏事做尽

功耗低，发热低，续航也是非常优秀。以前几乎不可能的断电开发，现在终于实现了，也是直接突破我的认知了。

最后如果你要用虚拟机，请务必上 32G 以上的机型，Apple Silicon 是改不了内存的。