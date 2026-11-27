# 修改记录：修复远程播放时加载圈永不消失

## 概览

| 项 | 内容 |
|---|---|
| 现象 | Android TV 上播放 SMB 视频时，画面已经开始播放，但屏幕中央的 VLC 圆锥加载动画一直旋转不消失（用户描述为「木播放转圈圈」） |
| 影响范围 | 远程媒体（SMB / HTTP 流）播放，尤其是首次播放、媒体库尚未完成启动时 |
| 修复文件 | `application/vlc-android/src/org/videolan/vlc/PlaybackService.kt`（唯一文件） |
| 改动规模 | +33 / -11 |
| 验证设备 | 192.168.1.110:5555，Android 5.1.1 (API 22)，arm64-v8a |
| 基线提交 | `baeb1edae` Bump the app version to 3.7.1 |
| 修复分支 | `fix/loading-spinner-stuck` |

---

## 一、问题定位过程

### 1.1 环境限制

目标电视是一台特殊机型的 Android 5.1 设备，排查中遇到以下限制，需要绕过：

- **无 logcat**：`/system/bin/logcat` 不存在，`log` 命令不支持 `-d`，`logcat_his` 无输出。设备上仅有 `log -p <优先级> -t <tag> message` 这一写接口。解决方案：在 App 内写文件日志，通过 `run-as org.videolan.vlc.debug cat files/xxx.txt` 读取。
- **`/sdcard` 只读**：`screencap -p /sdcard/x.png` 报 `Read-only file system`。解决方案：先写 `/data/local/tmp/`，再 `adb pull`。
- **shell 用户非 root**：`su` 不存在，`/data/data/...` 直接读取 Permission denied。解决方案：利用 debug 包可 `run-as` 的优势读应用私有目录。
- **Git Bash 路径转换**：`adb pull /data/local/tmp/x.png` 会被 MSYS 转换成 `C:/Program Files/Git/data/...`。解决方案：命令前加 `MSYS_NO_PATHCONV=1`。

### 1.2 复现与排除

逐步缩小范围，确认只有特定路径触发：

| 尝试的启动方式 | 结果 |
|---|---|
| `VIEW` intent 直接指定 SMB URI 冷启动播放 | 正常，无圆锥叠图 |
| TV 主页 browse UI → 本地网络 → SMB → 文件点击播放 | **复现**，圆锥持续旋转 |
| 带 `start_time` extra 从中间位置恢复播放 | **复现** |
| 播放第二个 SMB 文件（驯龙高手2） | **复现** |

### 1.3 关键证据

#### 证据 A：视图树显示加载层处于可见状态

视频明明在渲染，但加载层没有被隐藏：

```bash
adb shell "dumpsys activity top | grep player_overlay_loading"
```

修复前：

```
AppCompatImageView{... V.ED.... ........ 880,460-1040,620 app:id/player_overlay_loading}
```

`V` = Visible。视图尺寸 160x160，位于屏幕正中，正是圆锥叠在画面上的那一块。

修复后：

```
AppCompatImageView{... I.ED.... ......ID 880,460-1040,620 app:id/player_overlay_loading}
```

`I` = Invisible。

#### 证据 B：三层探针日志锁定断点

在三个环节埋入文件日志后，逐步定位到唯一的阻塞点（时间戳单位为毫秒，相对进程启动）：

```
[396] svc-loop: msg=CbAdd                      ← actor 正常消费
[422] svc-addCallback: cb=VideoPlayerActivity callbacks=2
[802] svc-loop: msg=UpdateMeta                 ← actor 收到 UpdateMeta
[803] updateMeta: enter
[803] updateMeta: afterInitSession
[805] updateMeta: bob-enter                    ← 进入 withContext(Dispatchers.Default)
[819] svc-onMediaPlayerEvent: type=259 send=true
[823] updateMeta: artwork-branch enter
[823] updateMeta: getFromMl enter              ← 进入后永久挂起，再无 exit
[1006] startLoading: isLoading=false isPlaying=false
[2445] plm-onEvent: type=274 vout=1            ← 视频已出画（Vout）
[2754] plm-onEvent: type=260                   ← Playing 事件已产生
[3075] svc-onMediaPlayerEvent: type=260 send=true   ← 入队成功
[3146] svc-onMediaPlayerEvent: type=259 send=true   ← Buffering=100 也入队成功
```

