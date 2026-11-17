# Snapshot Bench 窗口级 perf 采集 · 执行手册（ARM 宿主机）

代码改动已完成，**执行者不需要修改任何源码**。按本文档构建、采集、分析即可。

被测范围：仅延迟类用例（`snapshot_creation*`、`snapshot_resume*`、`concurrent_resume`）。不覆盖吞吐用例。

## 0. 已实现的能力

`crates/benchmarks/benches/snapshot_benchmark.rs` 中已加入 perf 门控：

- 环境变量 `AENV_BENCH_PERF_CTL`：指向 `perf record --control=fifo:...` 的 ctl FIFO
- 环境变量 `AENV_BENCH_PERF_ACK`：可选 ack FIFO，用于等待 perf 确认命令生效
- 仅在计时区间（`pause`、`resume_from_snapshot_config`）内自动 `enable` / `disable` 采样
- bounded 与 Criterion（`AENV_BENCH_FULL=1`，即 `--profile-time` 路径）两条执行路径均已接入
- 并发用例通过引用计数合并窗口，不会相互关闭
- 未设置上述环境变量时，行为与改动前完全一致

**必须使用 `dev_perf` 分支**：门控代码只存在于该分支，`dev` 分支没有。

执行前确认：

```bash
cd /path/to/AgentENV
git switch dev_perf          # 门控代码在此分支
grep -c "AENV_BENCH_PERF_CTL" crates/benchmarks/benches/snapshot_benchmark.rs
# 输出应 >= 1；为 0 说明分支不对
```

## 1. 前置检查

```bash
# perf 是否支持运行时门控
perf record --help | grep -A3 -- '--control'
perf record --help | grep -- '-D'

# ARM 上的可用事件（优先软件事件 cpu-clock）
perf list | grep -E 'cpu-clock|cycles|task-clock'

# 必须能以 root 运行 perf
cat /proc/sys/kernel/perf_event_paranoid

# 工具链
cargo --version && rustc -vV | grep '^host'
```

判定：

| 检查项 | 不满足时 |
| --- | --- |
| 无 `--control` / `-D` | 停止，记录 perf 版本，改走第 7 节降级方案 |
| 无 `cycles` | 全程用 `-e cpu-clock` |
| 无 root | 停止并反馈 |

## 2. 构建

```bash
cargo check -p agentenv-benchmarks --benches
cargo bench -p agentenv-benchmarks --bench snapshot --no-run
```

若 `cargo check` 报错，停止执行并把完整错误贴回，不要自行修改源码。

## 3. Firecracker 二进制（通常无需操作）

若已把下载的二进制替换为自建版本，**本节可完全跳过**，不需要改任何 config。

默认查找路径（[cfg.rs L767-L779](src/cfg.rs)）：

```
<AENV_DEPS_PATH>/firecracker/<version>/firecracker
```

启动时的校验逻辑（[deps/mod.rs L126](src/deps/mod.rs)）只检查该文件**存在且非空**，不校验哈希、不校验版本，也不会重新下载覆盖。因此就地替换即可生效。

仅在二进制放在其他路径时才需要：

```bash
mkdir -p /tmp/aenv-perf
cp config/default.toml /tmp/aenv-perf/config.toml
# 编辑 /tmp/aenv-perf/config.toml，在 [firecracker] 段加入：
#   binary_path = "/abs/path/to/your/firecracker"
# 然后导出 AENV_CONFIG_PATH=/tmp/aenv-perf/config.toml
```

要求：绝对路径、可执行、非空；自建二进制保留符号（`CARGO_PROFILE_RELEASE_DEBUG=true` 或保留 `.debug` 文件），否则火焰图只有地址。

## 4. 执行采集

