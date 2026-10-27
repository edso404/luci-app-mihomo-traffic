---
title: 全链路零 ubus 依赖与系统资源精简架构优化
date: 2026-10-27
type: refactor
artifact_contract: ce-unified-plan/v1
product_contract_source: ce-brainstorm
execution: code
---

## Goal Capsule

**Objective:** 彻底消除插件运行时对系统 ubus IPC 的直接客户端调用与连接句柄管理负担，改由纯内存、内核 proc 文件系统与静态 UCI 读取实现本机识别与进程状态探测，根除文件描述符耗尽和总线异常风险。
**Means:** 移除所有 `import { connect } from 'ubus'`；`mihomo_traffic.uc` 纯直读 UCI 与 `/proc/net/`；`luci.mihomo-traffic` 状态检查改扫 `/proc`；常驻进程启动期固化本机 IP。
**Product Authority:** 仅覆盖 `luci-app-mihomo-traffic` 内部的本机 IP 探测、别名解析热路径与服务状态查询；不改变前端交互与数据 TSV 存储结构。
**Open Blockers:** None.

## Product Contract

### Summary
实施全链路 100% 零 ubus 客户端化重构，核心库 `mihomo_traffic.uc` 纯靠静态 UCI 与内核网络接口识别本机特征，rpcd 插件 `luci.mihomo-traffic` 剥离冗余的 ubus 事件并改为读取 `/proc` 进程状态，采集端 `mihomo-traffic run` 实施启动期冷热隔离，彻底切断所有跨进程 ubus 连接。

### Problem Frame
此前为了自动将路由器自身流量标为“(本机)”并高亮在线，在核心库中引入了 `ubus call network.interface dump`。在后台常驻轮询和高频查询场景下，密集的 IPC 调用与未妥善释放的 Socket 导致单进程文件描述符瞬间达到 1024 限制，连带把系统总线守护进程 `ubusd` 拖垮，导致全路由器 LuCI 瘫痪。尽管通过加缓存与关闭句柄进行了就地止血，但根本矛盾在于：识别本机 IP 与检查进程存活完全可以在 Linux 本地完成，引入外部 ubus 客户端属于设计过度与反模式。

### Key Decisions
- **完全废弃 ubus 客户端连接** (session-settled: user-directed — chosen over ubus 单例长连接复用: 状态检查与 IP 获取完全可以通过内核文件直读，无需保留 ubus 客户端长连接及其断线重连复杂度) Governs R1, R2, R4.
- **本机 IP 双源直读** (session-settled: user-approved — chosen over 仅读 UCI: 结合 UCI `/etc/config/network` 与 Linux 内核 `/proc/net/if_inet6`、`/proc/net/route`，兼顾静态配置与动态 IPv6/WAN 特征) Governs R1, R3.
- **采集端冷热分层与启动期固化** (session-settled: user-approved — chosen over 每次别名解析时重新扫描系统配置: 本机 IP 在启动时固定加载到内存 Set 中，每秒采集与 30 秒轮询热路径零系统调用读取) Governs R3.

### Requirements

#### 核心库架构退耦
- R1. `root/usr/share/ucode/mihomo_traffic.uc` 必须彻底移除对 `ubus` 模块的引用（`import { connect } from 'ubus'`），退耦为无 IPC 依赖的纯工具库。
- R2. `local_router_ips()` 的本机识别仅通过静态读取 UCI 网络配置（`/etc/config/network` 中的 `ipaddr` 和 `ip6addr`）与系统内核 `/proc/net/if_inet6` 提取本机环回（127.0.0.1、::1）及接口 IP。

#### RPC 服务精简与进程探测
- R3. `root/usr/share/rpcd/ucode/luci.mihomo-traffic` 必须彻底移除对 `ubus` 模块的引用，不再保留任何 `connect()` 调用。
- R4. RPC 接口中的 `status` 进程探测改为直接遍历 `/proc/[0-9]+/cmdline` 匹配 `mihomo-traffic run` 并提取真实 PID，替代原先对 `service.list` 的 ubus 依赖。
- R5. RPC 接口中的 `clear` 历史清理逻辑仅保留 `/etc/init.d/mihomo-traffic restart` 执行，移除冗余的 `service.event` ubus 广播。

