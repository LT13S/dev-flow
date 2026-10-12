# dev-flow

让 AI 编程助手按固定流程改代码的 skill：先定规模，读项目档案，调研清楚，把要拍板的事一次问完，再动手实现，最后拿刚跑出来的证据说明做完了没有。前端、后端、移动端通用。

## 流程

| 步骤 | 做什么 |
|---|---|
| 0 定规模 | 分小、中、大三档，决定走哪些步骤 |
| 1 读项目档案 | 项目的命令、规矩和坑；没有档案就按模板生成一份 |
| 2 调研 | 找已有的相似实现，查清改动会影响到哪里 |
| 3 一次拍板 | 要用户定的问题一轮问完，给出带编号的拍板摘要 |
| 4 实现 | 照项目已有的写法改，功能日志打全 |
| 5 验证 | 自己把程序跑起来验证，出一份测试结果文档 |
| 6 收尾 | 汇报、核对拍板摘要，问用户要不要提交 |

## 安装

### 用 npx skills（Claude Code、Codex、Cursor 等都能装）

```sh
npx skills add LT13S/dev-flow --skill dev-flow -g -a claude-code -y
```

要装给别的工具，把 `-a claude-code` 换成对应的名字，比如 `-a codex`。

### 用 Claude Code 插件市场

```sh
claude plugin marketplace add LT13S/dev-flow
claude plugin install dev-flow@dev-flow
```

### 让它在改代码时自动用上

在 `~/.claude/CLAUDE.md` 里加一行：

```
改代码的任务（开发功能、修 bug、照另一端实现做、重构）先用 dev-flow skill；纯问答和一眼能改完的小改动不用。
```

## 更新

- 用 npx skills 装的：`npx skills update -g dev-flow -y`
- 用插件装的：先 `claude plugin marketplace update dev-flow`，再 `claude plugin update dev-flow@dev-flow`

## 目录

```
skills/dev-flow/
├── SKILL.md                 主流程
└── references/
    ├── profile-template.md  项目档案的生成方法和模板
    ├── test-report.md       测试结果文档的格式
    ├── debugging.md         修 bug
    ├── large-work.md        大任务：总纲、分阶段、交接
    ├── parity.md            照另一端或旧版本的实现做
    ├── ui-work.md           涉及界面
    └── stacks/              技术栈指南：ios、android、flutter、web、backend
```
