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

种子准备统一使用两个来源，不要依赖任何外部路径：

**1. selfseed 离线 swarm（内容读取场景专用，有真实做种者）**

仓库自带一键脚本 `ci/run_self_seed_env.sh`，在本机自建一个离线 swarm（tracker + 一个做种者），产物固定为单文件种子 `selfseed.torrent`（4 MiB 全 `a` payload，16 pieces × 256 KiB）。

```bash
cd <repo>
./ci/run_self_seed_env.sh            # 构建并启动 tracker+seeder，保持前台运行
# 产物：ci/selfseed/output/selfseed.torrent
#       ci/selfseed/output/payload.txt
#       ci/selfseed/output/tracker.url
```

该脚本内部写死 `info.name="selfseed"`，内部文件路径为 `<mount>/data/<source_path>/selfseed.torrent/selfseed`（目录名是完整种子文件名，含 `.torrent` 后缀）。
**该脚本是前台进程，必须先启动它（另开终端保持运行），`ci/selfseed/output/selfseed.torrent` 生成后才能并入 `$SEED_DIR`——顺序不能颠倒。**
内容读取场景必须使用这个种子：命名种子没有做种者，读内容约 40s 后返回 ENODATA（No seeder connected），无法完成下载。

**2. Docker 变体**

`run_self_seed_env.sh` 不透传 `--announce-host` / `--tracker-bind`，Docker 内运行的 torrentfs 无法直连宿主机 loopback。此时直接跑二进制（`<host 可达 IP>` 为容器可达的宿主机地址）：

```bash
cargo build --locked --release --example torrentfs-selfseed-env --quiet
target/release/examples/torrentfs-selfseed-env \
  --tracker-bind 0.0.0.0 --announce-host <host 可达 IP> \
  --payload ci/selfseed/output/payload.txt \
  --torrent-out ci/selfseed/output/selfseed.torrent \
  --url-out ci/selfseed/output/tracker.url
```

`--payload` / `--torrent-out` / `--url-out` 显式指定，避免默认相对路径 `payload.txt` 在仓库根找不到文件；产物与脚本相同。场景 12 的容器内容读取需要用这种 announce 可达的种子（或 `docker run --network host`，见场景 12）。

**3. 命名种子：内嵌 python3 生成器 + 种子目录合并（无做种者）**

下面脚本纯标准库、确定性产出命名种子。这些种子只用于验证目录结构、文件列表、`.stats` 格式等**不读取内容**的场景。脚本同时把 selfseed 产物并入 `$SEED_DIR`：

```bash
SEED_DIR=/tmp/torrentfs-seeds
mkdir -p "$SEED_DIR"

# 1) selfseed 产物并入（必须先已运行 run_self_seed_env.sh）
cp <repo>/ci/selfseed/output/selfseed.torrent "$SEED_DIR/"

# 2) 命名种子：python3 内联执行，标准库即可，不依赖外部脚本文件
python3 - "$SEED_DIR" <<'PYEOF'
import sys
import hashlib


def bencode(x):
    if isinstance(x, int):
        return b"i" + str(x).encode() + b"e"
    if isinstance(x, bytes):
        return str(len(x)).encode() + b":" + x
    if isinstance(x, str):
        b = x.encode()
        return str(len(b)).encode() + b":" + b
    if isinstance(x, list):
        return b"l" + b"".join(bencode(i) for i in x) + b"e"
    if isinstance(x, dict):
        out = bytearray(b"d")
        for k in sorted(x.keys()):
            out += bencode(k)
            out += bencode(x[k])
        out += b"e"
        return bytes(out)
    raise TypeError(type(x))


def build_torrent(path, name, entries):
    piece_len = 262144
    total = sum(s for _, s in entries)
    full, rem = total // piece_len, total % piece_len
    data = b"a" * (full * piece_len) + (b"b" * rem if rem else b"")
    pieces = bytearray()
    pos = 0
    while pos < len(data):
        pieces += hashlib.sha1(data[pos:pos + piece_len]).digest()
        pos += piece_len
    files = [{"length": s, "path": list(p)} for p, s in entries]
    info = {"name": name, "piece length": piece_len, "pieces": bytes(pieces)}
    if len(files) == 1:
        info["length"] = files[0]["length"]
    else:
        info["files"] = files
    with open(path, "wb") as f:
        f.write(bencode({"announce": "http://127.0.0.1:16969/announce", "info": info}))


out = sys.argv[1]
build_torrent(f"{out}/ubuntu.iso.torrent", "ubuntu.iso", [(("ubuntu.iso",), 4194304)])
build_torrent(f"{out}/debian.torrent", "debian.iso", [(("debian.iso",), 1048576)])
build_torrent(f"{out}/fedora.torrent", "fedora.iso", [(("fedora.iso",), 524288)])
build_torrent(f"{out}/freebsd.torrent", "freebsd.iso", [(("freebsd.iso",), 262144)])
build_torrent(f"{out}/movie.torrent", "movie",
              [(("file_A",), 1048576),
               (("file_B",), 262144),
               (("subdir", "file_C"), 524288)])
PYEOF

ls -1 "$SEED_DIR"
```

**种子 ↔ 场景映射**：

| 种子 | info.name | 大小 | 用途 |
|------|-----------|------|------|
| selfseed.torrent | selfseed | 4 MiB（16×256 KiB） | 所有内容读取场景（Peers>0、dd/cat、缓存） |
| ubuntu.iso.torrent | ubuntu.iso | 4 MiB | 单文件结构/元数据、Pieces 格式 |
| debian.torrent | debian.iso | 1 MiB | 单文件结构、小文件 |
| fedora.torrent | fedora.iso | 512 KiB | 单文件结构、元数据 churn |
| freebsd.torrent | freebsd.iso | 256 KiB | 单文件结构 |
| movie.torrent | movie | 多文件 | 多文件结构、选择性下载 |

后续场景中的 `cp "$SEED_DIR"/*.torrent` 均指向该目录。

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
cp "$SEED_DIR/ubuntu.iso.torrent" /tmp/torrentfs-test-mnt/metadata/

# 用户期望：data/ 下出现对应目录
ls /tmp/torrentfs-test-mnt/data/
# → ubuntu.iso.torrent/

# 用户期望：进入目录看到文件
ls -la /tmp/torrentfs-test-mnt/data/ubuntu.iso.torrent/
# → ubuntu.iso  （文件名、大小正确）

# 用户期望：文件大小合理
stat /tmp/torrentfs-test-mnt/data/ubuntu.iso.torrent/ubuntu.iso
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
cp "$SEED_DIR/ubuntu.iso.torrent" /tmp/torrentfs-test-mnt/metadata/os/linux/
cp "$SEED_DIR/debian.torrent" /tmp/torrentfs-test-mnt/metadata/os/linux/
cp "$SEED_DIR/movie.torrent" /tmp/torrentfs-test-mnt/metadata/media/movies/

