# iOS 技术栈指南

适用于原生 iOS 项目（Swift / Objective-C，UIKit / SwiftUI），以及 Flutter、React Native 工程里的 `ios/` 部分。

这里只写所有 iOS 项目都成立的做法和坑。项目特有的命令、路径和坑写进项目档案。

## 生成档案时怎么推断

| 要确定什么 | 怎么查 |
|---|---|
| 工程入口 | 有 `.xcworkspace` 就用 `-workspace`（用了 CocoaPods 必须用它），否则用 `-project` |
| 工程是不是工具生成的 | 有 `project.yml` 是 XcodeGen；有 `Project.swift` 或 `Tuist/` 目录是 Tuist。这两种都不要手改 `.xcodeproj`，改完配置重新生成 |
| 主 App 的 scheme | `xcodebuild -list -json -workspace <W>.xcworkspace`（或 `-project`）。workspace 会把 Pods 的 scheme 也一起列出来，主 App 的一般和 target 同名 |
| 多环境、多渠道 | 看 scheme 和 configuration 的对应关系，比如 `Debug-<渠道>`。问清楚验证时用哪一个 |
| 依赖管理 | `Podfile` → CocoaPods；`Package.resolved` 或工程里有 Swift Package 引用 → SPM；`Cartfile` → Carthage |
| 最低系统版本 | `xcodebuild -showBuildSettings -workspace <W> -scheme <S> \| grep IPHONEOS_DEPLOYMENT_TARGET` |
| 有没有单测 | 找 `*Tests` target 和测试文件。有 target 但几乎没有测试的，按「没有」处理，并问用户测试策略 |
| 代码规范 | 有 `.swiftlint.yml` 用 `swiftlint lint --quiet <改动的文件>`；有 `.swiftformat` 用 `swiftformat --lint <改动的文件>`（不加 `--lint` 会直接改文件）；有 `.swift-format` 用 `xcrun swift-format lint --strict <改动的文件>`（Xcode 自带）。只查这次改动的文件 |
| 多语言文案 | `*.lproj/Localizable.strings`（旧格式）或 `Localizable.xcstrings`（String Catalog，JSON 格式） |
| 本地持久化 | Core Data（`.xcdatamodeld`）、Realm（`import RealmSwift`）、SwiftData（`@Model`）、GRDB 等 |
| 界面验证工具 | 模拟器能不能跑（见下文）；`command -v idb appium` 看本机装了哪些 |

## 编译验证

**默认命令**（不需要签名，也不需要模拟器或真机）：

```sh
xcodebuild -workspace <W>.xcworkspace -scheme <S> -configuration Debug \
  -destination 'generic/platform=iOS' CODE_SIGNING_ALLOWED=NO build \
  > /tmp/<项目>-build.log 2>&1; echo "exit=$?"
```

- **为什么用 `generic/platform=iOS`**：它不依赖具体设备，还能避开一类常见问题：有些三方 SDK（音视频、硬件厂商的 SDK 居多）缺模拟器架构，模拟器一编译就报找不到模块，其实和你的改动无关。
- **需要模拟器时**（跑测试、看界面）：用 `-destination 'platform=iOS Simulator,name=<机型>'`。可用机型看 `xcrun simctl list devices available`。
- **判断成败**：看退出码，或者日志里有没有 `** BUILD SUCCEEDED **`。别接 `| tail` 或 `| tee` 看结果，退出码会变成管道里最后一个命令的。想用 xcbeautify，就写 `set -o pipefail && xcodebuild ... | xcbeautify`。
- **找错误**：

  ```sh
  grep -nE ': (fatal )?error: |^error: |Undefined symbols|^ld: |PhaseScriptExecution failed' /tmp/<项目>-build.log | head -50
  ```

  按「位置: error:」这种固定格式匹配，免得把日志里普通的 error 字样当成错误；签名、链接、构建脚本的错误也能抓到。还是找不到的，看日志末尾「The following build commands failed:」那一段，里面写了是哪一步失败的。
- **提速**：改动只在某个有独立 scheme 的模块里（framework、SPM 包、pod）时，可以先只编这个 scheme，阶段末尾再编整个 App。首次编译和增量编译的耗时差很多，两个都记进档案。
- **同一时间只跑一个构建**：多个会话共用同一份 DerivedData，并发编译会报 `database is locked`，这不是代码的问题。确实要并行，就给每个会话单独指定 `-derivedDataPath`，代价是第一次要全量编译。
- **卡在解析依赖**：SPM 要能访问依赖所在的仓库（多数在 GitHub），卡在 Resolving Package Graph 时先查网络和代理。`-list`、`-showBuildSettings` 也会先解析依赖，一样会卡。依赖没变时加 `-onlyUsePackageVersionsFromResolvedFile -skipPackageUpdates`：只用 Package.resolved 里记的版本，也不去远端检查更新。
- **报 `Could not resolve package dependencies:`，后面却是空的**：真正的原因在 DerivedData 里这个工程的 `Logs/Package/*.xcactivitylog`。文件是 gzip 压缩的，用 `gunzip -c` 解开看，多半是连不上 GitHub。
- **workspace 和 xcodeproj 里各有一份 Package.resolved**，记的版本可能不一样。编译入口（`-workspace` 还是 `-project`）要和平时开发用的一致，不然拉到的依赖版本不一样。
- **报了没改过的文件的错**：先确认是不是别的会话正在改那个文件，或者 DerivedData 里还有旧产物，别急着去改它。

