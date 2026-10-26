# 03 · Wear OS 签名校验与 AppKey 协议（Mismatched certificate / v3 lineage）

> 整个项目最核心的机制文档。理解这一篇，"为什么改了还是不通"的大部分问题都有答案。

## 1. 重签名破坏了什么

Android 要求 APK 签名，修改官方 APK 后必须重签。重签名的直接影响：

```
官方包:  (包名, 官方证书) ──► 各处校验通过
修改包:  (包名, 你的证书) ──► 所有"包名+证书"校验失败
```

不是单点失败，而是整个信任体系里的校验点同时失败。Wear 生态的校验点尤其密集，因为 Google 把"包名 + 证书摘要"用作跨设备、跨进程的**运行时身份**，不只是安装校验（见 §2）。

## 2. AppKey：穿戴生态的运行时身份

**AppKey = （包名, 签名证书 SHA-1 摘要）**。它出现在：

1. **binder 调用声明**——App 调用 WearableService 时自报身份；
2. **消息通道 token / 能力（capability）上报**——每个节点通告的能力都挂在 AppKey 下；
3. **消息信封**——手机↔手表的每条消息都携带发送方 AppKey，**接收端校验不过即静默丢弃**（仅留一行日志）。

手表上的 WearableService 为已安装 App 维护本地认知，如 `AppKey[com.fitbit.FitbitMobile, 29a4514c…]`。收到消息时比对信封声明与本地认知，不一致则：

```
W/WearableService: Mismatched certificate: AppKey[<声明包名>, <声明摘要>]
W/WearableService: Failed to deliver message … action=<消息路径>, dataSize=N
```

**注意方向性**：错误可能出现在手机侧或手表侧日志，取决于发送方是谁、接收端本地认知是什么。同一根因在两端表现不同，这是排障时最容易误判的地方。

## 3. 三类处理方式

| 方式 | 修改对象 | 效果 | 适用 |
|---|---|---|---|
| **白名单放行** | 校验的**执行方**（国行 Wear OS App） | "包名 == X → 直接通过" | 校验双方都在可控 App 内 |
| **摘要替换** | 校验执行方 | 把硬编码的官方摘要换成自己的摘要 | 自检类校验（如 user 构建自检） |
| **AppKey 重写** | AppKey 的**构造点** | 出站身份改写为（官方包名, 官方摘要） | 接收端不可修改（手表 / 服务器） |

AppKey 重写的可行性来自一个协议事实：AppKey 的"逻辑包名"与"物理包名"在官方实现里**本就允许分离**（Google 为 `.cn` 版自己就这样用）。把 Bridge App 的出站身份映射为官方 Fitbit 身份在协议层是常规操作，手表无需任何修改。

## 4. 校验点清单（本项目实测）

复现时按此表逐点排查：

| # | 位置 | 检查 | 通过条件 | 失败特征 | 处理 |
|---|---|---|---|---|---|
| 1 | 国行 Wear OS 入站 | 调用方包名验签 | 包名在白名单且证书匹配 | `API_UNAVAILABLE`，客户端卡 logo | 白名单放行 |
| 2 | 国行 Wear OS 入站 | 第二层调用方校验 | 同上，另一方法 | `SecurityException` | 同上（两个入口都要放） |
| 3 | 国行 Wear OS 自身 | user 构建自检：内置摘要 == 自身签名摘要 | 摘要一致 | 弹"不兼容的 companion 应用" | 摘要替换 |
| 4 | 手机→手表 | 消息信封 AppKey == 手表本地认知 | 官方包名+官方摘要 | 手表 `Mismatched certificate` | AppKey 重写 |
| 5 | 手表→手机 | capability 上报的 AppKey == 手机本地认知 | 同上 | 手表能力表恒为空 | 手机侧本地认知改为官方值 |
| 6 | 手表侧应用 | 监听服务准入：调用方 uid 拥有 GMS 且过 Google 签名校验，或 CN 开关放行 CN wear 的 uid | uid + 签名 | `WearableLS: Caller is not GooglePlayServices; caller UID: N` | 不可改（手表）；影响有限 |
| 7 | manifest 信任声明 | `wear-trusted-peer-packages` 风格的硬编码摘要 | 声明匹配 | 静默不生效 | 确认即可，多数 App 未声明 |
| 8 | GMS broker（第三方 App 走 GMS 时） | 逐证书官方名册 | Google 官方证书 | `GoogleCertificatesRslt: not allowed` | **不可改**（GMS 进程），换路线 |
| 9 | GMS 账号 provider | 调用方签名 | Google 官方证书 | 枚举账号返回空/异常 | 改用 AccountManager 直查（需 GET_ACCOUNTS 权限） |
| 10 | GMS token 签发 | 客户端注册绑定官方证书 | 官方签名 + 服务器注册 | `UNREGISTERED_ON_API_CONSOLE`（注意负缓存假象，重启判别） | **不可改**，此路不通 |