# 用户期望：data/ 下镜像相同结构
ls /tmp/torrentfs-test-mnt/data/os/linux/
# → ubuntu.iso.torrent/  debian.torrent/

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
mv /tmp/torrentfs-test-mnt/metadata/os/linux/ubuntu.iso.torrent \
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
cp "$SEED_DIR/ubuntu.iso.torrent" /tmp/torrentfs-test-mnt/metadata/cat-a/
cp "$SEED_DIR/debian.torrent" /tmp/torrentfs-test-mnt/metadata/cat-a/
cp "$SEED_DIR/fedora.torrent" /tmp/torrentfs-test-mnt/metadata/

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
cp "$SEED_DIR/ubuntu.iso.torrent" /tmp/torrentfs-test-mnt/metadata/cat-a/
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
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/os/linux/
sleep 1
# === 全量读取 ===
# 用户操作：尝试读取一个文件
cat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed > /tmp/test-output.iso

# 用户期望：
# - 命令不立即报错（即使数据尚未下载）
# - 数据逐渐下载（可能较慢，但不应无限阻塞）
# - 最终完整输出文件
# - 输出文件内容正确（与种子 hash 一致）

# 验证输出完整性
ORIG_SIZE=$(stat --format=%s /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed)
OUTPUT_SIZE=$(stat --format=%s /tmp/test-output.iso)
echo "Original: $ORIG_SIZE  Output: $OUTPUT_SIZE"
# 期望：大小一致

# === 部分读取（seek/skip） ===
# 读取文件中间 1KB（从 1MB 偏移处开始）
dd if=/tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed    of=/tmp/test-chunk.bin bs=1024 skip=1024 count=1 2>&1
# 期望：正确读取 1024 字节

# 读取文件开头 4KB
dd if=/tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed    of=/tmp/test-head.bin bs=4096 count=1 2>&1
# 期望：正确读取 4096 字节

# 读取文件末尾 1KB
FILE_SIZE=$(stat --format=%s /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed)
TAIL_OFFSET=$((FILE_SIZE / 1024 - 1))
dd if=/tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed    of=/tmp/test-tail.bin bs=1024 skip=$TAIL_OFFSET count=1 2>&1
# 期望：读取最后 1024 字节，不越界

# === 多次读取（验证缓存加速） ===
time cat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed > /dev/null
echo "First read done"

time cat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed > /dev/null
echo "Second read (should be faster due to cache)"

# === 读取超出文件大小 ===
# 尝试从超出的偏移量读取 → 应返回 0 字节，不 panic
dd if=/tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed    of=/tmp/test-eof.bin bs=1 skip=999999999 count=1 2>&1
echo "EOF read exit code: $?"
# 期望：返回 0（正常 EOF），不报错

# === 查看文件大小（不应为 0） ===
ls -la /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed
stat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed
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
stat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/
stat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed
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
rm /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed
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

### 场景 6：并发读取测试

用户可能同时打开多个终端窗口，或在多个程序里同时读取相同/不同文件。并发场景重点验证：**不崩溃、不丢数据、读出的内容与串行读取完全一致**。

```bash
# 准备：挂载 + 写入两个种子（一个大文件种子，一个小文件种子）
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache &
sleep 1
mkdir -p /tmp/torrentfs-test-mnt/metadata/big /tmp/torrentfs-test-mnt/metadata/small
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/big/
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/small/
sleep 1
BIG=/tmp/torrentfs-test-mnt/data/big/selfseed.torrent/selfseed
SMALL=/tmp/torrentfs-test-mnt/data/small/selfseed.torrent/selfseed
```

#### 6.1 单文件多进程并发全量读

多个进程同时读**同一个文件**，每个进程的输出必须与串行读取一致：

```bash
# 5 个进程同时 cat 同一文件，各自落盘
for i in 1 2 3 4 5; do
  cat "$BIG" > /tmp/conc-read-$i.iso &
done
wait
echo "并发读退出码: $?"

# 完整性：所有输出 md5 一致，且与文件系统 stat 大小一致
for i in 1 2 3 4 5; do
  md5sum /tmp/conc-read-$i.iso
done
stat --format=%s "$BIG"
# 期望：5 个 md5 完全相同；输出大小 == stat 大小；进程全部退出
```

#### 6.2 多文件并发读

不同进程同时读不同文件，互不干扰：

```bash
cat "$BIG" > /tmp/conc-big.iso &
cat "$SMALL" > /tmp/conc-small.iso &
cat "$BIG" > /tmp/conc-big-2.iso &
wait
md5sum /tmp/conc-big.iso /tmp/conc-big-2.iso
# 期望：两个 big 输出 md5 一致；small 读取正常返回
```

#### 6.3 并发读 + 错位偏移读（模拟播放器 seek 行为）

多个 dd 进程从**不同偏移**并发读同一文件，每个分片必须精确：

```bash
FILE_SIZE=$(stat --format=%s "$BIG")
# 并发 8 个不同偏移读取（头部/中部/尾部交叉）
for off in 0 1024 4096 $((FILE_SIZE/4)) $((FILE_SIZE/2)) $((FILE_SIZE*3/4)) $((FILE_SIZE-4096)) 999999; do
  dd if="$BIG" of=/tmp/conc-off-$off.bin bs=1 skip=$off count=4096 2>/dev/null &
done
wait
# 期望：每个 dd 正常退出（exit 0）；越界偏移（999999）返回 0 字节不报错
# 校验：非越界偏移的 dd 输出大小 == 4096（头部偏移 0 时 bs=4096 count=1 同理）
```

#### 6.4 并发读 + metadata 写入（读写混合）

读取进行的同时，向 metadata/ 写入/删除种子，两者互不阻塞、不崩溃：

```bash
# 后台持续读大文件
while true; do cat "$BIG" > /dev/null || break; done &
READER=$!

# 同时做 metadata 操作
cp "$SEED_DIR/fedora.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 1
rm /tmp/torrentfs-test-mnt/metadata/fedora.torrent
sleep 1
cp "$SEED_DIR/fedora.torrent" /tmp/torrentfs-test-mnt/metadata/

# 读取进程应仍在正常运行
kill -0 $READER 2>/dev/null && echo "reader alive" || echo "reader died"
kill $READER 2>/dev/null
wait $READER 2>/dev/null
# 期望：metadata 操作 exit 0；reader 未崩溃；data/ 镜像与 metadata/ 一致
```

#### 6.5 高并发压力测试（20+ 并发读）

