# Web 前端技术栈指南

适用于 React、Vue、Svelte、Angular 等 Web 前端，也包括 Next.js、Nuxt 这类全栈框架的前端部分。

这里只写所有 Web 前端项目都成立的做法和坑。项目特有的命令、端口和坑写进项目档案。

## 生成档案时怎么推断

| 要确定什么 | 怎么查 |
|---|---|
| 包管理器 | 先看 `package.json` 的 `packageManager` 字段（或 `devEngines.packageManager`），再看锁文件：`pnpm-lock.yaml` 是 pnpm，`yarn.lock` 是 yarn（有 `.yarnrc.yml` 的是 Yarn 2 及以上），`package-lock.json` 是 npm，`bun.lock` 或 `bun.lockb` 是 bun。锁文件不止一个时，看 CI 用的哪个，还定不了就问。只用这一个，不要混用 |
| 执行项目里装的工具（下文写作 `<执行>`） | npm：`npm exec --no --`（`--` 不能省，省了参数会被 npm 吞掉）；pnpm：`pnpm exec`；yarn：`yarn`；bun：`bun run`。不要用裸的 `npx <工具>`：项目里没装时，它不提示就从网上下载，下到的可能是同名的别的包（比如没装 TypeScript 时跑 `npx tsc`） |
| Node 版本 | `.nvmrc`、`.node-version`、`package.json` 的 `engines.node`。和 `node -v` 对不上的，写进档案并告诉用户 |
| 常用命令 | `package.json` 的 scripts：dev、build、lint、test，类型检查可能叫 typecheck、type-check 或 check。逐个看脚本实际跑什么：有的 lint 脚本自带 `--fix`（比如 create-vue 生成的 `eslint . --fix`），有的 build 已经带了类型检查（比如 `tsc -b && vite build`） |
| 是否 monorepo | 有 `pnpm-workspace.yaml`，或 `package.json` 里有 `workspaces` 字段，就是；有 `turbo.json`、`nx.json` 说明用 Turborepo、Nx 编排任务。命令要按包过滤，比如 `pnpm --filter <包名> <命令>`、`npm run <脚本> -w <包名>`、`turbo run <任务> --filter=<包名>` |
| 测试 | vitest、jest、playwright、cypress 的配置文件。vitest 的配置也可能写在 `vite.config.*` 里 |
| 样式体系 | tailwind（v3 有 `tailwind.config.*`；v4 一般没有这个文件，主题写在 CSS 的 `@theme` 里）、CSS 变量、设计 token 文件，以及用的组件库（shadcn 有 `components.json`，antd 等看依赖） |
| 环境变量 | `.env.example`；哪些变量会暴露给浏览器（带 `NEXT_PUBLIC_`、`VITE_`、`PUBLIC_` 这类前缀的）。`.env.local` 这类本地文件不提交，缺了问用户 |
| 界面验证工具 | 依赖里有没有 `@playwright/test` 或 `playwright`；浏览器装没装（macOS 在 `~/Library/Caches/ms-playwright`）；会话里有没有浏览器控制工具（比如浏览器类的 MCP）。别用 `npx playwright --version` 来查，没装时它会直接下载 |

## 验证

- 类型检查、lint、相关的单测一般几秒到几十秒，每改完一块就跑。build 比较慢，放到阶段末尾跑。
- **判断成败**：输出重定向到文件，再看退出码。别接 `| tail` 或 `| tee` 看结果，退出码会变成管道里最后一个命令的。

  ```sh
  <包管理器> run build > /tmp/<项目>-build.log 2>&1; echo "exit=$?"
  ```

- **找错误**：

  ```sh
  perl -pe 's/\e\[[0-9;]*m//g' /tmp/<项目>-build.log | grep -nE 'TS[0-9]{4,}: |^ +[0-9]+:[0-9]+ +error |error during build|[Bb]uild failed|\[ERROR\]|ERROR in |[0-9] ERROR "|Failed to compile|Type error:|Build error occurred|Error occurred prerendering|[Ff]ound [1-9][0-9]* errors?' | head -50
  ```

  能匹配 tsc（含 vue-tsc）、svelte-check、ESLint、Vite、esbuild、webpack、Next.js 的错误格式。前面先去掉颜色码，否则带颜色的输出匹配不上。不要直接搜 error：成功的日志里也常有（警告、`0 errors`、叫 error 的页面路由）；`Module not found` 在成功的 build 里也可能只是警告。命中后看前后几行：Vite 的具体原因在 `error during build:` 下面一两行，ESLint 的文件名在问题行上方。都没命中，看日志末尾 50 行。