```bash
cd /path/to/AgentENV

mkfifo /tmp/perf.ctl /tmp/perf.ack
chmod 666 /tmp/perf.ctl /tmp/perf.ack   # bench 以非 root 运行，perf 以 root 运行

# ARM 上为 AARCH64_UNKNOWN_LINUX_GNU；不要写死 x86_64
HOST_ENV=$(rustc -vV | sed -n 's/^host: //p' | tr '[:lower:]-' '[:upper:]_')
CARGO_BIN=$(command -v cargo)

# 所有 AENV_* 都通过 env 内联传入，不依赖 sudo -E（sudoers 可能清空变量）
sudo env \
  HOME="$HOME" PATH="$PATH" \
  CARGO_HOME="${CARGO_HOME:-$HOME/.cargo}" \
  RUSTUP_HOME="${RUSTUP_HOME:-$HOME/.rustup}" \
  AENV_HOME_PATH=/tmp/aenv-perf/home \
  AENV_RUNTIME_PATH=/tmp/aenv-perf/run \
  AENV_DEPS_PATH=/var/lib/aenv/deps \
  AENV_BENCH_FULL=1 \
  AENV_BENCH_PERF_CTL=/tmp/perf.ctl \
  AENV_BENCH_PERF_ACK=/tmp/perf.ack \
  CARGO_TARGET_${HOST_ENV}_RUNNER="$PWD/scripts/run-with-capabilities.sh" \
  perf record \
    -g --call-graph dwarf \
    -e cpu-clock -F 999 \
    -D -1 --control=fifo:/tmp/perf.ctl,/tmp/perf.ack \
    -o /tmp/aenv-perf/perf.data -- \
    "$CARGO_BIN" bench -p agentenv-benchmarks --bench snapshot -- \
    '^snapshot_creation$' --profile-time 120
```

> 二进制不在默认 deps 路径时，在上面的 `env` 列表中再加一行 `AENV_CONFIG_PATH=/tmp/aenv-perf/config.toml`。

参数要点：

| 参数 | 作用 |
| --- | --- |
| `-D -1` | 启动即禁用采样，等门控 |
| `--control=fifo:ctl,ack` | 由 bench 自动开关 |
| `-e cpu-clock -F 999` | ARM 通用软件事件；`cycles` 可用时可替换 |
| `--call-graph dwarf` | 解栈失败改 `--call-graph fp` |
| `'^snapshot_creation$'` | 正则锚定单个用例，避免命中 `_1gdisk`/`_1gmem` |
| `--profile-time 120` | 持续迭代 120s，提供足够窗口 |

可替换的用例名（逐个替换过滤参数，每次换一个 `-o` 输出文件）：

```
^snapshot_creation$
^snapshot_creation_1gdisk$
^snapshot_creation_1gmem$
^snapshot_resume_cold$
^snapshot_resume$
^concurrent_resume$
```

注意：

- 必须 `sudo` 运行 perf；bench 自身以非 root 运行（`run-with-capabilities.sh` 通过 `setpriv` 赋予 `CAP_NET_ADMIN`/`CAP_SYS_ADMIN`）
- 全部 `AENV_*` 通过 `env` 内联传入，不要依赖 `sudo -E`
- 顺序不能反：perf 必须是 bench 的父进程，否则 FIFO 打开会阻塞
- 首次运行需要解析默认镜像（会访问 registry），后续命中 `$AENV_HOME_PATH/benchmarks/rootfs-image.json` 缓存
- `^concurrent_resume$` 固定使用 50 并发沙箱（不受 `AENV_BENCH_CONCURRENCY` 影响），资源不足时会跳过该用例

## 5. 结果分析

### 5.1 确认门控与子进程

```bash
sudo perf report -i /tmp/aenv-perf/perf.data --stdio \
  --sort comm --percent-limit 0 | head -20
```

- 出现 `firecracker` 且样本数 > 0 → 子进程已覆盖
- 总样本数为 0 → 门控未生效，见第 6 节
- 门控本身**不打印任何日志**（bench 静默写 FIFO），只能通过样本数量与 `comm` 分布判断是否生效
- 交叉验证：`sudo perf script -i /tmp/aenv-perf/perf.data | wc -l`，应远大于 0 且只覆盖若干段短时间窗口
- 数据里会混入约 1 秒的预热窗口：`--profile-time` 模式下 criterion 先跑 1 秒预热，门控在预热迭代里同样生效。占比很小，不单独剔除

