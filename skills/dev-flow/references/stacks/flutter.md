# Flutter 技术栈指南

适用于 Flutter 工程的 Dart 部分（`lib/`、`test/`、`integration_test/`、`pubspec.yaml`）。原生部分（`ios/`、`android/`）读 [ios.md](ios.md) 和 [android.md](android.md) 里「宿主是 Flutter 或 React Native 时」那一节。

这里只写所有 Flutter 项目都成立的做法和坑。项目特有的写进项目档案。

下文的 `flutter`、`dart` 命令，用 fvm 管理版本的项目一律换成 `fvm flutter`、`fvm dart`。

## 生成档案时怎么推断

| 要确定什么 | 怎么查 |
|---|---|
| 项目要求的 Flutter 版本 | `.fvmrc`（fvm；旧版 fvm 是 `.fvm/fvm_config.json`），或者 `.tool-versions`（asdf）、CI 配置里写的版本。`pubspec.yaml` 的 `environment` 和 `pubspec.lock` 末尾的 `sdks:` 只是最低要求 |
| 怎么调用 flutter | 有 `.fvmrc` 的用 `fvm flutter ...`，也可以直接用 `.fvm/flutter_sdk/bin/flutter`（`.fvm/` 一般不进 git，本机用 fvm 装过才有）。`command -v fvm` 找不到，或者 fvm 提示要下载这个版本：环境问题，告诉用户 |
| 本机的版本 | `dart --version`。Flutter 版本读 SDK 目录（flutter 命令所在 bin 目录的上一级）下的 `bin/cache/flutter.version.json`，较老的版本是 SDK 根目录的 `version` 文件。不要用 `flutter --version`，见「常见坑」 |
| 工程类型 | `.metadata` 里的 `project_type`。module 是嵌进原生 App 的模块，它的 `.android/`、`.ios/` 是工具生成的，不要改；plugin、package 一般在 `example/` 里跑起来验证 |
| 是不是多包仓库 | 根目录 `pubspec.yaml` 有 `workspace:`（pub workspace：依赖在根目录装，只有一份 pubspec.lock），或者用了 melos（有 `melos.yaml`，或根目录 pubspec 里有 `melos:`）。档案里写清各个包在哪、命令在哪个目录跑 |
| 运行和构建参数 | README、`scripts/`、Makefile、CI 配置、`.vscode/launch.json`、`.idea/runConfigurations/` 里实际用的 `flutter run`、`flutter build` 命令：`--flavor`、`-t <入口文件>`、`--dart-define`、`--dart-define-from-file`，以及其他参数（比如 `--no-tree-shake-icons`）。原样记进档案 |
| 多渠道（flavor） | `ios/` 的 scheme、`android/` 的 productFlavors，以及 `main_<渠道>.dart` 这类入口文件 |
| 生成代码 | dev_dependencies 里有 build_runner（常配合 freezed、json_serializable），说明改了带注解的源文件要重新生成。build_runner 的版本看 `pubspec.lock`。生成的 `*.g.dart`、`*.freezed.dart` 有没有提交进 git：`git ls-files '*.g.dart'` |
| 多语言 | 有 `l10n.yaml` 或 `lib/l10n/*.arb` 的是官方的 gen-l10n；也可能是 easy_localization（翻译放在 assets 下的 JSON 等文件里）、GetX 的 `Translations` 类等其他方案 |
| 代码规范 | `analysis_options.yaml`。配了 custom_lint（比如 riverpod_lint）的，这些规则 analyze 不报，要另跑 `dart run custom_lint`。项目要不要求 dart format，看 CI 里有没有 `dart format --set-exit-if-changed` |
| 测试 | `test/` 下的单元测试和 widget 测试，`integration_test/` 下的集成测试。搜 `matchesGoldenFile` 看有没有 golden 测试（和基准截图比对） |

## 验证

