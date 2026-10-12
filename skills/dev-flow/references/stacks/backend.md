# 后端技术栈指南

适用于服务端应用和接口，比如 Go、Python、Node.js、Java / Kotlin、Rust。

这里只写所有后端项目都成立的做法和坑。项目特有的命令、端口、环境和坑写进项目档案。

## 生成档案时怎么推断

| 要确定什么 | 怎么查 |
|---|---|
| 语言和包管理 | `go.mod` → Go；`Cargo.toml` → Rust；`pom.xml` → Maven，`build.gradle(.kts)` → Gradle；`package.json` → Node.js，包管理器看锁文件（见 [web.md](web.md)）；Python 看锁文件：`uv.lock` → uv，`poetry.lock` → Poetry，只有 `requirements*.txt` 的用 pip 加虚拟环境 |
| 命令从哪来 | 先看 Makefile、justfile、Taskfile、package.json 的 scripts 和 CI 配置，项目包好的命令（比如 `make test`）优先。有 `./gradlew`、`./mvnw` 就用它们，不用全局装的 gradle、mvn。Python 命令要在项目环境里跑：`uv run <命令>`、`poetry run <命令>`，或者 `.venv/bin/<命令>` |
| 运行时版本 | `go.mod` 的 `go` 和 `toolchain` 行、`.python-version`、`.nvmrc` 或 `package.json` 的 `engines`、`rust-toolchain.toml`、`.tool-versions`；Java 看构建文件里的 toolchain 或 Java 版本配置。和本机 `--version` 对不上的，按下文「配置和环境」处理 |
| 快速检查 | Go：`go build -o /dev/null ./... && go vet ./...`（vet 会连测试文件一起检查；不加 `-o /dev/null` 时，如果只匹配到一个 main 包，会在当前目录留下一个可执行文件）；Rust：`cargo check`（加 `--all-targets` 连测试一起查）；Python：看 pyproject 和 CI，一般是 `ruff check --no-fix` 加 mypy 或 pyright；TypeScript：`<包管理器> exec tsc --noEmit`；Java / Kotlin：`./gradlew compileJava` 或 `compileKotlin`（连测试代码一起用 `testClasses`），Maven 用 `./mvnw -q test-compile` |
| 代码规范 | 只用检查模式，只查改动的文件，见下文「代码规范」 |
| 测试 | `go test ./...`、`pytest`、`cargo test`、vitest 或 jest、`./gradlew test` 或 `./mvnw test`。弄清楚哪些测试需要数据库等外部依赖，常见的放法：Go 的 build tag（`//go:build integration`，跑时加 `-tags integration`）、pytest 的 marker（`-m`）、Maven Failsafe 的 `*IT` 类（`mvn verify` 才跑，`mvn test` 不跑）、Gradle 单独的任务。用了 Testcontainers 的，要本机 Docker 在跑 |
| 本地依赖服务 | `compose.yaml` 或 `docker-compose.yml` 里需要哪些服务（数据库、缓存、消息队列），映射到本机哪个端口 |
| 数据库迁移 | `migrations/` 目录，或 Alembic、Django、Flyway、Liquibase、Prisma、golang-migrate 等工具。搞清楚怎么新建、怎么执行、怎么回滚，以及服务启动时会不会自动执行迁移 |
| 接口契约 | OpenAPI / Swagger、protobuf、GraphQL schema，以及有没有据此生成代码、生成命令在哪（Makefile、`//go:generate`、scripts） |
| 配置 | `.env.example` 和配置文件；本地运行读哪个文件、哪个 profile，数据库等连接默认指向哪里；密钥从哪里来 |
| 本地运行 | 启动命令、端口、健康检查接口（Spring Boot 装了 actuator 的是 `/actuator/health`）、日志打到终端还是文件 |

## 验证

