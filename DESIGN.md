# Harness² 设计文档

> 本文基于仓库源码（`harness2/` 包、`docs/`、`pyproject.toml`、`tests/`）整理，描述项目的目标、架构、数据流与关键设计决策。

## 1. 项目概述

Harness²（harness2）是一个面向开放世界的**递归智能体套件优化框架**（Recursive Agent Harnessing）。这里的 "harness" 指围绕模型使其成为智能体的整个外部系统：系统提示、护栏、记忆、工具指引、子智能体、技能、脚本与插件——共 8 个可编辑组件（c1..c8）。"Harnessing" 则是持续优化这套系统的过程。

核心思想是在**冻结模型权重**的前提下，通过两层递归提升智能体在新任务上的表现：

- **任务级递归（Task-level recursion）**：对当前任务执行对比式的 **Propose–Probe–Compose** 循环——提议对 harness 的编辑（不变量维度 INV / 领域维度 DOM 两个方向），用候选 harness 实际探测执行效果，再基于执行间的行为对比合成更优 harness。全程不依赖真值（ground-truth）反馈。
- **Harnessing 级递归（Harnessing-level recursion）**：每个任务完成后，把被采纳的 harness 与有证据支撑的经验教训写入按领域组织的持久化**经验库（Library）**，用于指导后续任务的优化。

框架是即插即用的：支持 **parallel / sequential 两种递归模式**、**OpenCode / Codex 两种执行底座（substrate）**、**JobBench / LAB / WorkBuddy-Bench 三个专业基准**，并且整个方法与具体基准、具体智能体框架完全解耦。

## 2. 整体架构与模块划分

系统由三条正交的轴组成：**方法（method）、底座（substrate）、基准（benchmark）**，由 `core/spec.py` 中的协议（Protocol）与值对象（`BenchSpec`、`ChannelSpec`、`RunView`）作为接缝粘合。

```
CLI 层  (harness2/cli)         harness2 / harness2-judge / harness2-report / harness2-setup
                               / harness2-check / harness2-demo / harness2-bridge
   │
方法层  (harness2/core)         调度、递归、证据、提议、选择、库、裁判、报告
   │          │
   │          └── 接缝: BenchmarkAdapter / BenchSpec / RunView  (core/spec.py)
   │
底座层  (harness2/substrate)    OpenCode / Codex 两种智能体框架适配
基准层  (harness2/benchmarks)   JobBench / LAB / WorkBuddy 三个基准适配器
```

### 2.1 CLI 层（`harness2/cli/`）

命令行入口均为对 core 的薄封装（argparse），对应 `pyproject.toml` 的 7 个 console scripts：

- `run_sweep.py`（`harness2`）：一次优化扫描的主入口——遍历任务划分、按领域并行、按流顺序执行任务；支持 `--dry-run` 预演；真实运行前离线检查裁判可用性（避免扫完才发现无法评分）。
- `judge.py` / `report.py`：对已采纳的选择进行评分与汇总，要求与优化运行使用完全相同的 bench/executor/k/mode/seed。
- `setup.py` / `check.py`：基准依赖安装与预检（`install.py`、`preflight.py` 提供实现）。
- `demo.py` / `bridge.py`：无需模型凭据的本地确定性演示；LiteLLM 代理桥接。

### 2.2 方法层（`harness2/core/`）

| 模块 | 职责 |
|---|---|
| `spec.py` | **唯一接缝**：`BenchSpec`（静态描述一个 (benchmark, substrate) 单元，纯值不含代码路径）、`BenchmarkAdapter` / `SubstrateLike` 协议、`ChannelSpec`（proposer/selector 通道到模型的映射）、`RunView`（一次 rollout 的结果视图，永不携带分数） |
| `schedule.py` | 命名空间（`ns()`：`<bench>-<substrate>-<mode><k>[-s<seed>][-<tag>]`）、种子化任务流、驱动器（单任务 / 单领域 / 整个 sweep） |
| `recursion.py` | 两种任务级递归模式的编排：`run_parallel`（宽度）与 `run_sequential`（深度），每阶段含一次无效提议重试，重试失败则回退继承当前 harness |
| `proposer.py` | 提议者（improver）会话：在隔离上下文工作区中给模型文件读写能力，从 `context/harness` 收割编辑结果 |
| `context.py` | **证据树**——模型获取证据的唯一途径（见 §3.2） |
| `tree.py` | c1..c8 组件树的读/写/校验/差分，按 `SurfaceSpec` 参数化；编辑契约是"省略即继承" |
| `selection.py` | 从匿名化的最终产出中做选择，并对其优化证据做反思（reflection） |
| `library.py` | 按领域组织的经验库：`<root>/<run-tag>/<mode>/<domain>/`，存放完整采纳树 + 经验教训 |
| `ledger.py` | 预算账本：强制恒等式 R = 4k rollout、P = 3k 改进阶段，含重复标签检测 |
| `judge.py` / `report.py` | 评分与汇总；裁判失败记为 `None`（从分母剔除），绝不计为 0 |
| `runmark.py` | 运行完成标记（`.adapt_complete`）+ harness 指纹，防止跨配置复用未完成的 rollout |
| `prompts.py` | 方法发送的所有提示词；基准文本只作为值（`BenchPrompts` 插槽）注入，不做分支 |

