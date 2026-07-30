# torrentfs QA 用户模拟测试

## 概述

本技能指导 QA agent 以**真实用户**视角测试 torrentfs FUSE 文件系统。不同于纯验收标准对照，你需要模拟用户的实际操作流程：挂载、浏览、读写、组织文件、犯错并观察系统反馈。

**配合技能**：先加载 `qa-testing-workflow`（通用 QA 流程），本技能提供 torrentfs 专属的用户场景和测试矩阵。

## 前提条件

### 构建与挂载

```bash
# 1. 构建项目
cd <repo_root>
cargo build --release

# 2. 准备挂载点和状态目录
mkdir -p /tmp/torrentfs-test-mnt
mkdir -p /tmp/torrentfs-test-state

# 3. 挂载（前台方便观察日志，使用自定义 db 和 cache 路径）

# 自定义 db 和 cache 路径（推荐测试用）：
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache

# 后台模式：
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache &
```

### 测试种子准备

准备至少以下类型的 .torrent 文件：
- 小文件种子（单文件 < 1MB，如文本/图片）
- 大文件种子（单文件 > 100MB，如 Linux ISO）
- 多文件种子（3+ 文件，含子目录结构）
- 无效种子（手动创建非法文件，或空文件）

如项目仓库中有现成种子文件，直接复用；否则从公开 tracker 下载或自行创建。

**项目种子文件路径**：`/workspace/torrentfs/*.torrent`（含 10 个种子，包括 Ubuntu ISO）

列出可用种子：
```bash
ls /workspace/torrentfs/*.torrent
```

### 环境清理

每次测试前：
```bash
fusermount -u /tmp/torrentfs-test-mnt 2>/dev/null
rm -rf /tmp/torrentfs-test-state
rm -rf /tmp/torrentfs-test-mnt
mkdir -p /tmp/torrentfs-test-mnt /tmp/torrentfs-test-state
```

---

## 用户模拟哲学

> **你不是在"测试"，你是在"使用"它。**

真实用户不会：
- 按验收标准清单逐条跑
- 小心翼翼地只做"对的"操作
- 知道内部实现细节

真实用户会：
- 按直觉操作，犯错后看反馈
- 期望行为与常规文件系统一致
- 对不符合直觉的行为感到困惑或挫败

因此每个测试场景都应回答：**「用户会不会困惑？体验是否顺畅？」**

---

## 用户旅程场景

### 场景 1：初次使用 —— 写入种子并浏览内容

用户拿到 torrentfs，想下载一个 Ubuntu ISO。

```bash
# 用户操作：把种子放进 metadata
cp ubuntu.iso.torrent /tmp/torrentfs-test-mnt/metadata/

# 用户期望：data/ 下出现对应目录
ls /tmp/torrentfs-test-mnt/data/
# → ubuntu.iso.torrent/

# 用户期望：进入目录看到文件
ls -la /tmp/torrentfs-test-mnt/data/ubuntu.iso.torrent/
# → ubuntu-25.10-desktop-amd64.iso  （文件名、大小正确）

# 用户期望：文件大小合理
stat /tmp/torrentfs-test-mnt/data/ubuntu.iso.torrent/ubuntu-25.10-desktop-amd64.iso
```

**检查要点**：

| 检查项 | 命令 | 期望 |
|--------|------|------|
| 种子写入无错误 | `cp` 返回 0 | 无报错 |
| data/ 镜像出现 | `ls data/` | 目录名与种子文件名一致 |
| 内部文件可见 | `ls -la data/<seed>/` | 文件名、大小与种子内元数据一致 |
| 文件为只读 | `stat data/<seed>/<file>` | 权限 0444 |
| 目录为只读 | `stat data/<seed>/` | 权限 0555 |

### 场景 2：子目录组织

用户有多个类别的种子，希望按分类组织。