- **类型检查**：
  - 有脚本用脚本。没有时，普通项目用 `<执行> tsc --noEmit`。`tsconfig.json` 里有 `references` 的要用 `<执行> tsc -b`：Vite 模板就是这样，根配置的 `files` 是空的，在根目录跑 `tsc --noEmit` 什么都不查，直接通过。`tsc -b` 会写 `.tsbuildinfo` 文件，没设 `noEmit` 的子项目还会输出编译结果。
  - tsc 不查 `.vue`、`.svelte`、`.astro` 文件，分别用 `vue-tsc`、`svelte-check`、`astro check`；Nuxt 用 `nuxt typecheck`。
  - Next.js、SvelteKit、Nuxt 有一部分类型是生成的（`.next/types`、`.svelte-kit`、`.nuxt`）。新检出的代码还没生成过，类型检查会报一堆找不到类型：先跑 `next typegen`、`svelte-kit sync`、`nuxt prepare`（项目脚本里一般已经有）。
- **lint 和格式**：只对这次改动的文件跑检查模式：`<执行> eslint <文件>`、`<执行> prettier --check <文件>`、`<执行> biome check <文件>`、`<执行> stylelint <文件>`。带 `--fix`、`--write` 的写法（`eslint --fix`、`prettier --write`、`biome check --write`、`stylelint --fix`）会直接改文件，对目录跑会改到一大片无关文件。ESLint 的退出码：0 是没有 error（可能有 warning），1 是有 error，2 是配置或运行出错。
- **测试**：一次性跑完用（前面加 `<执行>`）：`vitest run <文件>`、`jest <文件>`、`playwright test <文件>`、`cypress run --spec <文件>`。项目的 test 脚本可能是监听模式（比如 `jest --watch`，老版本的 `vitest`），会卡住不退出；`cypress open`、`playwright test --ui` 是交互界面，也不要用。
- **build**：
  - build 通过不等于类型没问题：Vite 和用 Vite 的框架只转译、不查类型，Nuxt 的 build 默认也不查；`next build`、`ng build` 会查。
  - SSR 框架有些错误只在 build 或运行时才出现（比如预渲染时访问了 `window`、`document`），所以阶段末尾必须 build。关键改动在 build 后用 `vite preview`、`next start`、`nuxt preview` 这类命令把生产包跑起来，再验一遍。
  - Next.js 15 及以前，dev 和 build 共用 `.next` 目录，dev server 开着时跑 build 会把它弄坏：先停 dev，build 完再重启。16 起 dev 输出到 `.next/dev`，可以同时跑。

## 界面验证与日志

项目已有的 e2e 测试照常跑；不为了验证往仓库里新加测试文件。一次性的脚本和截图放测试结果目录。

- **dev server**：
  - 启动前先看端口有没有被占：`lsof -nP -iTCP:<端口> -sTCP:LISTEN`。被占的多半是别的会话或用户开的，不要杀，换个端口。
  - 在后台启动，输出写进日志，记下端口和 PID：`nohup <包管理器> run dev > /tmp/<项目>-dev.log 2>&1 & echo $!`。工具自带后台运行功能的，用工具的。
  - 地址以日志里打印的为准：没写死端口时，Vite、Next.js 遇到端口被占会自动换下一个。也别把 `localhost` 换成 `127.0.0.1`：dev server 默认绑定 localhost，可能只监听 IPv6，`127.0.0.1` 连不上。
  - 换端口：Vite 加 `--port <端口>`，Next.js 加 `-p <端口>`。经 npm 脚本传参要加 `--`：`npm run dev -- --port 5174`；pnpm、yarn 直接写在脚本名后面（pnpm 会把多写的 `--` 原样传给命令，参数就不生效了）。
  - Next.js 16 起，同一个项目只能开一个 `next dev`。报 `Unable to acquire lock`，说明已经有人开着：不要删锁文件，问用户能不能直接用那个。
  - 打开页面前，先用 `curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:<端口>/<路径>` 确认能访问（SSR 页面出错会返回 500）。
  - 收尾只停自己启动的：kill 记下的 PID，再用 lsof 确认端口已经释放。还有进程占着的，确认是自己启动的那个拉起来的，再停。
- **截图**：手机尺寸和桌面尺寸各截一张：

  ```sh
  <执行> playwright screenshot --viewport-size="390,844" http://localhost:<端口>/<路径> <测试结果目录>/mobile.png
  <执行> playwright screenshot --viewport-size="1440,900" http://localhost:<端口>/<路径> <测试结果目录>/desktop.png
  ```

  - 要整页加 `--full-page`。内容是异步加载的，加 `--wait-for-selector=<选择器>` 等它出现再截，否则可能截到加载中的样子。深色模式加 `--color-scheme=dark`。
  - 只要尺寸就用 `--viewport-size`。`--device` 会换成设备默认的浏览器（iPhone 是 WebKit），没装就跑不起来。
  - 报 `Executable doesn't exist` 是浏览器没装：先问用户，同意后执行 `<执行> playwright install chromium`。浏览器装在用户缓存目录，不改项目；浏览器版本跟 Playwright 版本绑定，所以要用项目里的 Playwright 来装。
  - 项目没装 Playwright：问用户能不能临时用 `npx playwright@<版本>`（下载到 npm 缓存，不改项目）。不要为了验证把它加进项目依赖。
