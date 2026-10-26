# 05 · Pixel Watch 的 Fitbit 应用激活与同步协议：消息路径、Binder 直连与 Bridge 实现

> 最后一环：手表端 Fitbit 应用如何激活、健康数据如何上云、以及一个 25KB 桥接 App（Bridge）的实现。
> 本文档包含穿戴消息层的完整路径表与字段语义——这部分是纯 wire 协议（功能事实），与反编译代码无关。

## 1. 手表端 Fitbit 应用的锁定态

配对/迁移完成后，手表上的 Fitbit 应用仍处于未登录锁定态：它不自行发起登录，而是等待手机侧 Fitbit 生态发送激活消息。手表上点"在手机上继续"实质是在等一条带账号凭据的消息——国行环境下这条消息无法到达（官方 Health 被 GMS broker 拒绝，见 §5），手表持续等待。这是"手表端 Fitbit 应用转圈"的原因之一（另一个是网络，见 §4.3）。

## 2. 消息路径注册表（wire 协议）

手表端 Fitbit 通过 Wear 能力系统通告服务，手机侧按路径发送消息。完整路径表（手表 APK 的 manifest 与能力注册中公开可见）：

**能力路径**（capability，用于发现）：

```
auth:1.1 / auth:1.2          登录协议版本（决定激活方式，见 §3）
fitbitmobile / nightlight    Fitbit 服务 / 夜间模式
sync/v1/request / sync/v1/request/schedule    同步触发
curr_fitbit_user/v1/data     当前用户数据
v1/query_series / v1.1/... / v1.2/query_series    数据查询
v1/latest_site_sync_times    最近同步时间
device_capabilities/v1/feature_enablement     设备能力
v1/structured_workout (+v2)  结构化锻炼
voice_cue / branding / onboarding / v1/sync_status_process
```

**消息路径**（message，用于通信）：

```
/auth/info / auth/request / pair/status          激活与配对状态
/sync/v1/request / sync/v1/request/schedule      同步触发（两种语义，见 §4）
/v1/query_series / voice_cue                     数据查询 / 语音提示
```

## 3. 激活协议：auth:1.2 与 auth:1.1

手机侧根据手表通告的 auth 能力版本选择激活方式：

| 版本 | 机制 | 载荷要点 |
|---|---|---|
| `auth:1.2`（新版，手表通告即首选） | Gaia 直传：手机把混淆后的谷歌账号标识（obfuscatedGaiaId）直接发往手表 `/auth/request`，手表凭它完成账号绑定确认 | protobuf：`1:"1.2"`（协议版本）、`2:账号标识`、`3:设备类型`、`4:认证方式=2`、`5:兼容设备列表` |
| `auth:1.1`（旧版回退） | WebView OAuth：手表拉起手机网页授权 → 取 code → 换 token → 经 `/auth/info` 下发 | 需要 WebView 与 OAuth 全链路，国行环境环节多，能用 1.2 就用 1.2 |

**应答**：手表经 `/pair/status` 返回状态，`1=SUCCESS`；`2=MISSING_GAIA`（账号标识缺失/无效——先核对手机侧是否真的取到账号标识）；`6=账号不一致`（手表已有账号与发来的不符，需先清除手表账号）；`9=配对进行中`。

**激活成功后的关键事实**：手表端 Fitbit 获得账号身份，此后不再依赖手机侧任何 Fitbit App——它自身具备完整的云同步能力。

激活载荷的实际构造代码（Bridge 的 `Pb.java`；字段序即 wire 序）：

```java
public static byte[] encodeAuthRequest(String gaiaId, String deviceType, int[] compat) {
    ByteArrayOutputStream o = new ByteArrayOutputStream();
    str(o, 1, "1.2");            // 协议版本
    str(o, 2, gaiaId);           // 混淆账号标识（obfuscatedGaiaId）
    str(o, 3, deviceType);       // 设备类型
    if (compat != null && compat.length > 0) {
        ByteArrayOutputStream p = new ByteArrayOutputStream();
        for (int v : compat) varint(p, v);
        tag(o, 5, 2);            // 兼容设备列表：packed int32
        varint(o, p.size());
        write(o, p.toByteArray());
    }
    return o.toByteArray();
}
```

## 4. 同步协议

### 4.1 触发（手机 → 手表）

| 路径 | 语义 |
|---|---|
| `/sync/v1/request/schedule` | 调度式：手表先查网络——有网立即回执（status=3）并**异步**开始同步；无网拒绝（错误 4013，回执 6/5） |
| `/sync/v1/request` | 直发式：网络检查后**等待同步完成**才返回最终状态 |