```bash
# 用户操作：创建分类目录
mkdir -p /tmp/torrentfs-test-mnt/metadata/os/linux
mkdir -p /tmp/torrentfs-test-mnt/metadata/media/movies

# 用户操作：分别放入种子
cp ubuntu.torrent /tmp/torrentfs-test-mnt/metadata/os/linux/
cp debian.torrent /tmp/torrentfs-test-mnt/metadata/os/linux/
cp movie.torrent /tmp/torrentfs-test-mnt/metadata/media/movies/

# 用户期望：data/ 下镜像相同结构
ls /tmp/torrentfs-test-mnt/data/os/linux/
# → ubuntu.torrent/  debian.torrent/

ls /tmp/torrentfs-test-mnt/data/media/movies/
# → movie.torrent/
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| data/ 子目录结构镜像 metadata/ | source_path 对应 |
| 同一 source_path 下多种子共存 | 各自独立目录 |
| 空目录的 data/ 镜像 | 空目录存在（或不存在但创建种子时自动创建） |

### 场景 3：删除和重命名

```bash
# 用户操作：重命名种子
mv /tmp/torrentfs-test-mnt/metadata/os/linux/ubuntu.torrent \
   /tmp/torrentfs-test-mnt/metadata/os/linux/ubuntu-25.10.torrent

# 用户期望：data/ 下对应变化
ls /tmp/torrentfs-test-mnt/data/os/linux/
# → ubuntu-25.10.torrent/  debian.torrent/

# 用户操作：删除种子
rm /tmp/torrentfs-test-mnt/metadata/os/linux/debian.torrent

# 用户期望：data/ 下对应消失
ls /tmp/torrentfs-test-mnt/data/os/linux/
# → ubuntu-25.10.torrent/  （debian 已消失）
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| metadata/ rename → data/ rename | 同步更新 |
| metadata/ unlink → data/ 清理 | 目录消失，DB 记录清除 |
| metadata/ rmdir | 允许（空目录）；非空拒绝或递归删除 |

### 场景 3b：种子文件移动（跨目录迁移）

用户经常需要重新组织种子分类，将种子从一个目录移动到另一个目录。

```bash
# 准备：创建多级目录结构并放入种子
mkdir -p /tmp/torrentfs-test-mnt/metadata/cat-a
mkdir -p /tmp/torrentfs-test-mnt/metadata/cat-b
cp /workspace/torrentfs/ubuntu.iso.torrent /tmp/torrentfs-test-mnt/metadata/cat-a/
cp /workspace/torrentfs/debian.torrent /tmp/torrentfs-test-mnt/metadata/cat-a/
cp /workspace/torrentfs/fedora.torrent /tmp/torrentfs-test-mnt/metadata/

# 1. 从根目录移动到子目录
mv /tmp/torrentfs-test-mnt/metadata/fedora.torrent    /tmp/torrentfs-test-mnt/metadata/cat-b/
# 期望：mv 成功，exit 0

# 验证 data/ 镜像
ls /tmp/torrentfs-test-mnt/data/cat-b/
# 期望：fedora.torrent/ 出现
ls /tmp/torrentfs-test-mnt/data/
# 期望：fedora.torrent/ 不再出现（已移走）

# 2. 从子目录移动到根目录
mv /tmp/torrentfs-test-mnt/metadata/cat-a/debian.torrent    /tmp/torrentfs-test-mnt/metadata/
# 期望：mv 成功，exit 0

# 验证 data/ 镜像
ls /tmp/torrentfs-test-mnt/data/
# 期望：debian.torrent/ 出现
ls /tmp/torrentfs-test-mnt/data/cat-a/
# 期望：debian.torrent/ 不再出现，只剩 ubuntu.iso.torrent/

# 3. 跨子目录移动
mv /tmp/torrentfs-test-mnt/metadata/cat-a/ubuntu.iso.torrent    /tmp/torrentfs-test-mnt/metadata/cat-b/
# 期望：mv 成功，exit 0

# 验证 data/ 镜像
ls /tmp/torrentfs-test-mnt/data/cat-b/
# 期望：fedora.torrent/ + ubuntu.iso.torrent/ 两个目录
ls /tmp/torrentfs-test-mnt/data/cat-a/
# 期望：空（或目录不存在）

# 4. 移动到不存在的目录 - 应失败
mv /tmp/torrentfs-test-mnt/metadata/cat-b/ubuntu.iso.torrent    /tmp/torrentfs-test-mnt/metadata/nonexistent/ 2>&1
echo "exit code: $?"
# 期望：非 0，ENOENT 或 "No such file or directory"

# 5. 移动后验证种子仍可读取
stat /tmp/torrentfs-test-mnt/data/cat-b/fedora.torrent/
# 期望：目录存在，权限正确

# 6. 移动后验证 source_path 已更新
ls /tmp/torrentfs-test-mnt/data/cat-b/
# 期望：之前移入的种子都在此目录下

# 7. 移动后验证旧位置无残留
ls /tmp/torrentfs-test-mnt/data/cat-a/ 2>&1
# 期望：空目录或无此目录

# 8. 同名移动（覆盖）—— 同一目录下同种子名
cp /workspace/torrentfs/ubuntu.iso.torrent /tmp/torrentfs-test-mnt/metadata/cat-a/
mv /tmp/torrentfs-test-mnt/metadata/cat-b/ubuntu.iso.torrent    /tmp/torrentfs-test-mnt/metadata/cat-a/ubuntu.iso.torrent 2>&1
echo "exit code: $?"
# 期望：按文件系统语义处理（覆盖或拒绝，需定义行为）

# 9. 移动后一致性检查
diff <(cd /tmp/torrentfs-test-mnt/metadata/ && find . -name '*.torrent' | sed 's|^./||' | sort)      <(cd /tmp/torrentfs-test-mnt/data/ && find . -type d -not -name '.' -not -regex '.*/.*/.*' | sed 's|^./||' | sort)
# 期望：metadata/ 种子文件与 data/ 一级目录一一对应
```