- 快速检查和相关单测每改完一块就跑。需要数据库的集成测试放到阶段末尾。JVM、Rust 这类首次构建和增量构建耗时差很多，两个都记进档案。
- **跑单个测试**：
  - Go：`go test ./<包目录> -run '^TestXxx$'`
  - pytest：`pytest <文件>::<测试名>`，或者 `-k <关键字>`
  - Rust：`cargo test <测试名的一部分>`
  - jest：`<包管理器> exec jest <文件> -t '<测试名>'`；vitest：`<包管理器> exec vitest run <文件> -t '<测试名>'`（`run` 保证跑完就退出，不进监听模式）
  - Gradle：`./gradlew :<模块>:test --tests '<类名>.<方法名>'`
  - Maven：`./mvnw -pl <模块> test -Dtest='<类名>#<方法名>'`。加了 `-am` 的，再加 `-Dsurefire.failIfNoSpecifiedTests=false`，否则没有匹配测试的模块会让构建失败
- **结果被缓存时没有真跑**：Go 测试显示 `(cached)`，Gradle 的 test 任务显示 `UP-TO-DATE`。依赖数据库等外部状态的测试，Go 加 `-count=1`，Gradle 加 `--rerun`（7.6 起有；更早的用 `cleanTest test`）。
- **跑测试前确认它连的库是本地的**：很多集成测试会清表，或者建库删库。
- **判断成败**：输出重定向到文件，再看退出码：

  ```sh
  <命令> > /tmp/<项目>-<步骤>.log 2>&1; echo "exit=$?"
  ```

  别接 `| tail` 或 `| tee` 看结果，退出码会变成管道里最后一个命令的。jest 把结果写在 stderr，`2>&1` 不能省。
- **找错误**：按各工具固定的格式匹配，免得把日志里普通的 error 字样当成错误：

  ```sh
  # Go：编译和 vet 的报错（文件:行:列）、测试失败、panic、数据竞争
  grep -nE '\.go:[0-9]+(:[0-9]+)?: |--- FAIL|^FAIL|^panic: |DATA RACE' <日志>
  # Rust：编译错误（带后两行，里面有位置）、测试失败、panic
  grep -nE -A 2 '^error(\[E[0-9]+\])?: |FAILED|panicked at' <日志>
  # Python：pytest 的失败汇总和断言行、mypy 和 pyright 的报错、末尾的统计
  grep -nE '^(FAILED|ERROR) |^E  |: error: | - error: |[1-9][0-9]* (failed|error)' <日志>
  # Node.js：tsc 报错、jest 和 vitest 的失败
  grep -nE 'error TS[0-9]+|FAIL |● |[1-9][0-9]* failed' <日志>
  # Gradle：Kotlin 和 javac 的报错、失败的任务和测试
  grep -nE '^e: |\.java:[0-9]+: |FAILED$|^FAILURE: |What went wrong' <日志>
  # Maven
  grep -nE '^\[ERROR\]|BUILD FAILURE' <日志>
  ```

  - javac 的报错会跟着系统语言变成中文（比如 `错误: 找不到符号`），所以按「文件.java:行号: 」匹配，不按 error 这个词。
  - tsc 输出到文件时，是一行一条的 `文件(行,列): error TS…`。环境里设了 `FORCE_COLOR` 时会夹带颜色控制符，匹配不上，加 `--pretty false`。
  - 还是找不到的，看日志末尾（`tail -n 80 <日志>`），各工具的汇总都在最后。JVM 项目的测试详情看报告：Gradle 在各模块的 `build/reports/tests/test/index.html`，Maven 在各模块的 `target/surefire-reports/`。

## 代码规范

只用检查模式，只查这次改动的文件。右列的写法会直接改文件，还会改到别人正在改的文件：

| 语言 | 只检查（用这个） | 会改文件（别用） |
|---|---|---|
| Go | `gofmt -l <文件>`：列出格式不对的文件，有输出就是有问题（退出码仍是 0）；`golangci-lint run --fix=false ./<目录>/...` | `gofmt -w`；`golangci-lint run --fix`；不加 `--diff` 的 `golangci-lint fmt` |
| Python | `ruff check --no-fix <文件>`；`ruff format --check <文件>`；`black --check <文件>` | `ruff check --fix`；不加 `--check` 的 `ruff format`、`black` |
| Node.js | `prettier --check <文件>`；`eslint <文件>`；`biome check <文件>` | `prettier --write`；`eslint --fix`；`biome check --write` |
| Rust | `cargo fmt --check`；`cargo clippy` | 不加 `--check` 的 `cargo fmt`；`cargo clippy --fix` |
| Java / Kotlin | `./gradlew spotlessCheck` 或 `./mvnw spotless:check`；`./gradlew ktlintCheck` | `spotlessApply`、`spotless:apply`；`ktlintFormat` |

