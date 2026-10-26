# 04 · Wear OS 配对流程与跨手机迁移协议（phone switching）

> 配对 = 手表与国行手机建立 Wear 通道；迁移（phone switching）= 手表带着账号从一台手机转移到另一台。
> 本文按流程顺序描述协议，每步给出可观测特征。前提：docs/01 的三层障碍已按 docs/03 的方式处理。

## 0. 前提：手表初始化需要完整环境

Pixel Watch 首次开机设置需要完整 Google 服务环境，国行手机连配对都无法完成，因此手表先在**另一台刷港版固件的三星 S22** 上完成初始设置与账号登录。国行手机上是"迁移接入"场景。

路线选择：

- **路线 A（本项目采用）**：S22 初始化 + 账号登录 → 迁移到国行手机（§4，账号保留）；
- **路线 B**：S22 初始化后重置手表 → 国行手机全新配对。流程更简单，但手表无账号，健康功能受限。

## 1. 配对步骤引擎（Watch App 侧）

Watch App 的设置向导是步骤流水线引擎：发现 → CDM 关联 → 蓝牙配对 → 连接建立 → 账号 → 条款 → … → 完成。两个对补丁友好的机制：

1. **步骤可用性**：每个步骤有 `isAvailable()` 判定，引擎对不可用步骤自动跳过（官方本来就用它跳过区域不适用的步骤）。让某个步骤消失的正确方式是把它的可用性改为 false，而不是硬跳；
2. **步骤分发器的报错**：插入了没有处理器的步骤会在分发器抛 `Unsupported step`——看到这句说明有步骤被塞进引擎但无人处理。

**国行环境实测需要处理的步骤**：

| 步骤 | 国行的问题 | 处理 |
|---|---|---|
| 中国持久服务步骤 | 引擎无条件插入，但分发器无处理器 → 崩溃 | 不插入（去掉插入指令） |
| 条款确认（ToS） | 请求新版服务端事务（tx=96 一族），国行 Wear OS 版本没有该接口 → `API_NOT_SUPPORTED` → fatal 报错页 | 步骤标记不可用，走官方跳过路径 |
| 账号选择/登录 | 见 §3 | 见 §3 |

## 2. 连接建立链路（服务侧）

配对进入连接阶段后，链路按以下顺序建立，卡住时按序检查：

```
① 蓝牙 bond        系统 BOND_STATE_CHANGED → BONDED
② 配置写入         putConfig（binder 事务 tx=0x14）→ 配置表
③ RFCOMM 客户端    服务起客户端线程拨号手表；手表也可能反向连回（服务端模式）
④ 握手             hello 消息交换：能力协商、迁移标志核对
⑤ 节点注册/路由     本机 nodeId 初始化、路由表建立
⑥ 数据通道         App ↔ 手表的消息/数据项通道（通道状态 1=开启 2=互通 4=关闭）
```

**配置表**是服务侧核心状态，每条配置含 `Name`（手表蓝牙地址）、`Type`（1=普通连接 / 5=迁移）、`Migrating` 标志等。相关 binder 事务：

| 事务 | 语义 |
|---|---|
| tx=0x14 (20) | putConfig（写配置） |
| tx=0x15 (21) | removeConfig（删配置） |
| tx=0x16 (22) | getConfigs（列配置） |
| tx=0x0f (15) | getConnectedNodes |

**国行特有的问题：配置表为空时无人创建配置**——官方流程里配置由国行 Wear OS 手机端 App 写入，本机没有安装。处理：空值自举补丁（蓝牙 bond 成功事件或首次查询时，表空则按标准字段建配置并触发连接）。注意 docs/02 教训 5：自举代码必须放进官方的就绪执行器线程，在 binder 线程同步访问数据库会触发初始化竞态。

**握手协议差异的实测结论**：国行 hello 消息比国际版少三个字段，均为无行为影响的兼容字段；版本号差异走标准 min 协商。**协议本身没有实质差异**——国行与国际的区别全部在准入与开关层，不在协议层。这个结论可以省掉后来者大量的协议比对工作。

**连接状态码字典**（`ConnectionStateEvent`）：

| 状态 | 含义 |
|---|---|
| 0 | Connected 已连接 |
| 1 | handshake in progress 握手中 |
| 2 | handshake complete 握手完成 |
| -3 | No bluetooth connection 无蓝牙 |
| -4 | Did not receive connect msg 未收到对端连接消息 |
| -6 | Migration status mismatch 迁移标志不一致 |
| -7 | Connect message malformed 连接消息畸形 |

## 3. 账号步骤（配对中的账号处理）

三个事实与处理：

1. **账号枚举被 GMS 签名校验拒绝**（docs/03 表 #9）——Watch App 枚举 Google 账号走 GMS provider，重签名包被拒。处理：改用 `AccountManager.get(context).getAccountsByType("com.google")` 直查（manifest 补 `GET_ACCOUNTS` 权限并 `pm grant`）。
2. **账号直传被 GMS 拒绝**：向导把选中账号经 GMS SmartDevice 通道直传手表，同样被签名校验拒绝（`callingPackage … is not authorized`，无限重试）。处理：改走 Watch App 自己的官方跳过路径（非监管设备可跳过，且有正式的上报与完成回调）——账号留给手表自行登录或经迁移带入。
3. **手表侧辅助登录会崩 App**：手表点"登录"拉起手机侧 GMS 认证流程，重签名包收到 DEVELOPER_ERROR 后异常无人捕获 → App 进程崩溃。处理：在失败处理器入口拦截该错误码（打日志后不投递），保持 App 存活。