#### 采集热路径冷热隔离
- R6. `root/usr/bin/mihomo-traffic` 在 `run()` 守护进程启动初始化时固化一次本机 IP 列表与主机名，在每秒数据聚合与每 30 秒别名轮询的热路径中仅进行纯内存查找，避免频繁读取磁盘文件与 UCI。

### Scope Boundaries
- **In Scope:**
  - `root/usr/share/ucode/mihomo_traffic.uc` 中的本机 IP 识别与无依赖改造。
  - `root/usr/share/rpcd/ucode/luci.mihomo-traffic` 中的状态检查与数据清空改造。
  - `root/usr/bin/mihomo-traffic` 中守护进程对本机配置的加载时机优化。
  - `tests/smoke.sh` 冒烟测试套件对无 ubus 环境的兼容性保证。
- **Out of Scope:**
  - 前端 JavaScript（`htdocs/luci-static/resources/view/mihomo-traffic.js`）的展示逻辑与数据契约（保持字段完全向后兼容）。
  - TSV 存储层与日汇总合并算法。
  - 设备别名固化策略与 ARP 离线判定规则。

### Acceptance Examples
- AE1. 全局静态代码扫描：在 `root/` 目录下搜索关键字 `ubus`，除 `acl.d/*.json` 声明供 LuCI 访问本插件的 RPC 权限外，所有 ucode/可执行脚本中不再包含任何 `connect()` 或 ubus 客户端调用。
- AE2. 本机设备识别验证：在无 `ubusd` 运行的环境下（如测试容器中），`local_router_ips()` 仍能正常提取 `127.0.0.1`、`::1` 及静态 LAN IP，别名解析仍准确呈现 `路由器主机名(本机)`。
- AE3. 运行状态查询验证：通过 `ubus call luci.mihomo-traffic status`（由 rpcd 服务端接收）查询时，依然能准确返回 `running: true` 与真实的 `pid`，结果与 `pgrep -f "mihomo-traffic run"` 完全吻合。
- AE4. 文件描述符稳定性验证：后台常驻运行 `mihomo-traffic run`，监控 `/proc/<pid>/fd`，打开的文件描述符始终保持在初始水位（≤ 15），即便长时间运行或前端高频刷新，句柄数零增长。

---

## Planning Contract

**Product Contract preservation:** Product Contract unchanged.

### Key Technical Decisions
- KTD1. **纯文件与内存驱动的状态发现 (Zero-IPC)**: 完全放弃任何客户端 `connect()` 调用。Linux `/proc` 与 UCI 均是本地纯文件/内存接口，读取无任何 Socket 句柄泄漏风险，执行速度快一个数量级。Covers R1, R3.
- KTD2. **IPv6 内核转译算法**: 读取 `/proc/net/if_inet6` 时，内核以 32 位十六进制输出地址（如 `00000000000000000000000000000001`）。通过标准每 4 字符加冒号并压缩连续零块格式化为合法的 IPv6 字符串（如 `::1`），精准纳入本机特征。Covers R2.
- KTD3. **Proc 遍历进程匹配器**: 实现轻量函数 `find_process(pattern)`，读取 `/proc` 下的 PID 目录及其 `cmdline`，使用正则 `match(cmd, pattern)` 快速定位常驻后台进程，替代重量级的 `ubus call service list`。Covers R4.

### High-Level Technical Design