```bash
# 20 个并发读者混读大/小文件 + 目录遍历
for i in $(seq 1 10); do
  cat "$BIG" > /dev/null &
  cat "$SMALL" > /dev/null &
done
for i in 1 2 3; do
  find /tmp/torrentfs-test-mnt/data/ | wc -l &
done
wait
echo "压力测试退出码: $?"
# 期望：全部正常退出，无 hang、无 panic、无 segfault
# dmesg 检查（可选）：dmesg | tail -20 无 torrentfs 相关崩溃记录
```

#### 6.6 并发读数据一致性（结果比对）

并发读输出与串行读输出逐字节一致：

```bash
# 串行基准
cat "$BIG" > /tmp/serial.iso
# 并发 5 路
for i in 1 2 3 4 5; do cat "$BIG" > /tmp/conc-$i.iso & done
wait
for i in 1 2 3 4 5; do
  cmp -s /tmp/serial.iso /tmp/conc-$i.iso && echo "conc-$i: 一致" || echo "conc-$i: 不一致 ✗"
done
# 期望：5 路全部"一致"
```

**检查要点**：

| 检查项 | 命令 | 期望 |
|--------|------|------|
| 单文件多进程并发读 | 5×`cat` 同文件 | 全部成功，md5 一致 |
| 多文件并发读 | 同时 cat 不同文件 | 互不干扰，内容正确 |
| 错位偏移并发读 | 多进程不同 offset `dd` | 分片精确，越界返回 0 字节 |
| 读 + metadata 写混合 | 后台 cat + cp/rm 种子 | 不崩溃、不丢事件、镜像一致 |
| 高并发 20+ 读 | 20 进程混读 + find | 无 hang / panic / segfault |
| 并发 vs 串行一致性 | `cmp` 逐字节比对 | 完全一致 |
| 并发读 + 缓存 | 第二次并发读 | 命中缓存，无重复下载 |
| 共享同一文件句柄 | 同一进程内多 fd 读 | 各 fd 偏移独立，不互相影响 |

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

cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/os/linux/
sleep 1
# 2. 首次读取文件（触发下载）
time cat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed > /dev/null
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
time cat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed > /dev/null
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
time cat /tmp/torrentfs-test-mnt/data/os/linux/selfseed.torrent/selfseed > /dev/null
echo "Post-remount read time: (should hit cache, not re-download)"

# 7. 删除种子后缓存清理
rm /tmp/torrentfs-test-mnt/metadata/os/linux/selfseed.torrent
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
BATCH_DIR="/tmp/torrentfs-test-mnt/metadata/batch-import"

mkdir -p "$BATCH_DIR"
# 复制所有可用种子（含命名种子与 selfseed 种子）
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

### 场景 11：分层 .stats 虚拟文件 —— 全局 / 种子 / 目录统计

优化后 `.stats` 分为三层：根目录（全局）、种子叶子目录（单种子）、中间目录（聚合）。