### 2.3 底座层（`harness2/substrate/`）

`base.py` 定义 `Substrate` 协议：`surface()`（c1..c8 可编辑面）、`materialise()`（把树落成 substrate 原生文件）、`execute()`、`read_trajectory()`、`preflight()`。

- `opencode.py`：OpenCode 底座，携带 `OPENCODE_SURFACE` 与三种安装模式（`workspace_agents_md` / `home_dir_jinja` / `workspace_config`），bun 运行的 CLI 契约与事件流轨迹渲染器。
- `codex.py`：codex-cli 底座，单一 `codex_root` 安装模式，`codex exec -C <ws>` 契约与线程项轨迹渲染器。部署差异体现为 `Install.extras` 中的值，而非子类。

### 2.4 基准层（`harness2/benchmarks/`）

每个基准一个适配器类，实现 `BenchmarkAdapter` 协议的全部方法（任务、拆分、提示词、启动器体、交付物、评分）：

- `jobbench.py`：JobBench，rollout 按 LABEL 寻址（运行存储按数据集寻址、全设置共享）。
- `lab.py`：LAB（harvey-labs），上游工具以子进程运行；在向 improver 暴露任务数据前剥离评分标准。
- `workbuddy.py`：WorkBuddy-Bench，执行是 Harbor 批处理；模型可见的交付物一律经 `wb_render.py`。
- `tasks.py`：任务发现与用户自定义划分（不内置任务清单）。

### 2.5 支撑目录

- `third_party/`：基准运行器叠加层（jb_runner、lab_runner、wb_runner）与补丁（`patches/*.diff`）。
- `dependencies/`：经 pin 的外部依赖安装目标（`config.DEPENDENCY_PINS` 固定 commit）。
- `scripts/`：安装、划分、渲染等辅助脚本；`tests/test_workflow.py`：覆盖执行、证据与采纳契约的回归测试。

## 3. 核心数据流

### 3.1 一次优化运行（单任务）

以 `harness2 --bench lab --k 1 --mode parallel` 为例：

1. **调度**：`schedule` 按 seeded stream 选定任务，建立运行命名空间与工作目录 `runs/<ns>/<task>/`。
2. **基线执行**：跑 k 个共享基线 rollout（`harness_dir=None` 即未修改的基础 harness），记入账本。
3. **Propose**：并行提出 k 个 INV（不变量）候选 + k 个 DOM（领域）候选——每个候选是一个 improver 会话，读取证据树、编辑 `context/harness/` 下的 c1..c8 树。
4. **Probe**：每个候选树经校验（`tree.validate` + 适配器 `validate_tree`）后落盘为 harness 目录，实际执行一次 rollout，产出 `RunView`。
5. **Compose**：k 个合成阶段（parallel 模式为 `agg1..aggk`）读取共享证据 + 候选池 + 各阶段决策记录，合成最终 harness 并再各跑一次 rollout。
6. **选择与反思**：`selection` 从匿名化的最终产出中选择一个，并反思优化证据。
7. **Harnessing 级递归**：采纳的完整 harness 与经验教训写入该领域的经验库，供同领域后续任务检索。
8. **评分与报告**（独立命令）：`harness2-judge` 只对被选择的运行评分；`harness2-report` 汇总为基准报告指标。

账本在结束时断言恒等式：R = 4k（k 共享基线 + 2k 候选 + k 最终）、P = 3k（k INV + k DOM + k 合成），标签重复即失败——保证每个被计入的候选都真实产出过。

### 3.2 证据树（`context.py`）

证据以**文件**而非提示词拼接的方式送达模型，提示词只是一段简短的指针（`MISSION.md`），因此没有任何证据被截断以塞进 prompt：

```
context/task.md                  任务原文（已剥离评分标准）
context/rollouts/<label>.md      完整轨迹，共享 [ACTION]/[OBSERVATION]/[ERROR] 语法
context/deliverables/<label>.md  交付物（经裁判自己的提取器）
context/harness/<rel>            正在被编辑的 c1..c8 树
context/library/INDEX.md         同领域早期任务及其完整 harness
context/pool/<label>/            候选 harness + 其 rollout（仅 aggregate/select 阶段）
MISSION.md                       要做什么
```

### 3.3 Harness 树与编辑契约

c1..c8 八组件（system prompt、guardrails、memory、tool guidance、subagents、skills、scripts、plugins）表示为 `dict[str, str]` 的树，由 `SurfaceSpec` 描述各底座的路径、容量上限、frontmatter 规则与 README 种子。提议只发出**变更的部分**（省略即继承），校验失败的信息以可操作散文原样回喂给提议者做一次重试。

## 4. 关键技术设计决策

