---
name: claude-seo (入口 / entry map)
description: >
  本仓库的「入口文档」——面向 AI Agent 的导航地图。Claude SEO 是一套 Claude Code plugin 形态的
  SEO 分析套件：25 个 sub-skill + 18 个 sub-agent + 53 个 Python 执行脚本 + 8 个可选 MCP 扩展，
  覆盖技术 SEO、E-E-A-T 内容质量、Schema 标记、站点地图、Core Web Vitals、本地 SEO / Maps、
  外链、GEO（AI 搜索）、语义聚类、SXO、漂移监控、电商 SEO、国际化 hreflang 与 Google 官方 API。
  当任务涉及「某个网站/页面在搜索引擎与 AI 搜索里的表现」时，先读本文件再进内部文件。
note: >
  这不是被 Claude Code 自动加载的 Skill 清单——真正被加载的单元是 `skills/<name>/SKILL.md`
  共 25 份（主编排器为 `skills/seo/SKILL.md`）与 `agents/*.md` 共 18 份。
  本文件是「先读我」的仓库级索引与路由，不参与注册：`install.sh` 只把 `skills/seo/*` 拷到
  `~/.claude/skills/seo/`，仓库根的本文件不在安装产物内。
---

# Claude SEO · 入口导航（Entry Map）

> **给 AI Agent 的一句话**：本仓库不是「SEO 知识文章集」，而是一套**可执行的审计流水线**——
> 由一个编排器把任务分发给 18 个专职 sub-agent，底下是 53 个真会去抓页面、调 Google API、
> 跑 Lighthouse、出 PDF 的 Python 脚本。你的正确姿势是**选命令、走编排、让脚本取真实数据**，
> 而不是凭模型记忆去点评一个网站。

## 0. 你该怎么用这份文档

1. 先看 **§1 这是什么**、**§2 一条命令上手**，建立整体认知与运行时前提。
2. 拿用户需求去 **§5 路由表**，定位到具体 `/seo` 命令与具体 `SKILL.md`。
3. 要细节时读 **§7 权威文件**，不要只凭本文摘要下判断。
4. 动手前读 **§6 硬约束**——尤其「运行时未就绪」「不许 pip install」「本仓是 fork」三条。

**判断是否启用**：任务的目标是**让某个网址被搜索引擎或 AI 搜索更好地找到、理解、引用**（收录、
排名、结构化数据、页面体验指标、AI Overviews 引用、本地商户曝光）——用本 Skill。
纯文案创意、投放/邮件/社媒运营、与具体 URL 无关的营销策略——不用本 Skill（见 §8）。

---

## 1. 这是什么

一个 Claude Code **plugin**（`.claude-plugin/plugin.json`，`name: claude-seo`，`version: 2.2.5`，
MIT，上游作者 AgriciDaniel）。三层架构：指令（SKILL.md）→ 编排（agents）→ 执行（scripts）。

| 组成 | 位置 | 实测数量 | 作用 |
|---|---|---|---|
| Sub-skills | `skills/<name>/SKILL.md` | **25** | 每个是一条 `/seo <command>` 的指令书；`skills/seo/` 是主编排器 |
| Sub-agents | `agents/*.md` | **18** | 审计时并行拉起的专职分析员，**只能经 Agent 工具调用，不许用 Bash 起** |
| 执行脚本 | `scripts/*.py` | **53** | 真正抓页面 / 调 API / 出报告；一律经 `claude-seo run` 调用 |
| 按需引用 | `skills/seo/references/*.md` | **13** | 阈值与规范（CWV、E-E-A-T、schema 类型、质量闸门、本地信号、Maps）——用到才读 |
| 可选扩展 | `extensions/<vendor>/` | **8** | dataforseo / firecrawl / banana / ahrefs / bing-webmaster / profound / seranking / unlighthouse |
| 质量钩子 | `hooks/hooks.json` | 1 条 | PostToolUse 匹配 `Edit\|Write` → 跑 `hooks/validate-schema.py` 校验 schema |
| 数据 / 模板 | `data/google-updates.json`、`schema/templates.json` | — | Google 更新台账、JSON-LD 模板 |