```bash
# 1. 挂载并写入种子到三个不同 source_path
./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache &

sleep 1

mkdir -p /tmp/torrentfs-test-mnt/metadata/os/linux
mkdir -p /tmp/torrentfs-test-mnt/metadata/os/bsd
mkdir -p /tmp/torrentfs-test-mnt/metadata/media

cp "$SEED_DIR/ubuntu.iso.torrent" /tmp/torrentfs-test-mnt/metadata/os/linux/
cp "$SEED_DIR/debian.torrent" /tmp/torrentfs-test-mnt/metadata/os/linux/
cp "$SEED_DIR/freebsd.torrent" /tmp/torrentfs-test-mnt/metadata/os/bsd/
cp "$SEED_DIR/movie.torrent" /tmp/torrentfs-test-mnt/metadata/media/
sleep 1

# ===== 测试 1：根目录 .stats（全局概览） =====
echo "=== Root .stats ==="

# 1a. 可见性
ls -la /tmp/torrentfs-test-mnt/
# 期望：-a 可见 .stats

ls -la /tmp/torrentfs-test-mnt/.stats
# 期望：显示为常规文件

# 1b. 全局内容段（纯英文，compact）
cat /tmp/torrentfs-test-mnt/.stats | grep "Overview"
cat /tmp/torrentfs-test-mnt/.stats | grep "Global Rates"
cat /tmp/torrentfs-test-mnt/.stats | grep "Connections"
cat /tmp/torrentfs-test-mnt/.stats | grep "Torrents"
cat /tmp/torrentfs-test-mnt/.stats | grep "Cache"
# 期望：每项都有对应内容

# 1c. 根目录 .stats 不包含逐种子详情
cat /tmp/torrentfs-test-mnt/.stats | grep -c "torrent: "
# 期望：0（无 "torrent: " 标题行）

# 1d. 全局统计包含种子计数
cat /tmp/torrentfs-test-mnt/.stats | grep "Total:"
# 期望：Total: 4 或类似（4 个种子）

# 1e. stat 属性
stat /tmp/torrentfs-test-mnt/.stats
# 期望：权限 0444，regular file

# 1f. 只读保护
echo "test" > /tmp/torrentfs-test-mnt/.stats 2>&1
echo "write exit: $?"
# 期望：Permission denied

rm /tmp/torrentfs-test-mnt/.stats 2>&1
echo "rm exit: $?"
# 期望：Operation not permitted


# ===== 测试 2：种子叶子目录 .stats（单种子统计） =====
echo "=== Torrent Leaf .stats ==="

# 2a. 种子目录下可见 .stats
ls -la /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/
# 期望：包含 .stats

# 2b. 读取单种子统计
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats
# 期望：输出包含该种子的状态、速率、peers、缓存

# 2c. 内容段检查
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats | grep "torrent: "
# 期望：顶部有 "torrent: ubuntu.iso"

cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats | grep "Status"
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats | grep "Rates"
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats | grep "Peers"
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats | grep "Cache"
cat /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats | grep "info_hash"
# 期望：每个段都存在

# 2d. 不同种子内容不同
cat /tmp/torrentfs-test-mnt/data/os/linux/debian.torrent/.stats | grep "torrent: "
# 期望：显示 "torrent: debian.iso"，与 ubuntu 不同

cat /tmp/torrentfs-test-mnt/data/media/movie.torrent/.stats | grep "torrent: "
# 期望：显示 "torrent: movie..."

# 2e. 只读保护
echo "test" > /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats 2>&1
echo "write exit: $?"
# 期望：Permission denied

rm /tmp/torrentfs-test-mnt/data/os/linux/ubuntu.iso.torrent/.stats 2>&1
echo "rm exit: $?"
# 期望：Operation not permitted


# ===== 测试 3：中间目录 .stats（聚合统计） =====
echo "=== Directory .stats ==="

# 3a. data/ 根目录有 .stats
ls -la /tmp/torrentfs-test-mnt/data/
# 期望：可见 .stats

# 3b. 中间目录有 .stats
ls -la /tmp/torrentfs-test-mnt/data/os/
# 期望：可见 .stats

ls -la /tmp/torrentfs-test-mnt/data/os/linux/
# 期望：可见 .stats

# 3c. 读取 data/ 聚合统计
cat /tmp/torrentfs-test-mnt/data/.stats
# 期望：汇总所有根级种子（4 个：ubuntu, debian, freebsd, movie）

# 3d. 读取中间目录聚合统计
cat /tmp/torrentfs-test-mnt/data/os/.stats
# 期望：汇总 os/ 下所有种子（3 个：ubuntu, debian, freebsd）

cat /tmp/torrentfs-test-mnt/data/os/linux/.stats
# 期望：汇总 os/linux/ 下所有种子（2 个：ubuntu, debian）

# 3e. 内容段检查
cat /tmp/torrentfs-test-mnt/data/os/linux/.stats | grep "directory: "
# 期望：顶部有 "directory: /data/os/linux"

cat /tmp/torrentfs-test-mnt/data/os/linux/.stats | grep "Summary"
cat /tmp/torrentfs-test-mnt/data/os/linux/.stats | grep "Torrents:"
# 期望：显示 2 个 torrents

cat /tmp/torrentfs-test-mnt/data/os/linux/.stats | grep "Torrents --"
# 期望：种子列表段包含两个种子摘要行

# 3f. 只读保护
echo "test" > /tmp/torrentfs-test-mnt/data/os/.stats 2>&1
echo "write exit: $?"
# 期望：Permission denied

rm /tmp/torrentfs-test-mnt/data/os/.stats 2>&1
echo "rm exit: $?"
# 期望：Operation not permitted


# ===== 测试 4：实时性验证 =====
echo "=== Real-time update ==="

# 写入新种子后根目录 .stats 更新
cp "$SEED_DIR/fedora.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 1
cat /tmp/torrentfs-test-mnt/.stats | grep "Total:"
# 期望：Total: 5（从 4 变为 5）

# 新种子所在目录的 .stats 更新
cat /tmp/torrentfs-test-mnt/data/.stats | grep "Torrents:"
# 期望：Torrents: 5


# ===== 测试 5：没有 .stats 的目录 =====
echo "=== No .stats in internal torrent sub-dirs ==="

# Torrent 内部的子目录不应有 .stats
ls -la /tmp/torrentfs-test-mnt/data/ubuntu.iso.torrent/ 2>&1 | grep "stat"
# 期望：种子内部目录（TorrentDir）不包含 .stats

# 目录内常规文件不显示 .stats
ls -la /tmp/torrentfs-test-mnt/metadata/ 2>&1 | grep "stat"
# 期望：metadata 目录不包含 .stats

fusermount -u /tmp/torrentfs-test-mnt
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 根目录 .stats 可见 | `ls -la /mount/` 列出 `.stats` |
| 根目录 .stats 内容段 | Overview / Global Rates / Connections / Torrents / Cache |
| 根目录不含逐种子详情 | 无 "torrent: " 标题行 |
| 种子目录 .stats 可见 | `ls -la` 种子目录下列出 `.stats` |
| 种子 .stats 内容 | 有 torrent: 标题、Status、Rates、Peers、Cache、info_hash 段 |
| 不同种子 .stats 不同 | 内容随种子变化 |
| data/ .stats 可见 | `ls -la /mount/data/` 列出 `.stats` |
| 中间目录 .stats 可见 | 所有 SourcePathDir 层级列出 `.stats` |
| 中间目录聚合正确 | /data/os/linux 聚合 2 个种子，/data/os 聚合 3 个 |
| 目录 .stats 内容段 | Summary / Torrents / Rates / Peers / Cache |
| 数据实时更新 | 新种子写入后 .stats 种子数增加 |
| 三层只读保护 | write/rm 在三层均被拒绝 |
| 种子内部无 .stats | TorrentDir 子目录不包含 .stats |
| metadata/ 无 .stats | metadata/ 目录不包含 .stats |
| stat 权限正确 | 0444，regular file |


---

### 场景 12：Docker 容器内启动 → 宿主机目录访问测试

验证 torrentfs 在 Docker 容器内启动后，宿主机能否通过共享挂载传播正常访问 FUSE 文件系统。核心测试路径：**宿主机在 `$HOST_DIR` 侧操作**，而非 `docker exec` 进容器执行。torrentfs 项目自带 `Dockerfile` 和 `entrypoint.sh`，支持 FUSE 设备直通和 `rshared` 挂载传播。

```bash
# 准备：构建 Docker 镜像
cd <repo_root>
docker build -t torrentfs:test .
echo "镜像构建完成"
docker images torrentfs:test
```

#### 12.1 宿主机目录准备与共享挂载传播

容器需要宿主机的 FUSE 设备和共享挂载传播：

```bash
# 创建宿主机侧目录（用于容器与宿主机共享 torrentfs 挂载点）
HOST_DIR=/tmp/torrentfs-docker-host
mkdir -p "$HOST_DIR"
# 标记为共享挂载
mount --bind "$HOST_DIR" "$HOST_DIR"
mount --make-shared "$HOST_DIR"
echo "宿主机共享目录已设置: $HOST_DIR"

