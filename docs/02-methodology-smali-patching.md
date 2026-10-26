# 02 · Android APK 逆向方法：apktool / jadx / smali 补丁工作流

> 工具链、工作循环、补丁的类型与适用场景、日志系统设计、以及十条按代价排序的经验教训。
> 例子来自本项目，方法适用于任何「反编译 → smali 修改 → 重签 → 装机验证」的 Android 逆向项目。

## 1. 工具链

| 工具 | 用途 | 要点 |
|---|---|---|
| **jadx** | 反编译成 Java 只读树 | 只用来**理解**。jadx 对协程/混淆代码偶有渲染错误，结论必须回 smali 核对 |
| **apktool** | 反编译成 smali 可改树 + 回编 | 修改的唯一入口 |
| **apksigner / zipalign** | 对齐 + 签名 | 自建测试密钥；v1/v2/v3 全签 |
| **keytool / apksigner lineage** | 证书取证 | 提取官方 APK 证书摘要；**必须看 lineage 完整证书链**（docs/03 §5，本项目代价最高的错误） |
| **adb** | 装机 / logcat / dumpsys / screencap | 双端日志对照是排障核心手段 |
| 自建索引脚本 | 为 jadx 树建 class/method/caller/callee 索引 | 见 §2 |

构建脚本化（PowerShell）：`apktool b → zipalign → apksigner → adb install` 一条命令出包，产物按 `App-版本号.apk` 归档。每一轮可安装产物都保留，出问题可逐版本回退对照。脚本核心（路径与密码替换为你自己的）：

```powershell
java -jar apktool.jar b <反编译目录> -o <app>-unsigned.apk
zipalign -f -p 4 <app>-unsigned.apk <app>-aligned.apk
apksigner sign --ks <你的.jks> --ks-pass pass:<密码> --out <app>-signed.apk <app>-aligned.apk
apksigner verify --verbose <app>-signed.apk
adb install -r <app>-signed.apk
```

## 2. 大代码库必须先建索引

Watch App / 国行 Wear OS / Health 都是百万行级、全混淆（三字母类名）的代码库，逐文件搜索效率极低。做法：

1. jadx 只跑一次，产出只读 Java 树；
2. 写一个小脚本（百行级）扫描整棵树，为每个类/方法建索引：标签（实现接口、父类、方法名字符串）、调用关系（callers / callees）；
3. 查询走索引：按 tag / 类名 / 调用者查询，秒级返回。

**定位锚点的优先级**（从稳定到不稳定）：

1. **日志字符串**——混淆不改字符串。例如搜 `"Unsupported step"` 可直接定位配对引擎的分发器；
2. **manifest 组件名 / action / 权限字符串**；
3. **协议常量**（消息路径、事务码、错误码）；
4. 反编译控制流（协程状态机在 jadx 里经常渲染错误——本项目至少两次条件极性被 jadx 误导，最终以 smali 的条件跳转指令为准）。

## 3. 核心工作循环

```
① Java 树定位     在 jadx 树里读懂原始语义——"理解"的唯一入口
② smali 锚点      在 apktool 树里找到对应方法，确认寄存器/标签现场
③ 最小改动        只动必须动的指令：入口返回、跳转改向、常量替换
④ 回读校对        改完把整个方法体读一遍，确认寄存器与标签完好
⑤ 登记            补丁语义与改动写入登记表（Java 语义 → smali 改动 的映射）
⑥ 构建装机        一次只验证一个改动
⑦ 双端日志对照     App 出站事务 vs 服务入站事务对账（见 §5）
```

**一次只改一个变量。** 结论无效就回滚，不留半成品。被新修改取代的旧修改必须删除，补丁集保持最小、尽量接近原始包。

**装机后必须用日志证明补丁生效。** 本项目真实事故：一个迁移互斥门补丁两度实现损坏（寄存器顺序错误 / 丢失 move-result），但构建成功、安装成功——直到发现该打的日志一条都没出现，才确认补丁未生效。"装上了"不等于"生效了"。