## 4. 跨手机迁移（phone switching）

迁移是官方"换手机"功能：手表从旧手机（S22）改投新手机（国行 Fold8），账号保留。链路与需要逐层处理的问题，按流程顺序：

```
① 设备发现门（Watch App）
   扫描能看到已配对他人的手表（RSSI 正常），但点击设备报
   ERROR_PHONE_SWITCHING_UNSUPPORTED —— Watch App 先查 GMS"换机功能是否开放"，
   国行侧答不支持 → 拦在发现阶段（无任何蓝牙动作）。
   ▸ 处理：Watch App 侧该 feature 查询改为恒"支持"。

② 迁移配置准入（服务端）
   Watch App 创建迁移配置：putConfig[Type=5, Migrating=true]。
   国行 Wear OS 服务端对迁移配置做支持检查（读远端开关，默认关闭）
   → 返回 4014 FEATURE_DISABLED，配置创建失败。
   ▸ 处理：服务端该检查改为恒真。检查收敛在一个方法，多处调用全覆盖。

③ 握手迁移标志
   手表 hello 消息携带迁移标志（手表侧已同意迁移）。手机握手层核对
   "本地配置的迁移标志 == 消息的迁移标志"，不等 → 状态 -6 断开连接。
   实测问题：握手线程可能携带换配置前的旧配置对象（Migrating=false）
   → a. 以非迁移身份对迁移态手表通信，被手表断开；
     b. 握手后续所有迁移分支被绕过。
   ▸ 处理：握手入口处，配置表中存在迁移配置时把本地配置的迁移位置 true。
     ⚠ 约束：该处理必须限定在"配置表中存在迁移配置"的窗口内。
     若无条件全局生效，日常 Type=1 连接也会被当成迁移握手，
     稳定复现 -7（见⑥）。

④ 账号匹配
   迁移握手最后，服务端发起账号匹配：枚举本机 Google 账号，与手表当前
   账号比对（取得到 ID 比 ID，取不到自动降级为纯名字比对）。
   - 枚举同样被 GMS 签名校验拒绝 → AccountManager 回退（同 §3.1）；
   ▸ 兜底：枚举结果为空时（账号可见性限制），将已知账号直接注入匹配表；
   - 匹配失败的官方行为是拉起选账号 Activity——但该 Activity 未导出，
     服务寄宿进程无权启动 → SecurityException 未捕获 → 宿主进程 FATAL。
   ▸ 处理：匹配失败分支不走 UI，按"无条件继续完成迁移"的官方设计走完
     （上报账号不匹配结果码后仍走 onMigrationSucceeded）。

⑤ 迁移完成
   onMigrationSucceeded → 手表自行断开旧手机（手表侧执行）→ 连接新手机。
   账号保留，手表上已登录的服务不丢失。
```

迁移成功时的实测日志时间线（手机侧节选；`<...>` 为占位符，需自行替换）：

```
putConfig  ConnectionConfiguration[Name=<手表蓝牙地址>, Type=5, Migrating=true, ...]
conn.connect → RFCOMM 拨号 → handshake（迁移位核对通过，无 -6 / -7）
AMATCH      account matched
onMigrationSucceeded
ConnectionStateEvent: state: Connected（node=<nodeId>）
→ 手表侧随后自行断开旧手机
```

**⑥ 迁移相关代价最高的问题**：`Migrating=true` 只允许出现在真正迁移那一次（Type=5 配置天然携带），日常 Type=1 连接必须保持 false。曾因无条件全局置位，日常重连全部被手表按迁移协议校验、判 -7 `Connect message malformed`，且无法通过重启恢复（配置已污染），最终清空服务配置表重来。排障看到 -6/-7 时，先查配置表里 `Migrating` 标志的分布。

**另一个竞态**：迁移流程自身会建蓝牙 bond，而 bond 成功事件会触发空值自举（§2）抢跑建立普通连接线程，以普通身份对迁移态手表通信，被手表快速断开。两种解法：给自举加"存在迁移配置则让路"的判断（注意 docs/02 教训 3——这类条件补丁最容易实现损坏而不自知），或依赖 ③ 的窗口限定处理兜底。

## 5. 排障顺序（配对/迁移总表）

```
1. App 出站 tx vs 服务入站 tx        → 请求是否送达（准入/进程问题）
2. 最后一条步骤日志                  → 向导卡在哪个阶段
3. 配置表内容（getConfigs）           → Type/Migrating 分布是否正确
4. conn.* → rfcomm.* → handshake     → 连接进行到哪一段
5. ConnectionStateEvent 状态码        → 按字典定位（-6/-7 查迁移标志，-3/-4 查蓝牙）
6. 手表侧日志                        → Mismatched certificate（docs/03）/
                                        no live listeners（docs/05 §3）
```

下一篇 [05 - Pixel Watch 的 Fitbit 应用激活与同步协议](05-fitbit-activation-sync-bridge.md)。