## 新建文件、改资源后要做的事

- **把新文件登记进工程**：
  - Xcode 16 开始有同步文件夹（pbxproj 里是 `PBXFileSystemSynchronizedRootGroup`），文件放进目录就自动参与编译。
  - 传统分组（`PBXGroup`）必须把文件登记进 target。判断方法：在 `project.pbxproj` 里搜同目录下某个已有的文件名，能搜到就是传统分组。
  - 登记时用 `xcodeproj` 这个 ruby gem（CocoaPods 依赖它）。先试 `ruby -e 'require "xcodeproj"'`，报找不到的，多半是 Homebrew 装的 CocoaPods，gem 在它自己的目录里，命令前加 `GEM_HOME="$(brew --prefix cocoapods)/libexec"`。**不要手改 pbxproj 文本**，很容易把工程改坏。

    ```sh
    ruby -e 'require "xcodeproj"; p = Xcodeproj::Project.open("<X>.xcodeproj"); t = p.targets.find { |x| x.name == "<Target>" }; g = p.main_group.find_subpath("<组路径>", true); t.add_file_references([g.new_reference("<文件名>.swift")]); p.save'
    ```

  - Xcode 正开着这个工程时，外部改过的 pbxproj 有时不会被加载。改完提醒用户重新打开工程。
  - XcodeGen 或 Tuist 生成的工程，改完重新生成即可，不碰 `.xcodeproj`。
- **CocoaPods 本地开发 pod**（Podfile 里写 `:path =>` 的那种）：增删文件后要执行 `pod install`，否则新文件不参与编译，会报 `cannot find ... in scope`。
- **图片**：放进 `.xcassets`。矢量图用 PDF 或 SVG，并勾选 Preserve Vector Data。SF Symbols 的每个图标都有最低系统版本要求，要对照项目的最低版本。
- **权限**：新用到相机、定位、蓝牙、麦克风、相册、健康、运动这类能力时，必须在 Info.plist 里加对应的 `NS...UsageDescription`，否则一调用就崩溃。后台模式、HealthKit 等还要打开对应的 Capability。
- **多语言文案**：每种语言的文件都要加上新 key，key 的写法和项目现有的保持一致。改完做格式检查：
  - `.strings`：`plutil -lint <文件>`
  - `.xcstrings`：`python3 -m json.tool <文件> > /dev/null`（plutil 校验不了 JSON）

## 持久化模型变更

改了存储模型的字段，一定要做迁移，否则老用户升级后一启动就崩溃，或者数据丢失：
- **Core Data**：新建一个模型版本。能自动迁移的，用轻量迁移。
- **Realm**：升 `schemaVersion`。改字段名、改类型时要写迁移代码。
- **SwiftData**：用 `VersionedSchema` 加 `SchemaMigrationPlan`。

验证时要**先装旧版本，再覆盖安装新版本**。只测全新安装发现不了这类问题。

## 界面验证与自动化

- **模拟器**：
  - 截图：`xcrun simctl io booted screenshot <文件>.png`
  - 点按：装了 idb 就用 idb；项目已有 UI 测试的照常跑，不为验证新写 XCUITest。
- **真机**：
  - 列出设备：`xcrun devicectl list devices`
  - 截图：`xcrun devicectl device capture screenshot --device <UDID> --destination <文件>.png`（Xcode 27 起才有；更早的版本用 WebDriverAgent 或 Appium 截图）
  - 签名：装到真机的包要签名，签名方式按档案。报签名错误（没登录 Apple 账号、找不到证书、描述文件里没有这台设备）是环境问题，停下来告诉用户，不要自己改工程的签名设置（Team、Bundle ID、证书）。
  - 安装：`xcrun devicectl device install app --device <UDID> <App 路径>`。装大包尽量插数据线：走 Wi-Fi 装几百 MB 的包要十几二十分钟，插线只要一两分钟。
  - 启动：`xcrun devicectl device process launch --device <UDID> <bundleId>`。真机要解锁才能启动 App，锁着会报 Locked。自动化测试前请用户把自动锁定设成永不。
  - 点按：需要 WebDriverAgent 或 Appium。都没搭好时，先问用户要不要搭，搭好一次以后都能自动测；用户暂时不搭，才请用户按步骤操作，你负责截图和判断
