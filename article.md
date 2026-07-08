# 和 BBZQ、NPatch、API102 缠斗的一天

> 原文发布在：[https://nastl3e.cn/posts/bbzq-npatch-api102/](https://nastl3e.cn/posts/bbzq-npatch-api102/)

今天原本只是想做一件看上去很朴素的事：把 BBZQ 部署到手机里的 B 站 9.0.0。

结果它像一只不愿进猫包的猫：你觉得已经抱住了，它从 API 版本里滑走；你觉得终于装上了，它又从设置同步里钻出去；你准备长叹一声，它还会在日志里淡淡留下一句：`Read only implementation`。

好消息是，最后这只猫被抱回来了。

这篇记录不放安装包，不分享改包成品，只记一下今天真正有价值的部分：判断方向、读日志、修兼容层，以及怎么确认“看起来装上了”和“真的生效了”不是一回事。

## 第一幕：我们一开始走错了门

最开始的直觉是：既然 BBZQ 是 Xposed 模块，那就找一个管理器导入模块，让它在 B 站里跑起来。

听起来很合理，直到我们去看项目 README 和实际工具界面，发现这条路不太对。LSPatch 管理界面里没有我们预期的导入入口，而 BBZQ 的 README 更像是在提示：它需要的是能把模块服务和目标应用运行时对接起来的环境。

于是路线从“管理器模式”切换到了 **NPatch 内嵌模式**：

- 把 BBZQ 模块嵌进目标应用。
- 让目标应用启动时加载 BBZQ。
- 保留 BBZQ 独立设置页，用来调整功能开关。

这一步的关键经验是：不要只看工具名字相似，不要只凭“Xposed 模块都差不多”来判断。Android 生态里的 patch / loader / manager / service 几个词经常像同一家族里脸很像的亲戚，但它们负责的事可能完全不同。

## 第二幕：装上不等于生效

后来模块确实能加载了，日志里也能看到：

```text
Loaded in tv.danmaku.bili on Vector(1), api=102
```

这句话很容易让人开心。

但开心早了。实际打开 B 站后，勾选功能、保存、重启，发现功能没有生效。这个状态最迷惑：它不是“完全没加载”，也不是“安装失败”，而是模块像坐在教室里，书也翻开了，但作业本没有带。

继续看日志，开始出现真正的线索：

```text
settings=remote write UnsupportedOperationException: Read only implementation
```

翻译成人话就是：BBZQ 的运行时想写远程设置，但 NPatch 提供给它的那套偏好设置实现是只读的。

也就是说，模块本体能进 B 站，hook 点也能扫到，但开关状态没有顺利同步进去。于是所有功能都像被蒙在鼓里：用户在设置页说“开”，运行时在 B 站里听到的还是“关”。

## 第三幕：API102 不是一个数字，是一张欠条

这次最显眼的卡点叫 `api=102`。

一开始 NPatch 的现代模块加载层更接近旧 API。BBZQ 这边需要的却是 API 102 的服务能力。只把版本号改成 102 是不够的，因为 API 不是贴纸，背后有接口契约。

这次补的核心方向有两块：

- 让 NPatch 的现代加载器按 API 102 对外工作。
- 把 API 102 需要的新服务接口补齐，哪怕某些能力在 NPatch 本地服务里只能返回“不支持”。

比如热重载相关接口就属于这种情况。BBZQ 不一定真的需要 NPatch 支持完整热重载，但接口缺了，现代模块框架就会像接电话时发现对方少了半根电话线：不一定马上炸，但迟早奇怪。

所以这部分修复不是“让某个功能变强”，而是“让服务契约完整”。这在兼容层里特别常见：很多时候 bug 不是业务逻辑写错，而是你答应对方有一扇门，结果墙上只画了一扇门。

## 第四幕：设置同步才是真凶

修完 API102 兼容后，BBZQ 能加载，hook 也能安装，但功能还是没有按预期全部生效。

继续沿日志往下挖，问题集中到了“设置如何从 BBZQ 独立 App 传给嵌在 B 站里的运行时”。

在 NPatch 内嵌模式下，有两个世界：

- B 站进程里的 BBZQ 运行时。
- 独立安装的 BBZQ 设置界面。

这两个世界要同步设置，需要一座桥。对现代 Xposed 模块来说，这座桥就是 `XposedService` Binder。

问题是，之前 Binder 没有稳定送到已安装的 BBZQ 模块 Provider。于是设置页保存了，B 站里的运行时却没收到。

后来的修复思路是：

- 在宿主 `Application.attach` 之后再发送 Binder，避免太早执行时应用上下文还没准备好。
- 为嵌入模块构造 `VectorXposedService`。
- 调用已安装模块的 `content://io.github.bbzq.XposedService`，把 Binder 送过去。

最终我们想看到的日志是：

```text
Sent XposedService binder to installed module io.github.bbzq
```

这句话出现后，事情就从“玄学试试”变成了“链路闭合”。

## 第五幕：一次误判，以及 force-stop 的小陷阱

中间还有一个很有意思的误判。

为了测试干净，我们一度同时强行停止了 B 站和 BBZQ。结果日志里出现：

```text
Unknown authority io.github.bbzq.XposedService
Failed to find provider info for io.github.bbzq.XposedService
```

第一眼看像是 Binder 转发又失败了。

但问题其实是：被手动强停后的 BBZQ Provider 暂时不可用。这个测试方式过于“干净”，干净到把我们要测试的桥也拆了。

后来换成只重启 B 站，不强停 BBZQ，再看日志，链路就正常了：

```text
Sent XposedService binder to installed module io.github.bbzq
BBZQ runtime installed 30 hook(s)
HomeRecommendAd removed 2 item(s)
```

这也是今天一个很实用的经验：测试不是越狠越好。Android 里的 force-stop 会改变应用可被唤起、Provider 可用性等状态。你以为自己在“清场”，其实可能把舞台灯也关了。

## 最后的有效状态

最终确认的有效状态大概是：

- 目标应用是 B 站 `9.0.0`。
- 部署方式是 NPatch 内嵌模式，不依赖 NPatch 管理器常驻。
- BBZQ 独立 App 仍然保留，用于设置页和服务同步。
- 日常使用不需要手动开启 Shizuku 或 NPatch 管理器。
- 改设置后，建议打开/保持 BBZQ 可用，然后只重启 B 站。
- 不要在启动 B 站前手动强停 BBZQ，否则设置服务 Provider 可能不可用。

最终判断“真的生效”的证据不是“安装成功”，也不是“模块加载成功”，而是日志里出现实际 hook 的行为，例如：

```text
HomeRecommendAd removed
```

这才是猫真的进包了。

## 今天学到的几件小事

第一，README 要认真读。我们一开始差点把时间花在不合适的 LSPatch 路线上，后来回头读项目说明，才把方向掰回 NPatch 内嵌模式。

第二，兼容层的问题不要只改版本号。`api=102` 不是一枚徽章，而是一组接口承诺。版本号改了，接口没补齐，就像穿了白大褂但忘了带听诊器。

第三，“能加载”和“能工作”之间隔着设置、服务、进程、Provider、Binder 这些地下管线。用户看到的只是一个开关，底下可能是一整个城市排水系统。

第四，日志是会说话的。只是它说话像猫：不直接告诉你“我饿了”，只会把杯子推下桌子。`Read only implementation`、`Unknown authority`、`Sent XposedService binder` 这些杯子碎片，拼起来就是答案。

第五，分享这类经验时，最好分享方法论、排障路径和补丁思路，而不是分享改好的安装包。前者能帮人理解系统，后者只会制造更多不可控变量。

## 如果以后还要继续

这次修复更像是一次本地兼容验证。以后如果要做得更漂亮，可以考虑把 NPatch 侧的兼容修复整理成更小、更清晰的补丁，再去和上游项目讨论。

到那时，才需要 fork 原仓库、开分支、提交 PR。

而今天这篇博客，只负责记录一只 API102 野猫如何被我们从日志草丛里慢慢哄出来。
