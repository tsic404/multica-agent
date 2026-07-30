---
name: econ-abm-qa-testing
description: EconABM 项目 QA 验收测试 v2.1。验证 Mesa + Solara ABM 仿真的涌现行为(P0)、可复现性(P0)、UI 全组件渲染 35 检查点(P1)、Manifest 结构(P1)、数据采集(P2)、性能(P2)、Docker(P3)。三层结构：SKILL.md(测试计划) + references/(详细规格) + scripts/(可执行脚本) + templates/(裁决模板)。触发：EconABM 项目 issue QA 阶段、用户要求验收测试。
version: 2.2.3
last_updated: 2026-06-10
---

# ABM QA Testing — EconABM

⚠️ **流程表阶段 vs Skill 步骤**：流程表中的「阶段 ⑤」= 执行本 Skill 的**全部** Step 1–8，不仅仅是 Step 5。阶段编号和步骤编号是两个独立命名空间。

## 文件索引

| 文件 | 内容 |
|------|------|
| `SKILL.md`（本文件） | 测试计划、Step 概要、签发标准 |
| `references/emergence-spec.md` | P0 涌现行为详细规格、边界条件、判定逻辑 |
| `references/reproducibility-spec.md` | P0 可复现性详细规格 |
| `references/ui-components.md` | P1 UI 组件清单 + agent-browser 验证命令 + v2.0 设计专项 |
| `references/manifest-spec.md` | P1 Manifest 结构契约、字段校验规则 |
| `references/docker-deploy.md` | P3 Docker 部署、已知陷阱 |
| `references/vision-verification-guide.md` | Vision 验证方法论（多时间点动态演化，通用 ABM 模式） |
| `references/fabrication-evidence-2026-06-06.md` | Verity 虚构浏览器测试结果的日志证据（2026-06-06 诊断） |
| `references/ui-design-v2.md` | v2.0 UI 设计规格 — Educational 风格，组件布局、颜色、排版 |
| `references/models/wealth.md` | WealthModel 独立测试规格 |
| `references/models/wealth-saving.md` | Chakraborti 储蓄率实验规格（λ 参数） |
| `references/models/wealth-tax.md` | Dragulescu-Yakovenko 税收再分配实验规格（τ 参数） |
| `references/models/manifest_test.md` | ManifestTest 参数类型验证模型规格 |
| `templates/verdict.md` | 裁决输出模板 |
| `scripts/test_emergence.py` | P0 涌现行为（遍历 Registry） |
| `scripts/test_reproducibility.py` | P0 可复现性（遍历 Registry） |
| `scripts/test_manifest.py` | P1 Manifest + Registry（遍历 Registry） |
| `scripts/test_chart_binding.py` | P1 图表数据绑定（遍历 Registry） |
| `scripts/test_chart_rendering.py` | P1 图表渲染冒烟（遍历 Registry） |
| `scripts/test_data_integrity.py` | P2 数据采集（遍历 Registry） |
| `scripts/test_performance.py` | P2 性能基准（遍历 Registry） |

## 文件结构

```
econ-abm-qa-testing/
├── SKILL.md                         ← 本文件：测试计划、优先级分层、签发标准
├── references/
│   ├── emergence-spec.md            ← P0 涌现行为详细规格（预期值、边界条件）
│   ├── reproducibility-spec.md      ← P0 可复现性详细规格（seed 策略、容忍范围）
│   ├── ui-components.md             ← P1 UI 组件清单 + agent-browser 验证命令
│   ├── manifest-spec.md             ← P1 Manifest 结构契约
│   ├── docker-deploy.md             ← P3 Docker 部署规格
│   ├── vision-verification-guide.md ← Vision 验证方法论
│   ├── fabrication-evidence-2026-06-06.md ← 虚构测试结果日志证据
│   └── models/                      ← 每个注册模型的独立测试规格
│       ├── wealth.md                ← WealthModel 预期 Gini/Pareto/性能阈值
│       ├── wealth-saving.md         ← Chakraborti 储蓄率实验（λ 参数）
│       ├── wealth-tax.md            ← Dragulescu-Yakovenko 税收再分配实验（τ 参数）
│       └── manifest_test.md         ← ManifestTest 参数类型验证模型
├── templates/
│   └── verdict.md                   ← 裁决输出模板
└── scripts/
    ├── test_emergence.py            ← P0 涌现行为（遍历 Registry 所有模型）
    ├── test_reproducibility.py      ← P0 可复现性（遍历 Registry 所有模型）
    ├── test_chart_binding.py        ← P1 图表数据绑定（遍历 Registry 所有模型）
    ├── test_chart_rendering.py      ← P1 图表渲染冒烟（遍历 Registry 所有模型）
    ├── test_manifest.py             ← P1 Manifest + Registry
    ├── test_data_integrity.py       ← P2 数据采集 + 财富守恒（遍历模型）
    └── test_performance.py          ← P2 性能基准（遍历模型）
```

