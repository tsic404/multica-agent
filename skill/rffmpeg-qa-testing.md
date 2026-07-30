# rffmpeg QA 用户模拟测试

## 概述

本技能指导 QA agent 以**真实用户**视角测试 rffmpeg 远程转码系统。rffmpeg 是三层架构的 ffmpeg 替代品：CLI → Server（HTTP REST + WebSocket）→ Worker（远程 GPU 执行节点）。用户期望行为与原生 ffmpeg 完全一致。

**配合技能**：先加载 `qa-testing-workflow`（通用 QA 流程），本技能提供 rffmpeg 专属的用户场景和测试矩阵。

## 前提条件

### 构建

```bash
cd <repo_root>
go build -o bin/rffmpeg ./cmd/cli/
go build -o bin/rffmpeg-server ./cmd/server/
go build -o bin/rffmpeg-worker ./cmd/worker/
```

### 启动测试环境（每次测试前按序执行）

```bash
# 1. 清理
rm -f /tmp/rffmpeg-test.db

# 2. Server
./bin/rffmpeg-server --port 18999 --data-dir /tmp/rffmpeg-test-data &

# 3. Worker 配置
cat > /tmp/rffmpeg-worker-config.json << 'EOF'
{"server_url": "http://localhost:18999", "name": "test-worker-1", "work_dir": "/tmp/rffmpeg-worker-work"}
EOF

# 4. Worker
./bin/rffmpeg-worker --config /tmp/rffmpeg-worker-config.json &

# 5. 验证
curl -s http://localhost:18999/api/v1/workers | jq .
```

### 测试媒体

```bash
mkdir -p /tmp/rffmpeg-test-media
ffmpeg -f lavfi -i testsrc=duration=10:size=1280x720:rate=30 -c:v libx264 /tmp/rffmpeg-test-media/test-720p.mp4
ffmpeg -f lavfi -i sine=frequency=440:duration=5 /tmp/rffmpeg-test-media/test-audio.mp3
dd if=/dev/urandom of=/tmp/rffmpeg-test-media/test-broken.mp4 bs=1024 count=10
ffmpeg -f lavfi -i testsrc=duration=120:size=1920x1080:rate=30 -c:v libx264 /tmp/rffmpeg-test-media/test-large.mp4
```

### 清理

```bash
pkill -f rffmpeg-server 2>/dev/null; pkill -f rffmpeg-worker 2>/dev/null
rm -rf /tmp/rffmpeg-test-data /tmp/rffmpeg-worker-work /tmp/rffmpeg-worker-config.json
```

---

## 用户模拟哲学

> **用户不知道 rffmpeg 的存在。他们只知道自己要用 ffmpeg 转码一个视频。**

真实用户不会关心 Server/Worker 架构、逐条对照验收标准、小心翼翼避开边界情况。他们会直接敲 ffmpeg 命令（替换为 rffmpeg），期望输出和原生 ffmpeg 完全一致，对不符合 ffmpeg 行为的表现感到困惑或挫败。

每个测试场景都应回答：**「用户当它是 ffmpeg 用，会不会出问题？」**

---

## 场景 1：基础转码

用户最简单的操作：

```bash
./bin/rffmpeg -i /tmp/rffmpeg-test-media/test-720p.mp4 -c:v libx264 output.mp4
```

**检查**：CLI 接受原生 ffmpeg 参数无报错；stderr 实时输出进度（帧数/fps/bitrate）；`output.mp4` 存在且 `ffprobe` 正常；同参数下与原生 ffmpeg 输出 `diff` 一致。

✅ **PASS**：以上 4 项全部通过。

## 场景 2：复杂参数组合

真实用户的花式操作：

```bash
# 多输入
./bin/rffmpeg -i video.mp4 -i audio.mp3 -c:v copy -c:a aac -shortest merged.mp4
# 滤镜链
./bin/rffmpeg -i input.mp4 -vf "scale=640:480,drawtext=text='test':x=10:y=10" out.mp4
# 编码器选项
./bin/rffmpeg -i input.mp4 -c:v libx264 -preset fast -crf 23 -c:a aac -b:a 128k out.mp4
# 时间裁剪
./bin/rffmpeg -ss 00:00:05 -i input.mp4 -t 00:00:03 clip.mp4
# 提取音频
./bin/rffmpeg -i input.mp4 -vn -c:a libmp3lame -q:a 2 audio.mp3
# rffmpeg 与 ffmpeg 选项混用
./bin/rffmpeg --server http://localhost:18999 -i input.mp4 -c:v libx264 out.mp4
```