- **依赖**：自己改了依赖，执行 `flutter pub get`。没改依赖、只是为了跑起来装依赖，用 `flutter pub get --enforce-lockfile`：lock 对不上就直接失败，不会悄悄改 `pubspec.lock`。`flutter analyze`、`test`、`build`、`run` 默认也会先自动跑一次 pub get，加 `--no-pub` 跳过（前提是依赖已经装好）。
- **静态检查**：`flutter analyze <改动的文件或目录>`。几秒到一分钟，每改完一块就跑。
  - 默认 error、warning、info 任何一条都会让退出码变成 1（`--fatal-infos`、`--fatal-warnings` 默认打开）。
  - 老项目常常本来就有一堆 info，退出码总是 1。这时动手前先对要改的文件跑一次，把结果记下来，改完对比，确认没有新增 error、warning。不要用 `git stash` 临时撤掉改动来对比，会把别的会话的改动一起收走。
- **格式**：项目要求 dart format 的（CI 里有检查，或者挑两个没改过的文件跑下面的命令都是干净的），改完只检查自己改的文件：`dart format --output=none --set-exit-if-changed <文件>`，退出码 1 表示要重排。要修就只对这些文件跑不带参数的 `dart format <文件>`。
  - 没改过的文件也报 `Changed`：项目没统一格式化，不要格式化，免得整个文件重排、混进大量无关改动。
  - 不带 `--output=none` 的 `dart format` 和 `dart fix --apply` 都会直接改文件。只想看 dart fix 的建议用 `dart fix --dry-run`。
- **测试**：`flutter test <测试文件>`，平时只跑相关的，阶段末尾再跑全量。
  - 按用例名筛选用 `--plain-name '<名字>'`。输出写进文件时加 `-r expanded`：默认的 compact 在一行里反复刷新，写进文件很难读。
  - 退出码 0、末尾是 `All tests passed!` 才算通过；失败时末尾是 `Some tests failed.`。
  - `integration_test/` 下的测试要连着设备跑：`flutter test integration_test/<文件> -d <设备>`，耗时按整包构建算。
  - golden 测试失败时，差异图在测试文件旁边的 `failures/` 目录。基准图和生成它的平台有关（常在 CI 的 Linux 上生成），本机对不上，先确认是不是平台差异。
  - 已有的测试照常跑，不往项目里加新的测试文件来验证。纯 Dart 的逻辑（模型、解析、工具函数）要单独验证的，在测试结果目录写一次性脚本，用 `dart run --packages=<项目>/.dart_tool/package_config.json <脚本>` 跑。脚本不能引用 Flutter 的包，纯 Dart 环境里没有 `dart:ui`。
- **构建**：`flutter build apk --debug`；iOS 用 `flutter build ios --debug --no-codesign`。`flutter build` 不写模式时默认是 release，慢很多。加上档案里记的 `--flavor`、`-t`、`--dart-define` 等参数。比较慢，放到阶段末尾。
  - 测试用的是 release 包的，阶段末尾再按 release 构建一次：release 和 profile 才做 AOT 编译和图标字体裁剪，有些错误只在这时出现。
  - 同一个项目同一时间只跑一个构建，`build/`、`.dart_tool/` 是共用的。
  - iOS 这一侧的注意事项（会顺带 pod install 等）见 ios.md。
- **判断成败、找错误**：输出重定向到文件再看退出码：`<命令> > /tmp/<项目>-<步骤>.log 2>&1; echo "exit=$?"`。别接 `| tail`，退出码会变成 tail 的。

  ```sh
  grep -nE '^ *(error|warning) (•|-) |\.dart:[0-9]+:[0-9]+: Error: |\(Xcode\): ' /tmp/<项目>-<步骤>.log | head -50
  ```

  三种分别是：analyze 的 error、warning 行（`flutter analyze` 用 `•` 分隔，`dart analyze` 用 `-`）；构建、运行、测试时的 Dart 编译错误；`flutter build ios` 整理出来的 Xcode 报错（这种只列错误）。Android 原生部分的报错按 android.md 的找法。还找不到的，看日志末尾 Flutter 的失败总结，比如 `Gradle task assembleDebug failed with exit code 1`、`Failed to build iOS app`、`Target kernel_snapshot_program failed`。

## 界面验证与日志