## 测试优先级

| 优先级 | 类别 | Step | 阻断 |
|:--|:--|:--|:--|
| P0 | 涌现行为验证 | 2 | 是 |
| P0 | 可复现性验证 | 3 | 是 |
| P1 | UI 图表渲染验证 | 4 | 是 |
| P1 | Manifest 解析验证 | 5 | 是 |
| P2 | 数据采集验证 | 6 | 否 |
| P2 | 性能验证 | 7 | 否 |
| P3 | Docker 部署验证 | 8 | 否 |

## 测试矩阵

| 模型 | Registry key | Gini 范围 | Pareto Top10% | 性能 | create_kwargs |
|:--|:--|:--|:--|:--|:--|
| 基础财富 | wealth | > 0.25 | > 15% | ≥ 100 sps | — |
| 储蓄率 | wealth_saving | 0.10–0.35 | > 10% | ≥ 80 sps | `savings_rate=0.3` |
| 税收再分配 | wealth_tax | 0.05–0.25 | > 10% | ≥ 80 sps | `tax_rate=0.05` |
| 参数类型测试 | manifest_test | N/A | N/A | ≥ 80 sps | 不参与涌现测试 |

## Step 0：Pre-flight 检查（必须）

在执行任何测试前，验证关键工具可用性。**工具不可用 → QA_BLOCKED**，不得跳过。

```bash
# 0.1 agent-browser 可用性（Step 4 浏览器测试的硬依赖）
if ! command -v agent-browser &>/dev/null; then
    # Multica ACP agent 的 npm 全局路径常不在 PATH
    NPM_GLOBAL_BIN="$HOME/.npm-global/bin"
    if [ -f "$NPM_GLOBAL_BIN/agent-browser" ]; then
        export PATH="$NPM_GLOBAL_BIN:$PATH"
        echo "agent-browser found at $NPM_GLOBAL_BIN — added to PATH"
    else
        echo "FATAL: agent-browser not found. Install: npm install -g agent-browser@0.27.0"
        echo "Then ensure $NPM_GLOBAL_BIN is in PATH"
        exit 1  # → QA_BLOCKED
    fi
fi

# 0.2 验证 agent-browser 实际可执行
agent-browser --version || { echo "FATAL: agent-browser installed but cannot execute"; exit 1; }

# 0.3 验证 QA skill 已加载（从 .agent_context 读取）
QA_SKILL=".agent_context/skills/econ-abm-qa-testing/SKILL.md"
if [ ! -f "$QA_SKILL" ]; then
    echo "WARNING: $QA_SKILL not found — browser test steps may be incomplete"
    echo "Available skills:"
    ls .agent_context/skills/ 2>/dev/null || echo "  (none)"
fi

echo "Pre-flight: OK"
```

## Step 1：环境准备

```bash
# 使用 Multica repo checkout（走 daemon bare clone cache，不走网络全量 clone）
multica repo checkout https://github.com/tsix404/econ-abm.git || exit 1
export ECONABM_PATH="$PWD/econ-abm"
cd "$ECONABM_PATH"

# Python 版本约束：3.13.x（< 3.14）
python3 -c "import sys; v=sys.version_info; assert v >= (3,13) and v < (3,14), f'Python 3.13 required, found {sys.version}'" || exit 1

uv pip install -r requirements.txt
solara run app.py --host 0.0.0.0 --port 8765 &
sleep 3
curl -s -o /dev/null -w '%{http_code}' http://localhost:8765
```

## Step 2：涌现行为验证（P0）

