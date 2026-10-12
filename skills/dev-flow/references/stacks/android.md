# Android 技术栈指南

适用于原生 Android 项目（Kotlin / Java），以及 Flutter、React Native 工程里的 `android/` 部分。

这里只写所有 Android 项目都成立的做法和坑。项目特有的写进项目档案。

## 生成档案时怎么推断

| 要确定什么 | 怎么查 |
|---|---|
| 模块 | `settings.gradle(.kts)` 里的 `include` |
| 依赖版本 | 有 `gradle/libs.versions.toml`（版本目录）的，加依赖、改版本都写在这里，不在 build.gradle 里直接写版本号 |
| 构建变体 | `build.gradle(.kts)` 里的 `buildTypes` 和 `productFlavors`。可用任务看 `./gradlew tasks --all` |
| 快速编译检查 | `./gradlew :<模块>:compile<变体>Kotlin`；纯 Java 模块用 `compile<变体>JavaWithJavac` |
| 整包构建 | `./gradlew :app:assemble<变体>` |
| 单测 | `./gradlew :<模块>:test<变体>UnitTest`。仪器测试 `connected<变体>AndroidTest` 需要连着设备 |
| 代码规范 | `./gradlew :<模块>:lint<变体>`。光跑 `lint` 只查默认变体，别的渠道的代码会漏掉。配了 ktlint、detekt、spotless 的，用检查任务（`ktlintCheck`、`detekt`、`spotlessCheck`），别跑 `ktlintFormat`、`spotlessApply`，它们会直接改文件 |
| 系统版本 | `minSdk`、`targetSdk` |
| 本地持久化 | Room（`@Database` 的 `version`）、SQLDelight、DataStore 等 |
| 界面验证工具 | `adb devices` 能不能看到设备或模拟器。adb 不在 PATH 里的，去 `$ANDROID_HOME/platform-tools` 找（macOS 默认在 `~/Library/Android/sdk/platform-tools`）。有没有配 Appium、Espresso、UI Automator |

## 编译验证

- 用项目里的 `./gradlew`，不要用全局安装的 gradle。
- 首次构建慢、增量构建快，两个耗时都记进档案。
- 判断成败看退出码和 `BUILD SUCCESSFUL`。找错误：`grep -nE '^e: |\.java:[0-9]+: |error:|FAILURE: Build failed' <日志>`（Kotlin 的错误行以 `e: file:///...:行:列` 开头；javac 的报错会跟着系统语言变成中文，比如 `错误: 找不到符号`，所以按「文件.java:行号: 」匹配）。还看不出原因的，看日志里 `* What went wrong:` 那一段，Gradle 会在那里写明是哪个任务、为什么失败。
- 同一时间只跑一个构建，否则会等锁甚至超时。
- 依赖下载失败，先查代理设置：项目的 `gradle.properties` 和全局的 `~/.gradle/gradle.properties` 都可能配了。可以加 `--offline` 试试只用本地缓存能不能编过。网络和代理是环境问题，告诉用户，不要改项目里的代理配置。

## 新建文件、改资源后要做的事

- **组件和权限**：新的 Activity、Service、Receiver、Provider 要在 `AndroidManifest.xml` 里注册。新权限要声明，危险权限还要在运行时申请。
- **新模块**：要加进 `settings.gradle`，用到它的模块也要声明依赖。
- **资源**：文件名只能用小写字母、数字和下划线。
- **多语言文案**：各个 `values-xx/strings.xml` 要同步加 key。字符串里的单引号要写成 `\'`，XML 特殊字符也要转义，否则编译报错。
- **混淆**：release 包开了 R8 或 ProGuard 时，反射、序列化（Gson 等）、JNI 用到的类要加 keep 规则，否则只有 release 包会崩溃。AGP 8 起默认用 R8 完整模式，比以前裁得更狠。
- **签名**：release 包用项目的签名配置（keystore）。缺 keystore 或密码是环境问题，告诉用户，不要改 `signingConfigs`，也不要把 keystore 和密码提交进仓库。

## 持久化模型变更

- **Room**：升 `version` 并提供 `Migration`，或者明确接受用 `fallbackToDestructiveMigration` 清空数据。项目导出了 schema 的，要把新生成的 schema json 一起提交。
- 验证时先装旧版本，再用 `adb install -r` 覆盖安装新版本，测升级路径。

## 界面验证与日志

- **设备准备**：手机要打开 USB 调试，并在手机上点「允许」（`adb devices` 显示 unauthorized 就是还没允许）。屏幕要亮着、解锁，点按才有效。不少国产手机还要在开发者选项里额外打开开关（比如小米的「USB 调试（安全设置）」），模拟点按、安装才不会失败。这些请用户在手机上操作。
- **安装和启动**：安装用 `adb install -r <apk>`（覆盖安装，保留数据）。签名不同的包、版本更低的 release 包装不上，要先卸载（debug 包降级可以加 `-d`）；卸载会清掉 App 的数据（登录状态、本地记录），先问用户。启动用 `adb shell monkey -p <包名> -c android.intent.category.LAUNCHER 1`，重启先 `adb shell am force-stop <包名>`。
- **截图**：`adb exec-out screencap -p > <文件>.png`
- **点按**：`adb shell input tap <x> <y>`（像素坐标）；输入文字用 `adb shell input text`，空格写成 `%s`。界面结构用 `adb shell uiautomator dump` 导出到手机的 `/sdcard/window_dump.xml`，`adb pull` 下来读控件的文字和位置（bounds）。项目已有的 UI 测试（Espresso、UI Automator、Appium）照常跑，不为验证新写测试。
- **权限弹窗**：有的手机 `pm grant` 授不了权限，可以直接点系统的授权弹窗。
- **日志**：`adb logcat --pid=$(adb shell pidof -s <包名>)`，App 重启后进程号会变，要重新取；按标签过滤用 `adb logcat -s <TAG>`。要留完整日志，先 `adb logcat -c` 清掉旧的，再 `adb logcat -v threadtime > <文件>` 在后台跑。
- 连着多台设备时，每条 adb 命令都要加 `-s <序列号>`；别动别的会话正在用的设备或模拟器。

## 宿主是 Flutter 或 React Native 时

这一节只讲原生这一侧要注意的。改的是 Dart 代码时，读 [flutter.md](flutter.md)。

- 先弄清改动边界，写进档案。
- 不要手改生成出来的文件，比如 Flutter 的 `GeneratedPluginRegistrant`。
- 构建：Flutter 用 `flutter build apk --debug [--flavor <渠道>]`；React Native 用 `cd android && ./gradlew assembleDebug`，JS 部分由 Metro 提供。
- 反过来，原生工程以源码方式嵌入 Flutter 模块的（add-to-app），编译前先在 Flutter 模块目录跑 `flutter pub get`，生成模块里的 `.android/`。这个目录每次都会重新生成，别改它。

## 危险信号

| 冒出这个念头 | 实际情况 |
|---|---|
| debug 包没问题就行 | release 包开了混淆，反射、序列化用到的类可能被裁掉。关键改动要打 release 包验证 |
| 模型只是加了个字段 | Room 要升版本并提供迁移 |
| 文案里有单引号，直接写 | 要转义，否则编译报错 |
| 新页面写好就能跳转过去 | Manifest 里没注册，一跳就崩溃 |
| 跑一下 spotlessApply、ktlintFormat 整理格式 | 会改整个仓库；只跑检查任务，只看改动的文件 |
| 装不上，先卸载再装 | 卸载会清掉 App 数据，先问用户 |
| 签名报错，改一下 signingConfigs | 签名是环境问题，告诉用户处理，不改项目的签名配置 |
