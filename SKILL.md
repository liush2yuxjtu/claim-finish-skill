---
name: claim-finish
description: >
  交付前最终验收关卡。在宣布任务完成、声称工作完毕之前，系统性核查 7 项交付物：
  (1) proposal.md 需求提案, (2) validation.md 验收标准, (3) VC/用户/开发者三方文档,
  (4) 白盒测试 + 快照报告, (5) 用户操作手册, (6) release/final/ 干净交付包,
  (7) Playwright 可回放脚本。
  完成后生成单页 HTML 验收报告并用 Chrome 打开。
  仅由用户手动触发：/claim-finish、交付检查、最终验收、claim finish。
  禁止被 hook 或其他 skill 自动调用。
---

# Claim Finish — 交付前最终验收关卡

运行 7 个 checkpoint，依次核查每一项交付物。每项记录 ✅/⚠️/❌ 三级状态。全部完成后生成 HTML 验收报告。

**阻塞规则：** 任何 ❌ 存在时，不得宣告任务完成，必须列出修复清单后停止。

---

## Subagent 优先执行

- 如当前 harness 支持 subagents，默认把 checkpoint 扫描委托给后台 subagents；主代理只负责 orchestration、阻塞判断、最终 HTML 报告装配与最终宣告。
- 后台 worker 优先使用 Haiku-class 或同类低成本快速模型；把跨 checkpoint 的综合判断、争议裁决和最终报告拼装留给主代理。
- 建议并行拆分为 4 个有界 worker：
  1. Worker A：CP1 + CP2
  2. Worker B：CP3 + CP5
  3. Worker C：CP4 + CP7
  4. Worker D：CP6
- 每个 worker 只返回结构化结果：`status`、简短 `notes`、关键指标、原始文件或日志路径。不要把整份原文、大段测试日志或完整 HTML 直接贴回主对话。
- 大体量原始内容应优先写入磁盘工件，再由主代理按需读取并装配到 `REPORT_DATA`，以节省主代理上下文。
- 若任一 worker 发现 `❌`，应立即回传阻塞原因；主代理停止“可发布”路径，改为汇总修复清单。
- 若当前环境不支持 subagents 或不支持模型路由，则回退到单代理执行，但仍保持“先写磁盘工件、后按需读取”的上下文节制策略。

---

## Checkpoint 流程

### CP1 — proposal.md

**目标：** 需求提案文档存在且章节完整。

1. 在项目根目录（含 `docs/`）搜索 `proposal.md`（大小写不敏感），排除 `node_modules/`、`.git/`、`release/`。
2. 读取文件，确认包含：背景/Background、目标/Goals、范围/Scope、验收标准/Acceptance Criteria 四类章节（名称可变，内容对应即可）。
3. 确认无 `TODO`/`TBD`/`[待填写]` 等未完成标记。

**状态判定：**
- ✅ 文件存在且四类章节均有实质内容
- ⚠️ 文件存在但缺少 1–2 个章节
- ❌ 文件不存在，或缺少验收标准章节（硬性阻塞）

---

### CP2 — validation.md

**目标：** 验收标准已书面化，并与 proposal.md 的验收条目一一对应。

1. 在项目中搜索 `validation.md`（同上排除规则）。
2. 读取文件，检查：条目列表存在、每条有测试方法说明、有实际测试结果（通过/部分通过/失败）。
3. 对比 CP1 中提取的验收标准条目，列出未覆盖项。

**状态判定：**
- ✅ 文件存在，所有验收条目均已覆盖并有结果
- ⚠️ 文件存在但有条目未覆盖或结果缺失
- ❌ 文件不存在（硬性阻塞）

---

### CP3 — 三方文档

**目标：** VC 路演手册、用户手册、开发者 INDEX.md 三份文档均存在且无占位符。

搜索以下三类文档（文件名模式灵活匹配，实质内容对应即可）：