**检查要点**：

| 检查项 | 命令 | 期望 |
|--------|------|------|
| 根到子目录 mv | `mv metadata/seed metadata/sub/` | data/ 镜像同步移动 |
| 子目录到根 mv | `mv metadata/sub/seed metadata/` | data/ 镜像同步移动 |
| 跨子目录 mv | `mv metadata/a/seed metadata/b/` | data/ 镜像同步迁移 |
| 目标不存在 | `mv metadata/seed metadata/nonexist/` | ENOENT，不创建孤儿 |
| 移动后读取 | `stat data/sub/seed/` | 正常访问，不报错 |
| source_path 更新 | 查看 data/ 结构 | 旧位置无残留，新位置正确 |
| 重复 mv 不丢事件 | 快速连续 mv 3 次 | 每次 data/ 正确同步 |
| 删除后移入 | 旧位置 rm 后同位置 mv 入新种子 | 新种子正常出现 |
| 移动时正在读取 | 后台 cat 同时 mv | 读取正常完成或优雅报错 |



### 场景 4：读取文件（懒加载触发点）

这是 torrentfs 的核心价值——用户只在实际读文件时才触发下载。

```bash
# === 全量读取 ===
# 用户操作：尝试读取一个文件
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso > /tmp/test-output.iso

# 用户期望：
# - 命令不立即报错（即使数据尚未下载）
# - 数据逐渐下载（可能较慢，但不应无限阻塞）
# - 最终完整输出文件
# - 输出文件内容正确（与种子 hash 一致）

# 验证输出完整性
ORIG_SIZE=$(stat --format=%s /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso)
OUTPUT_SIZE=$(stat --format=%s /tmp/test-output.iso)
echo "Original: $ORIG_SIZE  Output: $OUTPUT_SIZE"
# 期望：大小一致

# === 部分读取（seek/skip） ===
# 读取文件中间 1KB（从 1MB 偏移处开始）
dd if=/tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso    of=/tmp/test-chunk.bin bs=1024 skip=1024 count=1 2>&1
# 期望：正确读取 1024 字节

# 读取文件开头 4KB
dd if=/tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso    of=/tmp/test-head.bin bs=4096 count=1 2>&1
# 期望：正确读取 4096 字节

# 读取文件末尾 1KB
FILE_SIZE=$(stat --format=%s /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso)
TAIL_OFFSET=$((FILE_SIZE / 1024 - 1))
dd if=/tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso    of=/tmp/test-tail.bin bs=1024 skip=$TAIL_OFFSET count=1 2>&1
# 期望：读取最后 1024 字节，不越界

# === 多次读取（验证缓存加速） ===
time cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso > /dev/null
echo "First read done"

time cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso > /dev/null
echo "Second read (should be faster due to cache)"

# === 读取超出文件大小 ===
# 尝试从超出的偏移量读取 → 应返回 0 字节，不 panic
dd if=/tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso    of=/tmp/test-eof.bin bs=1 skip=999999999 count=1 2>&1
echo "EOF read exit code: $?"
# 期望：返回 0（正常 EOF），不报错

# === 查看文件大小（不应为 0） ===
ls -la /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso
stat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso
# 期望：size 显示完整大小，不是 0
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| read 不立即失败 | 启动下载并阻塞等待 |
| 全量读取完整性 | 输出大小与 stat 一致 |
| 下载完成返回正确数据 | 内容 hash 与种子一致 |
| 多次 read 同一文件 | 使用缓存，第二次明显更快 |
| 部分读取 — 头部 | `dd bs=4096 count=1` 正确 |
| 部分读取 — 中间 | `dd bs=1024 skip=1024 count=1` 正确 |
| 部分读取 — 尾部 | 最后 1KB 不越界 |
| 读取超出文件大小 | 返回 0 字节，exit 0，不 panic |


### 场景 4b：data/ 目录浏览与遍历

用户想了解文件系统的整体结构——浏览 data/ 目录树。

```bash
# 浏览 data/ 根
ls -la /tmp/torrentfs-test-mnt/data/
# 期望：列出所有种子对应的目录

