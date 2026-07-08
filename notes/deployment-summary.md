# 部署复盘摘要

## 最终结论

BBZQ 最终通过 NPatch 内嵌模式在 B 站 `9.0.0` 中生效。

关键验证日志包括：

```text
Loaded in tv.danmaku.bili on Vector(1), api=102
Sent XposedService binder to installed module io.github.bbzq
BBZQ runtime installed 30 hook(s)
HomeRecommendAd removed
```

## 主要判断

- 不采用 LSPatch 管理器导入路线。
- 不 fork 原项目来写经验贴；经验贴放个人博客和本专题仓库。
- NPatch 内嵌模式下，BBZQ 独立 App 仍然有价值：它负责设置界面和服务同步。

## 遇到的问题链路

### 1. API102 兼容层不完整

BBZQ 需要现代 Xposed API 102 能力。只修改版本号不够，还需要补齐服务接口契约。

修复方向：

- 让现代加载器对外按 API 102 工作。
- 补齐 API 102 新增接口，例如热重载注册相关方法。
- 对 NPatch 本地暂不支持的能力返回明确的“不支持”，而不是让接口缺失。

### 2. RemotePreferences 只读

日志线索：

```text
settings=remote write UnsupportedOperationException: Read only implementation
```

含义：

- 模块运行时能加载。
- hook 点能扫描。
- 但功能开关不能可靠写入远程设置。

修复方向：

- 为 NPatch 的远程偏好设置实现补写入逻辑。
- 将设置 diff 应用到宿主侧存储。
- 通知偏好设置监听器刷新。

### 3. XposedService Binder 没有送到已安装模块

现象：

- BBZQ 设置页保存后，B 站内嵌运行时仍读不到更新后的开关。

修复方向：

- 在宿主 `Application.attach` 后再通知已安装模块 Provider。
- 为模块构造 `VectorXposedService` Binder。
- 调用 `content://io.github.bbzq.XposedService` 并发送 Binder。

期望日志：

```text
Sent XposedService binder to installed module io.github.bbzq
```

### 4. force-stop 导致的测试误判

错误测试方式：

- 同时强停 B 站和 BBZQ 后再启动 B 站。

可能出现：

```text
Unknown authority io.github.bbzq.XposedService
Failed to find provider info for io.github.bbzq.XposedService
```

更贴近真实使用的验证方式：

- 不手动强停 BBZQ。
- 只重启 B 站。
- 查看 Binder 转发和 hook 生效日志。

## 使用经验

- 日常使用 B 站不需要启动 Shizuku 或 NPatch 管理器。
- BBZQ 独立 App 不建议手动强停。
- 改完 BBZQ 设置后，只重启 B 站更稳。
- 判断生效要看实际 hook 行为，而不只是安装成功或模块加载成功。

## 不在仓库中提供的内容

- 修改后的 APK。
- 原始 APK。
- 签名文件。
- 密钥。
- 完整设备日志。

这个仓库只保留排障记录和经验摘要。
