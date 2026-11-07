## 修复：v1.0.0 的「未接管 / 接口不可用 / 加载不出源」

先说结论：**这不是优化代码的问题，是打包方式的问题。**

`v1.0.0` 上传的 `fnmedia-ext-1.0.0-public.fpk` 是**公开构建**——`proxy/features.py` 里

```python
DDUO = False   ADULT = False   CATVOD = False   DRPY = False   GUARD = False
```

而官方本机安装的是 **full 构建**，这五项全为 `True`。公开构建在打包时由
`tools/public_strip.py` 把对应模块与第三方资产整块删除，**丢掉的正是部分在线资源站的解析通道**，
所以现象就是「未接管、接口不可用、加载不出源」。

`v1.0.1` 改为以**官方 full 构建 v0.2.39** 为基底重新打包，**保留全部特性开关，未做任何剥离**。

## 本版改了什么

### 一、基座换成 full 构建（本次核心修复）

| 项 | v1.0.0 | v1.0.1 |
|---|---|---|
| 基座 | 公开构建 | **官方 full 构建 v0.2.39** |
| `DDUO` / `CATVOD` / `DRPY` / `GUARD` | `False`（已剥离） | **`True`** |
| 包体 | 897 KB | 116 MiB |
| 资源站解析通道 | 部分缺失 | **完整** |

### 二、优化全部烘焙进源码

v1.0.0 需要手工跑补丁脚本；本版**装完即用，不再需要任何补丁脚本**。

播放链路（源自 v1.0.0）：

- **分片流式直发** — `hls_seg` 边下边传，首字节 ≈ 上游首字节；旧实现 `body = r.content`
  要整段下完才回传，播放器把「整片下载时间」当成起播等待
- **线路竞速「先成功者胜」** — `asyncio.FIRST_COMPLETED`，拿到第一个可用线路即取消其余，
  不再用 `gather` 等最慢的那条
- **按实测吞吐选线** — 播放时顺带记录每条线真实 KB/s（指数滑动平均，零额外流量），
  避开「延迟低但吞吐烂」的线路
- **熟线单发直连** — 实测 ≥200 KB/s 的线路不再参与并发竞速
- **进程内 DNS 缓存** — 600s TTL / 2048 条，只影响本进程，不碰 `/etc/resolv.conf`
- **接管等待窗 16s → 180s**（`FNMEDIA_STAGE_WAIT`）— 修片库大时「启动又关闭」
- **榜单等待窗** — 详情 0.12s → 2.5s（`FNMEDIA_CHART_DETAIL_WAIT`），
  播放侧加 12s 上限（`FNMEDIA_CHART_PLAY_WAIT`）
- **systemd 单元补 `ExecStartPre`** — 等 venv 就位，修开机自启 203/EXEC

搜索与索引（本版新增内置）：

- 搜索注入预算 6s → 8.5s，在线结果不再被整体丢弃
- `search_all` 支持「到点返回已完成源」
- 4 字母以上英文片名（jojo、Bleach）也触发在线搜索
- 搜索合并改增量重建索引
- 海报磁盘缓存 + 后台预取
- 虚拟库 guid 兼容、海量四库首屏预热、封面改用 `sys/img` 前缀

## 实测验证

本机（fnOS 1.2.1000 / trim.media 0.9.8-1）全新安装后逐项验收：

| 项目 | 结果 |
|---|---|
| 应用中心 | 飞牛影视在线扩展 Turbo / **v1.0.1** / running |
| 特性开关 | DDUO / CATVOD / DRPY / GUARD **均 True** |
| `/v/api/ext/status` | 200，16 个内置源 / 启用 10 |
| 在线搜索 | 「流浪地球」「庆余年」「Bleach」均 12 条全在线 |
| 榜单四库 | 热门电影 / 动漫新番 / 海量电影 / 海量剧集 均有条目 |
| 海报代理 | 200，33,745 B 真实 JPEG |
| 分片流式 | 6 MB 分片 TTFB **0.65s**、总 5.24s（TTFB 仅占 12%；旧整段缓冲会等于总耗时） |
| 冷启动 | 安装后首次请求偏慢（后台预热榜单/海报缓存），预热后 TTFB 稳定 ~1s |

## 安装

应用中心 → 手动安装 → 选 `fnmedia-ext-1.0.1-full.fpk` → 勾选免责声明 → 安装。

前置条件与注意：

1. **必须先安装官方「飞牛影视」并让它运行**（`cmd/install_init` 会检查 `/var/run/trim_media.sock`）。
2. **已装旧版请先卸载**。CLI 覆盖安装会被应用中心拒（`10236`）；
   应用中心界面里同 `appname` 高版本一般按升级处理，但仍建议先卸载再装以避免残留。
3. 安装时需要联网（`cmd/install_callback` 会 `pip install fastapi uvicorn httpx` 建 venv）。
4. 本包**不含** `state/`、`config.json`，也不含任何 token / cookie；资源站为上游内置默认资产。

```
SHA-256: 见下方资产 fnmedia-ext-1.0.1-full.fpk.sha256
```

## 署名

- **开发者 / 版权**：[yujian0815](https://github.com/yujian0815) · [fnmedia-ext](https://github.com/yujian0815/fnmedia-ext)（MIT）
- **发布者**：[无名的小卒](https://github.com/cjm2004) · [cjm2004/fnmedia-ext-turbo](https://github.com/cjm2004/fnmedia-ext-turbo)

本包保留原作者全部版权声明。原版问题请提给[原仓库](https://github.com/yujian0815/fnmedia-ext)；
本仓库只受理这些优化与打包相关的问题。

## 免责

本仓库**不提供、不内置、不分发任何影视资源、资源站地址或播放链接**，与上游一致。
所有在线内容来自使用者自行配置的第三方接口，请自行确认合法性。