# 递归遍历整个 data/
find /tmp/torrentfs-test-mnt/data/ -type f -o -type d | head -50
# 期望：不崩溃，不挂死，输出完整

# stat 各个条目
stat /tmp/torrentfs-test-mnt/data/
stat /tmp/torrentfs-test-mnt/data/os/
stat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/
stat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso
# 期望：所有路径 stat 正常返回，文件大小 > 0，权限正确

# 检查 data/ 镜像 metadata/ 结构
diff <(cd /tmp/torrentfs-test-mnt/metadata/ && find . -name '*.torrent' | sort)      <(cd /tmp/torrentfs-test-mnt/data/ && find . -type d | sort)
# 期望：种子文件对应的 data/ 子目录结构一致

# 读取目录（opendir/readdir）
ls -R /tmp/torrentfs-test-mnt/data/ > /dev/null
# 期望：不报错
```

**检查要点**：

| 检查项 | 命令 | 期望 |
|--------|------|------|
| data/ 根目录可列 | `ls data/` | 列出所有种子目录 |
| 递归遍历不崩溃 | `find data/` | 完整输出，无 hang |
| 任意深度 stat | `stat data/a/b/c/seed/file` | 返回正确元数据 |
| metadata/ 与 data/ 镜像 | `diff` | 结构一致 |
| 空 data/ 目录 | 无种子时 `ls data/` | 空，不报错 |

### 场景 5：用户犯错 —— 预期被拒绝的操作

真实用户会尝试这些，系统应给出明确反馈。

```bash
# 错误1：向 data/ 写入
echo "hello" > /tmp/torrentfs-test-mnt/data/foo.txt
# 期望：Permission denied 或 Read-only file system

# 错误2：删除 data/ 下的文件
rm /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso
# 期望：Read-only file system

# 错误3：在 data/ 下创建目录
mkdir /tmp/torrentfs-test-mnt/data/newdir
# 期望：Read-only file system

# 错误4：在 data/ 下重命名
mv /tmp/torrentfs-test-mnt/data/foo /tmp/torrentfs-test-mnt/data/bar
# 期望：Operation not permitted 或跨设备链接错误

# 错误5：放入无效 .torrent 文件
echo "garbage" > /tmp/bad.torrent
cp /tmp/bad.torrent /tmp/torrentfs-test-mnt/metadata/
# 期望：cp 报错（EINVAL），data/ 下不出现对应目录
```

**检查要点**：

| 操作 | 期望错误 | 用户理解 |
|------|---------|---------|
| write to data/ | `ENOSYS` 或 `EROFS` | "这是只读的" |
| unlink from data/ | `EROFS` | "不能在这里删除" |
| mkdir in data/ | `EROFS` | "不能在这里创建" |
| rename in data/ | `EXDEV` 或 `EPERM` | "跨设备不支持" |
| invalid .torrent | 写入时 `EINVAL` | "种子文件无效" |
| 写入无效后 data/ 无对应 | 不出 ghosts | 干净 |

### 场景 6：并发操作

用户可能同时打开多个终端窗口。

```bash
# 终端1：读取一个大文件
cat /tmp/torrentfs-test-mnt/data/ubuntu.torrent/large.iso > /dev/null &

# 终端2：同时浏览同一目录
ls -la /tmp/torrentfs-test-mnt/data/ubuntu.torrent/

# 终端3：同时在另一个种子目录浏览
ls -la /tmp/torrentfs-test-mnt/data/debian.torrent/