- **模拟器跑不了的情况**：有的 SDK 只提供真机架构，或者缺某种模拟器架构，这时只能用真机。在档案里写明。

## 日志

- 沿用项目已有的日志方式（系统的 `Logger`、CocoaLumberjack，或者项目自己的封装），不要新加 `print`。
- 功能日志的级别要保证测试用的包里看得到：Release 包里 debug 级别的日志常被去掉。
- `#if DEBUG` 里的代码在 Release 包里不会执行。先确认测试用的是哪个 configuration。
- **真机日志**：`xcrun devicectl device process launch --device <UDID> --console --environment-variables '{"OS_ACTIVITY_DT_MODE":"YES"}' <bundleId>`。这样 os_log 一类的日志也会输出到控制台。App 被杀掉后，要重新用这条命令启动。
  - 要看 debug 级别的日志，环境变量再加 `"OS_ACTIVITY_MODE": "debug"`，Release 包里的 debug 日志也能看到。
  - Ad Hoc、TestFlight 这类不可调试的包，`--console` 里没有 App 的日志：插 USB 用 `idevicesyslog -u <UDID> --no-colors -p <进程名> -o <文件>`（libimobiledevice，brew 安装）。抓日志期间别跑任何 devicectl 命令，跑过之后就收不到 App 的日志了。
  - `--console` 输出的换行是 `\r\n`，grep 用 `$` 匹配行尾会漏掉，先 `tr -d '\r'`。
- **模拟器日志**：`xcrun simctl spawn booted log stream --predicate 'process == "<进程名>"'`

## UIKit 常见坑

- 更新界面必须在主线程。
- iOS 13 开始支持多 scene，`UIApplication.shared.keyWindow` 已经废弃，可能返回 nil，依赖它弹出的视图会不声不响地不显示。要从 `connectedScenes` 里取当前窗口。
- 子视图超出父视图 bounds 的部分看得见，但点不到，因为 hitTest 不会把触摸分发过去。要让父视图的范围把子视图包住。
- 父视图上的点击手势会抢走子视图的点击，但 UIButton、UISwitch、UISegmentedControl 这些标准控件的单击不受影响，系统会优先交给控件。被抢的多是自定义的 UIControl 子类或自己处理触摸的视图：先设 `cancelsTouchesInView = false`；控件还是收不到点击的话，再把 `delaysTouchesEnded` 关掉。
- 判断页面是否被 pop，别用 `viewWillDisappear` 加 `isMovingFromParent`：侧滑返回刚开始它就是 true，用户中途取消了也会误触发。一次性的收尾动作放进 `didMove(toParent:)`，判断 parent 为 nil，再加一个防止重复执行的标志。

## 宿主是 Flutter 或 React Native 时

这一节只讲原生这一侧要注意的。改的是 Dart 代码时，读 [flutter.md](flutter.md)。

- 先弄清改动边界：只能改原生部分，还是 Dart 或 JS 也能改。写进档案。
- 不要手改生成出来的文件：Flutter 的 `ios/Flutter/Generated.xcconfig`、`GeneratedPluginRegistrant.*`，以及 React Native codegen 的产物。
- 直接用 xcodebuild 编译之前，要先执行 `flutter pub get`（或 `npm install`）和 `pod install`，把配置生成好。`flutter build ios --no-codesign` 会顺带执行 pod install，可能覆盖手工放进 Pods 的东西。
- iOS 14 以上，Flutter 的 Debug 包离开调试器就没法在真机上单独启动。真机测试要装 Release 或 Profile 包。
- 多渠道（flavor）一般对应单独的 scheme，以及 `Debug-<渠道>` 这类 configuration。

## 危险信号

| 冒出这个念头 | 实际情况 |
|---|---|
| 模拟器编译不过，说明代码有问题 | 先看是不是三方 SDK 缺模拟器架构，换成 `generic/platform=iOS` 再编 |
| 编译命令后面接个 `\| tail` 看结果 | 退出码被管道吞掉了。输出重定向到文件，再看退出码 |
| 文件建好了自然会参与编译 | 先看它在同步文件夹、传统分组、开发 pod 还是生成的工程里 |
| 模型只是加了个字段 | 持久化模型的任何改动都要迁移 |
| 直接调用定位、蓝牙就行 | Info.plist 缺用途说明，一调用就崩溃 |
| 全新安装测过了，没问题 | 覆盖安装的升级路径还没测 |
| 手改一下 pbxproj 最快 | 很容易把工程改坏，用 xcodeproj gem |
| 签名报错，改一下 Team 或 Bundle ID 就好 | 签名是环境问题，告诉用户处理，不改工程的签名设置 |
| 跑一下 swiftformat 整理格式 | 不加 `--lint` 会改整个仓库；只对改动的文件跑 `--lint` |
