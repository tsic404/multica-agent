# 上游 Dockerfile 变更检测

## ⚠️ 核心原则：上游 Dockerfile 是参考，不是替换目标

**docker-images 所有项目使用统一的 fetcher 两阶段构建模式**，上游 Dockerfile 假设源码已在构建上下文中（`COPY . .`），不需要 fetcher 阶段。

检测到上游变更后，任务是：**将上游 runtime 逻辑适配到 fetcher 模式中**，而不是：
- ❌ 直接复制/替换上游 Dockerfile（会丢失 fetcher 阶段）
- ❌ 把上游 bind-mount 方式照搬过来（docker-images 统一用 git clone）
- ❌ 建议「移除 builder 阶段」

✅ 正确做法：保留 builder 阶段（git clone），将上游新增的依赖、构建步骤、文件 COPY 改写为 `COPY --from=builder`。

## 上游对照表

| 本地项目 | 上游仓库 | 上游 Dockerfile 路径 | 默认分支 | 语言/生态 |
|---------|---------|---------------------|---------|----------|
| `honcho` | `plastic-labs/honcho` | `Dockerfile` | main | Python/FastAPI |
| `multica-server` | `multica-ai/multica` | `Dockerfile` | main | Go |
| `multica-web` | `multica-ai/multica` | `Dockerfile.web` | main | Node.js/Next.js |
| `mineru` | `opendatalab/MinerU` | `docker/global/Dockerfile` | master | Python |
| `mem0` | `mem0ai/mem0` | `server/Dockerfile` | main | Python |
| `mem0-dashboard` | `mem0ai/mem0` | `server/dashboard/Dockerfile` | main | Node.js |
| `camofox-browser` | `jo-inc/camofox-browser` | `Dockerfile` | master | Node.js |

跳过：`homeassistant`（使用预构建镜像）

## 特殊项目类型

### 双源项目（camofox-browser）
同时依赖两个上游：
- 源码：`jo-inc/camofox-browser`（Node.js 服务端，git clone）
- 二进制：`daijro/camoufox`（Firefox 魔改浏览器，下载 Release zip）

额外检查：上游 package.json 新依赖/ESM切换、camoufox 新 Release tag、源码目录新增文件。

### bind-mount 上游（camofox-browser v1.9+, honcho 最新版）
上游使用 `RUN --mount=type=bind` 时：
- **不迁移到 bind-mount**，在 builder 阶段用 curl 下载或保持 git clone 来模拟
- 例如：`--mount=type=bind,source=dist,target=/dist` → builder 阶段 curl 下载到 /workspace/，runtime COPY --from=builder

### Python 项目（mineru, mem0, honcho）
注意：pip install 方式变更、模型预下载增大镜像体积、--break-system-packages 标志。

### Go 项目（multica-server）
注意：ldflags 构建参数、新源码目录 COPY、CGO 需求。

### Node.js 项目（camofox-browser, multica-web, mem0-dashboard）
额外检查 package.json：ESM/CJS 切换、新依赖、engines.node 要求。
缓存优化：先 COPY package.json 装依赖，再 COPY 源码。