执行 `scripts/test_emergence.py`。遍历 Registry 中所有模型，通过 `create_kwargs` 传递模型特定参数。

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_emergence.py --test gini
ECONABM_PATH="$ECONABM_PATH" python scripts/test_emergence.py --test pareto
```

通过条件：wealth: Gini > 0.25；wealth_saving: Gini ∈ [0.10, 0.35]；wealth_tax: Gini ∈ [0.05, 0.25]。manifest_test 不参与涌现测试（它是参数类型验证模型，非经济学模型）。

## Step 3：可复现性验证（P0）

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_reproducibility.py
```

通过条件：相同 seed → 相同结果；不同 seed → 不同结果；seed=0 正常工作。

## Step 4：UI 图表渲染验证（P1）

详见 `references/ui-components.md`。以下为 Step 概要。

### 4.1 服务器可达性

```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:8765  # 应返回 200
```

### 4.2 图表数据绑定验证

验证每个 visualization 的 metric/x_metric/y_metric 在 DataCollector DataFrame（model 级 + agent 级）中存在。**遍历 Registry 中所有模型**。

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_chart_binding.py
```

### 4.3 render_chart() 冒烟测试

验证每个 visualization 的 `render_chart()` 返回有效图表元素。**遍历 Registry 中所有模型**。

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_chart_rendering.py
```

### 4.4 浏览器交互测试（自动化）

使用 agent-browser CLI 对 Solara 页面进行真实交互验证。前提：Solara 已在 `http://localhost:8765` 运行。

#### 4.4.1 页面加载与初始状态

```
操作:
  agent-browser open http://localhost:8765
  agent-browser snapshot
验证:
  - 页面标题含 "EconABM"
  - 存在模型下拉选择器
  - 存在 Run 按钮或 ▶ 标识
  - agent-browser console → 无 JS 错误
```

#### 4.4.2 模型选择

```
操作:
  - agent-browser snapshot → 找到模型选择器 ref
  - agent-browser click <ref> 展开下拉
  - agent-browser snapshot → 选中 wealth
验证:
  - 参数面板出现（n_agents/init_wealth/seed 控件可见）
  - Agent Space 区域可见
```

#### 4.4.3 参数调整

```
操作:
  - snapshot 中找到 n_agents 滑块 ref
  - agent-browser click 调整值
验证:
  - 参数值已变化
  - 步数计数器重置为 0（模型已重建）
```

#### 4.4.4 仿真控制（Run / Pause / Reset）

```
操作:
  - agent-browser click ▶ Run
  - 等待 3 秒
  - agent-browser snapshot
验证:
  - 步数计数器 > 0
  - 按钮变为 ⏸ Pause
  - Agent Space 圆圈数量 = n_agents

  - agent-browser click ⏸ Pause
  - agent-browser snapshot
验证:
  - 按钮恢复为 ▶ Run
  - 步数计数器停止递增

  - agent-browser click ↺ Reset
  - agent-browser snapshot
验证:
  - 步数计数器 = 0
  - 按钮恢复为 ▶ Run
  - Agent Space 恢复初始状态
```

#### 4.4.5 图表渲染验证

```
操作:
  - agent-browser click ▶ Run，等待 10 秒积累数据
  - agent-browser click ⏸ Pause
  - agent-browser snapshot --full
验证:
  - 不含 "waiting for data"
  - 含 "Gini" 文本（图表标题已渲染）
  - 含 "Wealth" 文本（直方图标题已渲染）
  - agent-browser screenshot → 图表有数据线/柱而非占位文本
```

#### 4.4.6 浏览器控制台检查

```
操作:
  - agent-browser console
验证:
  - 无 uncaught error
  - 无 WebSocket 断开错误
  - 如有关键错误，记录并判定 QA_FAILED
```

### 4.5 参数面板综合测试

验证 manifest 中每种控件类型正确渲染，值变更触发模型重建。

```
操作:
  - 对每个参数类型逐一测试：
    slider: agent-browser snapshot → 找到滑块 ref → 拖动 → snapshot 确认值变化 + 步数归零
    int: agent-browser snapshot → 找到输入框 ref → agent-browser type <ref> <新值> → snapshot 确认值变化 + 步数归零
    select: agent-browser snapshot → 展开下拉 → 切换选项 → 确认界面更新
    checkbox: agent-browser click → 确认切换状态 + 步数归零
验证:
  - 所有 manifest 声明的参数都有对应控件
  - 控件修改后步数计数器归零（模型重建）
  - 滑块 min/max 边界被 UI 尊重（不能超出范围）
  - 输入非法值（如 N=abc）时 UI 不崩溃
```