- 配置文件也能打开自动修复：ruff 的 `fix = true`、golangci-lint 的 `issues.fix: true`。所以检查命令里的 `--no-fix`、`--fix=false` 不能省。
- 项目包好的 lint 命令（`make lint`、`npm run lint`），先看里面有没有 `--fix`、`--write`、`-w`，有就别直接跑。
- Go 的 vet、golangci-lint 按包跑（`./<目录>/...`），不按单个文件：只给部分文件时，同一个包里别的文件的定义找不到，会报 undefined。
- `cargo fmt --check`、Spotless、ktlint 查的是整个项目。输出里只看改动的文件，别人留下的老问题不在这次范围内。

## 起服务、调接口

接口改动要在本地把服务跑起来，用 curl 实际调一遍，请求、状态码和响应的关键部分记进测试结果文档。

**先确认会连到哪里**：
- 启动前看服务实际会连哪个数据库、缓存和第三方服务：`.env`、配置文件、profile，以及 shell 里已经设置的环境变量。dotenv（Python、Node）、Node 的 `--env-file`、docker compose 替换 `${变量}` 时，默认都是 shell 里已有的变量优先，`.env` 里的不会覆盖它。
- 连的不是本机或本地容器的，先问用户。打印连接串时把密码遮掉。

**起本地依赖**：
- 先 `docker compose ps` 看哪些已经在跑，在跑的直接用。
- 只起需要的服务：`docker compose up -d --wait <服务>`（`--wait` 等服务进入 running 或 healthy 状态才返回）。记下哪些是你起的。
- 端口冲突（`port is already allocated`、`address already in use`）：查是谁占的，不动别人的进程，问用户。

**起服务**：
- 先查端口有没有人在听：`lsof -nP -iTCP:<端口> -sTCP:LISTEN`（Linux 也可以用 `ss -ltnp`）。已经有进程在听的，它可能是改动前的旧版本，也可能是用户自己起的：不拿它验证，也不杀它，换个端口或问用户。
- 在后台启动，日志写进文件：`<启动命令> > /tmp/<项目>-server.log 2>&1 &`。带热重载、监听文件变化的启动命令不会自己退出，一定要放后台。
- 等它就绪：

  ```sh
  curl -sS --retry 60 --retry-delay 1 --retry-connrefused -o /dev/null -w '%{http_code}\n' http://localhost:<端口>/<健康检查路径>
  ```

  连接被拒、503 这类会每秒重试一次，最多等约一分钟，最后打印状态码。`000` 是一直没连上，去看日志；`404` 说明服务起来了，只是没有这个路径。
- 记下真正在听端口的 PID：`lsof -nP -iTCP:<端口> -sTCP:LISTEN -t`。`$!` 拿到的往往只是外层进程（`go run`、`npm run`、`./mvnw spring-boot:run`、`./gradlew bootRun` 都是这样），真正的服务是另一个进程；只杀外层，有的服务还会继续占着端口（`go run` 就是这样）。
- 改了代码要重启服务（或者确认热重载已经生效）再调。

**调接口**：
- 看状态码和响应体：`curl -sS -o /tmp/<项目>-resp.json -w '%{http_code}\n' http://localhost:<端口>/<路径>`；要看响应头用 `curl -sS -i`。
- 发 JSON：加 `-H 'Content-Type: application/json' -d '<JSON>'`（带 `-d` 默认就是 POST，别的方法用 `-X PUT` 这类）；请求体长的放进文件，用 `-d @<文件>`。
- 不加 `-f`：加了以后，4xx、5xx 时 curl 直接以退出码 22 结束，响应体也看不到，而错误响应往往正是要验证的。
- 检查 JSON：`jq . <文件>` 看全貌；`jq -e '<条件>' <文件>` 当断言用，条件为 false 或 null 时退出码是 1。没装 jq 的，用 `python3 -m json.tool <文件>`。
- 要令牌的接口，令牌放环境变量：`-H "Authorization: Bearer $TOKEN"`。贴进测试结果文档时打码。
- gRPC 接口用 grpcurl：服务开了反射，`grpcurl -plaintext localhost:<端口> list` 就能看到有哪些服务；没开的，要加 `-import-path <目录> -proto <文件>`。