1. **三轴分离（方法 × 底座 × 基准）**：`core` 只依赖 `BenchSpec` / `BenchmarkAdapter` / `SubstrateLike` 三个协议，新增基准或底座不需要改方法层；`spec.py` 的模块 docstring 明确了这是唯一的 benchmark seam。
2. **提示词做值不做分支**：`prompts.py` 中所有阶段提示词跨基准完全一致，基准差异只经 `BenchPrompts` 插槽注入；`BenchPrompts` 甚至静态禁止插槽文本中出现 "benchmark/evaluation/grader/rubric/score" 等词，从结构上防止评测泄漏进优化提示。
3. **证据即文件**：拒绝 prompt 内截断，完整轨迹与交付物以文件树呈现，规避"截断制造缺陷"的问题（`BenchSpec.obs_trunc` 下限 10,000 并在构造期强制）。
4. **预算恒等式作为自检**：`Ledger.assert_identity` 把 R=4k / P=3k 变成运行结束的硬断言，重复标签（两个阶段写同一 run id 导致静默覆盖）优先于计数被检查——把静默失败转化为显式崩溃。
5. **`RunView` 永不携带分数**：方法层与评测完全隔离；`judge.score` 只在 `core.judge` 中、且只在选择提交后被调用；裁判失败是 `None` 不是 0，也不缓存为 0。
6. **回退而非崩溃的改进阶段**：提议无效重试一次后回退继承当前 harness（`fell_back`），保证 k 预算下的流程总能推进；真实故障仍被记录并在 `ModeResult.errors` 中暴露。
7. **Run 完成标记 + 指纹**：`runmark` 显式记录"由这段代码、在这份 harness 指纹下完成"，输出存在不等于运行完成，跨配置的 rollout 永不复用——支撑幂等 resume。
8. **并发但确定性记账**：rollout 与提议阶段用 `ThreadPoolExecutor` 并发，账本在池排空后按调用方稳定顺序写入，使并发运行与串行运行记录的标签序列完全一致（可复现性）。
9. **配置即环境**：`config.py` 用 `.env` + shell 覆盖 + `HARNESS2_*` 变量管理全部部署设置，依赖按 commit pin 安装到 `dependencies/`，第三方叠加经补丁管理（`install.PATCHES`），保证可重复安装。
10. **资金安全前置检查**：sweep 真实运行前离线检查裁判可用性、`preflight` 探测依赖与运行时，"花钱之前"拦截可预见的失败。

## 5. 目录结构说明

```
.
├── harness2/                  # 主包
│   ├── core/                  # 方法层：调度、递归、证据、提议、选择、库、裁判、报告
│   ├── substrate/             # 智能体框架底座：opencode.py / codex.py / base.py(协议)
│   ├── benchmarks/            # 基准适配器：jobbench / lab / workbuddy / tasks
│   ├── cli/                   # 7 个 console script 的 argparse 薄封装
│   ├── config.py              # 部署设置、依赖 pin、模型解析
│   ├── install.py             # 可重复安装（依赖 + 运行器叠加补丁）
│   ├── preflight.py           # 基准依赖与运行时就绪检查
│   └── demo.py                # 无模型凭据的本地确定性演示适配器
├── third_party/               # 基准运行器叠加层与补丁（保留上游许可）
├── dependencies/              # pin 过的外部依赖安装目标（非源码）
├── scripts/                   # 安装 / 划分 / 渲染辅助脚本
├── tests/                     # test_workflow.py：执行、证据、采纳契约的回归测试
├── docs/                      # SETUP.md / BENCHMARKS.md / EXPERIMENTS.md
├── assets/                    # 图表与演示素材
├── pyproject.toml             # 打包与 7 个 console scripts 定义
└── DESIGN.md                  # 本文档
```

## 6. 后续演进建议

1. **扩大底座与基准矩阵**：`Substrate` / `BenchmarkAdapter` 协议已经稳定，可低成本接入新的智能体框架与专业基准；建议为协议补充一致性测试夹具（现有 `tests/test_workflow.py` 只覆盖 workflow 契约，可抽象出适配器合规性套件）。
2. **经验库检索策略**：当前检索是"按领域目录构造式"的（同目录即同领域，整树携带、不摘要），任务量增大后可引入向量/主题索引做跨领域迁移实验，但需保持可复现性（库内容与任务流顺序绑定）。
3. **并发与资源治理**：`PHASE_WIDTH`、`PROPOSER_CONCURRENCY` 为静态常量，长 sweep 可暴露为调度参数并加全局限额（API 速率、Docker 并发、磁盘），同时把账本扩展为跨任务聚合视图。
4. **观测与成本归因**：账本已按 rollout/session 记录 USD 与缓存命中，可进一步输出按阶段/维度的成本分解与 `fell_back` 率看板，帮助定位无效提议的模式。
5. **评测防泄漏自动化**：`BenchPrompts` 的禁词检查是静态护栏，可增加运行期网络限制的校验（README 已提示 executor 网络不受限、评分标准可能在线可达），将防泄漏从约定升级为强制检查。
6. **测试覆盖**：`benchmarks/` 三个适配器（合计 3000+ 行）缺乏直接单测，建议对拆分逻辑、交付物提取、评分解析补充基于 fixtures 的测试，降低外部依赖版本漂移带来的回归风险。
