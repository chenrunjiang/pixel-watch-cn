# 01 · Pixel Watch 国行为什么配对失败：GMS 平台限制与国行 Wear OS 架构

> 本文解释 Pixel Watch 在国行手机上无法使用的精确技术原因。
> 结论来自实测日志与反编译行为分析；混淆类名随版本变化，本文一律用「行为 + 可观测特征」描述，便于在任何版本上重新定位。

## 1. 国行手机的 Google 服务形态

国行三星手机出厂带 GMS（Google 移动服务）框架，但是裁剪版：GMS 内部有一层基于系统 feature 声明的"国行环境可用性"检查（系统分区中 `services.cn.google.xml` 一类配置），大量功能对国行设备直接返回不可用。Wear 服务就在被整体关闭的名单里。

可观测证据——任何 App 通过 GMS 请求 Wear 服务时，GMS 进程输出：

```
WearableService(GMS): onGetService: Wear is not available on this device.
Calling package: <调用方包名>
```

这条日志来自 GMS 进程，不可修改。这是第一层障碍：**GMS 平台级功能开关**。

## 2. 关键架构事实：手机上有两个 WearableService

国行手机上实际存在两套 Wear 服务，这是理解整个项目的前提：

| | GMS 内的 WearableService | 国行 Wear OS App 内的 WearableService |
|---|---|---|
| 宿主 | `com.google.android.gms`（系统级，不可修改） | `com.google.android.wearable.app.cn`（普通 App，可反编译修改） |
| 服务对象 | 所有第三方 App 的公共代理（broker） | 国行 Wear OS 手机端 + Pixel Watch App 的专用直连通道 |
| 国行可用性 | 被平台功能开关关闭 | 正常工作 |
| 对调用方的要求 | 逐证书校验（Google 官方名册） | 签名校验，只认 Google 官方证书 |

右列是本项目的主要修改对象。它是一个功能完整的穿戴服务：配对管理、RFCOMM 传输、节点路由、消息通道、配置数据库都在，只是通过签名校验把调用方限制在官方生态内。

**绑定方式**（manifest 层公开事实，任何 App 都可以尝试）：

```
Intent { action = "com.google.android.gms.wearable.BIND"
         component = com.google.android.wearable.app.cn 内的
         com.google.android.gms.wearable.service.WearableService }
```

绑定调用的实际代码（出自本项目 Bridge 源码）：

```java
Intent intent = new Intent("com.google.android.gms.wearable.BIND")
        .setPackage("com.google.android.wearable.app.cn");
boolean ok = ctx.bindService(intent, connection, Context.BIND_AUTO_CREATE);
```

## 3. Pixel Watch App 的中国模式

Pixel Watch App（`com.google.android.apps.wear.companion`，手机上显示为 Watch，下文称 Watch App）原生内置双模逻辑：

```
启动配对流程时：
  检查本机是否安装国行 Wear OS App，且其签名 == 内置的 Google 官方证书摘要
    ├─ 通过 → 中国模式：所有 Wear 调用直连国行 Wear OS App
    └─ 不通过 → GMS 通道 → 平台功能开关 → UNSUPPORTED_PLATFORM
```

正版环境下三方签名全部匹配，中国模式自动生效。修改后的 Watch App 与国行 Wear OS App 都必须重签名，这个检测（验国行 Wear OS App 的签名）必然失败，Watch App 被送进 GMS 死路。

处理方式：把检测方法改为恒定返回 true（方法入口直接返回，原逻辑保留为不可达代码）。Watch App 随即把全部流量切到国行 Wear OS App。通道对了，剩下的是第二层障碍。

## 4. 三层障碍总览

| 层 | 位置 | 检查内容 | 失败表现 | 处理方式 |
|---|---|---|---|---|
| ① GMS 平台功能开关 | GMS 进程内（不可改） | 国行环境禁用 Wear | `Wear is not available on this device` | 改走国行 Wear OS App（Watch App 强制中国模式） |
| ② 手机侧调用方签名校验 | 国行 Wear OS App 内 | 调用方包名 + 证书验签；自身 user 构建自检内置摘要 | `API_UNAVAILABLE` / 卡 logo / 弹"不兼容" | 服务端放行调用方包名；自检摘要替换为自己证书 |
| ③ 手表侧 AppKey 校验 | 手表 WearableService（不可改，也无需改） | 消息信封的 AppKey ==（官方包名, 官方证书摘要） | 手表日志 `Mismatched certificate: AppKey[...]`，消息静默丢弃 | 手机侧出站时声明官方 AppKey（见 docs/03） |

一个重要推论：三层障碍里有两层位于可以修改的普通 App（国行 Wear OS App）内。这是不 root 也能完成的根本原因。

## 5. 官方 Health App（第三方）为什么也连不上

官方 Health App（`com.fitbit.FitbitMobile`）管理 Wear 设备走 GMS 的公共 broker 通道：

```
WearableService(GMS): onGetService: Wear is not available on this device.
Calling package: com.fitbit.FitbitMobile
```

官方原版、官方签名，同样被拒。这是结构性限制：走 GMS broker 就过不了第 ① 层。

Health App 与 Watch App 一样内置 CN 双模检测（认国行 Wear OS App），但其检测要求国行 Wear OS App 为官方签名，与重签名版互斥，于是回落 GMS 通道被拒。官方 Health 不能改（改动会失去云端 token 能力，见 docs/05 §5），所以第三方链路的解法不是修改它，而是自己写一个 Bridge App 顶替它与手表通信——见 docs/05。

## 6. 设计意图（有助于预判边界）

- **签名隔离**：国行 Wear OS App（`.cn` 包名）是独立发布的包，Google 用证书摘要把国行与国际两个信任域分开，防止跨生态随意互通与伪造客户端。
- **平台功能开关是合规分层**：功能是否在某市场开放由系统 feature 与服务端开关控制，代码不从国行包里删除——所以国行包的功能是完整的，只是被禁用。
- **手表只验 AppKey**而不验证"对端是否真的是官方 App 在运行"，因为它假设手机侧 WearableService 已完成准入。这个假设正是 Bridge 方案可以利用的空间。

## 7. 环境自检（动手前确认）

在国行手机上验证本文前提：

```bash
# 国行 Wear OS App 存在且启用
adb shell pm list packages | grep wearable.app.cn

# 平台功能开关确实存在（任选一个走 GMS 通道的 App 触发，观察日志）
adb logcat -s WearableService | grep "not available"

# 国行 Wear OS App 的服务可绑定（manifest 层检查）
adb shell dumpsys package com.google.android.wearable.app.cn | grep -A2 "wearable.BIND"
```

下一篇 [02 - Android APK 逆向方法](02-methodology-smali-patching.md)讲怎么改：工具链、工作循环、以及怎样避免把 App 改崩溃。
