# BrowserGraph 技术方案

## 1. 项目目标

1. 给定网站 URL，Agent 自动扫描网站并提取可交互能力。
2. Agent 将网站能力抽象为统一 DSL（能力地图），包含页面、组件、动作、流程、selector 候选等信息。
3. Runner 读取 DSL，解析为执行计划，并生成或执行 Playwright 代码。
4. 通过 DSL 实现网站能力复用，避免重复编写脆弱脚本。

---

## 1.1 目标与价值主张

### 产品目标

- 打通 `scan -> map -> run`，在目标站点类型上稳定执行核心流程。
- 首次抽象后，在复跑场景的 `cost/speed/accuracy` 超越 `browser-use` 与 `stagehand`。
- 在多站点与高并发场景下保持低波动、可审计、可维护。
- 让同类网站能力模板可复用，降低新增站点接入成本。

### 价值主张（对用户）

- **一次抽象，持续复用**：将网站能力固化为 `site.dsl.json`，避免重复从零编排。
- **低成本可预测执行**：运行期以确定性 Runner 为主，降低模型调用依赖与 token 消耗。
- **更高稳定性**：selector 候选、fallback、重试和断言统一治理，降低脚本脆弱性。
- **更快交付自动化**：首次编码效率高于手写 Playwright，且后续改版恢复更快。

### 量化目标（建议）

- **首次编码效率**：`T_first_script` 相比手写 Playwright 缩短 `>= 40%`。
- **复跑成本**：`cost_per_success` 相比 `browser-use/stagehand` 降低 `>= 50%`。
- **复跑速度**：`P95 latency` 相比 `browser-use/stagehand` 降低 `>= 30%`。
- **准确率**：核心流程 `first_pass_success_rate >= 90%`，重试后 `>= 96%`。
- **稳定性**：`flaky_rate <= 5%`，页面改版后 `MTTR <= 1` 工作日。

---

## 2. 核心理念

项目以“能力复用”而不是“脚本复用”为中心，采用分层抽象：

- **Signal 层**：DOM、ARIA、URL、网络请求、事件、状态变化。
- **Element 层**：可交互元素及其 selector 候选、稳定性评分。
- **Capability 层**：登录、搜索、筛选、提交、下单等语义能力。
- **Flow 层**：能力组合形成业务流程（含前置条件、重试、回滚）。

DSL 不是简单 selector 字典，而是“可执行能力图”。

---

## 3. 系统架构

### 3.1 Scanner（扫描器）

输入：`baseUrl`、扫描配置（深度、白名单路径、登录态等）  
输出：`scan-artifacts/*` 原始证据

职责：

- 站点拓扑发现（页面可达图）
- 可交互元素提取（button/input/link/form）
- 动作后状态变化记录（DOM/URL/网络）
- 多环境扫描（可选：不同 viewport/语言/登录态）

建议技术：Playwright + DOM 观察器 + 网络监听

### 3.2 Abstractor（抽象器）

输入：扫描证据  
输出：`capability-graph.json`

职责：

- 元素聚类与同义入口归并
- selector 稳定性打分
- 动作语义识别（search/login/filter 等）
- 前置条件与后置结果抽取

### 3.3 Mapper（DSL 生成器）

输入：`capability-graph.json`  
输出：`site.dsl.json`

职责：

- 生成页面定义（Page）
- 生成元素定义（Element/Component）
- 生成能力定义（Capability）
- 生成流程定义（Flow）
- 写入运行策略（等待、重试、fallback）

### 3.4 Runner（执行器）

输入：`site.dsl.json` + 运行参数  
输出：执行结果 / 生成代码（Playwright）

职责：

- DSL 解析（Parser）
- 中间表示转换（IR）
- 执行计划生成（Planner）
- Playwright 执行或代码生成（Codegen）
- 失败诊断与反馈（便于 selector 自愈）

---

## 4. DSL 设计（v0.1）

采用 JSON-only，配合 JSON Schema 做强校验与版本兼容管理。

### 4.1 DSL 结构

- `site`: 站点元信息（id/baseUrl）
- `pages`: 页面定义（URL pattern + 元素）
- `capabilities`: 原子能力定义
- `flows`: 能力编排流程
- `policies`: 运行策略（超时、重试、fallback）

### 4.2 TypeScript 类型定义（核心）

