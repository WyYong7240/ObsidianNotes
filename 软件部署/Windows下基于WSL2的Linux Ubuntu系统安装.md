---
tags:
  - WSL
  - WSL2
  - Linux
  - Ubutnu
---

# 预先准备

> 让Windows系统具备能够使用WSL、WSL2的能力

1. 在控制面板中选中**程序与功能**
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202601192039585.png" alt="image.png" style="zoom:80%;" />
1. 点击 **启用或关闭Windows功能**，并在窗口中勾选 **虚拟机平台、适用于Linux的Windows子系统**
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202601192042306.png" alt="image.png" style="zoom:80%;" />
1. 重启电脑

# 安装Ubuntu并设施WSL版本

## 安装Ubuntu WSL

1. 在命令行中，输入

   ~~~sh
   wsl --install
   ~~~

2. 在安装完成后，输入如下命令查看已安装的WSL

   ~~~sh
   wsl -l -v
   ~~~
   <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202601192045661.png" alt="image.png" style="zoom:80%;" />

3. 运行指定的WSL，以Ubuntu为例

   ~~~sh
   wsl -d ubuntu
   
   // 如果需要停止运行，使用如下命令
   wsl -t ubuntu
   ~~~

## 设置Ubuntu WSL版本

* WSL1与WSL2的区别

  1. 架构差异

     | 特性                    | WSL 1                                                    | WSL 2                                            |
     | ----------------------- | -------------------------------------------------------- | ------------------------------------------------ |
     | 是否使用真实 Linux 内核 | ❌ 否，通过翻译层将 Linux 系统调用转换为 Windows 系统调用 | ✅ 是，运行由 Microsoft 编译优化的真实 Linux 内核 |
     | 是否基于虚拟机（VM）    | ❌ 否，直接集成在 Windows 内核中                          | ✅ 是，使用轻量级 Hyper-V 虚拟机运行 Linux 内核   |
     | 系统调用兼容性          | ⚠️ 部分兼容（依赖翻译层）                                 | ✅ 完全兼容（原生内核支持）                       |

     > - **WSL 1** ≈ “模拟 Linux”，把 Linux 命令“翻译”成 Windows 能懂的语言。
     > - **WSL 2** ≈ “真 Linux”，在轻量 VM 里跑一个完整的 Linux 系统。

  2. 性能对比

     | 场景                                                  | WSL 1          | WSL 2                                           |
     | ----------------------------------------------------- | -------------- | ----------------------------------------------- |
     | Linux 文件系统内操作（如 `git clone`, `npm install`） | 较慢           | ⚡ 快 2～5 倍，甚至 20 倍（尤其 I/O 密集型任务） |
     | 访问 Windows 文件系统（如 `/mnt/c/...`）              | ✅ 更快         | ❌ 较慢（跨 OS 文件访问有开销）                  |
     | 启动速度 & 资源占用                                   | 极轻量、启动快 | 略高内存占用，但仍是轻量级 VM                   |

     > - 把项目文件放在 **Linux 文件系统**（如 `～/project`）中使用 WSL 2，性能最佳。
     > - 如果必须频繁在 Windows 和 Linux 之间共享文件（如用 VS Code + Windows 工具编辑 Linux 项目），且无法迁移文件位置，可考虑 WSL 1。

  3. 功能支持对比

     | 功能                        | WSL 1                     | WSL 2                                                   |
     | --------------------------- | ------------------------- | ------------------------------------------------------- |
     | Docker 支持                 | ❌ 不支持（无内核特性）    | ✅ 原生支持（可运行 Docker Desktop 或 rootless Docker）  |
     | systemd 支持                | ❌ 不支持                  | ✅ 已支持（自 2023 年起官方启用）                        |
     | USB / 串口设备访问          | ✅ 支持（如 `/dev/ttyS0`） | ❌ 默认不支持（需额外工具如 `usbipd-win`）               |
     | IPv6                        | ✅ 支持                    | ✅ 支持                                                  |
     | 与 VMware / VirtualBox 共存 | ✅ 兼容                    | ⚠️ 需较新版本（VMware ≥15.5.5，VirtualBox 存在兼容问题） |
     | 网络模式                    | 桥接（与主机同网段）      | NAT（独立虚拟网络，IP 会变，端口需转发）                |

* 何时使用何系统

  1. **推荐使用** ***\*WSL 2\**** **的情况：**
     - 开发需要完整 Linux 兼容性（如 Docker、systemd、内核模块等）
     - 执行大量文件操作（构建、安装包、克隆仓库等）
     - 使用现代 Linux 工具链（如 Podman、Kubernetes in WSL）
  2. **考虑使用** ***\*WSL 1\**** **的情况：**
     - 项目**必须存储在 Windows 文件系统**（如 C:\projects），且频繁被 Windows 工具访问
     - 需要直接访问**串口设备**（如嵌入式开发 ESP32，且不想配置 usbipd）
     - 内存资源非常紧张（WSL 2 会缓存内存，可能暂占较多 RAM）
     - 使用旧版 VirtualBox 且无法升级

* 切换WSL版本，或者设置分发版本

  1. 将Ubuntu设置为WSL2

     ~~~sh
      wsl.exe --set-version Ubuntu-20.04 2
     ~~~

  2. 将Ubuntu设置为WSL1

     ~~~sh
      wsl.exe --set-version Ubuntu-20.04 1
     ~~~

# 将安装的WSL移动存储位置

> 所有的WSL默认安装在C盘，这很占用C盘空间

1. 关闭WSL

   ~~~sh
   wsl --shutdown
   ~~~

2. 导出指定WSL到其他位置，以Ubuntu为例

   ~~~sh
   # 创建目标目录
   mkdir D:\WSL
   
   # 导出为 tar 文件
   wsl --export ubuntu D:\WSL\ubuntu.tar
   ~~~

3. 注销原发行版，这会删除C盘中对应数据

   ~~~sh
   wsl --unregister ubuntu
   ~~~

4. 从D盘重新导入

   ~~~sh
   wsl --import ubuntu D:\WSL\Ubuntu D:\WSL\ubuntu.tar --version 2
   ~~~

   第二个参数 `D:\WSL\Ubuntu` 是新系统的根目录（会自动创建），里面将生成 `ext4.vhdx`。

   然后可以选择性的删除`D:\WSL\ubuntu.tar`

5. 查看并运行重新导入的WSL2 Ubuntu

   ~~~sh
   wsl -l -v
   
   wsl -d ubuntu
   ~~~

   