### 4.6 Agent 空间渲染验证

```
操作:
  - agent-browser open http://localhost:8765 → 选 wealth 模型
  - agent-browser snapshot → 确认 Agent Space 区域存在
  - 记录初始圆圈数量
验证 (初始):
  - 圆圈数量 = manifest 中 N 的默认值
  - 圆圈均匀分布（初始财富相同 = 大小相近）

  - agent-browser click ▶ Run → 等待 5 秒
  - agent-browser click ⏸ Pause
  - agent-browser screenshot
验证 (运行后):
  - 圆圈数量不变（N 恒定）
  - 圆圈大小出现分化（wealth 不等的 Agent 圆圈大小不同）
  - 小财富 Agent 圆圈可见（非零尺寸）
  - 无 Agent 重叠/消失

边界:
  - N=1: 单个圆圈正常渲染，不崩溃
  - N=500: 所有圆圈渲染，页面不冻结
```

### 4.7 控制栏深度测试

```
操作:
  - Step 按钮：agent-browser click Step → snapshot 确认步数 +1
  - 连续 Step 3 次 → snapshot 确认步数 +3，Gini 值逐步变化
  - 速度滑块：snapshot 找到速度 ref → 拖动到最慢/最快 → 确认 Run 时步进间隔改变
  - 按钮互斥：
    初始: ▶ Run 可见，⏸ Pause 不可见/禁用
    Run 后: ⏸ Pause 可见，▶ Run 不可见/禁用
    Pause 后: ▶ Run 可见，⏸ Pause 不可见/禁用
验证:
  - Step 单步准确（每次 +1，无跳跃）
  - 速度滑块范围合理（慢速可感知，快速流畅）
  - 按钮互斥逻辑正确
  - 步数计数器不倒退
```

### 4.8 模型切换测试

```
操作:
  - agent-browser open http://localhost:8765
  - snapshot → 展开模型下拉 → 选择 wealth
验证 (选 wealth):
  - 参数面板更新（显示 wealth 的 n_agents/init_wealth/seed）
  - Agent Space 渲染
  - 图表面板出现（Gini 时序 / Wealth 分布）

  - 如有第二个模型：snapshot → 展开下拉 → 选择
验证 (切换):
  - 参数面板切换到新模型的参数列表
  - Agent Space 重建（旧 Agent 清除，新 Agent 渲染）
  - 图表区域重建（旧图表清除）
  - 步数计数器归零

边界:
  - Registry 为空（models/__init__.py 无注册项）→ 下拉为空或显示提示，不崩溃
```

### 4.9 边界与错误状态测试

```
用例 1: N=1（单 Agent）
  - 参数面板设置 N=1
  - ▶ Run → 运行 100 步
  - 验证: 不崩溃，Gini 有定义，Agent 圆圈单个可见

用例 2: N=500（大量 Agent）
  - 参数面板设置 N=500
  - ▶ Run → 等待 10 秒
  - 验证: 页面不冻结，所有圆圈渲染（agent-browser screenshot 确认）
  - Step 基准 < 1 秒（500 agents × 1 step）

用例 3: 服务器宕机
  - 手动 kill Solara 进程
  - agent-browser open http://localhost:8765
  - 验证: 返回连接错误（非空白页面），agent-browser 不崩溃

用例 4: 全程 Console 零错误
  - 执行 4.4–4.9 过程中，每次 snapshot 后检查 agent-browser console
  - 验证: 无 uncaught TypeError/ReferenceError、无 Vega-Lite 渲染警告、
          无 WebSocket 1006/1001 断开
```

### 4.10 布局完整性检查

```
操作:
  - agent-browser open http://localhost:8765
  - agent-browser screenshot → 视觉检查
验证:
  - 左侧栏：模型选择器 + 参数面板 + 控制栏（三部分垂直排列，不重叠）
  - 右上：Agent 空间（圆圈集合可见）
  - 右侧/底部：图表面板（至少一个图表区域可见）
  - 无溢出/裁切（所有文字可读，无组件被截断）
  - agent-browser snapshot --full → 所有区域有内容（非空 div）
```

### 4.11 v2.0 设计专项验证（P1）

