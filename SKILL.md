---
name: claim-finish
description: >
  交付前最终验收关卡。在宣布任务完成、声称工作完毕之前，系统性核查 6 项交付物：
  (1) proposal.md 需求提案, (2) validation.md 验收标准, (3) VC/用户/开发者三方文档,
  (4) 白盒测试 + 快照报告, (5) 用户操作手册, (6) release/final/ 干净交付包。
  完成后生成单页 HTML 验收报告并用 Chrome 打开。
  仅由用户手动触发：/claim-finish、交付检查、最终验收、claim finish。
  禁止被 hook 或其他 skill 自动调用。
---

# Claim Finish — 交付前最终验收关卡

运行 6 个 checkpoint，依次核查每一项交付物。每项记录 ✅/⚠️/❌ 三级状态。全部完成后生成 HTML 验收报告。

**阻塞规则：** 任何 ❌ 存在时，不得宣告任务完成，必须列出修复清单后停止。

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

## HTML 验收报告

**所有 6 个 checkpoint 完成后**，生成单页验收报告。

1. 读取模板：`~/.claude/skills/claim-finish/assets/report-template.html`
2. 将以下占位符替换为实际数据：

| 占位符 | 替换内容 |
|--------|---------|
| `{{PROJECT_NAME}}` | 当前目录名 |
| `{{REPORT_DATE}}` | `date +%Y-%m-%d` 输出 |
| `{{CHECKPOINT_DATA}}` | 6 条 JSON 对象（见下方格式） |
| `{{TEST_TOTAL}}` | 测试总数（字符串，无数据填 `"—"`） |
| `{{TEST_PASS}}` | 通过数 |
| `{{TEST_FAIL}}` | 失败数 |
| `{{COV_LINE}}` | 行覆盖率整数（0–100，无数据填 `0`） |
| `{{COV_BRANCH}}` | 分支覆盖率整数 |
| `{{COV_FUNC}}` | 函数覆盖率整数 |
| `{{FILE_TREE}}` | `find release/final -type f \| sort` 输出，HTML 转义后填入 |
| `{{MANIFEST_COUNT}}` | MANIFEST.txt 行数 |
| `{{PACKAGE_SIZE}}` | `du -sh release/final` 输出 |

**CHECKPOINT_DATA 格式**（6 条，顺序固定）：

```json
[
  {"id":"cp1","icon":"📋","title":"proposal.md",     "status":"pass|warn|fail","notes":"简短说明"},
  {"id":"cp2","icon":"✔️", "title":"validation.md",  "status":"pass|warn|fail","notes":"简短说明"},
  {"id":"cp3","icon":"📚","title":"三方文档",         "status":"pass|warn|fail","notes":"简短说明"},
  {"id":"cp4","icon":"🧪","title":"白盒测试",         "status":"pass|warn|fail","notes":"简短说明"},
  {"id":"cp5","icon":"📖","title":"用户操作手册",     "status":"pass|warn|fail","notes":"简短说明"},
  {"id":"cp6","icon":"📦","title":"release/final/",  "status":"pass|warn|fail","notes":"简短说明"}
]
```

3. 将填充后的内容写入 `/tmp/claim-finish-report-YYYYMMDD.html`（日期用实际日期）。
4. 立即执行：`open -a "Google Chrome" /tmp/claim-finish-report-YYYYMMDD.html`

---

## 最终宣告规则

| 状态组合 | 操作 |
|---------|------|
| 全部 ✅ | 宣告 **LGTM — 可以发布** |
| 存在 ⚠️，无 ❌ | 宣告 **条件通过**，列出警告项，提示用户在报告中签字后再发布 |
| 存在任意 ❌ | **阻塞发布**，列出必须修复的条目清单，不得宣告完成 |