# 终端4：同时删除另一个种子
rm /tmp/torrentfs-test-mnt/metadata/old.torrent
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 并发 readdir + read | 不崩溃、不死锁 |
| 并发 unlink + 正在读取 | 优雅处理（读取完成或报错，不 segfault） |
| 并发多个种子写入 | 不冲突 |

### 场景 7：重启恢复

用户卸载后重新挂载，之前写入的种子应保留。

```bash
# 写入种子后卸载
fusermount -u /tmp/torrentfs-test-mnt

# 重新挂载（db 和 cache 路径相同）
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache

# 用户期望：之前写入的种子依然在 data/ 中可见
ls /tmp/torrentfs-test-mnt/data/
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 重新挂载后种子仍可见 | data/ 结构恢复 |
| 文件大小正确 | 从 DB 恢复，非重新下载 |
| resume data 保留 | 下载进度不丢失（如适用） |

---


### 场景 8：缓存检查

下载完成后检查缓存机制是否正常工作。

```bash
# 1. 记录缓存目录初始状态
CACHE_DIR="/tmp/torrentfs-test-state/cache"
ls -la "$CACHE_DIR" 2>/dev/null || echo "cache dir not found"
find /tmp/torrentfs-test-state/ -type f | head -20

# 2. 首次读取文件（触发下载）
time cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso > /dev/null
echo "First read time: (see above)"

# 3. 检查缓存是否已创建
find /tmp/torrentfs-test-state/ -type f -mmin -1
echo "Cache files created within last minute:"
find /tmp/torrentfs-test-state/ -type f -mmin -1 | wc -l
# 期望：有新增缓存文件

# 4. 检查缓存文件大小
CACHE_FILE=$(find /tmp/torrentfs-test-state/ -type f -mmin -1 | head -1)
if [ -n "$CACHE_FILE" ]; then
    ls -la "$CACHE_FILE"
    # 期望：大小 > 0，非空文件
fi

# 5. 再次读取（应命中缓存，更快）
time cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso > /dev/null
echo "Second read time: (should be much faster)"
# 期望：第二次明显快于第一次

# 6. 卸载后缓存保留
fusermount -u /tmp/torrentfs-test-mnt
echo "Before unmount cache:"
find /tmp/torrentfs-test-state/ -type f | wc -l

# 重新挂载
./target/release/torrentfs /tmp/torrentfs-test-mnt \n  --db /tmp/torrentfs-test-state/metadata.db \n  --cache /tmp/torrentfs-test-state/cache &

sleep 2

# 再次读取（应命中持久化缓存，不需要重新下载）
time cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.torrent/ubuntu.iso > /dev/null
echo "Post-remount read time: (should hit cache, not re-download)"

# 7. 删除种子后缓存清理
rm /tmp/torrentfs-test-mnt/metadata/os/linux/ubuntu.torrent
# 期望：对应缓存被清理（或延迟清理），不残留孤岛数据
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 首次读取后创建缓存 | state-dir 下有新增文件 |
| 缓存文件非空 | 大小 > 0 |
| 第二次读取命中缓存 | 耗时明显短于首次 |
| 卸载后缓存保留 | 文件数不变 |
| 重新挂载后缓存命中 | 读取快，不重新下载 |
| 删除种子清理缓存 | 对应缓存被移除，无残留 |
| 多次读取同文件 | 不重复下载，不创建重复缓存 |
| 并发读取 + 缓存写入 | 不冲突 |


### 场景 9：metadata/ 批量操作

用户一次性导入大量种子，验证系统在写入压力下的实时镜像和稳定性。

