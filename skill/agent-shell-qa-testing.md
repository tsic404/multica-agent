---
name: agent-shell-qa-testing
description: agent-shell 验收测试方法论。执行 doctor 与后端/模块测试时加载。
tags: [agent-shell, qa, testing, verification]
---

# agent-shell QA 测试

## 测试方法论

验收测试按两个维度组织：

1. **按真机环境**：每台物理/虚拟机的测试步骤（`references/env/*.md`）
2. **按后端/模块**：每个适配器或功能模块的测试用例（`references/cases/*.md`）

### 验收标准

每个测试用例输出：
- `PASS` — 行为符合规格
- `FAIL` — 行为不符合规格
- `BLOCKED` — 环境/工具链问题导致无法测试

所有测试用例 PASS 后，Verity 输出 `QA_PASSED`。

### doctor 命令基准

`agent-shell doctor` 是所有测试的前置条件。必须输出完整的健康报告：

```
✓ DE 检测        : <DE> (<session_type>, <wm>)
✓ 主后端        : <adapter> (bridge OK, N windows discoverable)
✓ D-Bus 桥接    : <service> reachable, roundtrip <N>ms
✓ 输入后端      : <backend> (keyboard + pointer)
  ├─ 降级1      : <fallback1>
  └─ 降级2      : <fallback2>
✓ 截图后端      : portal ScreenCast v<N> (PipeWire)
  └─ 降级1      : portal Screenshot v<N>
✓ AT-SPI        : enabled (a11y bus up, N apps)
⚠ 事件流        : <event script loaded status>
```

## 真机环境清单

| 环境 | 目标 | 主机名 (tailscale) | IP | 访问 |
|------|------|------|------|------|
| Dev2 | KDE 6.7.4 Wayland (Arch) | company-01 | 100.73.214.94 | `ssh tsic@100.73.214.94` |
| Dev2-deepin | DDE 25 (Deepin 25) X11 | Dev2-deepin（LAN，非 tailscale） | 192.168.122.216 | `ssh uos@192.168.122.216`（via ProxyJump company-01） |
| Dev2-107X | DDE 20 (UOS 20 Pro) X11 | Dev2-107X（LAN，非 tailscale） | 192.168.122.57 | `ssh uos@192.168.122.57`（via ProxyJump company-01） |
| 本机 GNOME | GNOME Shell 50.4 Wayland (Arch) | company-02（本机 hostname `workstation-01`，LAN `192.168.122.4`） | 100.94.174.69 | 本地；或 `ssh tsic@100.94.174.69`（免密） |
| uos-PC | DDE 20 Wayland (UOS 20 Pro) | uos-pc | 100.113.128.117 | `ssh uos@100.113.128.117`（免密） |

> **tailscale 实测（2026-08-30）**：本机 = `company-02` @ `100.94.174.69`（内核 hostname `workstation-01`，LAN `192.168.122.4`）；`100.107.109.109` = `company-03`（另一台机器，内核 hostname 同样为 `workstation-01`，非本机、非测试环境）；`100.73.214.94` = `company-01`；`100.113.128.117` = `uos-pc`。按主机名定位机器时以 tailscale 主机名为准：本机与 `company-03` 的内核 hostname 均为 `workstation-01`，只有 tailscale 名能区分。

## 测试用例结构

每个测试用例包含：

```
### TC-NNN: 测试标题

**前置条件**: ...
**测试步骤**:
1. ...
2. ...
**预期结果**: ...
**实际结果**: PASS/FAIL/BLOCKED
```

## 验证策略

1. 先跑 `agent-shell doctor` 确认环境健康
2. 按后端/模块的顺序执行测试用例
3. 每次测试记录 `agent-shell doctor` 输出作为基准
4. 失败时记录 `FAIL` 原因

## 模块引用

**环境验收（references/env/）**：

| 文件 | 内容 |
|------|------|
| `references/env/kde.md` | Dev2 KDE 6.7.4 环境验收步骤 |
| `references/env/dde25.md` | Dev2-deepin DDE 25 环境验收步骤 |
| `references/env/dde20.md` | Dev2-107X DDE 20 环境验收步骤 |
| `references/env/gnome.md` | 本机 GNOME 50 环境验收步骤 |
| `references/env/dde20-wayland.md` | uos-PC DDE 20 Wayland 环境验收步骤 |

**测试用例（references/cases/）**：

| 文件 | 内容 |
|------|------|
| `references/cases/kwin.md` | KWin 后端：窗口管理、工作区、事件流、版本兼容 (TC-001~006) |
| `references/cases/dde.md` | DDE 后端：DDE 专有接口、系统服务 (TC-101~106) |
| `references/cases/input.md` | 输入子系统：键盘、鼠标、滚动 (TC-201~205) |
| `references/cases/capture.md` | 截图/录屏子系统 (TC-301~304) |
| `references/cases/atspi.md` | AT-SPI 无障碍模块 (TC-401~404) |
| `references/cases/routing.md` | 语义路由与降级链 (TC-501~506) |

> **落地状态（2026-08-30）**：上表 `references/` 共 11 个文件均已落盘于工作区技能库（`multica skill files list` 可见 `references/env/*.md` 5 个 + `references/cases/*.md` 6 个），QA agent 加载本技能时可直接读取。提示词仓 `tsic404/multica-agent` 按仓库惯例仅版本化平铺的 `skill/agent-shell-qa-testing.md`，不单独存放 `references/` 目录；`agent-shell` 主仓不含本技能文件。