**查日志和数据**：
- 每调一次，看服务日志里这次请求前后的几行。搜报错：

  ```sh
  grep -nE 'Traceback|panic:|panicked at|Exception|Caused by:|Error:|ERROR|FATAL|level=error|"level":"error"' /tmp/<项目>-server.log
  ```

  项目日志格式不一样的，把匹配方式写进档案。看不到自己打的功能日志，先查日志级别配置。
- 写操作（增、删、改）不能只看响应，要去库里查数据是不是真的变了。只用只读查询，比如 `docker compose exec -T <数据库服务> psql -U <用户> -d <库> -c '<select 语句>'`，或者 `sqlite3`、`mysql`、`redis-cli`。密码不写在命令里打出来。

**收尾**：
- 只停自己启动的：服务用记下的 PID（`kill <PID>`），停完再用 lsof 确认端口空了；依赖服务用 `docker compose stop <服务>`。不用 `docker compose down`：它会把整个项目的容器都删掉，包括用户自己起的；加 `-v` 还会删数据卷，本地库的数据跟着没了。
- 验证用的 curl 序列、小客户端脚本是一次性的，放测试结果目录（见 [test-report.md](../test-report.md)），不进仓库。不为了验证往项目里加测试文件；已有的自动化测试照常跑。

## 数据库迁移

- **先确认连的是哪个库**：执行迁移命令前，看它读的是哪个连接配置。只对本机或本地容器里的库执行；共享的开发库、测试库、预发库、生产库一律不跑，需要的话告诉用户。
- **启动服务也可能在迁移**：Spring Boot 带 Flyway 或 Liquibase 时，一启动就自动执行迁移；有的项目在启动代码里调迁移。Hibernate 的 `ddl-auto` 是 `create` 或 `create-drop` 时，启动会删表重建。所以连着共享库启动服务，也可能等于在那个库上跑迁移。
- **已经提交、合并或在别的环境执行过的迁移文件不能改**，只能新增迁移来修正。Flyway、Liquibase 会核对已执行迁移的校验和，文件改了，迁移和启动都会报错。只在自己本地库跑过、还没提交的，先回滚再改。
- **要考虑已有数据**：新增非空字段要给默认值或回填。改表名、字段名时，自动生成的迁移常常变成「删掉旧的、加一个新的」，数据会丢（Alembic 的自动生成就识别不了改名），要手工改成重命名。大表上加索引、改字段类型可能长时间锁表。
- **能写回滚的就把回滚写好**，并在本地库上验证：升级 → 回滚 → 再升级都能过，再看一眼表结构对不对。工具不支持回滚的（比如 Flyway 社区版没有 undo），就新增一个反向迁移。
- **能先看 SQL 就先看**：Alembic `upgrade <起>:<止> --sql`、Django `sqlmigrate <应用> <迁移名>`、Liquibase `update-sql`。Django 改了模型，用 `makemigrations --check --dry-run` 确认没漏迁移。
- **会清数据或绕过迁移记录的命令**（名字里带 reset、clean、flush、drop、push 这类的）只对本地一次性的库用。
- **迁移中途失败**：golang-migrate 这类工具会把库标成 dirty，之后什么迁移都不跑。先查清楚执行到了哪一步，告诉用户，不要直接 `force` 改版本号。

## 配置和环境