## 4. 补丁类型与适用场景

本项目约 25 个有效补丁，全部落入六类：

| 类型 | 典型场景 | 写法 | 风险 |
|---|---|---|---|
| **开关恒真/假** | 区域门、功能门（"是否支持换机"、"是否国行环境"） | 方法入口直接 return 常量，原方法体保留为不可达代码 | 低；先确认调用方只有预期的几处 |
| **白名单放行** | 调用方验签、准入检查 | 校验方法入口加"包名 == 我们的包 → 直接通过"分支 | 低；同一检查可能有多个入口方法 |
| **常量替换** | 硬编码的官方证书摘要 | 摘要常量替换为自己证书的摘要 | 中；摘要取值有重大注意点，见 docs/03 §5 |
| **AppKey 重写** | 对端设备校验 AppKey | 在 AppKey 构造点把（包名, 摘要）改写为官方值 | 中；必须全域一致（binder 声明/通道 token/消息信封应出自同一构造点） |
| **步骤禁用** | 引用国行缺失接口的向导步骤 | 把步骤的可用性方法返回不可用，走引擎自带的跳过路径 | 低；优先用应用自带的 available=false 机制 |
| **空值自举** | 空表/空列表导致流程死等 | 查询入口补"为空时构造默认值"分支 | 高；注意线程与初始化时序（§7 教训 5） |

两个通用原则：

- **让原逻辑成死代码而不是删除**。回滚只需恢复一条跳转，对照时原逻辑还在原地。
- **补丁尽量打在上游收敛点**。例如 AppKey 只在一个工厂方法构造，改一处全域生效；分散打补丁必然遗漏。

## 5. 日志系统：方法入口打点 + 双端对账

排障依赖日志，但插日志本身就能把 App 弄崩（教训 1）。本项目规范：

- 统一 Tag + 组件前缀（如 `[SVC/handshake]`、`[APP/zpu]`），一条 `adb logcat -s <TAG>` 拉出全链路；
- **只打方法入口**：入口处寄存器状态确定（参数寄存器 + 已知空寄存器），方法中段寄存器都在使用中，插入调用必然覆盖；
- 辅助类封装（一个 smali 日志类），每个打点一行调用；
- **事务码对照表**：binder 事务码（tx=0x14 等）与语义（putConfig 等）做成字典，日志直接打印；
- **双端对账**：App 侧打出站事务、服务侧打入站事务。有出无入 = binder 未送达（查服务进程/准入）；有入无响应 = 服务内部异常；两侧都有但 UI 无反应 = 回调链断。这套对账把"配对卡死"类问题从猜测变成二分查找。

两个日志辅助类的实现（均出自本项目，可直接复用）。注入到被修改 App 的 smali 辅助类——它是一个**独立新增的文件**（不改动官方任何类，只被调用），每个打点因此只需一行调用：

```smali
# pwcn/Log.smali（项目新增文件，节选 i 方法；另有 w/e/obj/kvi/tx 同构方法）
.method public static i(Ljava/lang/String;Ljava/lang/String;)V
    .locals 2
    const-string v0, "[PWCN]["
    invoke-virtual {v0, p0}, Ljava/lang/String;->concat(Ljava/lang/String;)Ljava/lang/String;
    move-result-object v0
    const-string v1, "] "
    invoke-virtual {v0, v1}, Ljava/lang/String;->concat(Ljava/lang/String;)Ljava/lang/String;
    move-result-object v0
    invoke-virtual {v0, p1}, Ljava/lang/String;->concat(Ljava/lang/String;)Ljava/lang/String;
    move-result-object v0
    const-string v1, "PWCN"
    invoke-static {v1, v0}, Landroid/util/Log;->i(Ljava/lang/String;Ljava/lang/String;)I
    return-void
.end method
```

Bridge App 侧的等价封装——统一 Tag、固定前缀、logcat 与应用内日志双写：

```java
// Lg.java（节选）
public static void i(String msg) {
    Log.i("PWCN", "[BR] " + msg);
    Sink s = sink;                    // UI 日志回调
    if (s != null) s.line(msg);
}
```