```ts
export type Selector =
  | { css: string }
  | { aria: string }
  | { text: string }
  | { xpath: string }

export interface ElementDef {
  selectors: Selector[]
  stableScore?: number // 0~1
}

export interface PageDef {
  id: string
  urlPattern: string
  elements: Record<string, ElementDef>
}

export type Step =
  | { action: 'click'; target: string }
  | { action: 'fill'; target: string; args: { value: string } }
  | {
      action: 'waitFor'
      condition: {
        any?: Array<{ urlMatches?: string; elementVisible?: string }>
      }
    }

export interface CapabilityDef {
  id: string
  page: string
  preconditions: string[]
  steps: Step[]
}

export interface SiteDslV010 {
  dslVersion: '0.1.0'
  site: { id: string; baseUrl: string }
  pages: PageDef[]
  capabilities: CapabilityDef[]
  flows?: Array<{
    id: string
    use: Array<{ capability: string; with?: Record<string, string> }>
  }>
  policies?: { timeoutMs?: number; retry?: { maxAttempts?: number } }
}
```

### 4.3 JSON 标准与校验规范

- 基础 JSON 语法遵循 RFC 8259。
- Schema 采用 JSON Schema Draft 2020-12。
- 根 schema 建议命名：`schema/site-dsl.schema.json`。
- 每份 DSL 文件都包含版本字段：`dslVersion`（例如 `0.1.0`）。
- 加载链路：`parse JSON -> schema validate -> semantic validate -> IR`。

### 4.4 示例（JSON）

```json
{
  "dslVersion": "0.1.0",
  "site": {
    "id": "example-shop",
    "baseUrl": "https://example.com"
  },
  "pages": [
    {
      "id": "home",
      "urlPattern": "^https://example.com/$",
      "elements": {
        "searchInput": {
          "selectors": [
            { "css": "input[name='q']" },
            { "aria": "Search products" }
          ],
          "stableScore": 0.92
        },
        "searchButton": {
          "selectors": [
            { "css": "button[type='submit']" },
            { "text": "Search" }
          ]
        }
      }
    }
  ],
  "capabilities": [
    {
      "id": "search_product",
      "page": "home",
      "preconditions": [],
      "steps": [
        {
          "action": "fill",
          "target": "searchInput",
          "args": { "value": "${query}" }
        },
        {
          "action": "click",
          "target": "searchButton"
        },
        {
          "action": "waitFor",
          "condition": {
            "any": [
              { "urlMatches": "/search" },
              { "elementVisible": "resultsList" }
            ]
          }
        }
      ]
    }
  ]
}
```

---

## 5. Selector 稳定性与自愈策略

### 5.1 选择优先级（建议）

1. `data-testid` / `data-qa`
2. `aria role + accessible name`
3. 语义稳定文本（多语言需映射）
4. 结构化 CSS selector
5. XPath（最后兜底）

### 5.2 运行时 fallback

- 同一元素维护 selector 候选列表 + 评分。
- 主 selector 失败时按评分降级尝试。
- 降级成功后记录反馈，供后续 DSL 更新（人审后合并）。

---

## 6. Runner 设计要点（Playwright）

建议采用三步管线：

1. `DSL -> IR`：将 DSL 动作标准化；
2. `IR -> Plan`：注入等待、重试、分支策略；
3. `Plan -> Playwright`：执行或生成代码。

标准动作集建议：

- `click` / `fill` / `select` / `upload` / `press`
- `waitFor(url|selector|network|state)`
- `assert(visible|contains|count|url)`
- `extract(text|attr|json)`

---

## 7. CLI 设计（建议）

```bash
bgraph scan --url https://example.com --out ./artifacts
bgraph map --in ./artifacts --out ./site.dsl.json
bgraph run --dsl ./site.dsl.json --flow guest_search_flow --input ./input.json
bgraph codegen --dsl ./site.dsl.json --target playwright --out ./generated
```

---

## 8. 目录结构（Monorepo）

```text
browsergraph/
  packages/
    scanner/
    abstractor/
    dsl/
    runner/
    codegen-playwright/
    cli/
  examples/
    demo-shop/
      site.dsl.json
      input.json
  docs/
    dsl-spec.md
    architecture.md
```

---

## 9. Feature List（Checklist）