```
+---------------------------------------------------------------------------------+
|                               luci-app-mihomo-traffic                           |
+---------------------------------------------------------------------------------+
|                                                                                 |
|  [ 采集端: mihomo-traffic run ]         [ RPC端: luci.mihomo-traffic (rpcd) ]   |
|         |                                      |                                |
|  (启动初始化一次)                        (status / clear 查询)                   |
|         v                                      v                                |
|  [ 核心库: mihomo_traffic.uc ]          [ /proc 文件系统扫描器 ]                  |
|         |                                      |                                |
|         +-------------------+                  |                                |
|         |                   |                  |                                |
|         v                   v                  v                                |
|   /etc/config/network   /proc/net/if_inet6   /proc/[0-9]+/cmdline               |
|   (UCI 静态网络接口)     (内核全部 IPv6)      (纯本地无锁进程扫描)                  |
|                                                                                 |
+---------------------------------------------------------------------------------+
|                        * 全链路 100% 零 ubus.connect() *                         |
+---------------------------------------------------------------------------------+
```

---

## Implementation Units

### U1. 核心库无 ubus 退耦与本机 IP 双源直读
- **Goal:** 改造 `root/usr/share/ucode/mihomo_traffic.uc`，移除 `ubus` 模块导入与调用，通过 UCI 与 `/proc/net/if_inet6` 静态读取路由器自身所有 IP。
- **Requirements:** R1, R2, KTD1, KTD2
- **Dependencies:** None
- **Files:**
  - `root/usr/share/ucode/mihomo_traffic.uc`
- **Approach:**
  1. 移除文件开头的 `import { connect } from 'ubus';`。
  2. 重构 `local_router_ips()`：
     - 初始化 `ips = { '127.0.0.1': true, '::1': true }`；
     - 通过 `cursor().foreach('network', 'interface', ...)` 读取静态配置的 `ipaddr` 和 `ip6addr`（支持字符串与数组格式，自动剔除掩码）；
     - 读取 `/proc/net/if_inet6`，每行提取首字段 32 字符十六进制，分段转译为规范 IPv6 地址并加入 `ips`；
     - 维护模块级内存单例 `cached_router_ips`，首次获取后常驻内存，无需频繁扫盘。
- **Test scenarios:**
  - Happy path: `local_router_ips()` 返回对象包含回环地址 `127.0.0.1`、`::1` 以及 UCI 中配置的 LAN 口 IP。
  - Edge case: 系统不存在 `/proc/net/if_inet6` 时（某些定制极简内核），安全兜底不抛出异常。
  - Edge case: UCI 配置中 `ipaddr` 为带掩码格式（如 `192.168.1.1/24`）时，正确剥离掩码存入 `192.168.1.1`。
- **Verification:** 运行 smoke 测试验证 `local_router_ips()` 返回合法 IP 映射，且文件中无 `ubus` 导入。

### U2. rpcd 插件状态检测与清理逻辑零 ubus 改造
- **Goal:** 改造 `root/usr/share/rpcd/ucode/luci.mihomo-traffic`，移除 `ubus` 模块引用，改为直读 `/proc` 检测服务存活，精简清理命令。
- **Requirements:** R3, R4, R5, KTD1, KTD3
- **Dependencies:** U1
- **Files:**
  - `root/usr/share/rpcd/ucode/luci.mihomo-traffic`
- **Approach:**
  1. 移除文件顶部的 `import { connect } from 'ubus';`。
  2. 实现内部辅助函数 `get_daemon_status()`：
     - 遍历 `fs.lsdir('/proc') ?? []`，筛选出纯数字 PID 目录；
     - 读取 `/proc/<pid>/cmdline`，使用 `\0` 或空字符分割，若包含 `mihomo-traffic` 且包含 `run`，则判定服务存活，记录 `pid = +pid`；
     - 返回 `{ running: true, pid }`，未匹配到返回 `{ running: false, pid: null }`。
  3. 更新 `status` 方法调用 `get_daemon_status()` 填充 `running` 与 `pid` 字段，保留原有数据目录大小统计。
  4. 更新 `clear` 方法：保留 `/etc/init.d/mihomo-traffic restart` 执行，彻底删除已无意义的 `service.event` ubus 重启广播。
- **Test scenarios:**
  - Happy path: 当后台有包含 `mihomo-traffic run` 的进程时，`status` 返回 `running: true` 与对应 PID。
  - Failure path: 当后台无对应进程时，`status` 返回 `running: false` 与 `pid: null`。
  - Edge case: 遍历 `/proc` 时遇到临时消失的瞬时进程目录，捕获异常不中断扫描。