- **点按、看控制台和网络**：要操作页面、看报错的，写一个一次性脚本，用 `node <脚本>` 运行。脚本放在仓库外，直接 `require('playwright')` 找不到项目里装的包，要写绝对路径：

  ```js
  // check.cjs
  const { chromium } = require('<仓库>/node_modules/@playwright/test'); // 只装了 playwright 的，把包名换掉
  (async () => {
    const browser = await chromium.launch();
    const page = await browser.newPage({ viewport: { width: 390, height: 844 } });
    const problems = [];
    page.on('console', m => { if (m.type() === 'error') problems.push('console: ' + m.text()); });
    page.on('pageerror', e => problems.push('pageerror: ' + e.message));
    page.on('requestfailed', r => problems.push('failed: ' + r.url() + ' ' + r.failure()?.errorText));
    page.on('response', r => { if (r.status() >= 400) problems.push(r.status() + ' ' + r.url()); });
    await page.goto('http://localhost:<端口>/<路径>');
    await page.getByRole('button', { name: '<按钮文字>' }).click();
    await page.screenshot({ path: '<测试结果目录>/1-<简短名>.png', fullPage: true });
    console.log(problems.join('\n') || 'no console errors or failed requests');
    await browser.close();
  })();
  ```

  每个场景都看 `problems` 的输出：水合错误、接口 4xx/5xx、资源 404、跨域被拦，页面上可能看不出来。
- **服务端日志**：SSR 框架服务端的报错（页面 500、服务端取数据失败）打在 dev server 的日志里，不在浏览器里，看 `/tmp/<项目>-dev.log`。

## 常见坑

- **依赖和锁文件**：加减依赖会改动锁文件。只用档案里写的包管理器，锁文件的变更要一起提交。只是装依赖时，用不改锁文件的方式：`npm ci`（会先删掉整个 node_modules，别的会话开着 dev server 时别用）、`pnpm install --frozen-lockfile`、`yarn install --frozen-lockfile`（Yarn 2 及以上用 `--immutable`）、`bun install --frozen-lockfile`。报锁文件和 package.json 对不上，告诉用户，不要改用普通 install 去更新锁文件。
- **环境问题**：Node 版本不对、registry 或代理连不上（报 401、403、超时、ECONNREFUSED）、包管理器没装（Node 25 起不再自带 Corepack）、缺 `.env.local` 或密钥，都告诉用户处理。不要改 `.nvmrc`、`engines`、`.npmrc`，也不要把密钥写进仓库来绕过。
- **生成的代码**：GraphQL codegen、OpenAPI 客户端、Prisma client、路由类型这些，改了源头要重新生成，不要手改生成出来的文件。
- **SSR 水合**：服务端和客户端渲染结果不一致（用了当前时间、随机数，或者直接访问 window）会报水合错误。错误打在浏览器控制台里，页面看着可能是正常的。
- **环境变量**：带公开前缀的变量在 build 时就写死进产物，换环境要重新 build。改了 `.env` 要重启 dev server 再验。
- **dev 代理**：Vite 的 `server.proxy`、webpack 的 `devServer.proxy` 只在本地 dev 和 preview 时生效，部署后请求不经过它。本地没有跨域问题，不代表线上没有。改了接口地址或代理配置，汇报里写明线上要怎么配。
- **缓存**：改了依赖或用 `npm link` 链接了本地包，页面却没变化：重启 dev server，Vite 加 `--force` 重新预构建依赖；Next.js 先停 dev server，再删 `.next` 目录。
- **密钥**：不能写进前端代码，也不能放进带公开前缀的环境变量。
- **多语言 JSON**：改完校验格式：`node -e 'JSON.parse(require("fs").readFileSync(process.argv[1], "utf8"))' <文件>`。每种语言都要同步。

## 危险信号

| 冒出这个念头 | 实际情况 |
|---|---|
| dev 模式能跑就行 | build 可能失败（类型错误、SSR 问题），阶段末尾必须 build |
| build 过了，类型肯定没问题 | Vite、Nuxt 的 build 不查类型，要单独跑类型检查 |
| `tsc --noEmit` 没报错，类型没问题 | tsconfig 用了 `references` 时它什么都没查；`.vue`、`.svelte` 文件它也不查 |
| 跑一下 lint 脚本或 `prettier --write` 整理格式 | 有的 lint 脚本自带 `--fix`，会改一大片无关文件。先看脚本内容，只对改动的文件跑检查模式 |
| `npx` 一下这个工具 | 项目没装时它不提示就下载。用档案里的执行方式 |
| 页面截图看着没问题 | 还要看控制台报错和失败的请求 |
| 端口被占，杀掉占端口的进程 | 可能是别的会话或用户的 dev server。换端口，只停自己启动的 |
| 顺手换个包管理器装依赖 | 锁文件会乱 |
| 直接改生成的文件最快 | 下次一生成就被覆盖 |