**检查**：多输入正确处理；滤镜参数不截断；codec 选项正确传递；-ss/-t 参数正确；rffmpeg 专属选项不被传给 ffmpeg；特殊字符参数（空格/引号）正确传递。

✅ **PASS**：以上 6 种参数组合全部不报错且输出可播放。

## 场景 3：用户犯错 —— 错误体验

真实用户的常见错误：

```bash
# 文件不存在
./bin/rffmpeg nonexistent.mp4 out.mp4
# 参数不完整
./bin/rffmpeg -i input.mp4 -c:v
# 不支持的编码器
./bin/rffmpeg -i input.mp4 -c:v nonexistent_codec out.mp4
# 空命令
./bin/rffmpeg
# Server 不可达
pkill rffmpeg-server && ./bin/rffmpeg -i input.mp4 out.mp4
```

**检查**：文件不存在 → 明确提示文件名（无 panic/堆栈）；参数不完整 → 类似 ffmpeg 风格错误；不支持编码器 → 明确说明；空命令 → 显示帮助（与 ffmpeg 一致）；Server 不可达 → 3 秒内返回连接错误，不卡死。

✅ **PASS**：5 种错误各有清晰提示，无 panic 无堆栈，不卡死。

## 场景 4：服务端异常

后台 Server/Worker 故障时 CLI 的优雅处理：

```bash
# Server 宕机
pkill rffmpeg-server
./bin/rffmpeg -i input.mp4 out.mp4  # 期望：3s 内返回连接错误

# 无可用 Worker
pkill rffmpeg-worker
./bin/rffmpeg -i input.mp4 out.mp4  # 期望：明确提示无可用 Worker

# Worker 执行中崩溃
# （启动任务后 kill worker 进程）
# 期望：CLI 显示任务失败，不产生损坏文件

# 并发提交
for i in $(seq 1 10); do
  ./bin/rffmpeg -i input.mp4 -c:v libx264 "out_$i.mp4" &
done
wait  # 期望：全部完成或全部报错，无部分丢失

# Server 重启恢复
# 提交长任务 → 等3s → kill Server → 重启Server+Worker
# 期望：Server 恢复后继续调度，已提交任务不丢失
```

**检查**：Server 不可达 3s 内报错；Worker 不可用有明确提示；任务中断不产生损坏文件；并发全部完成无丢失；重启后任务恢复。

✅ **PASS**：以上 5 项全部通过。

## 场景 5：编码器改写与透明度

Worker 根据硬件能力改写编码器参数，CLI 必须透明告知。

```bash
# 未指定编码器 → 自动硬件加速
./bin/rffmpeg -i test-720p.mp4 output.mp4
# 期望：stderr 显示 [rffmpeg] Auto-selected encoder: h264_vaapi

# 不支持的编码器 → 降级
./bin/rffmpeg -i test-720p.mp4 -c:v h264_nvenc output.mp4
# 期望：[rffmpeg] h264_nvenc unavailable, fallback to h264_vaapi
# 跨格式降级应拒绝（如 AV1→H.264）

# --auto-hw flag 控制
./bin/rffmpeg --auto-hw -i test-720p.mp4 -c:v libx264 output.mp4
# 期望：[rffmpeg] --auto-hw: upgraded libx264 → h264_vaapi
# 不加 --auto-hw 时尊重用户选择，不改写

# 参数映射（crf→qp 等）
./bin/rffmpeg --auto-hw -i test-720p.mp4 -c:v libx264 -crf 23 -preset fast output.mp4
# 期望：stderr 显示映射详情，SSIM > 0.95

# Worker 能力上报与调度
curl -s http://localhost:18999/api/v1/workers | jq '.[] | {name, capabilities}'
# 期望：硬件编码任务优先分配给有该能力的 Worker
```