三个结论：

1. `plm-onEvent type=259 buffering=100.0` 确实产生了，`Buffering==100` 事件真实存在。
2. `svc-onMediaPlayerEvent ... send=true` 说明消息成功进入 channel。
3. 但此后 `svc-loop` 再无任何输出 —— actor 循环不再取消息。

即：**事件已入队，但消费方停摆**。`Buffering==100` 永远到不了 `VideoPlayerActivity.onMediaPlayerEvent`，因此 `stopLoading()` 从未被调用。

#### 证据 C：确认是挂起而非崩溃

在 actor 外层包了 `try/catch` 并记录 `svc-actor-DEAD`，该日志从未出现。说明 actor 协程没有抛异常退出，而是**被 suspend 卡住**。

---

## 二、根因分析

### 2.1 事件分发链路

```
libVLC native
   └─ MediaPlayer.onEventNative
        └─ PlaylistManager.mediaplayerEventListener.onEvent   (suspend)
             └─ service.onMediaPlayerEvent(event)
                  └─ cbActor.trySend(CbMediaPlayerEvent(event))   ← 入队
                       └─ cbActor 协程串行消费
                            └─ callback.onMediaPlayerEvent(event)  ← 各 UI 回调
```

`cbActor` 的消费循环是**顺序 await**的：

```kotlin
for (update in channel) when (update) { ... }
```

循环体里任何一个 `suspend` 调用未完成，下一条消息就取不到。它就是整条链路上唯一的串行瓶颈。

### 2.2 主因：`getFromMl()` 在 actor 内永久挂起

`updateMetadataInternal()` 里这段代码：

```kotlin
val mw = getFromMl { findMedia(media) }
```

`getFromMl` 的实现（`application/resources/.../util/Extensions.kt`）：

```kotlin
suspend inline fun <reified T> Context.getFromMl(crossinline block: Medialibrary.() -> T) =
    withContext(Dispatchers.IO) {
        val ml = Medialibrary.getInstance()
        if (ml.isStarted) block.invoke(ml)
        else {
            // 媒体库未启动：注册 onMedialibraryReady 监听，然后 startMedialibrary()
            suspendCancellableCoroutine { continuation -> ... }
        }
    }
```

关键点：**媒体库未就绪时，它会一直挂起，等待 `onMedialibraryReady` 回调**。

播放 SMB 远程媒体时，媒体库启动被推迟（需要扫描/初始化），`onMedialibraryReady` 迟迟不来，于是：

```
actor 消费 UpdateMeta → updateMetadataInternal() → getFromMl() 挂起
   ↓
for 循环停在这一条消息上，不再取 channel 中的后续消息
   ↓
Buffering==100 排在其后，永远得不到处理
   ↓
VideoPlayerActivity.stopLoading() 永不执行
   ↓
player_overlay_loading 保持 Visible，圆锥一直转
```

### 2.3 次因一：`setupScope()` 重复调用会丢弃已入队消息

`setupScope()` 被调用两次：

- `onCreate()` 第 740 行
- `onStartCommand()` 第 891 行

原实现每次都无条件 `cbActor = lifecycleScope.actor(...)`，第二次会**用新 channel 覆盖旧 channel**。旧 channel 里已经入队的消息就此无人消费，而 `callbacks` 列表挂在 Service 实例上保持不变。

实测时序证实了这次调用确实发生：

```
[309] setupScope: NEW actor created, previous=NONE
[394] setupScope: NEW actor created, previous=EXISTS   ← 第二次
```

**但这不是本次现象的直接原因**。实测显示第二个 actor 仍在正常工作（420/422、763 都有 `svc-loop` 输出），说明切换本身没有立即致死。它只会在「切换瞬间恰好在途」的消息上造成丢失，属于必须修的竞态隐患，但本次故障由 2.2 主导。

### 2.4 次因二：单条回调异常会终止整个 actor

原来的 `when` 表达式没有异常保护。任何一个 callback 抛异常都会让 actor 协程退出，之后**永久**不再分发任何事件，且没有任何日志。