- [ ] 无登录静态站点扫描
- [ ] 登录态站点扫描（Cookie/Session 注入）
- [ ] SPA 路由识别与状态跟踪
- [ ] 页面拓扑发现（站点可达图）
- [ ] 可交互元素提取（button/input/link/form）
- [ ] selector 候选生成与稳定性评分
- [ ] 页面定义自动生成（`pages`）
- [ ] 能力定义自动生成（`capabilities`）
- [ ] 流程编排生成（`flows`）
- [ ] JSON Schema 校验（Draft 2020-12）
- [ ] 语义校验（引用完整性、循环依赖检测）
- [ ] DSL 解析器（`site.dsl.json -> IR`）
- [ ] 执行计划器（等待/重试/分支/fallback）
- [ ] Playwright Runner 执行引擎
- [ ] Playwright 代码生成器（可读可改）
- [ ] 统一动作集（click/fill/select/upload/press/waitFor/assert/extract）
- [ ] 失败分类与错误诊断报告
- [ ] 执行报告输出（成功率、失败节点、耗时、重试）
- [ ] selector 自愈建议生成（候选更新）
- [ ] DSL 更新补丁输出（人工审核合并）
- [ ] 通用能力本体（Auth/Search/Filter/Paginate）
- [ ] 跨同类网站能力迁移机制
- [ ] Benchmark 工具链（config/results/report/replay）
- [ ] CI 门禁（schema + semantic + 示例回归）

---

## 10. 风险与应对

- **动态页面不稳定**：引入多信号等待（DOM + URL + Network）。
- **反爬与风控**：限制扫描频率，支持人工接管关键步骤。
- **多语言文案变化**：依赖 aria/test-id 优先，文本作为辅助锚点。
- **DSL 复杂度膨胀**：分层 schema + 能力模板化。

---

## 11. 开源治理建议

- License：Apache-2.0（企业友好）
- 文档优先：`dsl-spec`、`architecture`、`how-to-add-site`
- 建立 benchmark：按站点类型统计执行成功率和稳定性
- issue 标签：`good first issue`、`capability-template`、`selector-heal`

---

## 12. 下一步落地建议

1. 先实现 `scan -> map -> run` 的最小闭环（单站点）。
2. 固化 DSL schema（先少量字段，保证稳定）。
3. 增加 `search/login` 两个能力模板，验证复用价值。
4. 引入执行报告与失败回放，形成迭代数据基础。

---

## 13. 方案对比：DSL-First vs browser-use vs stagehand

本节用于技术选型决策，重点比较 `cost`（token 花费）、`accuracy`（成功率/稳定性）、`speed`（端到端时延）。

### 13.1 对比结论（简版）

- 一次性任务、探索未知页面：`browser-use` / `stagehand` 往往更快启动。
- 重复执行、高并发、生产稳定性：本方案（DSL-First）整体更优。
- 长期总成本：本方案通常更低（一次建模，多次确定性执行）。

### 13.2 指标对比（定性）

| 维度               | DSL-First（本方案）                           | browser-use                                  | stagehand                               |
| ------------------ | --------------------------------------------- | -------------------------------------------- | --------------------------------------- |
| Cost（token）      | 前期高，后期低；复用越多越省                  | 按步骤持续消耗，长流程成本高                 | 通常低于纯 Agent 探索，但仍依赖模型决策 |
| Accuracy（稳定性） | 上限高；可通过 schema、fallback、自愈持续提升 | 探索能力强，但结果受页面噪声和提示词波动影响 | 工程化较好，稳定性中等偏上              |
| Speed（首跑）      | 需先扫描建模，冷启动较慢                      | 冷启动快                                     | 冷启动较快                              |
| Speed（复跑）      | 热启动快；流程固定后速度稳定                  | 频繁决策导致时延波动                         | 视任务复杂度而定，通常中等              |
| 可维护性           | 最高；能力与执行解耦                          | 中等；逻辑散落于任务描述/脚本                | 中等偏上                                |
| 可移植性           | 高；可扩展多执行后端                          | 偏任务级，不强调能力资产                     | 偏 Playwright 生态                      |

### 13.3 成本模型（用于评估 ROI）

- `browser-use / stagehand`（任务时模型参与较多）  
  近似成本：`O(N * steps * llm_calls)`
- `DSL-First`（先建模后执行）  
  近似成本：`O(scan + map) + O(N * deterministic_run)`

其中 `N` 为重复执行次数。`N` 越大，DSL-First 相对优势越明显。