- **装到设备上**：`flutter devices` 看设备 ID。两种方式：
  - `flutter run -d <设备> <档案里的参数>`：构建、安装、启动一步完成，Dart 的日志直接在输出里。它会一直挂着：放后台跑，输出重定向到文件，看到 `Flutter run key commands.` 就是已经启动了，测完结束这个进程。不写模式是 debug。
  - 先 `flutter build`，再按 ios.md、android.md 的方法安装、启动、抓日志。Android 包在 `build/app/outputs/flutter-apk/app-<渠道>-<模式>.apk`（没有渠道就没有中间那段），iOS 真机包在 `build/ios/iphoneos/` 下。iOS 真机上脱离 `flutter run` 单独启动的，只能是 profile 或 release 包。
- **构建模式**：debug 包慢，右上角有 DEBUG 角标（项目没关掉的话），截图对照设计时别当成差异。看流畅度、启动速度要用 profile 或 release。`assert` 和 `if (kDebugMode)` 里的代码，profile、release 包里不执行。
- **热重载**：不会重新执行 `main()` 和已经执行过的 `initState`，全局变量、静态变量也不会重新初始化；改了原生代码或加了带原生代码的插件，要停掉重新运行。只有 debug 模式能热重载。后台跑的 `flutter run` 按不了 r：启动时加 `--pid-file <文件>`，`kill -USR1 $(cat <文件>)` 热重载，`kill -USR2` 热重启。最终验证以重新启动的 App 为准。
- **截图**：`flutter screenshot -d <设备> -o <测试结果目录>/<文件>.png`。不加 `-o` 会在当前目录（也就是仓库里）生成 `flutter_01.png`。支持 Android 设备和模拟器、iOS 模拟器；iOS 17 以上的真机不支持，用 ios.md 里的方法。
- **点按**：flutter 命令行没有点按功能，用 ios.md、android.md 里的工具。Flutter 整个界面画在一个原生视图里，这些工具读到的控件树里只有带文字或 Semantics 标签的组件（Key 看不到）；找不到元素就按截图上的坐标点。
- **界面上的报错**：组件构建出错时，debug 包显示红底报错，profile、release 包里只是一块灰色。debug 包的运行日志里搜 `EXCEPTION CAUGHT BY` 和 `Another exception was thrown`，比如布局溢出会报 `A RenderFlex overflowed by`。截图看着正常也要搜一遍。
- **日志**：
  - `print`、`debugPrint` 在 release 包里也会输出。Android 在 logcat 里的标签是 `flutter`（`adb logcat -s flutter`）；iOS 在设备日志里以 `flutter:` 开头，抓法见 ios.md。iOS 真机跑 release 包时，`flutter run` 的输出里可能没有这些日志，以设备日志为准。
  - `dart:developer` 的 `log()` 只在 DevTools 和 IDE 的调试控制台里看得到，`flutter run` 的输出和设备日志里都没有。功能日志不能只走它。
  - `flutter logs` 在 Android 上只显示 `flutter` 标签和崩溃信息，原生日志要直接用 adb logcat。
  - `flutter devices`、`run`、`install`、`screenshot`、`logs` 查找 iOS 真机时，内部会调用 `xcrun devicectl`：ios.md 里「抓日志期间别跑 devicectl」同样适用。

## 常见坑

- **依赖**：改了 `pubspec.yaml` 要执行 `flutter pub get`。App 项目要把 `pubspec.lock` 的变化一起提交（库项目一般不提交 lock）。
- **生成代码**：改了带 `@freezed`、`@JsonSerializable` 这类注解的类，要执行 `dart run build_runner build --delete-conflicting-outputs`，看退出码。
  - 这个参数是给 2.7 以前的 build_runner 的：不加的话，生成文件已经存在时会停下来问要不要删，非交互环境里直接失败。2.7 起这个参数被忽略，打一行警告也不影响。
  - 跑完看 `git status`：只该有自己改的源文件对应的生成文件变化。冒出一大批无关的生成文件变化，多半是生成器或 Dart 版本和上次生成时不同，不要一起提交，告诉用户。
  - 不要手改 `*.g.dart`、`*.freezed.dart` 这些生成出来的文件。
