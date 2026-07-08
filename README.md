# BBZQ + NPatch API102 折腾记录

这是一次把 BBZQ 跑进 B 站 9.0.0 的个人排障复盘。

更好读的博客版在这里：

- [和 BBZQ、NPatch、API102 缠斗的一天](https://nastl3e.cn/posts/bbzq-npatch-api102/)

## 这里有什么

- `article.md`：博客正文镜像，方便在 GitHub 上阅读和引用。
- `notes/deployment-summary.md`：更偏工程复盘的摘要，记录关键判断、问题链路和最终验证方式。

## 这里没有什么

这个仓库不发布、不托管、不传播任何改包成品、安装包、签名文件、密钥或完整设备日志。

它只记录排障思路和兼容层经验：怎么判断方向、怎么读日志、为什么 `api=102` 不只是一个版本号、以及为什么“模块加载了”不等于“功能生效了”。

## 关于 fork 原仓库

这类经验分享不需要 fork BBZQ 或 NPatch 原仓库。

fork 更适合这类情况：

- 要给上游提交代码修复。
- 要补充上游文档。
- 要提交可复现的 bug report / PR。

如果只是写一次个人经验复盘，放在自己的博客和专题仓库里更合适，也更不容易把“经验分享”和“分发改包”混在一起。

## 关键词

`Android`、`Xposed`、`NPatch`、`BBZQ`、`API102`、`RemotePreferences`、`XposedService Binder`
