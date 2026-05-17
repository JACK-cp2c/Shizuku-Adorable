# Suiku Ultra（Shizuku）
<div align="center">
  <img src="https://img.cdn1.vip/i/6a094979158ba_1778993529.webp" alt="Suiku Ultra Logo" width="150">
  <h1>Suiku Ultra：萌化版的shizuku </h1>
  <p align="center" style="font-size: 2.0em;">

# Background 背景
**此版本仅限简体中文的内容萌化🥲🥲**
开发需要 root 权限的应用程序时，最常见的方法是在 su shell 中运行一些命令。例如，有一个应用程序使用 pm enable/disable 命令来启用/禁用组件

-这种方法有很大的缺点：

1.极其缓慢（创建多个进程）

2.需要处理文本（超级不可靠）

3.可能性仅限于可用命令

4.即使 ADB 具有足够的权限，应用程序也需要根权限才能运行

Shizuku uses a completely different way. See detailed description below.

**Shizuku 使用了一种完全不同的方式。请参阅下面的详细描述。**
  <p align="center" style="font-size: 1.0em;">


# 作为第三方复刻，我们十分尊重原作.

<https://shizuku.rikka.app/>
<https://github.com/RikkaApps/Shizuku>
# Shizuku 的工作原理

首先要说明安卓应用如何调用系统API。
例如应用想要获取已安装应用列表，常规写法是调用  `PackageManager#getInstalledPackages()`
这本质是应用进程与系统服务进程之间的进程间通信（IPC），只是安卓框架帮我们封装好了底层逻辑。
 
安卓系统采用 Binder 机制实现这类进程间通信。
Binder 能让服务端获取客户端的用户标识（UID）和进程标识（PID），系统服务借此校验应用是否拥有对应操作权限。
 
通常安卓系统中，开发者可用的各类管理器类（如应用包管理器 PackageManager），
都对应系统服务进程里的一个系统服务（如应用包管理服务 PackageManagerService）。
可以简单理解：只要应用持有系统服务的 Binder 通信句柄，就能和该服务建立通信。应用在启动时会自动获取各类系统服务的 Binder 句柄。
 
Shizuku 的逻辑：先引导用户通过 ROOT 或 ADB 权限运行一个独立进程——Shizuku 服务端。
当第三方应用启动时，Shizuku 服务端的 Binder 句柄会同步发送给该应用。
 
Shizuku 最核心的作用是充当中间转发代理：接收应用的请求、转发给安卓系统服务、再将执行结果回传给应用。
具体可查看源码类：
 `rikka.shizuku.server.ShizukuService`  中的  `transactRemote`  方法
以及  `moe.shizuku.api.ShizukuBinderWrapper`  封装类。
 
最终实现效果：应用可以调用更高权限的系统API，且对开发者而言，调用方式几乎和直接使用原生系统API毫无区别。
# 开发者指南

## API & sample

https://github.com/RikkaApps/Shizuku-API

## 接口文档与示例代码

> 从 v11 旧版本迁移
 已有旧版适配的应用可照常使用，无需改动。
迁移指南：<https://github.com/RikkaApps/Shizuku-API#migration-guide-for-existing-applications-use-shizuku-pre-v11>
## 注意事项

 1.ADB 权限存在限制🌚

   ADB 权限本身有限，且不同安卓系统版本权限范围各不相同。
可在此查看系统赋予 ADB 的权限清单：
<https://github.com/aosp-mirror/platform_frameworks_base/blob/master/packages/Shell/AndroidManifest.xml>
   调用接口前，可通过两种方式校验权限：
- ShizukuService#getUid 判断 Shizuku 是否由 ADB 模式运行
- ShizukuService#checkPermission 校验服务端是否拥有足够操作权限
  
   
 2.安卓 9 隐藏 API 调用限制

   从安卓 9 开始，普通应用被系统限制调用隐藏非公开 API。
如需绕过限制，可使用开源方案：<https://github.com/LSPosed/AndroidHiddenApiBypass>


 3.安卓 8.0 与 ADB 适配问题

   目前 `Shizuku` 服务端获取应用进程的方式，是组合使用
`IActivityManager#registerProcessObserver` 和 `IActivityManager#registerUidObserver`（安卓 8.0 及以上），
保证应用启动时能正常接收 `Binder` 句柄。
但在安卓 8.0（API26）上，ADB 没有权限使用 `registerUidObserver`。
如果你的应用可能非活动页面（Activity）拉起进程，建议启动一个透明页面来触发 `Binder` 句柄推送.


 4.直接调用 transactRemote 接口须知

   - 不同安卓版本的系统接口定义存在差异，务必仔细适配版本；
且 `android.app.IActivityManager` 仅在 API26 及以上才有 AIDL 接口定义，`android.app.IActivityManager$Stub` 类也仅存在于 `API26` 版本。
   - `SystemServiceHelper.getTransactionCode` 可能无法获取正确的事务码，
例如 `android.content.pm.IPackageManager$Stub.TRANSACTION_getInstalledPackages` 在 `API25` 中不存在，
取而代之的是 `TRANSACTION_getInstalledPackages_47`。
官方已处理该类兼容问题，但不排除还有其他兼容场景；
推荐优先使用 `ShizukuBinderWrapper` 封装方式，可规避此类版本适配问题。


# 编译开发 Shizuku 本体

## 编译构建
 1.克隆源码（包含子模块）：git clone --recurse-submodules
 
 2.执行 Gradle 编译任务：
调试版：:manager:assembleDebug
正式版：:manager:assembleRelease
执行 :manager:assembleDebug 会生成可调试的服务端程序。
可对 shizuku_server 进程附加调试器进行源码调试。
注意：在安卓工作室（Android Studio）的「运行 / 调试配置」中，需勾选 始终通过包管理器安装，确保服务端使用最新编译代码。


## 开源许可证

本项目所有代码文件均遵循 Apache 2.0 开源协议。
依据 Apache 2.0 协议第 6 条规定，特别声明：

 1.禁止擅自使用 manager/src/main/res/mipmap*/ic_launcher*.png 图标资源(这个不用管，官方仓库才有😅)，仅允许用于展示 Shizuku 官方本体；
 
 2.禁止 将 Shizuku 用作应用名称、使用 `moe.shizuku.privileged.api` 作为应用包名，以及声明 moe.shizuku.manager.permission.* 系列权限。