## 6. 验证策略

- **里程碑拆小**：配对拆成"服务能绑定 → 配置能写入 → 传输能建立 → 握手能完成 → 节点能注册"，每段有独立日志证据再前进；
- **A/B 对照**：怀疑日志打点影响行为时，出一个全静音版本对照（本项目实测证明入口打点无害）；
- **真机双端**：手机侧 + 手表侧（手表可开 adb over Wi-Fi）同时抓日志。很多"手机没发"其实是"手表拒收"。

## 7. 十条经验教训（按代价排序）

1. **日志不能插在方法中段。** 中段寄存器在使用中，插入调用会覆盖它们——轻则数据错乱，重则字节码校验失败（VerifyError），服务进程每秒崩溃重启，表现为"卡 logo + 无任何流量"。本项目 v2 系列因此整体重做。
2. **smali 宽寄存器**：`J`/`D`（long/double）类型参数各占**两个**寄存器。按顺序数参数时漏算会导致 invoke 调用错位 → 整个类 VerifyError。
3. **改动必须整体回读 + 日志证明生效**（见 §3）。
4. **不要 force-stop 手表上的 GMS 进程。** 会破坏手表侧 wearable 会话状态，此后每次握手都报 `Connect message malformed`，重启、重置蓝牙均无法恢复，只能重新配对。手表侧只做只读操作（看日志、改网络设置）。
5. **服务 App 的数据库存在延迟初始化**：在 binder 线程同步访问其数据库的代码会在初始化竞态下抛 ClassCastException，进而引发客户端 SDK 重试风暴（实测一轮 2000+ 次重试）。空值自举类补丁必须放进官方的就绪执行器线程。
6. **`pm clear` 寄宿 GMS 的服务 App 可能引发崩溃风暴**：清数据删光其落盘模块后，服务重启需要重新落盘，但 App 处于后台时 Android 14 拒绝其启动服务 → 崩溃循环。解法：clear 前后 `adb shell cmd deviceidle tempwhitelist -d 300000 <包名>` 给临时白名单。
7. **v3 签名轮换的摘要取值**：`apksigner verify --print-certs` 只显示当前签名者，而运行时旧 API 返回的是 lineage 祖先证书。取错值 = 双向校验失败且极难定位。完整机制见 docs/03 §5。
8. **GMS 有按包名键控的负缓存**：测试签名包反复请求失败会污染缓存，之后装回官方包也继承失败（表现为官方包也报 UNREGISTERED_ON_API_CONSOLE），**重启即愈**。下"服务器拒绝"结论前先重启排除缓存因素。
9. **增量安装会破坏 run-as**：`adb install` 增量模式可能把数据目录权限建成 0755，`run-as` 直接拒绝。需要调试数据目录时先 uninstall 再 `--no-incremental` 安装。
10. **Android 系统级 HTTP 代理设置全局生效**：设备若残留一个不可达的系统代理（`settings global http_proxy`），所有出网请求都会失败（统一表现为连接异常/503），症状容易误导到应用层鉴权。排障第一步：`adb shell settings get global http_proxy` 确认为空。

## 8. 何时应当放弃当前路线

本项目两次关键转向，都是证据表明此路不通时果断停止：

- Health App 修改路线：走到最后确认**云端 token 签发绑定官方签名**（GMS 拒绝为重签包签发专有 scope），服务器级判决，客户端无解 → 转向 Bridge 方案，官方 Health 保持原版。
- "按依赖闭包拷贝官方 SDK 类"方案：三次运行时缺类（六万级类的依赖图无法收敛）→ 放弃拷贝，改纯手写 Binder 协议（25KB、零依赖）。

判断标准：**剩余障碍位于不可修改的进程（GMS / 服务器）且没有 AppKey 重写空间时，换路线。**

下一篇 [03 - Wear OS 签名校验与 AppKey 协议](03-signature-trust-and-appkey.md)。