| 类型 | 常见文件名 | 必检内容 |
|------|-----------|---------|
| VC 路演手册 | `vc-handout*`, `investor*`, `pitch*` | 产品定位、市场规模、核心指标 |
| 用户手册 | `user-handout*`, `user_guide*`, `getting-started*` | 快速入门、核心功能、FAQ |
| 开发者索引 | `INDEX.md`（docs/ 或根目录）, `DEVELOPERS.md` | 架构、API 目录、环境搭建 |

对每份文档：
1. 确认文件存在且非空（> 500 bytes）。
2. 检查是否含未完成标记：`TODO`、`PLACEHOLDER`、`[待填写]`、`TBD`、`FIXME`。
3. 读取前 30 行确认文档结构已成型。

**状态判定（三份分别记录）：**
- ✅ 找到文件、内容成型、无占位符
- ⚠️ 文件存在但含占位符
- ❌ 文件不存在（硬性阻塞）

---

### CP4 — 白盒测试 + 快照报告

**目标：** 探测项目测试框架，运行测试，收集覆盖率和快照结果。

**Step 1 — 探测框架**（按优先级顺序匹配，找到即停）：

```
Node/TypeScript:  package.json 含 jest/vitest/mocha → npm test / npx jest / npx vitest run
Python:           pytest.ini / conftest.py 存在    → pytest -v --tb=short
Go:               *_test.go 文件存在               → go test ./... -v -coverprofile=/tmp/cover.out
Makefile:         有 test 目标                     → make test
其他:             根据项目实际情况判断
```

**Step 2 — 运行测试，捕获结果**

根据探测结果选择正确命令执行，将输出保存到变量中备用。关键采集项：
- 总测试数、通过数、失败数
- 代码覆盖率（line / branch / function，若可获取）
- 失败测试名称及错误摘要（若有）
- 快照变更情况（Jest: `--updateSnapshot` 状态；Playwright: screenshots/video 路径）

**Step 3 — 状态判定：**
- ✅ 零失败，覆盖率 ≥ 60%
- ⚠️ 零失败，但覆盖率 < 60% 或无覆盖率数据
- ❌ 有任意 FAILED 测试（硬性阻塞）
- ⚠️ 项目无测试框架（记录说明，不阻塞）

---

### CP5 — 用户操作手册

**目标：** 面向最终用户的操作手册结构完整，可独立阅读执行。

1. 搜索用户手册主文档（`README.md`、`USAGE.md`、`user-guide*`、`*手册*`、`*操作指南*`），优先 docs/ 目录。
2. 检查以下章节是否存在（名称可变，内容对应即可）：
   - 安装/环境要求
   - 快速开始
   - 功能说明
   - FAQ 或 Troubleshooting
   - 版本说明/Changelog（或文件头部含版本号）
3. 检查命令示例中有无未替换占位符（`your-value-here`、`<YOUR_`、`示例值`等）。
4. 验证内部 Markdown 链接是否指向已存在的文件。

**状态判定：**
- ✅ 五类章节均存在，无占位符，链接有效
- ⚠️ 缺少 1–2 个章节，或有少量占位符
- ❌ 文件不存在或缺少安装/快速开始核心章节（硬性阻塞）

---

### CP6 — release/final/ 打包

**目标：** 构建不含开发残留的干净对外交付包。

**Step 1 — 构建交付目录**

```
输出路径：release/final/（先清空旧内容）
```

**复制策略**（根据项目实际结构自适应）：
- 文档类：`proposal.md`、`validation.md`、`README.md`、`LICENSE`、`CHANGELOG.md`、`docs/` 目录
- 源码：`src/`、`lib/`、`app/` 等（排除测试缓存、构建产物）
- 测试代码：`tests/`、`test/`、`spec/` 等（排除 cache、coverage 原始数据）
- 配置：`package.json`、`pyproject.toml`、`go.mod`、`Makefile`、`Dockerfile` 等

**Step 2 — 禁止项扫描**

交付包中不得含有以下任何内容：

```
*.log  |  .env  |  .env.*  |  *.local  |  node_modules/
__pycache__/  |  *.pyc  |  .pytest_cache/  |  .DS_Store
dist/build/  |  *.egg-info/  |  coverage/（原始数据）
```