触发载荷（protobuf，两路径同构）：`1:触发源字符串`（"user"=手动 / "client"=前台 / "tracker"=后台）、`2:枚举(手机端恒为1)`、`3:uuid`（随机，回执回显用于对账）、`4:事件类型(可选)`。实际构造（Bridge 的 `Pb.java`）：

```java
public static byte[] encodeSyncRequest(String trigger, String uuid) {
    ByteArrayOutputStream o = new ByteArrayOutputStream();
    str(o, 1, trigger);          // 触发源："user" / "client" / "tracker"
    varintField(o, 2, 1);        // 枚举，手机端恒为 1
    str(o, 3, uuid);             // 随机 uuid，回执回显用于对账
    return o.toByteArray();
}
```

### 4.2 回执语义（容易走弯路的地方）

官方手机端的触发调用不是"发消息"，而是**读调用**（request-response 语义，一次 binder 往返）——手表 handler 的**返回值就是响应**，协议中不存在另一条独立的"响应消息"。本项目曾按"发出后等待一条 `/sync/response` 消息"实现，等待窗口永远为空——协议里没有这条消息。若 Bridge 走纯 fire-and-forget 的 sendMessage，同样拿不到回执；要么实现读调用，要么不等待、通过状态回调（§7）感知结果。

### 4.3 数据路径

**同步数据不经过手机**。实测数据流：

```
手表进程 ──HTTPS──► fitbit 云（上传心率/步数/睡眠等）
手表进程 ◄──HTTPS── fitbit 云（拉取个人资料/目标等）
```

手机侧 App 在同步中只有两个作用：**触发**（发一条消息）与**查看状态**。手机端 Health 上显示的"手表数据"全部来自云端，不是蓝牙搬运。

先决条件：手表自身需要具备访问相应云端的网络环境（Wi-Fi 或经手机的蓝牙网络隧道）。排障提示：手表同步失败统一表现为云接口 503/UNAVAILABLE 时，先查手表系统级网络设置（含残留代理，见 docs/02 教训 10），再怀疑鉴权。

## 5. 手机侧的限制与 Bridge 的由来

官方 Health App 无法承担激活任务，三重限制（docs/01 §5 的展开）：

1. 官方 Health 管理 Wear 设备走 **GMS broker** 通道，被平台功能开关拒绝（官方签名也无效）；
2. Health 内置的 CN 双模检测要求国行 Wear OS App 为**官方签名**，与重签名版互斥，检测失败后回落 GMS → 限制 1；
3. 修改 Health 本身不可行：云端 token 签发绑定官方签名（docs/03 表 #10，服务器级判决），改了它就只能离线使用。

**结论：官方 Health 一个字节都不能动（它是云端数据的入口），手表激活由一个新角色完成——一个自己的、可以被国行 Wear OS 接受身份的 App，即 Bridge。**

## 6. Bridge 实现（纯协议实现，零官方代码）

### 6.1 连接协议（纯 Java Binder 直连）

不拷贝任何官方 SDK 类，全部按 wire 协议手写（产物 25KB、零依赖）。连接序列：

```
① bindService(action=com.google.android.gms.wearable.BIND,
              pkg=com.google.android.wearable.app.cn)
      → 取得 IGmsServiceBroker（GMS 通用服务代理，descriptor 可校验）
② transact(46) getService + 手写 GetServiceRequest（SafeParcel）
      字段：versionCode=6、serviceId=14(wearable)、gmsVersion=8600000、
           packageName=自己、supportsConnectionInfo=true
③ 回调取得 status + IWearableService binder
④ IWearableService 事务：
      12 = sendMessage(callback, node, path, bytes)
      15 = getConnectedNodes(callback)
      43 = getConnectedCapabilities(callback, int)
⑤ 回调（IWearableCallbacks）按 code 解包：
      7  = SendMessageResponse（f2=status）
      10 = GetConnectedNodesResponse（f2=status, f3=List<Node>{f2=id, f3=name}）
      23 = GetAllCapabilitiesResponse（f2=status, f3=能力表）
```

**两个准入问题**（都表现为 status=8）：

1. Bridge 的 manifest **必须声明** `com.google.android.gms.version` meta-data 且 ≥ 10200000——国行 Wear OS 对调用包做此检查，不满足则把包名加入**进程内黑名单**（此后 status=8；修改 manifest 后必须 `am force-stop` 该服务清除黑名单才生效）；
2. Android 11+ 包可见性：manifest 需要 `<queries>` 声明目标包，否则绑定与读取 meta-data 均不可见。