**检查**：每次改写有 `[rffmpeg]` 日志行含改写原因；默认尊重用户选择；--auto-hw 触发升级但不降级；参数映射正确（crf→qp）；Worker 能力上报对 Server 可见。

✅ **PASS**：改写有日志，无静默改写，参数映射正确，能力调度合理。

## 场景 6：安全认证与配额

TSI-710 安全底座。

```bash
# 无 token → 拒绝（不回退本地 ffmpeg）
unset RFFMPEG_TOKEN
./bin/rffmpeg -i test-720p.mp4 out.mp4  # 期望：exit code ≠ 0

# 错 token → 401
export RFFMPEG_TOKEN=wrong-token
./bin/rffmpeg -i test-720p.mp4 out.mp4  # 期望：HTTP 401，明确提示

# 正确 token → 正常
export RFFMPEG_TOKEN=test-token-12345
./bin/rffmpeg -i test-720p.mp4 out.mp4  # 期望：正常转码

# Rate limit 超限 → 429
for i in $(seq 1 15); do
  ./bin/rffmpeg -i test-720p.mp4 -c:v libx264 "out_$i.mp4" &
done
wait  # 期望：超出 limit 返回 429，已在队列的任务不受影响

# Timeout
./bin/rffmpeg -i test-large.mp4 -c:v libx264 --timeout 5 out.mp4
# 期望：5s 后终止，Worker 无残留 ffmpeg 进程，不产生部分输出
```

**检查**：无/错 token 拒绝；正确 token 通行；超限 429，不同 client 隔离计数；超时终止且子进程清理。

✅ **PASS**：认证三态正确，rate limit 生效，timeout 清理干净。

## 场景 7：数据通道与缓存

TSI-711 远程输入源、流式输出、Worker 缓存。

```bash
# HTTP 输入
./bin/rffmpeg -i http://example.com/test-video.mp4 -c:v libx264 out.mp4
# 期望：Worker 自行下载，不经过 CLI 上传

# 不可达 URL
./bin/rffmpeg -i http://nonexistent.example.com/video.mp4 out.mp4
# 期望：INPUT_UNREACHABLE 分类错误

# 流式输出到 stdout
./bin/rffmpeg -i test-720p.mp4 -f mp4 - > output_pipe.mp4
# 期望：数据实时流回，Ctrl+C 后 Worker 停止

# 缓存命中
./bin/rffmpeg -i test-720p.mp4 -c:v libx264 out1.mp4  # 首次
./bin/rffmpeg -i test-720p.mp4 -c:v libx264 out2.mp4  # 期望：[rffmpeg] Cache hit
# 不同参数不命中，TTL/LRU 驱逐生效
```

**检查**：远程输入 Worker 自行下载；不可达 URL 返回 INPUT_UNREACHABLE；流式输出实时不阻塞；缓存命中/参数差异化/TTL/LRU 正确。

✅ **PASS**：远程输入可拉取，流式实时，缓存命中参数隔离，驱逐正确。

## 场景 8：可观测性

TSI-712 ffprobe、进度、心跳、失败分类。

```bash
# ffprobe 端点
./bin/rffmpeg probe -i test-720p.mp4 -show_streams -of json
# 期望：JSON 含 _rffmpeg{worker_encoders, suggestion}，同步返回无延迟

# 进度与 ETA
./bin/rffmpeg -i test-large.mp4 -c:v libx264 out.mp4 2>progress.log
# 期望：0%→100% 线性增长，ETA 误差 <20%，每秒1-2次不刷屏

# Worker 心跳面板
curl -s http://localhost:18999/api/v1/workers | jq '.[] | {name, status, health}'
# 期望：health{gpu_util, gpu_mem, active_jobs, throughput_fps, last_heartbeat}
# 心跳 5-10s，离线后 status→offline

# 失败分类（6 种）
# INPUT_UNREACHABLE / ENCODER_UNSUPPORTED / DISK_FULL / TIMEOUT / WORKER_CRASH / FFMPEG_ERROR
# 期望：每种场景触发正确分类，retryable 标记正确
```