**Step 3 — 生成 MANIFEST.txt**

列出 `release/final/` 下所有文件的路径，追加总文件数和包体积（`du -sh`）。

**状态判定：**
- ✅ 禁止项扫描通过，MANIFEST.txt 已生成
- ❌ 发现任意禁止文件（硬性阻塞，必须清除后重扫）

---

### CP7 — Playwright 可回放脚本

**目标：** 项目包含可回放的 Playwright 端到端测试脚本，能够录制并回放关键用户操作流程，支持可选的前置设置脚本（setup pre-script）。

**Step 1 — 探测 Playwright 配置**

搜索以下标志文件（按优先级）：
- `playwright.config.ts` / `playwright.config.js` / `playwright.config.mjs`
- `package.json` 中含 `@playwright/test` 依赖
- `e2e/`、`tests/e2e/`、`tests/playwright/` 目录

**Step 2 — 检查回放脚本**

在以下目录搜索 Playwright 测试脚本（`*.spec.ts`、`*.spec.js`、`*.test.ts`、`*.test.js`）：
- `e2e/`、`tests/e2e/`、`tests/playwright/`、`tests/`、`test/`

对每个脚本检查：
1. 至少包含有意义的用户交互步骤（`page.goto`、`page.click`、`page.fill`、`page.locator`、`expect` 等）。
2. 脚本可独立描述一条完整用户操作路径（从打开页面到验证结果）。
3. 记录脚本总数和覆盖的用户流程。

**Step 3 — 检查回放配置**

读取 `playwright.config.*`，确认以下回放关键配置：
- `use.trace`：是否启用 trace 录制（`on` / `on-first-retry` / `retain-on-failure`）
- `use.video`：是否启用视频录制（`on` / `retain-on-failure`）
- `use.screenshot`：是否启用截图（`on` / `only-on-failure`）
- `outputDir` / `testDir`：输出目录是否合理

至少需要 trace 或 video 其中之一启用，以保证脚本可回放。

**Step 4 — 检查前置设置脚本（setup pre-script）**

搜索前置设置相关配置和文件：
- `playwright.config.*` 中的 `globalSetup` / `globalTeardown` 字段
- `playwright.config.*` 中 `projects` 数组里的 `setup` 依赖项（`dependencies: ['setup']`）
- `tests/setup/`、`e2e/setup/`、`fixtures/` 目录下的设置脚本
- `.auth/` 或 `playwright/.auth/` 状态文件（认证状态保存）

前置设置脚本允许包含：
- 环境初始化（数据库 seed、服务启动、测试数据注入）
- 认证状态准备（登录并保存 `storageState`）
- 依赖服务健康检查

记录：`globalSetup` 是否配置、setup project 是否存在、设置脚本列表。

**Step 5 — 试运行验证**

执行 `npx playwright test --list` 确认脚本可被解析发现（不实际运行全部测试）。采集：
- 可发现的测试总数
- 测试文件列表
- 是否有解析错误

**状态判定：**
- ✅ Playwright 已配置，至少 1 个回放脚本存在且含有意义交互步骤，`--list` 通过，trace 或 video 已启用
- ⚠️ Playwright 已配置但：回放脚本无实质交互 / `--list` 有警告 / trace 和 video 均未启用 / 无前置设置脚本（记录说明）
- ❌ 项目无 Playwright 配置或无任何回放脚本（硬性阻塞）

---

## HTML 验收报告（Index 首页）

**所有 7 个 checkpoint 完成后**，生成单页 Index 首页，内嵌所有 checkpoint 原始文件内容。

**此页面不是模板**——不使用 `{{}}` 占位符替换。HTML 结构和渲染逻辑已内置在 `~/.claude/skills/claim-finish/assets/report-template.html` 中，Claude 只需注入一个 `REPORT_DATA` JSON 对象即可。

### 生成步骤