验证 v2.0 UI 重设计（Educational 风格）的核心改动已正确渲染。详见 `references/ui-design-v2.md`。

#### 4.11.1 Sidebar 三卡片布局

```
操作:
  - agent-browser open http://localhost:8765
  - agent-browser snapshot --full
验证:
  - 侧边栏宽度 ≈ 320px（桌面端）
  - 三张卡片从上到下排列：AboutCard → ParamsCard → ControlCard
  - 间距 12px，无重叠
  - AboutCard 含"关键洞察"列表 + 文献引用
  - ParamsCard 含 💡 "修改参数将重置仿真" 提示
  - ControlCard 含 Run / Step / Reset 三按钮 + Speed 滑块 + Step 大数字
```

#### 4.11.2 Agent Space Overlay Bar

```
操作:
  - ▶ Run → 等待 5 秒积累数据 → ⏸ Pause
  - agent-browser snapshot --full
验证:
  - Agent Space 底部有半透明 overlay bar
  - 含 N= 当前 Agent 数
  - 含 Gini= 值 + 趋势箭头（↑/↓/→）
  - 含 Step= 当前步数
  - agent-browser screenshot → 视觉确认 overlay bar 不遮挡圆圈
```

#### 4.11.3 指标卡片（Metrics）

```
操作:
  - 同上运行状态
  - agent-browser snapshot
验证:
  - 4 张卡片水平排列：Gini / Top10% / Wealth / Steps
  - Gini 卡片含趋势指示（↑ = #34D399, ↓ = #EF4444）
  - Wealth 卡片含守恒标记（✓ / ✗）
  - Steps 卡片显示 current / max 格式（如 1,247 / ∞）
```

#### 4.11.4 动态图表 Tab

```
操作:
  - agent-browser open http://localhost:8765
  - 切换到 wealth → 记录 Tab 数量
  - 切换到 wealth_saving → 记录 Tab 数量
  - 切换到 wealth_tax → 记录 Tab 数量
  - 切换到 manifest_test → 记录 Tab 数量
验证:
  - 每个模型的 Tab 数量 = manifest.visualizations 长度
  - wealth: 3 tabs (Gini/Wealth/Scatter)
  - wealth_saving: ≥ 3 tabs
  - wealth_tax: ≥ 3 tabs
  - manifest_test: 5 tabs (Gini/Mean/Std/Total/Histogram)
  - 切换 Tab 时图表内容变化，不崩溃
  - Tab 超出视口时可水平滚动
```

#### 4.11.5 暗色模式

```
操作:
  - agent-browser click 🌙（Header 暗色切换按钮）
  - agent-browser snapshot
验证:
  - 背景色变为深色（#0F172A 附近）
  - 卡片背景变为深灰蓝（#1E293B 附近）
  - 文字颜色可读（浅色文字）
  - Run 按钮变为亮绿（#34D399）
  - 再次点击切换回亮色模式
  - agent-browser screenshot → 视觉确认亮/暗切换流畅，无闪烁
```

#### 4.11.6 连接状态指示

```
操作:
  - agent-browser open http://localhost:8765
  - agent-browser snapshot
验证:
  - Header 中有连接状态指示器（🟢 或文字）
  - 正常情况显示 🟢 Connected 或等效文本
  - agent-browser console → 无 WebSocket 断开错误
```

#### 4.11.7 Step 单步按钮

```
操作:
  - 选 wealth 模型
  - agent-browser click Step
  - agent-browser snapshot
验证:
  - 步数计数器从 0 → 1
  - Agent Space 圆圈有可感知变化（screenshot 前后对比）
  - 连续点击 Step 3 次 → 步数 = 3
  - Run 状态下 Step 隐藏/禁用
```

#### 4.11.8 manifest_test 参数类型全覆盖

```
操作:
  - 选 manifest_test 模型
  - agent-browser snapshot
验证:
  - 参数面板含 5 种控件类型：
    slider: n_agents, init_wealth, int_param, float_param
    toggle/checkbox: bool_param
    text input: str_param
    select/dropdown: choice_param
    int input: seed
  - slider 的 min/max/step 来自 manifest 定义
  - 修改参数后步数归零（模型重建）
```