### 5.2 用户态 / 内核态占比

```bash
sudo perf report -i /tmp/aenv-perf/perf.data --stdio \
  --sort dso --percent-limit 0 | head -40
```

分类：

- 内核态：`[kernel.kallsyms]`、`[kernel]`
- 用户态：其余（bench 二进制、`firecracker`、`libc`、`[unknown]`、`[vdso]`）

用户态占比 = 用户态样本 / 总样本。采样近似均匀时间采样，比例即时间占比。

### 5.3 热点

```bash
sudo perf report -i /tmp/aenv-perf/perf.data --stdio --sort symbol | head -60

# 火焰图（需要 FlameGraph 脚本）
sudo perf script -i /tmp/aenv-perf/perf.data | \
  stackcollapse-perf.pl | flamegraph.pl > /tmp/aenv-perf/flame.svg
```

### 5.4 可选：精确计数器

若 `cycles` 可用且 `perf stat` 支持 `--control`，把第 4 节的 `perf record ...` 换成：

```bash
sudo env HOME="$HOME" PATH="$PATH" CARGO_HOME="${CARGO_HOME:-$HOME/.cargo}" \
  RUSTUP_HOME="${RUSTUP_HOME:-$HOME/.rustup}" \
  AENV_HOME_PATH=/tmp/aenv-perf/home AENV_RUNTIME_PATH=/tmp/aenv-perf/run \
  AENV_DEPS_PATH=/var/lib/aenv/deps \
  AENV_BENCH_FULL=1 AENV_BENCH_PERF_CTL=/tmp/perf.ctl \
  CARGO_TARGET_${HOST_ENV}_RUNNER="$PWD/scripts/run-with-capabilities.sh" \
  perf stat -e task-clock,cycles:u,cycles:k \
    -D -1 --control=fifo:/tmp/perf.ctl \
    -- "$CARGO_BIN" bench -p agentenv-benchmarks --bench snapshot -- \
    '^snapshot_creation$' --profile-time 60
```

- 用户态占比 = `cycles:u / (cycles:u + cycles:k)`
- 平均并行度 = `task-clock / seconds time elapsed`（后者由 `perf stat` 输出）

## 6. 故障排查

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 样本数为 0 | 门控未生效 | 检查两个环境变量、FIFO 权限 666、`-D -1` |
| bench 启动即卡住 | FIFO 打开阻塞 | 确认 perf 是 bench 的父进程，不要先单独跑 bench |
| 只有 bench 进程，无 firecracker | perf 非 root，或误加 `-i` | 用 `sudo`；不要加 `-i` |
| 火焰图全是地址 | 二进制无符号 | 自建 Firecracker 保留调试信息 |
| `dwarf` 解栈失败 | ARM 缺 unwind 信息 | 改 `--call-graph fp`；构建加 `-C force-frame-pointers=yes` |
| 并发用例窗口偏长 | 多 worker 窗口合并，包含彼此的 stop/sleep | 属预期；单 worker 用例不受影响 |

## 7. 降级方案（perf 无 `--control`）

1. 不设置门控环境变量，直接 `AENV_BENCH_FULL=1` + `'^<case>$'` + `--profile-time`
2. 报告中明确说明 setup 阶段被计入
3. 用 `perf report --sort comm,dso` 人工排除镜像准备/启动阶段的调用栈

## 8. 验收标准（执行者需提交）

1. `cargo check -p agentenv-benchmarks --benches` 通过
2. 未设置门控环境变量时，bench 能正常输出延迟样本（回归验证）
3. 每个被测用例提交：
   - 用例名、`perf.data` 路径
   - 总样本数、用户态占比、内核态占比
   - CPU 时间 / 墙钟比值（平均并行度）
   - 样本中是否出现 `firecracker` comm
4. 附加：`perf report --sort symbol` 前 20 行

## 9. 清理

```bash
rm -f /tmp/perf.ctl /tmp/perf.ack /tmp/aenv-perf/perf.data
```

不要修改、提交 `crates/benchmarks/benches/snapshot_benchmark.rs`；该改动由仓库维护者管理。
