# 更新日志

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

### 打包

- 版本号 1.0.0（> 0.2.39，应用中心按升级处理，保留 state/）。
- 排除 .venv（安装时创建）、state/（用户数据）、spider-runner/android/.buildtools（206MB 编译工具链，运行时不需要）。