- **Verification:** 执行 `ucode -c` 语法检查，验证代码内无任何 `connect()` 存在。

### U3. 采集端启动期固化与热路径冷热隔离
- **Goal:** 改造 `root/usr/bin/mihomo-traffic`，使采集端常驻进程在启动时固化本机 IP，热路径杜绝系统调用。
- **Requirements:** R6, KTD1
- **Dependencies:** U1
- **Files:**
  - `root/usr/bin/mihomo-traffic`
- **Approach:**
  1. 在 `run()` 函数启动初始化时，预先调用一次 `host_names()` 与 `local_router_ips()`；
  2. 将提取到的本机 IP 集合在内存中标记固化；
  3. `get_alias()` 内部热路径保持纯内存哈希字典查询；
  4. 每 30 秒更新外部 DHCP 租约和 ARP 时，本机 IP 映射直接复用内存固化集，不再重复打 UCI 或扫描文件系统。
- **Test scenarios:**
  - Happy path: 持续采集过程中，本机产生的连接流量其别名恒定解析为 `主机名(本机)`。
- **Verification:** 常驻运行进程，确认其 CPU 与内存开销保持平稳，FD 打开数恒定。

### U4. 离线冒烟测试用例强化与代码基线净化
- **Goal:** 强化 `tests/smoke.sh` 测试用例，增加零 ubus 静态代码断言，验证在无 ubus 环境下的健壮性。
- **Requirements:** AE1, AE2, AE3, AE4
- **Dependencies:** U1, U2, U3
- **Files:**
  - `tests/smoke.sh`
- **Approach:**
  1. 在 `tests/smoke.sh` 中增加静态扫描检查：断言 `root/usr/share/ucode/mihomo_traffic.uc` 和 `root/usr/share/rpcd/ucode/luci.mihomo-traffic` 中均不包含 `from 'ubus'` 或 `connect()`；
  2. 验证 `local_router_ips()` 在没有 ubus 运行的测试环境下依然能提取回环地址及 UCI 模拟接口 IP；
  3. 验证别名解析优先级及本机标记逻辑正常运行。
- **Test scenarios:**
  - Happy path: 运行 `./tests/smoke.sh`，所有测试用例全部通过并输出 `all passed`。
- **Verification:** 检查 Git 提交差异，确认无多余变动。

---

## Verification Contract

### 自动验证
1. **静态代码审计:**
   ```sh
   grep -rn "connect" root/usr/share/ucode/ root/usr/share/rpcd/ root/usr/bin/
   ```
   **预期结果:** 没有任何匹配输出（0 匹配）。
2. **冒烟回归测试:**
   ```sh
   ./tests/smoke.sh
   ```
   **预期结果:** 编译通过，所有单元断言输出 `ok`，最后输出 `all passed`。

### 运行时手工验证（路由器真机）
1. 将修改后的代码部署至路由器：
   - 检查后台常驻进程 FD 数量：
     ```sh
     ls -l /proc/$(pgrep -f "mihomo-traffic run")/fd | wc -l
     ```
     **预期结果:** 句柄数量始终在 10~15 之间，绝不随时间递增。
2. 验证 LuCI 状态接口：
   - 执行 `ubus call luci.mihomo-traffic status`，确认返回正确的 `running: true` 与真实 PID。
   - 打开 LuCI Web 界面，确认仪表盘与表格正常渲染，本机流量标记为 `(本机)` 且常驻在线。

---

## Definition of Done
1. 代码中彻底清除所有 `ubus.connect()` 客户端调用，实现全链路零 ubus 客户端依赖。
2. 核心库 `mihomo_traffic.uc` 纯直读静态 UCI 与 Linux 内核 `/proc/net/` 接口。
3. `status` 状态探测通过 `/proc` 扫描本地实现，`clear` 彻底精简。
4. `tests/smoke.sh` 测试全部通过且包含零 ubus 静态断言。
5. 所有修改提交 Git 并保持提交记录语义清晰。