---

## 2. 一条命令上手

**运行时入口只有一个**——`bin/claude-seo`，它是 `scripts/runtime.py` 的壳，只有三个子命令：

```bash
./bin/claude-seo --help          # {setup,doctor,run}
./bin/claude-seo doctor          # 查运行时是否就绪，不改动系统
./bin/claude-seo run <script.py> [args]   # 跑某个 bundled 脚本
```

**首次必须先 setup**。全新检出时 `doctor` 的实测输出：

```
Runtime: setup required
Install mode: manual
Python: 3.12
Chromium: not installed
Reason: managed environment is missing
```

此时任何 `run` 都会**明确拒绝**（实测 `exit=3`）：

```
Claude SEO runtime is not ready. Run `/seo setup` and retry.
```

修复只有一条路：`./bin/claude-seo setup`（建隔离 venv + 装 Chromium）。
**venv 落点**：`manual` 模式落在仓库根；plugin 模式落在系统数据目录
（macOS `~/Library/Application Support/claude-seo`），可用 `CLAUDE_SEO_DATA_DIR` 覆盖。

**最小工作流**：`doctor` 确认就绪 → 选 §5 的命令 → 读对应 `skills/<name>/SKILL.md` 照做 →
需要真实数据时 `claude-seo run <script>` → 收口时按需出 PDF（`google_report.py`）。

---

## 3. 能力清单（25 个 sub-skill）

> 何时读本节：用户给了需求但你不确定该进哪条命令。列均为实测枚举。

| 命令 | Sub-skill 文件 | 回答什么问题 |
|---|---|---|
| `/seo audit <url>` | `skills/seo-audit/` | 全站审计，并行拉起多个 sub-agent，出 0–100 健康分 |
| `/seo page <url>` | `skills/seo-page/` | 单页深度分析 |
| `/seo technical <url>` | `skills/seo-technical/` | 技术 SEO 九类（可抓取性、可索引性、安全等） |
| `/seo content <url>` | `skills/seo-content/` | E-E-A-T 与内容质量 |
| `/seo content-brief <topic>` | `skills/seo-content-brief/` | 内容简报：目标词、大纲、内链 |
| `/seo schema <url>` | `skills/seo-schema/` | Schema.org 检测 / 校验 / 生成 |
| `/seo sitemap <url\|generate>` | `skills/seo-sitemap/` | XML 站点地图分析或生成 |
| `/seo images <url>` | `skills/seo-images/` | 图片 SEO：页面内审计、SERP 表现、文件优化 |
| `/seo geo <url>` | `skills/seo-geo/` | GEO：AI Overviews / ChatGPT / Perplexity 的可引用性 |
| `/seo local <url>` | `skills/seo-local/` | 本地 SEO：GBP、NAP 引用、评价、map pack |
| `/seo maps [cmd]` | `skills/seo-maps/` | Maps 情报：地理网格排名、GBP 审计、竞对半径 |
| `/seo backlinks <url>` | `skills/seo-backlinks/` | 外链档案：引荐域、锚文本分布、毒链 |
| `/seo cluster <seed>` | `skills/seo-cluster/` | 基于 SERP 的语义聚类与内容架构 |
| `/seo sxo <url>` | `skills/seo-sxo/` | 搜索体验优化：页面类型学、用户故事、人物画像 |
| `/seo drift baseline\|compare\|history <url>` | `skills/seo-drift/` | 建基线并按 17 条规则比对站点漂移 |
| `/seo ecommerce <url>` | `skills/seo-ecommerce/` | 电商 SEO：商品 schema、市场情报 |
| `/seo hreflang [url]` | `skills/seo-hreflang/` | 国际化 / hreflang 审计与生成 |
| `/seo plan <type>` | `skills/seo-plan/` | 按行业出战略规划 |
| `/seo programmatic [url\|plan]` | `skills/seo-programmatic/` | 规模化程序性 SEO |
| `/seo competitor-pages [url\|generate]` | `skills/seo-competitor-pages/` | 竞品对比页生成 |
| `/seo google [cmd] [url]` | `skills/seo-google/` | Google 官方 API：GSC、PageSpeed、CrUX、Indexing、GA4（另附 11 份 API 引用） |
| `/seo flow [stage]` | `skills/seo-flow/` | FLOW 框架提示词库集成 |
| `/seo dataforseo [cmd]` | `skills/seo-dataforseo/` | 实时 SEO 数据（**扩展镜像**，需装 DataForSEO MCP） |
| `/seo image-gen [use-case]` | `skills/seo-image-gen/` | SEO 素材图生成（**扩展镜像**，需装 Banana MCP） |
| （编排器） | `skills/seo/` | 路由表、行业识别、评分方法论、合成框架 |