```bash
# 1. 批量写入种子
echo "批量写入 20 个种子到 metadata/"
SEED_DIR="/workspace/torrentfs"
BATCH_DIR="/tmp/torrentfs-test-mnt/metadata/batch-import"

mkdir -p "$BATCH_DIR"
# 复制所有可用种子（含 Ubuntu ISO 等大种子）
cp "$SEED_DIR"/*.torrent "$BATCH_DIR/" 2>/dev/null
echo "写入数量: $(ls "$BATCH_DIR"/*.torrent 2>/dev/null | wc -l)"

# 用户期望：所有种子无报错写入

# 2. 实时检查 data/ 镜像
echo "检查 data/ 镜像..."
ls /tmp/torrentfs-test-mnt/data/batch-import/
# 期望：目录数与种子数一致

# 3. 创建多层子目录并分别放入种子
mkdir -p /tmp/torrentfs-test-mnt/metadata/cat-a/sub-1
mkdir -p /tmp/torrentfs-test-mnt/metadata/cat-a/sub-2
mkdir -p /tmp/torrentfs-test-mnt/metadata/cat-b

# 从批量目录 mv 种子到分类目录
for f in "$BATCH_DIR"/*.torrent; do
    # 轮流转到不同目录
    case $((RANDOM % 3)) in
        0) mv "$f" /tmp/torrentfs-test-mnt/metadata/cat-a/sub-1/ 2>/dev/null ;;
        1) mv "$f" /tmp/torrentfs-test-mnt/metadata/cat-a/sub-2/ 2>/dev/null ;;
        2) mv "$f" /tmp/torrentfs-test-mnt/metadata/cat-b/ 2>/dev/null ;;
    esac
done

# 验证 data/ 已同步更新
echo "=== cat-a/sub-1 ==="
ls /tmp/torrentfs-test-mnt/data/cat-a/sub-1/ 2>/dev/null | wc -l
echo "=== cat-a/sub-2 ==="
ls /tmp/torrentfs-test-mnt/data/cat-a/sub-2/ 2>/dev/null | wc -l
echo "=== cat-b ==="
ls /tmp/torrentfs-test-mnt/data/cat-b/ 2>/dev/null | wc -l

# 用户期望：data/ 结构与 metadata/ 完全镜像

# 4. 快速连续 rm 删除
echo "删除 cat-a 整个子树..."
rm -rf /tmp/torrentfs-test-mnt/metadata/cat-a

# 验证 data/ 同步清理
echo "data/cat-a 剩余:"
ls /tmp/torrentfs-test-mnt/data/cat-a/ 2>/dev/null || echo "(目录已正确清除)"
# 期望：data/cat-a 不存在或为空

# 5. 遍历最终 data/ 完整性
echo "最终 data/ 结构:"
find /tmp/torrentfs-test-mnt/data/ -type d | sort
echo "最终 metadata/ 结构:"
find /tmp/torrentfs-test-mnt/metadata/ -type f -name '*.torrent' | sort
# 期望：data/ 目录数 = metadata/ 种子文件数（一对一）
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 批量 cp 20+ 种子无报错 | 全部成功写入 |
| 实时镜像 | 写入后 data/ 立即出现对应目录（无延迟） |
| mv 种子到子目录 | data/ 同步 mv，不残留 |
| 多层目录结构镜像 | metadata/a/b/c/seed → data/a/b/c/seed/ |
| rm -rf 递归删除 | data/ 对应目录同步清理 |
| 批量操作后 data/ = metadata/ 一一对应 | 种子文件数 = data/ 目录数 |
| 操作过程中并发 ls data/ | 不崩溃、不挂死 |
| 快速连续操作无丢事件 | 无遗漏 mv/rm 事件 |

## 边界与压力测试

这些场景模拟"好奇的高级用户"行为：

| 测试 | 操作 | 期望 |
|------|------|------|
| 超大种子 | 写入包含 1000+ 文件的种子 | 不崩溃，readdir 性能可接受 |
| 极深路径 | metadata/a/b/c/d/e/f/seed.torrent | 不崩溃 |
| 特殊字符 | 种子文件名含空格/中文/emoji | 正确处理（或明确拒绝） |
| 种子名与内部文件名冲突 | 种子名为 file.txt 内含 file.txt | 正确处理 |
| 同 info_hash 不同 source_path | 同一 .torrent 复制到两个子目录 | 允许，各自独立 |
| 同 info_hash 同 source_path | 同一子目录重复写入 | 拒绝或覆盖（行为需定义） |
| 空目录 | data/ 下空目录的行为 | `ls` 返回空，不报错 |
| 卸载时正在读取 | `fusermount -u` 时仍有活跃读取 | 等待或强制卸载，不挂死 |
| 0 字节种子 | 空文件写入 metadata/ | EINVAL |
| 种子文件无读权限 | metadata/ 下种子 chmod 000 | 合理报错 |

### 场景 10：配置文件 —— libtorrent session 参数

torrentfs 支持通过 `--config` 参数传入 TOML 配置文件来覆盖 libtorrent session settings。

```bash
# 1. 无配置文件启动（向后兼容）
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache
# 期望：正常启动，使用所有默认值