本次实测中该分支并未触发（无 `svc-actor-DEAD` 日志），属于加固措施。

---

## 三、修改内容

### 3.1 主修复：为媒体库查询加超时兜底

```diff
--- a/application/vlc-android/src/org/videolan/vlc/PlaybackService.kt
+++ b/application/vlc-android/src/org/videolan/vlc/PlaybackService.kt
@@ -1231,7 +1244,13 @@
                     else -> {
                         //The media id may be 0 on resume
-                        val mw = getFromMl { findMedia(media) }
+                        //getFromMl() suspends until the medialibrary is ready. On a remote (SMB/HTTP)
+                        //media the medialibrary startup can be delayed, and this runs inside the
+                        //callback actor which consumes player events serially: blocking here stops
+                        //every later event, so Buffering==100 never reaches the player overlay and
+                        //the loading spinner never stops. Bound the wait and fall back to the
+                        //MediaWrapper we already have.
+                        val mw = withTimeoutOrNull(ML_LOOKUP_TIMEOUT) { getFromMl { findMedia(media) } } ?: media
                         val mediaId = MediaSessionBrowser.generateMediaId(mw)
                         artworkMap[mediaId] ?: ArtworkProvider.buildMediaUri(ctx, mw)
                     }
```

同步新增常量：

```diff
@@ -2030,6 +2049,9 @@
         private const val UPDATE_META = "update_meta"
 
+        /** Upper bound for a medialibrary lookup done while building the media metadata. */
+        private const val ML_LOOKUP_TIMEOUT = 2000L
+
```

以及 import：

```diff
+import kotlinx.coroutines.withTimeoutOrNull
```

**为什么是 2000ms**：媒体库正常就绪时该查询是毫秒级的，2 秒足够覆盖正常路径；超时后回退到已有的 `MediaWrapper` 对象（`media`），此时封面 URI 退化为非缓存路径，属于可接受的降级，不影响播放。

### 3.2 加固一：`setupScope()` 单实例化

```diff
@@ -838,17 +839,29 @@
     @OptIn(ObsoleteCoroutinesApi::class)
     private fun setupScope() {
+        // Called from onCreate() and again from onStartCommand(). Recreating the actor drops the
+        // previous channel while callbacks stay registered on this instance, so nothing consumes
+        // the queued messages anymore. Keep a single actor for the service lifetime.
+        if (::cbActor.isInitialized) return
         cbActor = lifecycleScope.actor(capacity = Channel.UNLIMITED) {
```

保证 Service 生命周期内只有一个 actor，`onStartCommand()` 的重复调用变成幂等操作。

### 3.3 加固二：逐条消息的异常隔离

```diff
-            for (update in channel) when (update) {
-                CbUpdate -> for (callback in callbacks) callback.update()
-                is CbMediaEvent -> for (callback in callbacks) callback.onMediaEvent(update.event)
-                is CbMediaPlayerEvent -> for (callback in callbacks) callback.onMediaPlayerEvent(update.event)
-                is CbRemove -> callbacks.remove(update.cb)
-                is CbAdd -> callbacks.add(update.cb)
-                ShowNotification -> showNotificationInternal()
-                is HideNotification -> hideNotificationInternal(update.remove)
-                UpdateMeta -> updateMetadataInternal()
-                UpdateState -> executeUpdate(true)
+            for (update in channel) {
+                // One failing callback must not terminate the actor: that would silently stop
+                // delivering player events to the UI (stuck loading spinner, frozen overlay).
+                try {
+                    when (update) {
+                        CbUpdate -> for (callback in callbacks) callback.update()
+                        is CbMediaEvent -> for (callback in callbacks) callback.onMediaEvent(update.event)
+                        is CbMediaPlayerEvent -> for (callback in callbacks) callback.onMediaPlayerEvent(update.event)
+                        is CbRemove -> callbacks.remove(update.cb)
+                        is CbAdd -> callbacks.add(update.cb)
+                        ShowNotification -> showNotificationInternal()
+                        is HideNotification -> hideNotificationInternal(update.remove)
+                        UpdateMeta -> updateMetadataInternal()
+                        UpdateState -> executeUpdate(true)
+                    }
+                } catch (t: Throwable) {
+                    Log.e(TAG, "Callback dispatch failed for ${update.javaClass.simpleName}", t)
+                }
             }
```