Bridge 的 manifest 关键声明：

```xml
<queries>
    <package android:name="com.google.android.wearable.app.cn"/> <!-- 包可见性 -->
</queries>
<application ...>
    <!-- 缺这条会被国行 Wear OS 拉黑（status=8），版本值需 ≥ 10200000 -->
    <meta-data android:name="com.google.android.gms.version" android:value="12451000"/>
    <!-- 供 Watch App 绑定的状态服务：组件名必须与官方 Health 内的服务同名（见 §7） -->
    <service android:name="com.fitbit.cronus.impl.wear.WearService" android:exported="true"/>
</application>
```

连接序列中两个关键步骤的实际代码（Bridge 源码 `Bridge.java`）。getService 请求体（SafeParcel 手写）：

```java
d.writeInterfaceToken("com.google.android.gms.common.internal.IGmsServiceBroker");
d.writeStrongBinder(callbacks);
d.writeInt(1);                                   // GetServiceRequest 非空
int pos = Safe.begin(d);
Safe.writeInt(d, 1, 6);                          // versionCode
Safe.writeInt(d, 2, 14);                         // serviceId = Wearable
Safe.writeInt(d, 3, 8600000);                    // gmsVersion
Safe.writeString(d, 4, "com.pwcn.fitbitbridge"); // packageName
Safe.writeBool(d, 12, true);                     // supportsConnectionInfo
Safe.end(d, pos);
broker.transact(46, d, r, 0);                    // IGmsServiceBroker.getService
```

sendMessage（事务码 12）的参数序：

```java
d.writeInterfaceToken("com.google.android.gms.wearable.internal.IWearableService");
d.writeStrongBinder(callback);   // IWearableCallbacks：code 7 = SendMessageResponse
d.writeString(nodeId);           // 目标节点
d.writeString(path);             // 消息路径，如 /auth/request
d.writeByteArray(payload);       // protobuf 载荷
wear.transact(12, d, r, 0);
```

### 6.2 SafeParcel 编码要点

GMS 系 binder 的参数是 SafeParcel 编码，手写编解码的关键规则：

| 元素 | 格式 |
|---|---|
| 对象头 | 魔数 `20293 \| 0xFFFF0000` + 总长度占位（末尾回填） |
| int/bool 字段头 | `fieldId \| (4<<16)`，后跟 4 字节 |
| long 字段头 | `fieldId \| (8<<16)`，后跟 8 字节 |
| 容器字段头（String/List） | `fieldId \| 0xFFFF0000` + 长度占位 + 内容 + 回填 |
| typed list 的每一项 | 前置 4 字节长度前缀（含项头+大小） |

解析侧必须**顺序读取 + 边界检查**，禁止按计算位置跳转（`setDataPosition` 越界会触发 native abort）。字段头写错的典型症状：服务端 CREATOR 解析异常、客户端 readException 抛 NPE——看到这类错误先检查 parcel 布局。

编码原语的实际实现（Bridge 的 `Safe.java`，与上表规则一一对应）：

```java
public static int begin(Parcel p) {            // 对象头：魔数 + 长度占位
    p.writeInt(20293 | 0xFFFF0000);
    p.writeInt(0);
    return p.dataPosition();
}
public static void end(Parcel p, int pos) {    // 末尾回填总长度
    int cur = p.dataPosition();
    p.setDataPosition(pos - 4);
    p.writeInt(cur - pos);
    p.setDataPosition(cur);
}
public static void writeInt(Parcel p, int field, int v) {   // 字段头 field|(4<<16)
    p.writeInt(field | (4 << 16));
    p.writeInt(v);
}
public static void writeLong(Parcel p, int field, long v) { // 字段头 field|(8<<16)
    p.writeInt(field | (8 << 16));
    p.writeLong(v);
}
public static void writeString(Parcel p, int field, String v) { // 容器 field|0xFFFF0000
    p.writeInt(field | 0xFFFF0000);
    p.writeInt(0);
    int pos = p.dataPosition();
    p.writeString(v);
    end(p, pos);
}
```

### 6.3 AppKey 重写

按 docs/03 §3 的方式，在国行 Wear OS 的 AppKey 构造点把 Bridge 包映射为：

```
（逻辑包名 = com.fitbit.FitbitMobile, 摘要 = 官方 lineage 祖先摘要 29a4514c…）
```

此后 Bridge 的所有出站消息（含激活与同步触发）在手表看来都来自官方 Fitbit App，手表无需修改。验证：手表侧日志 `onMessageReceived: AppKey[com.fitbit.FitbitMobile,29a4514c…]` 且无 `Mismatched certificate`。