## 5. v3 签名轮换（lineage）的摘要取值

**现象**：手机侧消息"成功发出"（发送 API 返回成功），但手表始终收不到，手表侧只有 `Mismatched certificate`。更迷惑的是：填入从 `apksigner verify --print-certs` 抄来的"官方摘要"仍然不匹配。

**根因**：现代 Google/Fitbit APK 使用 **v3 签名轮换（lineage）**——同一 APK 内含证书链：祖先证书（历史签名者）→ … → 当前证书。两个事实组合成陷阱：

1. `apksigner verify --print-certs` 只显示**当前签名者**（Signer #2，"Google Inc." 证书）；
2. Wear 生态计算 AppKey 用的是旧 API `getPackageInfo(pkg, GET_SIGNATURES)`，该 API 在 v3 轮换包上**为向后兼容返回 lineage 祖先证书**（Signer #1，"Fitbit, Inc." 证书）。

因此**运行时生效的摘要 = 祖先证书摘要**，与 apksigner 顶层显示值不同。以 Fitbit 为例（摘要为公开 APK 可提取的公开事实）：

```
lineage:  Signer #1（祖先）  SHA-1 = 29a4514c3b90b90cb6badc79614262195c6a5747   ← 运行时生效值
          Signer #2（当前）  SHA-1 = 80637b399756fe72817966fa4da2977a8e3efad6   ← apksigner 顶层显示值
```

**取证方法**（四源交叉，得二即可定案）：

```bash
# 源1: 完整证书链
apksigner lineage --print-certs --in official.apk

# 源2: 系统对该包的签名记录（past signatures 列出轮换历史）
adb shell dumpsys package <包名> | grep -A3 signatures

# 源3: 接收端日志里对端自报的 AppKey（最直接的"对端期望值"）
adb logcat -s WearableService | grep -i "AppKey\|Mismatched"

# 源4: 反编译树里内置的信任映射（官方硬编码的官方摘要）
```

以 Fitbit 官方 APK 为例，`apksigner lineage --print-certs` 的输出形态（摘要为公开可提取的事实）：

```
Signer #1 certificate DN: CN=Fitbit, Inc., ...
  SHA-1 digest: 29a4514c3b90b90cb6badc79614262195c6a5747   ← lineage 祖先，运行时生效值
Signer #2 certificate DN: CN=Google Inc., ...
  SHA-1 digest: 80637b399756fe72817966fa4da2977a8e3efad6   ← 当前签名者，apksigner verify 只显示这个
```

**原则：AppKey 重写使用的摘要，以"接收端日志里对端自报的值"为准**——那是接收端本地认知的直接暴露，比任何工具输出都权威。

## 6. 官方摘要参考表

以下均为公开 APK 中任何人可提取的公开事实（`keytool -printcert -jarfile official.apk`），列出供理解（复现时请从自己持有的官方包重新提取）：

| App | 包名 | 证书摘要（SHA-1） |
|---|---|---|
| Pixel Watch App | `com.google.android.apps.wear.companion` | `78f631ae…` |
| 国行 Wear OS | `com.google.android.wearable.app.cn` | `a197f921…` |
| Fitbit/Health（lineage 祖先，**运行时生效**） | `com.fitbit.FitbitMobile` | `29a4514c…` |
| Fitbit/Health（lineage 当前） | `com.fitbit.FitbitMobile` | `80637b39…` |

自己的重签名证书摘要：

```bash
keytool -list -v -keystore your.jks -alias your_alias
```

## 7. 多修改包共存的一致性

多个修改包共存时（本项目：修改版 Watch App + 修改版国行 Wear OS + Bridge）注意两条：

1. **signature 级权限要求同证书**：用同一 keystore 签所有自己的包，避免互相访问被系统权限拦截；
2. **AppKey 重写要全域一致**：binder 声明、通道 token、消息信封若出自不同构造点，必须确认是否共用一个工厂。本项目 AppKey 只有一个工厂方法，在工厂处改写即全域生效；改写后用日志验证每条出站消息携带的身份。

下一篇 [04 - Wear OS 配对流程与跨手机迁移协议](04-pairing-and-migration.md)。