1. **读取** `~/.claude/skills/claim-finish/assets/report-template.html` 全文。
2. **收集数据**：读取每个 checkpoint 涉及的原始文件内容（proposal.md 全文、validation.md 全文、三方文档全文、测试输出、用户手册全文、MANIFEST.txt、playwright 配置和脚本内容等）。
3. **构建 `REPORT_DATA` 对象**（见下方格式）。
4. **在 HTML 中找到注释行** `// window.REPORT_DATA = { ... };`，将其替换为实际的 `window.REPORT_DATA = { ... };`（JSON 内容内联，确保字符串中的 `<`、`>`、`&` 已转义）。
5. **写入** `/tmp/claim-finish-report-YYYYMMDD.html`（日期用实际日期）。
6. **立即执行**：`open -a "Google Chrome" /tmp/claim-finish-report-YYYYMMDD.html`

### REPORT_DATA 格式

```js
window.REPORT_DATA = {
  project: "当前目录名",
  date: "YYYY-MM-DD",
  checkpoints: [
    {
      id: "cp1",
      icon: "📋",
      title: "proposal.md",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "proposal.md", path: "docs/proposal.md", content: "原始文件全文内容..." }
      ]
    },
    {
      id: "cp2",
      icon: "✔️",
      title: "validation.md",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "validation.md", path: "validation.md", content: "原始文件全文内容..." }
      ]
    },
    {
      id: "cp3",
      icon: "📚",
      title: "三方文档",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "vc-handout.md", path: "docs/vc-handout.md", content: "..." },
        { name: "user-handout.md", path: "docs/user-handout.md", content: "..." },
        { name: "INDEX.md", path: "docs/INDEX.md", content: "..." }
      ]
    },
    {
      id: "cp4",
      icon: "🧪",
      title: "白盒测试",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "测试输出", path: "", content: "完整测试命令输出..." }
      ]
    },
    {
      id: "cp5",
      icon: "📖",
      title: "用户操作手册",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "README.md", path: "README.md", content: "原始文件全文内容..." }
      ]
    },
    {
      id: "cp6",
      icon: "📦",
      title: "release/final/",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "MANIFEST.txt", path: "release/final/MANIFEST.txt", content: "文件清单全文..." },
        { name: "目录树", path: "", content: "find release/final 输出..." }
      ]
    },
    {
      id: "cp7",
      icon: "🎭",
      title: "Playwright 回放脚本",
      status: "pass|warn|fail",
      notes: "简短状态说明",
      files: [
        { name: "playwright.config.ts", path: "playwright.config.ts", content: "配置文件全文..." },
        { name: "脚本列表 (--list)", path: "", content: "npx playwright test --list 输出..." },
        { name: "setup.ts", path: "tests/setup/setup.ts", content: "前置脚本内容（若存在）..." }
      ]
    }
  ],
  tests: {
    total: "10",
    pass: "10",
    fail: "0",
    coverage: { line: 80, branch: 70, func: 85 }
  },
  release: {
    fileTree: "find release/final 输出",
    manifestCount: "42",
    packageSize: "1.2M"
  }
};
```

**注意事项：**
- `files` 数组中的每项 `content` 字段包含**原始文件全文**，页面会在可折叠面板中展示
- 文件内容中的特殊字符（`<`、`>`、`&`、`\`、引号）必须正确 JSON 转义
- 文件不存在时 `content` 填 `"(文件不存在)"`，`files` 数组仍需保留对应条目
- `tests` 和 `release` 字段无数据时设为 `null`，页面会优雅降级
- 每个 checkpoint 的 `files` 数组应包含该 CP 检查过程中实际读取的所有文件

---

## 最终宣告规则

| 状态组合 | 操作 |
|---------|------|
| 全部 ✅ | 宣告 **LGTM — 可以发布** |
| 存在 ⚠️，无 ❌ | 宣告 **条件通过**，列出警告项，提示用户在报告中签字后再发布 |
| 存在任意 ❌ | **阻塞发布**，列出必须修复的条目清单，不得宣告完成 |