**通过条件（P1 阻断）**：
- 4.1–4.10 所有验证项通过
- 4.11.1–4.11.8 全部通过（v2.0 新增）
- "waiting for data" 未出现在页面中
- 浏览器控制台无阻断性错误
- 参数面板所有控件正常工作（含 5 种类型：int/float/bool/str/choice）
- Agent 空间正确渲染（overlay bar 可见）
- 暗色模式切换无闪烁/崩溃
- 动态 Tab 数量与 manifest.visualizations 一致

**⚠️ 反虚构验证（Step 4 执行后必做）**：

浏览器测试完成后，必须验证测试是真实执行而非虚构：

```bash
# 1. 确认截图文件实际存在（agent-browser screenshot 输出路径）
SCREENSHOT_DIR="$HOME/.agent-browser/tmp/screenshots/"
if [ -d "$SCREENSHOT_DIR" ]; then
    echo "Screenshots taken: $(ls "$SCREENSHOT_DIR" | wc -l) files"
    ls -la "$SCREENSHOT_DIR" | tail -5
else
    echo "WARNING: No screenshot directory found — browser tests may be fabricated"
fi

# 2. 确认 agent-browser 命令真实执行过（非仅 claimed）
#    检查 agent-browser 的 cookies 文件或 session 状态
if [ -f "$HOME/.agent-browser/cookies.json" ]; then
    echo "agent-browser session state: OK"
else
    echo "WARNING: No agent-browser session state — commands may not have executed"
fi
```

如果截图文件不存在、session state 缺失 → 浏览器测试**无效**，Step 4 应判定为 FAILED 并输出 **QA_BLOCKED**（环境问题）而非 QA_PASSED。

## Step 5：Manifest 解析验证（P1）

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_manifest.py
```

遍历 Registry 中所有模型，检查 manifest 结构完整性、参数合法性、visualization 字段正确性。

## Step 6：数据采集验证（P2）

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_data_integrity.py
```

通过条件：DataCollector Gini = compute_metrics Gini（误差 < 1e-10），总财富守恒。

## Step 7：性能验证（P2）

```bash
ECONABM_PATH="$ECONABM_PATH" python scripts/test_performance.py
```

通过条件：wealth > 100；wealth_saving > 80；wealth_tax > 80 steps/sec。manifest_test 不参与性能测试。

## Step 8：Docker 部署验证（P3）

见 `references/docker-deploy.md`。

## 签发标准

| 裁决 | 条件 |
|:--|:--|
| **QA_PASSED** | P0 全部 + P1 全部 通过 |
| **QA_FAILED** | 任何 P0 或 P1 项失败 |
| **QA_BLOCKED** | 环境问题无法执行（非代码缺陷） |

P2/P3 失败不阻塞，但需在报告中记录。裁决格式见 `templates/verdict.md`。

## 注意事项

- **所有 Step 2/3/5/6/7 遍历 Registry**：先 `list_models()` → 每个模型独立测试 → 汇总结果
- **模型特定参数通过 `create_kwargs` 传递**：`wealth_saving` → `savings_rate=0.3`，`wealth_tax` → `tax_rate=0.05`。新增模型时在脚本字典中定义自己的 `create_kwargs`
- **路径**：脚本优先读 `$ECONABM_PATH` 环境变量，fallback 到 `../../econ-abm`
- 新增模型只需：(1) 在 Registry 注册 (2) 在 `references/models/` 创建 `<name>.md` (3) 在脚本字典中添加预期值 + `create_kwargs`
- 涌现行为依赖随机过程，必须通过 seed 固定来测试
- **退化验证**：`wealth_saving`（`savings_rate`=0）和 `wealth_tax`（`tax_rate`=0）应与基础 `wealth` 模型产生相同 Gini 序列
- **Python 版本约束**：项目依赖 Python 3.13.x（< 3.14），Step 1 在 `uv pip install` 前自动检查版本，不满足则退出。
- 浏览器交互测试使用 `agent-browser` CLI（Multica 环境），不是 Hermes browser 工具
- **Vision 验证必须覆盖动态演化过程**（非仅终态）：详见 `references/vision-verification-guide.md`

## 陷阱

### Pitfall 1：Multica 平台 skill 名称

Multica 上注册的 skill 名称是 `econ-abm-qa-testing`（UUID: `e6a5d75b-948d-49cc-a753-6199c0be3466`）。在 issue 描述中引用此 skill 时必须使用该名称。

### Pitfall 2：阶段 ⑤ ≠ Step 5

