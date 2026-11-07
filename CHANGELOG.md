# 更新日志

## v1.0.1 — 2026-11-07

> **发布者**：无名的小卒 · [cjm2004/fnmedia-ext-turbo](https://github.com/cjm2004/fnmedia-ext-turbo)
> **开发 / 版权**：原开发者 [yujian0815](https://github.com/yujian0815)（MIT）

### 🐞 修复：v1.0.0「未接管 / 接口不可用 / 加载不出源」

根因是**打包方式**而非优化代码：v1.0.0 发布的是 `public` 构建，`proxy/features.py` 中
`DDUO / ADULT / CATVOD / DRPY / GUARD` 被改写为 `False`，对应模块与第三方资产由
`tools/public_strip.py` 整块剥离。丢掉的正是部分在线资源站的解析通道。

本版改为以**官方 full 构建 v0.2.39** 重打包，`features` 开关与官方 `0.2.39-full` 逐项一致，
**未做任何剥离**。

| 项 | v1.0.0 | v1.0.1 |
|---|---|---|
| 基座 | 公开构建 | 官方 full 构建 v0.2.39 |
| DDUO / CATVOD / DRPY / GUARD | False（已剥离） | **True** |
| 包体 | 897 KB | 116 MiB |
| 是否需打补丁 | 需要 | **不需要，已烘焙进源码** |

### ✨ 播放链路优化（全部内置，无需补丁脚本）

- 分片流式直发（`hls_seg` 边下边传，TTFB ≈ 上游首字节）
- 线路竞速「先成功者胜」（`asyncio.FIRST_COMPLETED`）
- 按实测吞吐选线（指数滑动平均，零额外流量）
- 熟线单发直连（≥200 KB/s 不再并发竞速）
- 进程内 DNS 缓存（600s TTL / 2048 条）
- 接管等待窗 16s → 180s（`FNMEDIA_STAGE_WAIT`）
- 榜单详情等待 0.12s → 2.5s、播放等待加 12s 上限
- systemd 单元补 `ExecStartPre`（修开机 203/EXEC）

### 🔍 搜索与索引修复（本版新增内置）

- 搜索注入预算 6s → 8.5s，`search_all` 到点返回已完成源
- 4 字母以上英文片名也触发在线搜索
- 搜索合并改增量重建索引
- 海报磁盘缓存 + 后台预取
- 虚拟库 guid 兼容、海量四库首屏预热、封面改用 `sys/img` 前缀

### ✅ 实测（fnOS 1.2.1000 / trim.media 0.9.8-1）

- 特性开关全 True；`/v/api/ext/status` 200，16 源 / 启用 10
- 在线搜索「流浪地球」「庆余年」「Bleach」均 12 条全在线
- 榜单四库均有条目；海报代理 200，33,745 B 真实 JPEG
- 6 MB 分片 TTFB **0.65s**、总 5.24s

### 📦 打包说明

不含 `state/`、不含 `config.json`、不含 token / cookie；资源站为上游内置默认资产。
代码版本 `repo/VERSION` 保持 `0.2.39`（基座版本），应用中心包版本为 `1.0.1`。

## v1.0.0 — 2026-11-04

基于原版 **fnmedia-ext v0.2.39**（完整构建）的播放链路优化版，原版全部能力保留。

### 优化

- **分片流式直发**（hls_seg / 新增 _try_stream_segment）：上游首字节即回传，实测首字节 1.5~55s → 0.09~0.70s；上游 4xx/5xx、PNG 外壳分片、content-encoding 三种情况自动回退整段缓冲（与旧行为一致）。
- **竞速先成功者胜**（_race_fastest）：asyncio.gather 改为 FIRST_COMPLETED + 取消其余任务，起播 11.67s → 2.93s。
- **熟线单发直连**：已知吞吐 ≥200 KB/s 的线路跳过并发竞速，3.68s → 2.34s。
- **按实测吞吐选线**（_LINE_SPEED / _line_score / _remember_speed）：边播边记录每条线路真实吞吐（EMA，零额外流量），未知线路给 150 KB/s 先验；自动避开 55~70 KB/s 的坑线。
- **进程内 DNS 缓存**：socket.getaddrinfo 600s TTL / 2048 条，冷解析 5.0~5.2s → 0.04~0.12s。
- **守护等待窗口**（supervise.py）：暂存 socket 等待 16s（硬编码）→ FNMEDIA_STAGE_WAIT（默认 180s）。修复片库较大时（3000+ 部，启动需 22.8s）必然超时退出、应用中心显示「启动又关闭」的问题。
- **systemd 单元加固**（deploy/fnmedia-ext.service.in）：新增 ExecStartPre 等待 venv 就位，修复开机时数据卷未挂载导致的 203/EXEC。

- **首点即播（榜单/热门 4 处）**：两阶段解析（快源 6s → 全量 25s）、详情等待 0.12s→2.5s、
  播放等待加 12s 上限、列表下发时后台预解析前 6 个未绑定卡片。首次点开不再「大小 0 / 路径加载中」，
  点播从 5.8~9.3s 降到 **0.02~0.03s**。

### 打包

- 版本号 1.0.0（> 0.2.39，应用中心按升级处理，保留 state/）。
- 排除 .venv（安装时创建）、state/（用户数据）、spider-runner/android/.buildtools（206MB 编译工具链，运行时不需要）。