- **环境问题告诉用户**：缺数据库或缓存、缺 `.env` 或密钥、依赖下载不下来（多半是网络或代理）、运行时版本对不上，都停下来说明缺什么、查到了什么。不为了跑起来去改项目配置，不改镜像源和代理设置（`.npmrc`、`pip.conf`、Maven 的 `settings.xml`、`go env -w`），不编造密钥，不把 `.env` 和密钥加进提交。
- **临时覆盖可以**：只是本地连接地址不对（比如 `.env` 里数据库主机写的是 compose 服务名 `db`，服务在本机直接跑要连 `localhost` 加映射出来的端口），而代码会读环境变量的，在启动命令前临时设变量，不改文件。
- **Go**：1.21 起，`go.mod` 要求的版本比本机新时，会自动下载对应的工具链，网络不通会卡住或报错。
- **Java**：JDK 比 Gradle 支持的新时，会报 `Unsupported class file major version <N>`（N 减 44 就是 Java 版本）。macOS 上 `/usr/libexec/java_home -V` 列出本机装了哪些 JDK；有合适的，在命令前临时设 `JAVA_HOME`；没有就告诉用户。不改 wrapper 和构建文件里的版本。
- **Python**：`uv run` 会先同步环境，pyproject 和 uv.lock 对不上时会顺手改写 uv.lock。不想动锁文件就加 `--locked`（对不上直接报错）。依赖没装的，不往系统自带的 Python 里 `pip install`。
- **时区和语言**：服务器和数据库常用 UTC，本机时区可能不一样（比如东八区）。涉及日期边界（今天、按天统计、过期时间）的改动，用 `TZ=UTC <测试命令>` 再跑一次。Java 在 macOS 上的默认语言跟系统设置走（设 `LANG` 不管用），按默认语言格式化的星期、月份会是中文，和服务器上不一样。

## 常见坑

- **接口兼容**：删字段、改类型、改含义都会让已发布的客户端出错，App 的老版本会在线上存在很久。优先只加不改，必要时加版本号。
- **配置和密钥**：不写死在代码里，也不提交。新增配置项要同步更新 `.env.example` 和部署配置；部署配置不在这个仓库的，汇报时提醒用户。
- **日志**：不要打印密钥、令牌和用户隐私数据。测试结果文档里贴请求和响应时也一样，要打码。
- **并发和重试**：写操作要考虑幂等，事务边界要清楚。
- **生成的代码**：protobuf、ORM、OpenAPI 生成的代码，改了源头要重新生成，不要手改。
- **依赖和锁文件**：加依赖用项目的包管理命令（`go get`、`uv add`、`poetry add`、`cargo add`、对应的 npm 或 pnpm 命令），锁文件（`go.sum`、`uv.lock`、`poetry.lock`、`Cargo.lock`、`package-lock.json` 等）的改动一起提交，不手改锁文件。

## 危险信号

| 冒出这个念头 | 实际情况 |
|---|---|
| 迁移写错了，直接改那个文件 | 已经提交或在别的环境执行过的迁移，只能新增一个迁移来修正；只在自己本地库跑过、还没提交的，先回滚再改 |
| 连测试库或预发库验证，更真实 | 共享的库不跑迁移、不跑会清数据的命令和测试；连着它启动服务也可能自动迁移。只用本地或一次性的库 |
| 服务一直开着，直接调接口 | 那可能是改动前的旧进程，或者是别人起的。重启自己启动的服务再调 |
| 端口被占、环境乱了，杀进程或 `docker compose down -v` 重来 | 只停自己启动的；`down -v` 会删数据卷，本地库的数据跟着没了 |
| 缺数据库、`.env` 或密钥，改下配置先跑起来 | 环境问题，告诉用户；不改项目配置，不编造密钥，不提交 `.env` |
| curl 返回 200 就算通过 | 还要看响应体；写操作要去库里查，确认数据真的变了 |
| 单测都过了就行 | 涉及数据库或外部服务的，要跑集成测试或在本地实际调用 |
| 这个字段客户端应该没在用 | 先查调用方；拿不准就保留 |
| 顺手跑一下 `gofmt -w`、`black`、`ruff format`、`cargo fmt` 整理格式 | 这些会直接改文件，还会改到别人的。只跑检查模式、只看改动的文件：`gofmt -l`、`black --check`、`ruff format --check`、`cargo fmt --check`；项目的 lint 脚本先看有没有带 `--fix` |
