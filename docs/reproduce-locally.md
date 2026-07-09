# 本地复现流程

本文只描述复现路线，不提供第三方闭源应用安装包。

## 前提

- Windows 或 Linux 构建环境。
- Android SDK / platform-tools。
- JDK 21。
- 自行合法获取的目标应用 APK。
- BBZQ 模块 APK。
- 修改后的 NPatch 构建产物，或自行按技术复盘修改 NPatch 后构建。

## 构建 NPatch

NPatch 当前构建需要 JDK 21。若系统默认 Java 版本过新导致 Gradle/Kotlin 解析失败，应显式指定 JDK 21。

PowerShell 示例：

```powershell
$env:JAVA_HOME = 'C:\path\to\jdk21'
$env:ANDROID_HOME = 'C:\Users\<User>\AppData\Local\Android\Sdk'
$env:ANDROID_SDK_ROOT = $env:ANDROID_HOME
$env:Path = "$env:JAVA_HOME\bin;$env:ANDROID_HOME\platform-tools;$env:Path"
```

构建：

```powershell
.\gradlew.bat :jar:buildRelease
```

## 打包目标应用

示例参数仅展示结构，路径请替换为本地文件：

```powershell
& 'C:\path\to\jdk21\bin\java.exe' `
  '-Djava.security.properties=C:\path\to\java-security-bc.properties' `
  -jar 'path\to\npatch-release.jar' `
  -f `
  -o 'outputs\npatched' `
  -l 1 `
  -m 'path\to\bbzq.apk' `
  'path\to\target-app.apk'
```

说明：

- `-m` 指向 BBZQ 模块 APK。
- 最后一个参数是用户自行准备的目标应用 APK。
- 输出目录会生成本地 patch 后的 APK。
- 本仓库不托管该输出 APK。

## 安装验证

如果设备上已有同包名但签名不同的目标应用，覆盖安装会失败。是否卸载旧版本取决于用户自己的数据备份情况。

验证时建议：

```powershell
$adb = 'C:\path\to\adb.exe'
$serial = '<device-serial>'

& $adb -s $serial logcat -c
& $adb -s $serial shell am force-stop tv.danmaku.bili
& $adb -s $serial shell monkey -p tv.danmaku.bili -c android.intent.category.LAUNCHER 1
Start-Sleep -Seconds 35
& $adb -s $serial logcat -d -v time |
  Select-String -Pattern 'BBZQ|NPatch-Loader|Sent XposedService binder|Read only implementation|UnsupportedOperationException|Unknown authority|HomeRecommendAd removed|startHook'
```

不要在这一步 force-stop `io.github.bbzq`，否则可能让 Provider 不可用，造成误判。

## 成功标准

需要看到：

```text
Loaded in tv.danmaku.bili on Vector(1), api=102
Sent XposedService binder to installed module io.github.bbzq
BBZQ runtime installed 30 hook(s)
HomeRecommendAd removed
```

不应看到：

```text
Read only implementation
UnsupportedOperationException
Unknown authority io.github.bbzq.XposedService
Could not notify installed module io.github.bbzq
```