fusermount -u /tmp/torrentfs-test-mnt

# 2. 创建最简配置文件
cat > /tmp/torrentfs-config.toml << 'TOML'
[connections]
listen_interfaces = "0.0.0.0:16881"
max_connections = 50

[cache]
cache_size = 536870912

[rate_limits]
download_rate_limit = 1048576
upload_rate_limit = 524288
TOML

# 3. 使用配置文件启动
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache \
  --config /tmp/torrentfs-config.toml
# 期望：正常启动，应用自定义配置

fusermount -u /tmp/torrentfs-test-mnt

# 4. 无效配置文件 - 应报错退出
echo "not valid toml {{{" > /tmp/torrentfs-bad.toml
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --config /tmp/torrentfs-bad.toml 2>&1
echo "exit code: $?"
# 期望：非 0 退出码，错误信息明确

# 5. 部分配置（只设部分字段，其余用默认值）
cat > /tmp/torrentfs-partial.toml << 'TOML'
[dht]
enabled = false

[encryption]
encryption_policy = 1
TOML

./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache \
  --config /tmp/torrentfs-partial.toml
# 期望：正常启动，DHT 禁用 + 加密启用，其余保持默认

fusermount -u /tmp/torrentfs-test-mnt

# 6. 清理
rm -f /tmp/torrentfs-config.toml /tmp/torrentfs-bad.toml /tmp/torrentfs-partial.toml
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 无 --config 启动 | 默认值生效，行为与旧版一致 |
| --config 有效文件 | 配置被应用，listen_interfaces / cache_size / rate_limits 生效 |
| --config 无效文件 | 非零退出 + 明确错误信息（非静默忽略） |
| 部分配置 | 仅覆盖指定项，其余使用默认值 |
| --db / --cache 与配置文件共存 | 不冲突，各自独立作用域 |

### 场景 11：.stats 虚拟文件 —— 系统状态查看

挂载点根目录下的 `.stats` 只读虚拟文件实时展示系统运行状态。

```bash
# 1. 挂载并写入种子
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache &

sleep 1

cp /workspace/torrentfs/ubuntu.iso.torrent /tmp/torrentfs-test-mnt/metadata/

# 2. 检查 .stats 是否可见
ls -la /tmp/torrentfs-test-mnt/
# 期望：-a 可见 .stats

ls -la /tmp/torrentfs-test-mnt/.stats
# 期望：显示为常规文件

# 3. 读取 .stats 内容
cat /tmp/torrentfs-test-mnt/.stats
# 期望：输出包含所有数据段

# 4. 检查各数据段存在
cat /tmp/torrentfs-test-mnt/.stats | grep "概况"
cat /tmp/torrentfs-test-mnt/.stats | grep "全局速率"
cat /tmp/torrentfs-test-mnt/.stats | grep "连接"
cat /tmp/torrentfs-test-mnt/.stats | grep "种子总览"
cat /tmp/torrentfs-test-mnt/.stats | grep "种子详情"
cat /tmp/torrentfs-test-mnt/.stats | grep "缓存详情"
cat /tmp/torrentfs-test-mnt/.stats | grep "性能"
# 期望：每项都有对应内容

# 5. 检查种子详情段包含种子名
cat /tmp/torrentfs-test-mnt/.stats | grep "ubuntu"
# 期望：种子详情段列出了 ubuntu 种子

# 6. 检查缓存详情段按 info_hash 聚合
cat /tmp/torrentfs-test-mnt/.stats | grep 'info_hash'
# 期望：至少有一个 info_hash 条目

# 7. 检查关联种子在 info_hash 下列出
cat /tmp/torrentfs-test-mnt/.stats | grep "source_path"
# 期望：每个 info_hash 条目下列出关联种子和 source_path

# 8. 实时性验证 —— 写入新种子后数据更新
cp /workspace/torrentfs/debian.torrent /tmp/torrentfs-test-mnt/metadata/
sleep 1
cat /tmp/torrentfs-test-mnt/.stats | grep "种子实例"
# 期望：种子数从 1 变为 2（或更多）

# 9. .stats 只读性
echo "test" > /tmp/torrentfs-test-mnt/.stats 2>&1
echo "write exit: $?"
# 期望：Permission denied

rm /tmp/torrentfs-test-mnt/.stats 2>&1
echo "rm exit: $?"
# 期望：Operation not permitted

# 10. stat .stats
stat /tmp/torrentfs-test-mnt/.stats
# 期望：权限 0444，类型为 regular file

fusermount -u /tmp/torrentfs-test-mnt
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| .stats 可见 | `ls -la` 列出 `.stats` |
| 内容包含所有数据段 | 概况 / 速率 / 连接 / 种子总览 / 种子详情 / 缓存详情 / 性能 |
| 种子详情含种子名 | 列出所有已写入种子 |
| 缓存按 info_hash 聚合 | `info_hash` 条目，非按种子 |
| 关联种子展开列出 | 每个 info_hash 下列出 source_path 和种子名 |
| 数据实时更新 | 新种子写入后再次 cat 种子数增加 |
| 只读保护 | write/rm 被拒绝 |
| stat 权限正确 | 0444，regular file |


---

## 输出格式

### PASS 报告（用户视角）

在 issue 评论中使用以下格式：

```
**QA_PASSED (User Simulation)**