---

## 四、验证结果

### 4.1 修复前后对比

| 检查项 | 修复前 | 修复后 |
|---|---|---|
| `player_overlay_loading` 视图状态 | `V.ED....`（可见） | `I.ED....`（不可见） |
| 圆锥是否叠在画面上 | 是，持续旋转 | 否，画面干净 |
| `Buffering=100` 是否被处理 | 否，actor 早被卡死 | 是，`stopLoading()` 正常执行 |

### 4.2 功能回归

安装修复后的 APK 到电视实测：

| 测试场景 | 结果 |
|---|---|
| 八仙！SMB 文件从头播放 | 正常，圆锥不叠加，画面干净 |
| 八仙！seek 到 15 分钟处播放 | 正常，`I.ED....` |
| 驯龙高手2 SMB 文件播放 | 正常，`I.ED....` |

### 4.3 构建产物

```
application/app/build/outputs/apk/debug/VLC-Android-3.7.1-debug-all.apk
```

122 MB，全 ABI 通用包，debug 签名，包名 `org.videolan.vlc.debug`。

---

## 五、排查中使用的辅助手段（已全部清理）

为定位问题临时增加、验证后已全部移除的内容：

- `application/vlc-android/src/org/videolan/vlc/util/LoadingTrace.kt` —— 写 `files/loading_trace.txt` 的探针工具类
- `PlaybackService.kt` 中的 `svc-loop` / `svc-addCallback` / `svc-dispatch` / `svc-onMediaPlayerEvent` / `svc-actor-DEAD` 等日志点
- `VideoPlayerActivity.kt` 中的 `onCreate` / `startLoading` / `stopLoading` / 事件类型日志点
- `PlaylistManager.kt` 中的 `plm-onEvent` 日志点

最终提交只保留 `PlaybackService.kt` 的功能性修改，`git diff baeb1edae HEAD` 净改动为 +33 / -11。

临时设备端文件（`/data/local/tmp/*.png`、应用私有目录下的 `loading_trace.txt`）不影响仓库。

---

## 六、提交信息

```
Fix video loading spinner never stopping after remote media playback

PlaybackService routes player events to its callbacks through a serial
actor. While a remote medium was already rendering, that actor stopped
consuming events, so Buffering==100 never reached the player view and
stopLoading() was never called: the loading indicator kept spinning on
top of the playing video.

Root cause: updateMetadataInternal() calls getFromMl(), which suspends
until the medialibrary is ready. With an SMB medium whose medialibrary
startup is delayed, the actor loop stayed suspended and every later event
queued behind it. Bound that lookup with withTimeoutOrNull and fall back
to the MediaWrapper already at hand.

Two related robustness fixes in the same dispatch path:

- setupScope() runs from onCreate() and again from onStartCommand().
  Recreating the actor abandons the previous channel, so a message sent
  just before the switch is never consumed even though callbacks stay
  registered on this instance. Keep a single actor for the service
  lifetime.
- A throwing callback terminated the whole actor, silently freezing event
  delivery. Contain the failure per message and log it instead.
```

---

## 七、备注

1. **术语准确性**：`PlaybackService` 的 `coroutineContext` 是 `Dispatchers.IO + SupervisorJob()`，所以 actor 是「串行消费」而不是「单线程阻塞」。`getFromMl()` 是 suspend 函数，它**挂起协程**等待回调，不是阻塞线程。描述该类问题时用 suspended 比 blocked 更准确。

2. **`ML_LOOKUP_TIMEOUT` 的合理性**：2 秒是一个保守值。如果媒体库在某些设备上初始化更慢，可以调大；但过大会重新引入「加载圈长时间不消失」的观感问题。当前值在正常路径下不会被触及。

3. **`callbacks` 未加同步保护**：`cbActor` 是唯一读写 `callbacks` 的地方，串行化保证了安全性。如果后续有人把 `callbacks` 的访问挪到 actor 之外，需要补上同步措施。

4. **绕过的设备限制不影响修复本身**：无 logcat、`/sdcard` 只读等问题是这台电视固件的特性，与 VLC 代码无关，修复在标准 Android 设备上同样适用。