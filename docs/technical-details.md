# 技术复盘：BBZQ 在 NPatch API102 下未生效

## 背景

BBZQ 是基于 libxposed API 102 的哔哩哔哩增强模块。目标环境中，普通管理器式流程不适用，最终采用 NPatch embedded mode：将模块嵌入目标应用，由目标应用启动时加载模块。

这次问题的迷惑点在于：模块并不是完全没加载。日志能看到 `Loaded in tv.danmaku.bili on Vector(1), api=102`，但用户在 BBZQ 设置页勾选功能、保存、重启目标应用后，功能没有实际生效。

最终定位到三个主要问题：

- NPatch 侧 API 102 服务契约不完整。
- RemotePreferences 写入路径是只读实现。
- BBZQ 独立设置 App 没有稳定收到 `XposedService` Binder。

## 现象

模块加载日志：

```text
Loaded in tv.danmaku.bili on Vector(1), api=102
Scan Errors: -
```

但功能开关未生效，部分 hook 显示为 disabled。早期关键错误：

```text
settings=remote write UnsupportedOperationException: Read only implementation
```

这说明运行时已经进入目标应用，但设置没有可靠同步到 embedded runtime。

## 问题一：API 102 不是单纯版本号

BBZQ 使用 libxposed API 102。NPatch 若只对外声明 API 102，而没有补齐相关服务接口，会造成现代模块服务能力不完整。

处理方向：

- modern loader 对外框架 API 调整为 `102`。
- 补齐 API 102 相关服务接口。
- 对 NPatch 当前不支持的能力返回明确的“不支持”，避免调用方遇到缺接口或半截实现。

相关修改点：

- `patch-loader/src/main/java/top/nkbe/npatch/loader/modern/VersionRouter.java`
- `patch-loader/src/main/java/top/nkbe/npatch/service/IntegrApplicationService.java`
- `patch-loader/src/main/java/top/nkbe/npatch/service/NeoLocalApplicationService.java`
- `patch-loader/src/main/java/top/nkbe/npatch/service/RemoteApplicationService.java`
- `share/android/src/main/java/org/lsposed/npatch/util/LocalInjectedModuleService.java`

示例方向：

```text
FRAMEWORK_API_VERSION = 102
registerHotReloadTarget(...) -> unsupported / delegated implementation
```

## 问题二：RemotePreferences 只读

日志：

```text
settings=remote write UnsupportedOperationException: Read only implementation
```

含义：

- BBZQ runtime 可以读取 remote preferences。
- 但设置更新无法写回 embedded runtime 使用的远程偏好设置。
- 用户在设置页保存后，目标应用进程仍读不到新的开关状态。

处理方向：

- 为 remote preferences editor 补充写入实现。
- 将 `clear`、`delete`、`put` 等 diff 应用到宿主侧 `SharedPreferences`。
- 写入后更新本地缓存并通知监听器。

相关修改点：

- `core/xposed/src/main/kotlin/org/matrix/vector/impl/VectorRemotePreferences.kt`
- `share/android/src/main/java/org/lsposed/npatch/util/LocalInjectedModuleService.java`

修复后，`Read only implementation` 不应再出现。

## 问题三：XposedService Binder 未稳定送达

NPatch embedded mode 下有两个关键运行侧：

- `tv.danmaku.bili`：目标应用进程中的 BBZQ runtime。
- `io.github.bbzq`：独立安装的 BBZQ 设置 App。

BBZQ 设置 App 需要收到 `XposedService` Binder，才能把设置同步到目标应用中的运行时。早期 Binder 注入发生得过早，`ActivityThread.currentApplication()` 可能还不可用，导致已安装模块 Provider 无法稳定收到 Binder。

处理方向：

- hook 宿主 `Application.attach(...)`。
- 在 attach 后构造 `VectorXposedService`。
- 调用已安装模块 Provider：`content://io.github.bbzq.XposedService`。
- 使用 `IXposedService.SEND_BINDER` 发送 Binder。

相关修改点：

- `patch-loader/src/main/java/top/nkbe/npatch/loader/LSPLoader.java`

成功日志：

```text
Sent XposedService binder to installed module io.github.bbzq
```

## 验证方式

只看 `Loaded in tv.danmaku.bili` 不够。至少需要同时确认：

```text
Loaded in tv.danmaku.bili on Vector(1), api=102
Sent XposedService binder to installed module io.github.bbzq
BBZQ runtime installed 30 hook(s)
HomeRecommendAd removed
```

负向检查：

```text
Read only implementation
UnsupportedOperationException
Unknown authority io.github.bbzq.XposedService
Could not notify installed module io.github.bbzq
```

这些不应在正常启动验证中出现。

## force-stop 测试陷阱

不要在验证前同时 force-stop 目标应用和 BBZQ。

如果手动强停 `io.github.bbzq`，Android 可能让其 Provider 暂时不可用，导致误导性日志：

```text
Unknown authority io.github.bbzq.XposedService
Failed to find provider info for io.github.bbzq.XposedService
```

更接近真实使用的验证方式：

1. 保持 BBZQ 独立 App 已安装且未被强停。
2. 只重启目标应用。
3. 查看 Binder 转发日志。
4. 查看实际 hook 行为日志。

## 使用结论

- NPatch Manager 不需要常驻。
- Shizuku 不需要日常开启。
- BBZQ 独立 App 需要保留。
- 改设置后，重启目标应用即可。
- 不要在启动目标应用前手动强停 BBZQ。