**检查**：ffprobe 注入 _rffmpeg 不污染原始字段；进度 ETA 误差 <20%；心跳指标合理；6 种失败分类全部覆盖且 retryable 标记正确。

✅ **PASS**：以上 4 项全部通过。

## 场景 9：自愈

TSI-713 Worker 宕机迁移与慢节点驱逐。

```bash
# Worker 宕机 → Job 重新入队
# 1. 启动 2 Worker → 提交长任务 → kill 执行中 Worker
# 期望：30s 检测离线，Job 重新入队分配给活 Worker，迁移事件记录到审计日志

# 慢 Worker 驱逐
# 期望：throughput 偏离中位数 3x → 标记 slow/evicted，不分配新 Job
# 已有 Job 不中断；throughput 恢复到 1.5x 内重新分配
# 空闲 Worker 不误判；新注册 Worker 前 N 个 Job 不参与驱逐评估
```

**检查**：30s 心跳超时检测；Job 重新入队不丢失；审计日志含完整迁移链（原Worker/目标/原因/重试次数）；慢驱逐 3x 触发，已有Job不中断，恢复后重分配。

✅ **PASS**：宕机迁移完整，慢驱逐不误判。

---

## 验收矩阵

| 维度 | 最少用例数 | 证据要求 |
|------|-----------|----------|
| Happy Path | 3 | 终端输出 + ffprobe |
| Error Handling | 3 | 错误信息文本 |
| Boundary Values | 2 | 极端参数组合结果 |
| Integration | 1 | CLI→Server→Worker 全链路日志 |

每个通过用例附可验证证据（命令 + 实际输出 + 验证方法）。

---

## 常见陷阱

1. **只用验收标准逐条对照** — 必须走完至少 2 个「用户旅程」场景
2. **忽略 stderr 输出体验** — 用户依赖进度条，必须验证 stderr 质量
3. **不对比原生 ffmpeg** — 至少 1 个用例做 `diff` 对比
4. **不测试 rffmpeg 选项与 ffmpeg 选项边界** — 专属选项不泄露给 ffmpeg
5. **忽略网络延迟** — CLI→Server→Worker 涉及 HTTP/WebSocket，注意延迟可接受性

---

## 快速检查清单

- [ ] 我用 rffmpeg 像用 ffmpeg 一样自然吗？
- [ ] stderr 输出与 ffmpeg 风格一致？编码器改写有 `[rffmpeg]` 日志？
- [ ] 错误提示友好（无 panic 无堆栈）？
- [ ] 至少 1 个用例与原生 ffmpeg diff 对比？
- [ ] Server/Worker 异常时 CLI 反馈清晰？
- [ ] Token 认证（无/错/正确）三态正确？Rate limit/Timout 生效？
- [ ] 流式输出实时？缓存命中/驱逐正确？
- [ ] ffprobe 注入 _rffmpeg 完整？进度 ETA 误差 <20%？
- [ ] 6 种失败分类全部触发正确？Worker 宕机迁移完整？
- [ ] 所有证据已收集（终端输出、ffprobe、diff 结果）？

---

## 签发标准

| 级别 | 内容 | 影响 |
|------|------|------|
| **P0** | 场景 1/2/3/6（基础转码/复杂参数/错误/认证）、场景 9（宕机迁移） | 阻断发布 |
| **P1** | 场景 4/5（服务端异常/编码器改写）、场景 8（进度/ETA） | 核心体验 |
| **P2** | 场景 7（数据通道）、场景 8（ffprobe/心跳/失败分类） | 完整功能 |
| **P3** | Rate Limit、Timeout、慢驱逐 | 边缘场景 |

### 门禁

| 裁决 | 条件 |
|------|------|
| **QA_PASSED** | P0 全部通过 + P1 全部通过 |
| **QA_FAILED** | 任何 P0 或 P1 项失败 |

### 裁决输出

```
**QA_PASSED** 或 **QA_FAILED**

Commit: `$COMMIT`
Branch: `$REMOTE_BRANCH`

| # | 场景 | 命令 | 退出码 | 结果 |
|---|------|------|:------:|------|
| N | <场景名> | <命令> | <code> | <单行 ≤80 字> |
```

