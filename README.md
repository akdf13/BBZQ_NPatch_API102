# BBZQ + NPatch API102 部署复盘

这个仓库用于记录一次将 BBZQ 跑在哔哩哔哩 `9.0.0` 上的排障过程。

内容包括：

- 为什么选择 NPatch embedded mode。
- 怎么解决现有NPatch版本不支持API102的问题。
- BBZQ 设置页和宿主进程之间的设置同步断掉如何解决。
- 如何通过日志判断 Binder、RemotePreferences 和 hook 是否真的工作。

## 仓库内容

- [docs/technical-details.md](docs/technical-details.md)：问题复盘和总结。
- [docs/reproduce-locally.md](docs/reproduce-locally.md)：复现和流程说明。

## 最终有效状态

- 目标应用：哔哩哔哩 `9.0.0`
- BBZQ：`v1.1.0-171`
- libxposed API：`102`
- 部署模式：NPatch embedded / integrated mode
- NPatch 配置：`useManager=false`
- 正常使用不需要 NPatch Manager 或 Shizuku 常驻
- BBZQ 独立 App 仍需保留，用于设置页面和 `XposedService` 同步

关键验证日志：

```text
Loaded in tv.danmaku.bili on Vector(1), api=102
Sent XposedService binder to installed module io.github.bbzq
BBZQ runtime installed 30 hook(s)
HomeRecommendAd removed
```

## 不提供的内容

本仓库不会上传：

- 哔哩哔哩原始 APK。
- 注入 BBZQ 后的哔哩哔哩完整 APK。
- 以及任何不适合公开分发的二进制文件。

需要复现时，请自行准备合法获取的目标应用 APK，在本地完成 patch。

## 相关项目

- BBZQ：<https://github.com/HSSkyBoy/BBZQ>
- NPatch：<https://github.com/MapleRecall/NPatch>

## 状态

当前文档先记录工程复盘。后续如果整理出可独立提交的 NPatch 兼容补丁，应优先向 NPatch 上游提交 PR，而不是在这里维护长期 fork。