---

## 4. 进阶：扩展与外部数据

> 何时读本节：用户要的数据本仓自身抓不到（关键词量、竞品外链、AI 引用份额、全站爬取）。

`extensions/<vendor>/` 各带安装脚本，装完才有对应命令。装法与鉴权读 `docs/MCP-INTEGRATION.md`。
凭据配置落在用户空间 `~/.config/claude-seo/`（`google-api.json` / `backlinks-api.json`），
**永远不进仓库**。查凭据是否就绪：`claude-seo run google_auth.py --check`、
`claude-seo run backlinks_auth.py --check`。

---

## 5. 需求 → 去哪（路由表）

| 用户想要… | 怎么做 / 读哪 |
|---|---|
| 「帮我全面看下这个网站的 SEO」 | `/seo audit <url>` → `skills/seo-audit/SKILL.md` |
| 「这页为什么排不上去」 | `/seo page <url>` → `skills/seo-page/SKILL.md` |
| 「Google 收录不了 / 抓取有问题」 | `/seo technical <url>` → `skills/seo-technical/SKILL.md` |
| 「加个结构化数据」 | `/seo schema <url>`；模板在 `schema/templates.json`，类型与弃用状态读 `skills/seo/references/schema-types.md` |
| 「Core Web Vitals 阈值是多少」 | 读 `skills/seo/references/cwv-thresholds.md`（**一律用 INP，不用 FID**） |
| 「怎么让 ChatGPT / AI Overviews 引用我」 | `/seo geo <url>` → `skills/seo-geo/SKILL.md` |
| 「本地门店没曝光」 | `/seo local <url>`；要地理网格排名再上 `/seo maps` |
| 「看竞品/我的外链」 | `/seo backlinks <url>`；免费源清单读 `skills/seo/references/free-backlink-sources.md` |
| 「围绕这个词做一批内容」 | `/seo cluster <seed>` → 产出可视化 `skills/seo-cluster/templates/cluster-map.html` |
| 「上线后有没有掉」 | 先 `/seo drift baseline <url>` 存基线，之后 `/seo drift compare <url>` |
| 「要真实的 GSC / GA4 / CrUX 数字」 | `/seo google <cmd> <url>`；先 `claude-seo run google_auth.py --check` |
| 「出一份能给客户看的报告」 | `claude-seo run google_report.py`（**唯一指定报告生成器**，A4 PDF） |
| 「命令跑不动 / 报 setup required」 | `./bin/claude-seo setup`；仍失败读 `docs/TROUBLESHOOTING.md` |
| 「怎么装 / 怎么接 MCP」 | `docs/INSTALLATION.md`、`docs/MCP-INTEGRATION.md` |
| 「它内部怎么组织的」 | `docs/ARCHITECTURE.md`、上游维护者指南 `CLAUDE.md` |
| 「完整命令参数表」 | `docs/COMMANDS.md` |

---

## 6. 硬约束与规范

**运行时**

- **脚本一律经 `claude-seo run <script.py>` 调用**，绝不用裸 `python3 scripts/xxx.py`。
- 报 `runtime is not ready` 时**只能跑 `setup`**，不许改用 `pip install` 兜底、不许装全局包。
- Sub-agent **只经 Agent 工具调用**，不许用 Bash 起。

**数据真实性（本仓最容易被违反的一条）**

- 所有结论必须来自脚本实际抓取或 API 实际返回。**取不到数就明说取不到**，不许用模型记忆
  编造排名、流量、外链数字。