模拟用户执行了以下旅程：

📋 场景覆盖：
- [x] 场景1 初次写入浏览 — 正常
- [x] 场景2 子目录组织 — 正常
- [x] 场景3 删除重命名 — 正常
- [x] 场景4 文件读取 — 数据正确
- [x] 场景5 错误拒绝 — 反馈清晰
- [x] 场景6 并发操作 — 无崩溃
- [x] 场景7 重启恢复 — 状态保留

🛡 边界测试：通过 (N 项)

用户体验评估：☑ 顺畅  ☐ 有小问题  ☐ 严重问题

@<assignee> 可以合并。
```

### FAIL 报告（用户视角）

```
**QA_FAILED (User Simulation)**

用户在以下场景遇到问题：

🔴 用户困惑 #1：[场景X]
- 用户做了什么：[具体命令]
- 用户看到什么：[实际输出/行为]
- 用户期望什么：[合理预期]
- 为什么这是问题：[用户体验影响]

🔴 用户困惑 #2：...

🟡 体验瑕疵：
- [不致命但让用户疑惑的行为]

边界测试：X/N 通过，失败项见上。

请修复后重新提交。
```

---

## Bug 报告模板（用户故事格式）

当发现 Bug 时，用用户故事描述而非技术术语：

```
Title: Bug: <一句用户视角描述>

As a <角色>, I want to <操作>, but <遇到的问题>.

Steps to reproduce:
1. 挂载 torrentfs
2. <具体 shell 命令>
3. <观察到的错误>

Expected: <用户期望的行为>
Actual: <实际发生的>
Impact: <对用户的影响：困惑/阻塞/数据丢失>
```

---

## 基本原则

1. **先做用户，再做 QA**——用直觉操作，不要按清单走
2. **错误信息面向用户**——返回的错误码/信息应让普通用户能理解发生了什么
3. **行为一致**——与常规文件系统的行为差异越小越好
4. **不猜测设计意图**——如果行为与 DESIGN.md 矛盾，这是 Bug，直接报告
5. **记录环境**——报告问题附版本、内核版本、FUSE 版本


---

## 签发标准

### 优先级分层

| 级别 | 内容 | 影响 |
|------|------|------|
| **P0** | 挂载/卸载（场景1）、浏览 data/（场景2）、文件读取正确性（场景4） | 阻断发布 |
| **P1** | 子目录组织（场景2）、删除重命名（场景3）、metadata/ 实时镜像 | 核心体验 |
| **P2** | 错误拒绝（场景5）、并发操作（场景6）、重启恢复（场景7）、缓存（场景8） | 完整功能 |
| **P3** | 批量操作（场景9）、边界压力测试（特殊字符/超大种子/极深路径） | 边缘场景 |

### 门禁

| 裁决 | 条件 |
|------|------|
| **QA_PASSED** | P0 全部通过 + P1 全部通过 |
| **QA_FAILED** | 任何 P0 或 P1 项失败 |

### 裁决输出格式

```
**QA_PASSED** 或 **QA_FAILED**

Commit: `$COMMIT`
Branch: `$REMOTE_BRANCH`

| # | 场景 | 命令 | 退出码 | 结果 |
|---|------|------|:------:|------|
| N | <场景名> | <命令> | <code> | <单行 ≤80 字> |
```