### 13.4 适用场景建议

- 选择 `browser-use`：快速 PoC、一次性任务、开放探索为主。
- 选择 `stagehand`：希望快速接入 Playwright 生态，同时保留部分智能交互能力。
- 选择 `DSL-First`：要做平台化、可复用能力资产、可审计与长期维护。

### 13.5 推荐落地策略（混合架构）

建议采用混合路径，兼顾首发速度与长期收益：

1. 用 `browser-use` / `stagehand` 做扫描与探索（发现页面能力）。
2. 将发现结果沉淀到 `site.dsl.json`（标准能力地图）。
3. 生产执行统一走 Runner（Playwright，确定性执行）。
4. 失败样本回写 DSL（selector 候选更新与稳定性评分）。

该策略可以在不牺牲迭代速度的前提下，逐步建立可复用的自动化资产。

---

## 14. Benchmark 方案（cost / accuracy / speed）

本节用于验证本方案是否在“首次抽象后”超越 `browser-use`、`stagehand`，以及在“首次编码效率”上超越手写 Playwright。

### 14.1 测试对象与基线

- **方案 A（本方案）**：`Scanner + Mapper + DSL Runner`。
- **方案 B**：`browser-use`（按官方推荐配置）。
- **方案 C**：`stagehand`（按官方推荐配置）。
- **方案 D**：人工手写 Playwright（同等经验开发者）。

### 14.2 测试集设计

- 站点类型：至少 3 类（如电商、文档站、后台 CRUD）。
- 每类站点：至少 5 个站点，总计不少于 15 个站点。
- 每站任务：至少 5 条核心流程（如登录、搜索、筛选、提交、分页）。
- 执行轮次：
  - **冷启动**：首次执行（无缓存、无历史反馈）。
  - **热启动**：首次抽象完成后重复执行 100 次。
  - **漂移测试**：模拟页面小改版后再执行 30 次。

### 14.3 指标定义（统一口径）

- **Cost**
  - `cost_per_success`：成功任务平均成本（美元或 token）。
  - `cost_per_1000_tasks`：每千次任务总成本。
  - `break_even_runs`：本方案总成本低于对手的最小复跑次数。
- **Speed**
  - `T_first_script`：从输入 URL 到首个可运行流程产出的耗时。
  - `latency_p50/p95/p99`：单任务端到端时延分位数。
  - `throughput`：固定时间窗口内完成任务数量。
- **Accuracy**
  - `first_pass_success_rate`：首轮执行成功率。
  - `retry_success_rate`：重试后成功率。
  - `flaky_rate`：同输入多次运行结果不一致比例。
  - `task_completeness`：输出结果字段完整率。

### 14.4 实验流程

1. 固定测试环境（网络、机器规格、浏览器版本、并发配置）。
2. 为四种方案分别执行同一测试集，记录全量日志与运行轨迹。
3. 冷启动场景对比首跑结果与首跑成本。
4. 热启动场景重复 100 次，统计稳定性、成本与时延分位数。
5. 漂移场景注入页面改动（文本、属性、DOM 层级）并复测恢复能力。
6. 输出汇总报告与显著性检验结论。

### 14.5 公平性与控制变量

- 所有方案使用同一站点列表、同一任务定义、同一成功判定标准。
- 统一超时与重试上限，禁止单方案使用额外人工干预。
- 模型相关方案需固定模型版本、温度和上下文窗口策略。
- 记录并公开失败样本，避免只统计成功案例。

### 14.6 胜负阈值（建议）

本方案判定“达标领先”需同时满足：

- **对 AI 爬虫（B/C）**：
  - 热启动 `cost_per_success` 至少低 `30%`；
  - 热启动 `latency_p95` 至少低 `20%`；
  - `first_pass_success_rate` 不低于对手，且 `retry_success_rate` 更高。
- **对手写 Playwright（D）**：
  - `T_first_script` 至少低 `30%`；
  - 页面小改版后的恢复耗时至少低 `40%`；
  - 在相同人力下可覆盖更多站点/流程。

### 14.7 交付物

- `benchmark/config.json`：测试集与环境配置。
- `benchmark/results/*.json`：原始指标与日志索引。
- `benchmark/report.md`：结论与图表（含失败分析）。
- `benchmark/replay/`：关键失败用例回放脚本或快照链接。