- 抓外部 URL 的脚本必须走 `scripts/url_safety.py`（挡私有 IP、回环、元数据端点、DNS 重绑定）。

**内容规则（已过时的建议会被判错）**

- 一律用 **INP**，不用 FID。
- **不许推荐 HowTo schema**（2023-09 弃用）。
- FAQPage：Google 已于 2026-05-07 对所有站点撤下 FAQ 富结果——存量只标 Info、不建议删、
  **不许为 Google SERP 收益推荐新建**；真实问答用 QAPage。
- 落地页规模闸门：30+ 时告警（要求 60%+ 独特内容），50+ **硬停**并要求用户说明理由。

**凭据**

- `.env`、`client_secret*.json`、`oauth-token.json`、`service_account*.json` 一律不入库。
- 配置只落 `~/.config/claude-seo/`；token 文件里不存 `client_secret`。

**真相来源与同步（本仓是 fork，与上游 CLAUDE.md 写的流程不同）**

- 本目录是 `RussellLe/claude-seo`，fork 自 **`AgriciDaniel/claude-seo`**。
  上游 `CLAUDE.md` 里的 `origin`/`aimh` 双远端发布流程**是上游的，对本 fork 不成立**——
  不要照着往上游推。
- 本 fork 的 `origin` = `https://github.com/RussellLe/claude-seo`，工作分支 `main`。
- 本仓被 `russell-harness` 以 submodule 挂在 `skills/seo/claude-seo`：**改了内容要
  commit + push 本仓，再回宿主仓 `git add skills/seo/claude-seo` 回写 gitlink**，两步一组。
- 跟进上游：`git remote add upstream https://github.com/AgriciDaniel/claude-seo` 后 fetch/merge。

**改仓时**：`SKILL.md` 控制在 500 行 / 5000 token 内；reference 文件 200 行内；
脚本要有 docstring、CLI、JSON 输出；目录名 kebab-case；改完跑 `python3 -m pytest tests/`。

---

## 7. 权威文件索引

| 文件 | 内容 | 何时读 |
|---|---|---|
| `skills/seo/SKILL.md` | 主编排器：命令表、行业识别、审计分发 15 步、评分权重、合成方法论 | 做任何 `/seo` 任务前 |
| `skills/<name>/SKILL.md` | 单条命令的完整指令书 | 已定位到命令后 |
| `agents/<name>.md` | 单个 sub-agent 的职责与输出契约 | 要理解审计并行分工时 |
| `skills/seo/references/*.md`（13 份） | 阈值 / 框架 / 清单 | 需要具体数值或判定标准时 |
| `docs/COMMANDS.md` | 全命令参数参考 | 忘了参数形态时 |
| `docs/ARCHITECTURE.md` | 三层架构与渐进披露设计 | 要改结构或加 skill 时 |
| `docs/INSTALLATION.md` / `docs/MCP-INTEGRATION.md` | 安装与 MCP 接入 | 环境没跑起来 / 要装扩展时 |
| `docs/TROUBLESHOOTING.md` | 故障排查 | 报错且 §2 的路子没解决时 |
| `CLAUDE.md` | 上游维护者指南（架构全景 + 开发/安全/报告规则） | 要改本仓代码时；**其发布流程一节对本 fork 不适用** |
| `scripts/runtime.py` | 运行时探测、venv 装配、数据目录判定 | `doctor` 结果看不懂时 |

---

## 8. 边界（不适用）

- **与具体 URL / 站点无关的营销工作**——文案、邮件、社媒、投放、红人、发布节奏：
  走同目录的 `marketingskills` 或 `aaron-marketing-skills`。
- **纯写作任务**（不带收录/排名目标的博客、文档）——本 Skill 的产出是审计与规范，不是成稿。
- **没有网络或不许外呼的环境**：本仓的价值几乎全在实抓与 API，离线时只剩规范文档可读，
  此时不要假装跑过审计。
- **要求给出确切排名 / 搜索量而无任何数据源接入时**：明确告知需要 Google API 凭据或
  DataForSEO 扩展，不要估。