### 6.4 功能闭环

```
激活：  发送 /auth/request（auth:1.2 载荷，§3）
        → 等待 /pair/status SUCCESS → 手表端 Fitbit 进入主页
触发：  发送 /sync/v1/request/schedule（fire-and-forget，不等待回执）
```

另配深链入口（`fitbit://` scheme 的 VIEW intent）：手表上点"在手机上继续"会拉起手机侧 Fitbit 生态，深链使 Bridge 被选中后自动完成对应动作，无需手动打开 Bridge 操作。深链处理的核心逻辑（Bridge 的 `MainActivity.java`）：

```java
// fitbit://auth/seamless?obfuscatedGaiaId=..&device_type=..&nodeId=..&action=sync
String gaia = q(u, "obfuscatedGaiaId", "obfuscated_gaia");
String dev  = q(u, "device_type", "peripheral_device_type");
String node = q(u, "nodeId", "node_id");
if ("sync".equals(q(u, "action"))) {       // action=sync：自动触发同步
    exec.execute(() -> doSync());
} else if (node != null || gaia != null) { // 手表发起登录：自动发送 /auth/request
    exec.execute(() -> doAuth());
}
```

## 7. Pixel Watch App 的状态回调（"正在同步…"问题）

**症状**：Watch App 手表页持续显示"正在同步…"，即使手表实际早已同步完成。

**根因**（典型的乐观更新缺回调案例）：Watch App 侧有自己的 Fitbit 状态机——

```
绑定手机侧 Fitbit 状态服务 → 注册回调 → 触发刷新成功后
状态乐观置为 Syncing → 等待服务端回调推送最终状态 → 显示已同步
```

Watch App 绑定的目标原本是官方 Health 内的状态服务；本项目把它改绑到 Bridge（Watch App 只匹配组件名，不做服务端签名校验）。Bridge 实现**同名服务**并补全回调协议：

- 事务 `1` = 触发刷新、`2` = 解绑设备、`3` = 注册回调（包体 = token + strongBinder）；
- 注册后按协议格式推送状态包（字段：状态码 / 节点 ID / 最近同步时间 / 用户名等，SafeParcel）。

状态推送的实际实现（Bridge 的 `WearService.java`；f4 用户名为展示字段，需自行替换）：

```java
private void pushStatus(String node) {
    Parcel d = Parcel.obtain();
    d.writeInterfaceToken("com.google.android.apps.fitbit.app.cronus.api.wear.IFitbitStatusServiceCallback");
    d.writeInt(1);                                    // transact code 1：状态推送
    int pos = Safe.begin(d);
    Safe.writeInt(d, 1, 1);                           // f1 状态码（1 = 已同步）
    Safe.writeString(d, 2, node);                     // f2 节点 ID
    Safe.writeLong(d, 3, System.currentTimeMillis()); // f3 最近同步时间
    Safe.writeString(d, 4, "<userName>");             // f4 用户名（展示用）
    Safe.writeLong(d, 5, System.currentTimeMillis()); // f5
    Safe.writeString(d, 6, "");                       // f6
    Safe.end(d, pos);
    cb.transact(1, d, r, 0);
}
```

回调接通后，Watch App 状态机从 Syncing 切换到 Synced，"正在同步…"消失。**一般化结论**：跨进程状态 UI 停留在中间态时，先查"乐观更新是否有对应的回调通路"，而不是怀疑业务本身。

## 8. 最终分工

| 角色 | 版本 | 职责 |
|---|---|---|
| 修改版 Pixel Watch App | 重签名 | 配对/迁移/手表管理 UI；Fitbit 状态展示（回调来自 Bridge） |
| 修改版国行 Wear OS | 重签名 | 全部 Wear 传输与身份中枢：调用方放行、AppKey 重写、迁移链开关 |
| Bridge（自行实现） | 自签名 | 手表端 Fitbit 激活 + 同步触发 + 状态回调（对 Watch App） |
| 官方 Health | **原版不动** | 云端数据查看、Fitbit Air 管理、账号登录态 |
| 手表 | **零修改** | 激活后自主上云；始终认为对端是官方 App |

**闭环逻辑**：手表自主上云 → 云端数据属于账号 → 官方 Health 以官方签名获取云端 token → 手机上看到手表数据。Bridge 只需要在"激活"与"触发"两个时刻以官方 AppKey 身份发送消息，日常数据面与它无关。

---

回到 [README](../README.md)。