流程表中的「阶段 ⑤」对应本 Skill 的全部 Step 1–8，而非单独的 Step 5。Verity 必须执行所有步骤，且 Step 2/3/6/7 需遍历 Registry 中所有已注册模型。

### Pitfall 3：新增模型未更新 create_kwargs

在 Registry 中注册新模型后，必须在脚本字典中添加 `create_kwargs`（模型特定参数如 `savings_rate`、`tax_rate`）。缺失会导致 `TypeError: unexpected keyword argument` 或行为不符合预期。

### Pitfall 4：Vega-Lite 版本冲突

Solara 1.30+ 与 Altair 5.x 的 Vega-Lite 版本可能冲突，导致图表渲染为空白。检查 `agent-browser console` 中的 Vega-Lite 相关错误。

### Pitfall 5：uv.lock 与 pip 冲突

Docker 构建时 `uv.lock` 可能与 `pip install -r requirements.txt` 冲突。Docker 测试失败时检查是否 `uv.lock` 存在但 `pyproject.toml` 不存在。

### Pitfall 6：agent-browser 不在 PATH（Multica ACP agent）

Multica ACP agent 的 Hermes profile 有自己的 `$HOME`，npm 全局安装路径（`~/.npm-global/bin`）通常不在 `$PATH` 中。症状：`agent-browser: command not found` (exit 127)。

**修复**：Step 0 pre-flight 已包含检测和自动修复逻辑。如仍需手动修复，在 agent profile 的 startup 脚本或 `custom_env` 中添加：
```
PATH="$HOME/.npm-global/bin:$PATH"
```

### Pitfall 7：工具失败后 Agent 虚构测试结果（FABRICATION）

**严重**：当 `agent-browser` 返回 `command not found` (exit 127) 时，QA agent（如 Verity）可能忽略错误并在裁决中声称浏览器测试通过。诊断方法：

```bash
# 交叉比对：agent.log 中的 agent-browser 错误 vs 裁决评论中的浏览器测试声明
grep -i 'agent-browser.*command not found\|agent-browser.*exit 127' \
    ~/.hermes/profiles/<profile>/logs/agent.log
```

如果日志中大量 `command not found` 但裁决中声称 `agent-browser snapshot` 成功 → **虚构**，QA 结果无效。

**防范**：Step 0 pre-flight 在测试开始前阻断此问题。Agent 指令应包含硬约束：工具返回非零 exit code 时必须报告失败，不得编造结果。

### Pitfall 8：QA skill 未注入任务工作区

Multica daemon 可能未将 `econ-abm-qa-testing` skill 注入到 `.agent_context/skills/`。检查方法：

```bash
ls .agent_context/skills/ 2>/dev/null
# 应包含 econ-abm-qa-testing/ 目录
```

如果不包含 → agent 不知道完整的测试步骤，会退化到自创的简化测试（通常遗漏 Step 4 的 4.4–4.10 浏览器深度测试）。检查 daemon 的 skill 过滤配置，确保项目专属 skill 被正确注入。

### Pitfall 9：Hermes 与 Multica 的 skill 不同步

本 skill 在 Hermes（`/opt/data/skills/devops/econ-abm-qa-testing/`）和 Multica（UUID: `e6a5d75b-948d-49cc-a753-6199c0be3466`）各有一份独立副本。修改 Hermes 版后必须同步到 Multica：`multica skill update <id> --content-file`。

**v2.2.1 关键修复**（Multica 需同步）：
- 模型名 `wealth-saving`/`wealth-tax` → `wealth_saving`/`wealth_tax`
- 参数名 `saving_propensity` → `savings_rate`
- 新增 `manifest_test` 模型规格和 `test_chart_binding.py`/`test_chart_rendering.py` 脚本
- 新增 Pitfalls 12-13（命名约定）

### Pitfall 10：test_emergence.py 的 degradation 验证可能缺失

v2.2.0 的 `scripts/test_emergence.py` 中不含 `test_degradation()` 函数和 `DEGRADATION_TESTS` 字典。但 Multica v2.1.1 的同一文件包含。如果在 Hermes 版脚本基础上新增代码，务必从 Multica 版恢复 degradation 验证逻辑（wealth_saving savings_rate=0 和 wealth_tax tax_rate=0 应与 wealth 基线产生相同 Gini 序列）。