# 准备种子目录
mkdir -p "$HOST_DIR/metadata"
cp "$SEED_DIR"/*.torrent "$HOST_DIR/metadata/" 2>/dev/null || true
# 注意：内容读取需 tracker 可达。本场景容器用 --network host 直连宿主机；若用 bridge 网络，请改用 'Docker 变体' 中以 --announce-host <host IP> 生成的 selfseed.torrent。
```

#### 12.2 启动容器并挂载 torrentfs

```bash
# 启动容器，直通 /dev/fuse 并将共享目录挂载到容器内
# --network host 让容器直连宿主机 tracker（16969）；bridge 网络下需用 --announce-host <host IP> 重新生成的 selfseed 种子
docker run -d --rm \
  --network host \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  --mount type=bind,source="$HOST_DIR",target=/mnt,bind-propagation=rshared \
  --name torrentfs-test \
  torrentfs:test /mnt

# 等待挂载完成
sleep 3
echo "容器状态:"
docker ps --filter name=torrentfs-test --format "{{.Status}}"

# 验证容器内 torrentfs 已挂载
docker exec torrentfs-test ls /mnt
# 期望：列出 metadata/ data/ 目录

# 验证宿主机也能看到 torrentfs 挂载
ls "$HOST_DIR"
# 期望：宿主机侧的 $HOST_DIR 也显示 metadata/ data/
```

#### 12.3 宿主机侧写入种子（通过共享挂载）

torrentfs 容器通过 `rshared` 挂载传播将 FUSE 挂载点暴露到宿主机 `$HOST_DIR`。宿主机应当能像操作本地 FUSE 挂载一样直接读写该目录——**不需要** `docker exec`。

```bash
# 宿主机侧：将种子放入 metadata/（通过共享挂载点）
HOST_DIR=/tmp/torrentfs-docker-host
mkdir -p "$HOST_DIR/metadata/linux"
cp "$SEED_DIR/ubuntu.iso.torrent" "$HOST_DIR/metadata/linux/"

# 期望：cp 成功，exit 0

# 宿主机侧：验证 data/ 镜像已通过 torrentfs 自动创建
ls "$HOST_DIR/data/linux/"
# 期望：显示 ubuntu.iso.torrent/ 目录（torrentfs 容器内自动镜像到 data/）

# 宿主机侧：浏览 data/ 下的文件
ls -la "$HOST_DIR/data/linux/ubuntu.iso.torrent/"
# 期望：显示内部文件列表（文件名、大小、权限与种子元数据一致）

# 宿主机侧：验证 stat 信息
stat "$HOST_DIR/data/linux/ubuntu.iso.torrent/"
# 期望：权限 0555

# 宿主机侧：创建子目录结构并写入多个种子
mkdir -p "$HOST_DIR/metadata/my-seeds"
cp "$SEED_DIR/debian.torrent" "$HOST_DIR/metadata/my-seeds/"
cp "$SEED_DIR/fedora.torrent" "$HOST_DIR/metadata/my-seeds/"

# 验证 data/ 镜像同步
ls "$HOST_DIR/data/my-seeds/"
# 期望：debian.torrent/  fedora.torrent/ 两个目录

# 宿主机侧：浏览种子内部文件
ls -la "$HOST_DIR/data/my-seeds/debian.torrent/"
# 期望：显示种子内包含的文件
```

#### 12.4 宿主机侧读取文件（触发懒加载）

宿主机通过 `$HOST_DIR/data/` 直接读取 torrentfs 文件，触发容器内 libtorrent 的懒加载下载。

```bash
HOST_DIR=/tmp/torrentfs-docker-host

cp "$SEED_DIR/selfseed.torrent" "$HOST_DIR/metadata/my-seeds/" && sleep 1
# 宿主机侧：全量读取文件（触发懒加载下载）
cat "$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" > /tmp/host-read.iso &
READ_PID=$!
sleep 5
# 检查读取是否仍在进行
kill -0 $READ_PID 2>/dev/null && echo "host reading in progress" || echo "host read completed or failed"
wait $READ_PID 2>/dev/null
echo "宿主机侧读取退出码: $?"
# 期望：正常读取（可能较慢，取决于下载速度）

# 宿主机侧：部分读取（seek 到中间偏移）
dd if="$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" \
  bs=4096 count=10 skip=0 2>/dev/null | wc -c
# 期望：输出 40960（40KB，10 个块）

# 宿主机侧：读取头部 4KB
dd if="$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" \
  bs=4096 count=1 2>/dev/null | wc -c
# 期望：4096

# 宿主机侧：再次读取（应命中缓存，更快）
time cat "$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" > /dev/null
echo "宿主机第二次读取完成（应命中缓存）"
# 期望：第二次明显快于第一次
```

#### 12.5 宿主机侧数据一致性验证

宿主机通过 `$HOST_DIR` 直接读取 torrentfs 数据，验证 `data/` 镜像的完整性和正确性。

```bash
HOST_DIR=/tmp/torrentfs-docker-host

# 宿主机侧：data/ 目录结构应与 metadata/ 种子文件一一对应
echo "metadata/ 种子文件:"
find "$HOST_DIR/metadata/" -name '*.torrent' | sort
echo "data/ 镜像目录:"
find "$HOST_DIR/data/" -type d -not -name '.' 2>/dev/null | sort | head -20
# 期望：data/ 下每个种子目录对应该种子在 metadata/ 中的 source_path

# 宿主机侧：stat 种子目录下的文件（验证大小和权限）
ls -la "$HOST_DIR/data/my-seeds/selfseed.torrent/"
# 期望：显示文件列表，大小 > 0，权限 0444

STAT_SIZE=$(stat --format=%s "$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" 2>/dev/null)
echo "文件大小: $STAT_SIZE"
# 期望：大小 > 0（种子元数据中的文件大小）

# 宿主机侧：部分读取（头部 16KB）
dd if="$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" \
  bs=4096 count=4 skip=0 2>/dev/null | wc -c
# 期望：16384（16KB）

# 宿主机侧：读取 linux/ 目录下的 ubuntu 种子
ls -la "$HOST_DIR/data/linux/ubuntu.iso.torrent/" 2>/dev/null
# 期望：种子目录可见，文件正常

# 宿主机侧：多次读取同一区域，内容应一致
dd if="$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" \
  bs=4096 count=4 skip=0 2>/dev/null | md5sum
dd if="$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" \
  bs=4096 count=4 skip=0 2>/dev/null | md5sum
# 期望：两次 md5 一致
```

#### 12.6 宿主机侧错误操作拒绝

宿主机对 `data/` 的写操作应被 torrentfs 拒绝（只读语义通过共享挂载传播到宿主机）。

```bash
HOST_DIR=/tmp/torrentfs-docker-host

# 宿主机侧：向 data/ 写入应被拒绝
echo "test" > "$HOST_DIR/data/test.txt" 2>&1
echo "exit: $?"
# 期望：Permission denied 或 Read-only file system

# 宿主机侧：删除 data/ 文件应被拒绝
rm "$HOST_DIR/data/my-seeds/selfseed.torrent/selfseed" 2>&1
echo "exit: $?"
# 期望：Operation not permitted

# 宿主机侧：在 data/ 下创建目录应被拒绝
mkdir "$HOST_DIR/data/newdir" 2>&1
echo "exit: $?"
# 期望：Read-only file system

# 宿主机侧（可选交叉验证）：容器内错误操作同样被拒绝
docker exec torrentfs-test touch /mnt/data/my-seeds/test 2>&1
echo "container touch exit: $?"
# 期望：Permission denied 或 Read-only file system
```

#### 12.7 容器重启后数据持久性（宿主机侧验证）

宿主机通过 `$HOST_DIR/metadata/` 写入种子，重启容器后宿主机仍应通过 `$HOST_DIR/data/` 看到相同的目录结构。

```bash
HOST_DIR=/tmp/torrentfs-docker-host

# 宿主机侧：写入种子到 metadata/ 下的 persistent/ 目录
mkdir -p "$HOST_DIR/metadata/persistent"
cp "$SEED_DIR/selfseed.torrent" "$HOST_DIR/metadata/persistent/"
cp "$SEED_DIR/ubuntu.iso.torrent" "$HOST_DIR/metadata/persistent/"

# 宿主机侧：验证 data/ 镜像同步
ls "$HOST_DIR/data/persistent/"
# 期望：selfseed.torrent/ 和 ubuntu.iso.torrent/ 存在

# 重启容器
docker restart torrentfs-test
sleep 3

# 宿主机侧：验证重启后数据仍在
ls "$HOST_DIR/data/persistent/"
# 期望：persistent/ 目录及种子仍在

# 宿主机侧：浏览种子目录文件
ls -la "$HOST_DIR/data/persistent/ubuntu.iso.torrent/"
# 期望：文件可见，stat 大小不为 0

# 宿主机侧：读取确认功能正常
dd if="$HOST_DIR/data/persistent/selfseed.torrent/selfseed" \
  bs=4096 count=1 2>/dev/null | wc -c
# 期望：4096（读取正常）

# 宿主机侧：再写入一个新种子（结构），验证重启后仍工作
cp "$SEED_DIR/debian.torrent" "$HOST_DIR/metadata/persistent/"
docker restart torrentfs-test
sleep 3
ls "$HOST_DIR/data/persistent/"
# 期望：selfseed.torrent/、ubuntu.iso.torrent/ 和 debian.torrent/ 都存在
```

#### 12.8 多容器访问同一挂载（挂载冲突检测）

TSI-2901 起，entrypoint 对同一宿主目录的并发挂载做冲突检测：两个容器通过
`rshared` 绑定同一宿主目录会叠加 FUSE 挂载并互相切断（ENOTCONN）。entrypoint
在挂载前对 mountpoint 目录取排他 `flock`（fd 9 持有整个容器生命周期），并探测
是否已有 FUSE 挂载，二者任一命中即拒绝启动（exit 101），且**不会切断第一个
容器的挂载**。

**情形 A：顺序启动 —— 容器2 拒绝 exit 101，容器1 不受影响**

```bash
HOST_DIR=/tmp/torrentfs-docker-host

# 前置：容器1（torrentfs-test）已由 12.2 正常挂载
docker ps --filter name=torrentfs-test --format "{{.Status}}"
# 期望：Up（容器1 运行中）

# 启动第二个容器，绑定同一共享目录 → 应拒绝启动 exit 101
docker run --rm \
  --network host \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  --mount type=bind,source="$HOST_DIR",target=/mnt2,bind-propagation=rshared \
  --name torrentfs-test-2 \
  torrentfs:test /mnt2 2>&1
echo "container2 exit: $?"
# 期望：exit 101；stderr 含 "[entrypoint] ERROR"

# 验证容器1 不受影响：挂载未被切断，仍可正常访问
docker exec torrentfs-test ls /mnt 2>&1
echo "container1 ls exit: $?"
# 期望：exit 0，列出 metadata/ data/（无 "Transport endpoint is not connected"）
```

**情形 B：同时启动 —— 恰好一个成功，另一个 exit 101**

```bash
# 用全新宿主目录做同时启动竞争（避免与容器1 已持有的 flock 冲突）
HOST_DIR2=/tmp/torrentfs-docker-host2
mkdir -p "$HOST_DIR2"
mount --bind "$HOST_DIR2" "$HOST_DIR2"
mount --make-shared "$HOST_DIR2"
mkdir -p "$HOST_DIR2/metadata"

# 同时启动两个容器，竞争同一共享目录
docker run -d --rm \
  --network host \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  --mount type=bind,source="$HOST_DIR2",target=/mnt-a,bind-propagation=rshared \
  --name torrentfs-test-a \
  torrentfs:test /mnt-a &
docker run -d --rm \
  --network host \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  --mount type=bind,source="$HOST_DIR2",target=/mnt-b,bind-propagation=rshared \
  --name torrentfs-test-b \
  torrentfs:test /mnt-b &
wait
sleep 3

# 恰好一个容器存活（挂载成功），另一个 exit 101
docker ps -a --filter name=torrentfs-test-a --format '{{.Names}}  {{.Status}}'
docker ps -a --filter name=torrentfs-test-b --format '{{.Names}}  {{.Status}}'
# 期望：一个 "Up ..."，另一个 "Exited (101) ..."

# 清理
docker rm -f torrentfs-test-a torrentfs-test-b 2>/dev/null || true
sudo fusermount3 -u "$HOST_DIR2" 2>/dev/null || sudo fusermount -u "$HOST_DIR2" 2>/dev/null || sudo umount "$HOST_DIR2" 2>/dev/null || true
rm -rf "$HOST_DIR2"
```

#### 12.9 无需 --privileged 运行

```bash
# 验证使用 --device /dev/fuse + --cap-add SYS_ADMIN 即可，不需要 --privileged
docker run -d --rm \
  --device /dev/fuse \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  --mount type=bind,source="$HOST_DIR",target=/mnt3,bind-propagation=rshared \
  --name torrentfs-test-no-priv \
  torrentfs:test /mnt3

sleep 3
docker exec torrentfs-test-no-priv ls /mnt3 2>&1
echo "无 --privileged 退出码: $?"
# 期望：正常挂载，ls 可列出目录

# 不使用 --device /dev/fuse — 应失败
docker run -d --rm \
  --cap-add SYS_ADMIN \
  --security-opt apparmor:unconfined \
  --mount type=bind,source="$HOST_DIR",target=/mnt4,bind-propagation=rshared \
  --name torrentfs-test-no-fuse \
  torrentfs:test /mnt4 2>&1
# 期望：容器启动失败或 entrypoint 报错（/dev/fuse 不存在）

# 清理
docker stop torrentfs-test-no-priv 2>/dev/null || true
docker rm -f torrentfs-test-no-fuse 2>/dev/null || true
```

#### 12.10 清理

> **重启容器前，先从宿主机卸载残留挂载点。** `docker stop` 后 torrentfs 退出时
> `bg.join()` 无限阻塞（代码侧缺陷，已独立提 TSI-2874），进程不会优雅退出，最终被
> 容器运行时 SIGKILL，宿主机挂载点残留 stale（读时报
> `Transport endpoint is not connected`）。**延长宽限期无法解决该问题**——无论宽限
> 多长都会被 SIGKILL。`docker stop` 后仍建议 `fusermount3 -u` 兜底；否则直接
> `docker start` 会报 `mkdir <mnt>: file exists`。

```bash
# 停止容器（残留 stale 挂载属已知缺陷 TSI-2874，宽限期无法规避）
docker stop torrentfs-test

# 卸载宿主机共享挂载：需 root/CAP_SYS_ADMIN，建议 sudo；fusermount 报
# `entry not found in /etc/mtab` 时 fallback `sudo umount`。
sudo fusermount3 -u "$HOST_DIR" 2>/dev/null || sudo fusermount -u "$HOST_DIR" 2>/dev/null || sudo umount "$HOST_DIR" 2>/dev/null || true
rm -rf "$HOST_DIR"

# 删除镜像（可选）
docker rmi torrentfs:test 2>/dev/null || true
echo "Docker 测试环境清理完成"
```

**检查要点**：

| 检查项 | 命令 | 期望 |
|--------|------|------|
| 宿主机共享目录设置 | `mount --make-shared` | 挂载传播标记成功 |
| 容器内挂载（验证容器正常启动） | `docker exec ls /mnt` | 列出 metadata/ data/ |
| 宿主机可见性 | `ls "$HOST_DIR"` | 宿主机也看到 metadata/ data/ |
| 宿主机侧写入种子 | `cp .torrent $HOST_DIR/metadata/` | 成功，exit 0 |
| 宿主机侧 data/ 镜像同步 | `ls $HOST_DIR/data/` | 写入后宿主机侧立即看到镜像 |
| 宿主机侧读取文件 | `cat $HOST_DIR/data/...` | 触发下载，正确返回数据 |
| 宿主机侧读取一致性 | `dd` + `md5sum` 比对 | 多次读取 md5 一致 |
| 宿主机侧错误操作拒绝 | `touch $HOST_DIR/data/` | Permission denied |
| 宿主机侧重启后数据持久性 | `docker restart` 后 `ls $HOST_DIR/data/` | 种子目录和文件仍存在 |
| 多容器挂载冲突（顺序） | 容器2 绑定同一宿主目录启动 | exit 101，容器1 挂载不受影响 |
| 多容器挂载冲突（同时） | 两容器同时启动竞争同一目录 | 恰好一个存活，另一个 exit 101 |
| 无需 --privileged | `--device /dev/fuse` 即可 | 正常工作 |
| 缺少 /dev/fuse | 不加 `--device` | 启动失败或报错 |

---

## 选择性下载与下载进度场景

用户只读了文件的一小段，系统就只下载那一小段附近的 piece，而不是整个文件。下面这些场景验证：`.stats` 里的 `-- Pieces --` 一行如何随用户的读取行为变化。

**前置条件**：构建并挂载（见「前提条件」），准备一个大文件种子（单文件，如 ubuntu.iso.torrent）和一个多文件种子（含 3+ 文件与子目录）。`.stats` 的 `-- Pieces --` 行用符号表示每个 piece 的状态，实现实际输出以下四种形式：

- `[x]` 已下载到缓存且未被访问过（hit_count = 0）
- `[X n]` 已下载到缓存且被访问过 n 次（X + 空格 + 数字 n）
- `[N]` 优先级为 N（向前方向递减，向后窗口内恒 1）
- `[]` 未下载且不在计划内

---

### 场景 13：复制种子后先看一眼统计（零下载）

用户把种子放进去，还没读任何文件，就想知道这个种子连上了多少做种者。

```bash
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5   # 等 tracker 连接（selfseed swarm 为 loopback-only，无 DHT）

cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep -A 2 "Peers"
# 期望：Peers > 0 或 Seeds > 0，不应看到 "0 peers / 0 seeds" 的健康警告
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 不读文件也能看到连接信息 | Peers 或 Seeds > 0 |
| 无可用做种者 | 出现健康警告，但 .stats 仍可正常读取 |

### 场景 14：只读文件一小段，只下载附近的 piece

用户用 dd 只读文件偏移 2MB 处的 1MB（selfseed 4 MiB，16 个 256 KiB piece），然后看 .stats 的 Pieces 行。

> **限制**：访问窗口默认 4096 MiB，selfseed 仅 4 MiB（16 piece），整个文件都在窗口内——「窗口外 `[]`」断言在本场景不可观测，需 >4 GiB 种子才适用。本场景只验证：读取偏移附近的 piece 被提升/下载，窗口内未读 piece 为 `[1]`。

```bash
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5

# 只读偏移 2MB 处的 1MB（落在 piece 8）
dd if=/tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed \
   bs=1M count=1 skip=2 of=/dev/null
sleep 5

cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep "Pieces"
# 期望：-- Pieces (N pieces, X each) -- 一行；覆盖读取偏移的那个 piece 为 [X n]（已缓存，n ≥ 1）或 [7]（正在下载）
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 只读一小段 | 只有读取偏移附近的 piece 被下载/提升 |
| 未读区域 | 窗口外 piece 保持 `[]`（不下载）；窗口内未读 piece 为 `[1]`（wanted，最低优先级） |
| 优先级从高到低 | 当前 piece `[7]`，向后 `[6]` `[5]` `[4]` `[3]`，窗口内其余 `[1]`，窗口外 `[]`，当前之前（窗口内）`[1]` |

### 场景 15：多文件种子只读其中一个文件

用户在多文件种子里只读文件 A，希望只有文件 A 对应的 piece 被下载，文件 B 不受影响。

> **限制**：仓库 `run_self_seed_env.sh` 只做单文件种子，无多文件做种者。`movie.torrent` 用于验证目录结构与 `.stats` 中多文件 piece 布局；「读文件 A 下载其 piece」的优先级断言需自备一个做种中的多文件种子，否则 `cat` 约 40s 后返回 ENODATA（No seeder connected）——此时请仅验证目录结构（`ls -R data/movie.torrent/`）与 Pieces 行格式。

```bash
cp "$SEED_DIR/movie.torrent" /tmp/torrentfs-test-mnt/metadata/   # 多文件种子
sleep 5

# 只读文件 A
cat /tmp/torrentfs-test-mnt/data/movie.torrent/movie/file_A > /dev/null
sleep 5

cat /tmp/torrentfs-test-mnt/data/movie.torrent/.stats | grep "Pieces"
# 期望：文件 A 的 piece 优先级依次递减（7→6→5→4→3→1，窗口外为 []），文件 B 的 piece 保持 []（不在被访问文件的 piece 范围内）
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 只读文件 A | 文件 A 的 piece 优先级依次递减 |
| 文件 B 不受影响 | 文件 B 的 piece 保持 `[]`（不在被访问文件的 piece 范围内） |

### 场景 16：读完整文件后，下载完的 piece 归入缓存

用户读完整个文件（selfseed 4 MiB），下载完成后再看 .stats，已下载的 piece 应标记为缓存、不再占用下载优先级。

```bash
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5

# 读完整文件（触发下载）
cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed > /dev/null
sleep 10   # 等下载完成

cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep "Pieces"
# 期望：下载完成的 piece 显示 [X n]（已缓存且被访问过，n ≥ 1），未下载的 piece 显示 []
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 下载完成 | piece 变为 `[X n]`（已缓存且被访问过，n ≥ 1），优先级归零 |
| 未下载 piece | 保持 `[]` |

### 场景 17：Pieces 行格式正确

用户想知道 .stats 里 `-- Pieces --` 一行的格式是否稳定、可读。

```bash
cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep -A 1 "Pieces"
# 期望：-- Pieces (N pieces, X each) -- 之后紧跟一行连续的 piece 标记
# 标记只出现 [] / [N] / [x] / [X N] 四种；piece 数量与种子信息一致
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 标记格式 | 只有 `[]` / `[N]` / `[x]` / `[X N]` |
| 数量一致 | piece 数量与种子元数据一致 |

### 场景 18：从文件中间开始读，靠前的内容仍保留低优先级

用户从文件中间读起，期望当前位置之前的内容不会被完全丢弃（保留低优先级，便于回放）。

> **限制**：selfseed 16 piece 全在默认窗口内，「窗口外 `[]`」不可观测；本场景只验证中间读取的当前 piece 提升与向后递减（6→5→4→3→1）、向前 piece 为 `[1]` 而非 `[]`。

```bash
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5

FILE=/tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed
SIZE=$(stat --format=%s "$FILE")
# 从文件中间读取 1MB
dd if="$FILE" bs=1M count=1 skip=$((SIZE/2/1048576)) of=/dev/null
sleep 5

cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep "Pieces"
# 期望：当前偏移的 piece = [7]，向后（文件末尾方向）6→5→4→3→1（窗口外 []），向前（文件开头方向，窗口内）[1]（非 []）
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 中间读取 | 当前 piece `[7]`，向后递减 6→5→4→3→1（窗口外 `[]`） |
| 向前内容 | 之前的内容 `[1]`，不被直接丢弃 |

### 场景 19：同时读两个文件，互不干扰

用户同时读多文件种子的两个文件，希望两个文件的下载进度各自独立。

> **限制**：同上——`movie.torrent` 无做种者，两个 `cat` 约 40s 后返回 ENODATA（No seeder connected）。优先级互不干扰的断言需自备多文件 seeder；否则仅验证结构。

```bash
cp "$SEED_DIR/movie.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5

cat /tmp/torrentfs-test-mnt/data/movie.torrent/movie/file_A > /dev/null &
cat /tmp/torrentfs-test-mnt/data/movie.torrent/movie/file_B > /dev/null &
wait
sleep 5

cat /tmp/torrentfs-test-mnt/data/movie.torrent/.stats | grep "Pieces"
# 期望：文件 A 和文件 B 对应的 piece 各自显示正确的优先级，互不干扰
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 并发读两文件 | 各自 piece 优先级正确 |
| 互不干扰 | 不互相覆盖或串扰 |

### 场景 20：缓存被淘汰后，重新读还能再下载

用户读过某段内容（已缓存），缓存被新内容挤掉后，重新读同一段应能再次下载。

```bash
# 1. 先读一小段，让它进入缓存
dd if=/tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed \
   bs=1M count=1 skip=2 of=/dev/null
sleep 5

# 2. 读文件其它内容（selfseed 共 16 个 piece），把缓存挤到上限，使之前那段被淘汰
cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed > /dev/null
sleep 10

# 3. 重新读之前那一段
dd if=/tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed \
   bs=1M count=1 skip=2 of=/dev/null
sleep 5

cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep "Pieces"
# 期望：被淘汰的那段重新显示 [7] 或 [X n]（被重新提升/下载，重读后 n ≥ 1）
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 淘汰后重读 | 该 piece 重新被提升 `[7]` 或重新下载并缓存 `[X n]`（n ≥ 1） |

### 场景 21：读取时预取（默认访问窗口约 4 GiB）

用户只读了一小段（selfseed 4 MiB，16 piece），系统会顺带把紧跟其后的一小段也标记为"接下来要下载"，但范围有限，不会预取整个文件。

> **限制**：selfseed 整个文件都在 4096 MiB 访问窗口内，「窗口外 `[]`」与「预取范围有限（窗口边界）」不可观测；本场景只验证读取位置之后的 piece 有 `[6][5][4][3]` 预取优先级。

```bash
cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5

# 只读偏移 2MB 处的 1MB
dd if=/tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed \
   bs=1M count=1 skip=2 of=/dev/null
sleep 5

cat /tmp/torrentfs-test-mnt/data/selfseed.torrent/.stats | grep "Pieces"
# 期望：读取位置之后的若干 piece 被预取（[6] [5] [4] [3]），窗口内其余为 [1]，窗口外为 []
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 读取后有预取 | 读取位置之后的 piece 有优先级（`[6]` `[5]` `[4]` `[3]`），不是 `[]` |
| 预取范围有限 | 紧跟其后 4 个 piece 高优先级，窗口内（约 4 GiB）其余 `[1]`，窗口外 `[]` |

### 场景 22：读取时日志保持精简

用户开启 debug 日志读文件，期望日志只有摘要，不刷屏。

```bash
RUST_LOG=debug ./target/release/torrentfs /tmp/torrentfs-test-mnt \
  --db /tmp/torrentfs-test-state/metadata.db \
  --cache /tmp/torrentfs-test-state/cache 2> /tmp/torrentfs-debug.log &
sleep 2

cp "$SEED_DIR/selfseed.torrent" /tmp/torrentfs-test-mnt/metadata/
sleep 5
dd if=/tmp/torrentfs-test-mnt/data/selfseed.torrent/selfseed \
   bs=1M count=1 skip=2 of=/dev/null

grep -i "piece" /tmp/torrentfs-debug.log
# 期望：可见读取摘要日志（一行）；没有逐 piece 的轮询刷新日志；异常时有 warn 日志
```

**检查要点**：

| 检查项 | 期望 |
|--------|------|
| 读取摘要日志 | 有，且简洁（一行） |
| 无逐 piece 刷屏 | 没有 per-piece 轮询循环日志 |
| 异常告警 | 异常时有 warn 日志 |

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
- [x] 场景6 并发读取测试 — 数据一致、无崩溃
- [x] 场景7 重启恢复 — 状态保留
- [x] 场景12 Docker 容器访问 — 宿主机一致性

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
| **P3** | 批量操作（场景9）、边界压力测试（特殊字符/超大种子/极深路径）、Docker 容器访问（场景12） | 边缘场景 |

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