- **多语言**（gen-l10n）：改了 arb 文件，执行 `flutter gen-l10n`（pubspec 里有 `generate: true`、又有 `l10n.yaml` 的，pub get 和构建时也会自动生成）。arb 是 JSON，改完用 `python3 -m json.tool <文件> > /dev/null` 查格式。每种语言都要加上新 key：漏了不报错，只打一行 `"<语言>": N untranslated message(s).`，界面上显示的是模板语言的文案。
- **平台通道**：Dart 端的通道名、方法名、参数名，必须和原生端完全一致。改了一边就要改另一边，否则编译时发现不了，要到运行时才报错（`MissingPluginException`，或者拿到 null）。用 Pigeon 生成通道代码的（dev_dependencies 里有 pigeon），改定义文件后重新生成两端的代码。
- **`--dart-define`**：项目靠它区分环境的，build、run、test 都要带同样的参数。漏了不报错，`String.fromEnvironment` 拿到的是默认值。
- **异步之后用 context**：`await` 之后再用 `BuildContext`，先检查 `mounted`，否则页面已经关掉时会出错。
- **flutter 命令本身**：
  - `flutter --version`、`flutter doctor` 会先联网拉取版本标签，网络不通时要卡一分多钟。其他命令每 3 天也会联网检查一次更新，加 `--no-version-check` 可以跳过。
  - 输出 `Waiting for another flutter command to release the startup lock...`：是别的 flutter 命令还没结束（常见是另一个会话卡在联网），等它结束。
- **环境问题**：下面这些都停下来告诉用户，不要改 `.fvmrc`、`pubspec.yaml`、`pubspec.lock` 或签名配置去迁就本机：
  - 本机 Flutter 版本和项目要求的不一致，fvm 没装，或者 fvm 提示要下载这个版本。
  - `pub get` 报 Dart SDK 版本不满足。
  - 没改依赖，`pubspec.lock` 却变了：Flutter SDK 钉死了 collection、meta 这些包的版本，换个 Flutter 版本 lock 就会变。把 lock 还原，不要提交。
  - `pub get` 连不上网。国内一般要配镜像（`PUB_HOSTED_URL`、`FLUTTER_STORAGE_BASE_URL`）或代理。
  - 签名报错，按 ios.md、android.md 的处理。
  - 另外要知道：用哪个 Flutter 跑过 pub get 或构建，`ios/Flutter/Generated.xcconfig`（模板默认不进 git）里的 `FLUTTER_ROOT` 就可能改成指向它，之后直接用 xcodebuild 编译也会用这个 Flutter。

## 危险信号

| 冒出这个念头 | 实际情况 |
|---|---|
| 改了 Dart 端的通道调用就行 | 原生端的接收也要改，或者先确认它已经支持 |
| 直接改生成的文件最快 | 下次一生成就被覆盖 |
| 热重载后看着没问题 | 初始化逻辑、全局和静态变量、原生代码的改动，热重载都体现不出来，重新运行再验证 |
| analyze 有几个警告，不影响运行 | 不能新增 error、warning。和改之前的结果比，新增的问题按 `analysis_options.yaml` 处理 |
| 跑一下 dart format（或 dart fix）整理格式 | 不带 `--output=none --set-exit-if-changed` 的 `dart format`、`dart fix --apply` 都会直接改文件，对目录跑就是一大片。只对改动的文件做检查 |
| 先 `flutter --version` 看一下版本 | 它会联网，网络不通卡一分多钟，还占着锁让别的 flutter 命令一起等。读 SDK 的 `bin/cache/flutter.version.json` |
| pub get 报 SDK 版本不够，改一下 environment；lock 跟着变了，一起提交 | 本机 Flutter 和项目要求的不一致，是环境问题：告诉用户，不改配置，变了的 lock 还原 |
| build_runner 跑完一堆生成文件变了，一起提交 | 只提交自己改的源文件对应的那几个。其余多半是生成器或 Dart 版本不同造成的，告诉用户 |
| 构建报怪错，先 `flutter clean` | 它会删 `build/`、`.dart_tool/`、iOS 的 `Generated.xcconfig`，还会 clean Xcode 工程：之后要重新 pub get、全量编译，别的会话正在跑的构建也会被打断。先按报错找原因 |
| golden 对不上，加 `--update-goldens` 更新一下 | 这会用现在的结果覆盖基准图，回归就成了新基准。先看 `failures/` 里的差异图，确认是有意改的界面再更新，并告诉用户 |