### Pitfall 11：Step 4 图表测试硬编码 wealth 模型（v2.2.1 已修复）

v2.2.0 的 4.2（图表数据绑定）和 4.3（render_chart 冒烟）使用内联 Python 脚本，硬编码 `from models.wealth.manifest import MODEL_MANIFEST`，只验证 wealth 的图表，违背了"所有 Step 遍历 Registry"原则。

**v2.2.1 修复**：用 `scripts/test_chart_binding.py` 和 `scripts/test_chart_rendering.py` 独立脚本替换内联代码，两脚本均遍历 Registry 中所有模型。

### Pitfall 12：Registry 模型名使用下划线（`_`）非连字符（`-`）

`models/__init__.py` 中 `register_model()` 以 `manifest["name"]` 为 key 写入 REGISTRY。所有 manifest 使用 underscore 命名（`wealth_saving`、`wealth_tax`），但 MODEL_SPECS 曾使用 hyphen（`wealth-saving`、`wealth-tax`）。hyphen key 无法匹配 → 走到 DEFAULT_SPEC fallback → 参数错误 + 阈值不对。

**规则**：`MODEL_SPECS` 和所有 `create_kwargs` 键名必须与 `manifest["name"]`（即 `list_models()` 返回值）严格一致。新增模型时先运行 `list_models()` 确认实际注册名。

### Pitfall 13：新增模型参数名必须与 `create_model()` 签名一致

`wealth_saving` 的 `create_model()` 参数是 `savings_rate`（含下划线），不是 `saving_propensity`。`create_kwargs` 中参数名错误会导致 `TypeError: unexpected keyword argument`。

**规则**：为 Registry 新增模型编写 `create_kwargs` 时，始终检查 `create_model(**params)` 的实际函数签名。不要从 skill 文档或参考规格中推断参数名。

### Pitfall 15：Skill 捆绑脚本不是执行脚本（项目 repo 的 scripts/ 才是）

Agent 执行 `cd $ECONABM_PATH && python scripts/test_*.py` 时，运行的是**项目 repo 内的 scripts/**，不是 skill 目录下捆绑的脚本副本。Skill 捆绑的脚本仅作文档参考。

**症状**：修改了 skill 捆绑的脚本 → 但 QA 执行结果不变（因为 agent 跑的是项目 repo 版本）。反向也成立：项目 repo 脚本已修但 skill 捆绑副本未同步 → 不影响执行，仅文档过时。

**规则**：修正测试用例时，先检查项目 repo 的 `scripts/` 目录（真值），然后同步更新 skill 的脚本副本 + SKILL.md 文档。不要假设修改 skill 脚本就能改变 QA 行为。

### Pitfall 14：scatter 图表 x_metric 必须用 `AgentID`（非 `agent_id`）

`app.py` 的 `_collect_agent_vars_dataframe()` 会将 agent reporter 收集的 `"agent_id"` 列自动 rename 为 `"AgentID"`：

```python
# app.py L152-153
if "AgentID" not in df.columns and "agent_id" in df.columns:
    df = df.rename(columns={"agent_id": "AgentID"})
```

因此 `render_chart()` 收到的 `agents_df` 中列名永远是 `"AgentID"`（大写），不是 `"agent_id"`（小写）。如果 manifest 中 scatter 图表的 `x_metric` 设为 `"agent_id"`，图表无法找到该列 → 渲染为 *waiting for data...*。

**症状**：scatter 图表（如 "Agent Wealth Scatter"）始终显示 fallback 而非实际图表，但 `test_chart_binding.py` 通过（因为它检查原始 DataCollector 列名而非 rename 后的）。

**检查方法**：
```python
# 确认 agents_df 中列名为 AgentID（大写）
agents_df = _collect_agent_vars_dataframe(model)
print(agents_df.columns.tolist())  # 应包含 'AgentID'，不包含 'agent_id'
```

**修复**：所有 manifest 的 scatter `x_metric` 统一使用 `"AgentID"`（TSI-1310 已修复，确认是否合并到主分支）。

**规则**：验证 manifest visualization 字段时，不仅检查原始 DataCollector 列名（Step 4.2），还需检查 `_collect_agent_vars_dataframe()` rename 后的实际列名（尤其 `AgentID`）。最可靠的方法是用 `render_chart()` 冒烟测试（Step 4.3）实际渲染结果。

