# BBZQ + NPatch API102 部署复盘

这个仓库用于记录一次将 BBZQ 跑在哔哩哔哩 `9.0.0` 上的排障过程。

目标不是发布改包成品，而是把这次真正有复用价值的内容沉淀下来：

- 为什么最终选择 NPatch embedded mode。
- 为什么 `api=102` 不是只改版本号。
- 为什么“模块加载了”不等于“功能生效了”。
- BBZQ 设置页和宿主进程之间的设置同步如何断掉。
- 如何通过日志判断 Binder、RemotePreferences 和 hook 是否真的工作。

## 仓库内容

- [docs/technical-details.md](docs/technical-details.md)：完整技术复盘。
- [docs/reproduce-locally.md](docs/reproduce-locally.md)：本地复现和构建流程说明。
- [docs/release-policy.md](docs/release-policy.md)：为什么不发布原始/改包 B 站 APK，以及和应用商店分发的区别。

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
- 签名文件、证书、密钥。
- 含有设备标识的完整日志。
- 任何不适合公开分发的二进制文件。

可公开分享的是技术路线、排障过程、构建脚本思路和校验方法。需要复现时，请自行准备合法获取的目标应用 APK，在本地完成 patch。

## 相关项目

- BBZQ：<https://github.com/HSSkyBoy/BBZQ>
- NPatch：<https://github.com/MapleRecall/NPatch>

## 状态

当前文档先记录工程复盘。后续如果整理出可独立提交的 NPatch 兼容补丁，应优先向 NPatch 上游提交 PR，而不是在这里维护长期 fork。
