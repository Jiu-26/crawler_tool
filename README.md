# crawler_tool — SignalX 统一采集与监测 Tool

SignalX 是面向行业新闻/社媒动态的自动监测与调研系统，分两层：**采集与监测 Tool 层（本仓库）** 与 **LLM 分析 agent（signalx-agent，独立仓库，不包含在本仓库）**。本仓库完全可独立运行；下游 agent 通过 HTTP API 或 Python Typed Tool SDK 调用本服务，用采集到的数据做事件抽取、合并与时间线生成。

## 核心能力

- **统一采集抽象**：`SourceAdapter → ContentItem(content.v1) → SourceReport`，8 个异构来源（南方周末/头条/央视网/搜狗微信/微博/微信/小红书/抖音）收敛为单一 Tool 接口，新来源仅需实现一个适配器；分页 cursor 绑定请求指纹与 `adapter_version`，结果可复现。
- **四级时间锚定**：① 列表层时间信号解析（头条 0.4.0 支持 9 类真实形态）→ ② 详情页补时间 `enrichTime`（JSON-LD → meta → 可见文本 → 内嵌脚本，逐级降级）→ ③ Wayback CDX 兜底（`--wayback-fallback`，默认关，连续失败 2 次熔断）→ ④ 证据级 grounding（在下游 agent 内实现）。证据不足不落值，兜底结果显式标注待人工复核。
- **规则×LLM 监测漏斗**（`crawler_tool/monitoring/`）：按配置查询轮询 → 规则漏斗（主体通道 + 可选 LLM 分诊 + 行业通道 + 高危旁路）→ 事件组聚合打分 → 告警落盘 + 每日 Markdown 日报；支持人工判读（accepted/rejected）。
- **人机协同采集闭环**：手动捕获投喂（零网络，信封自动探测）+ 服务端关键词捕获队列（`data/capture_queue.json` 持久化，agent 可推送）+ 会话视图/导出页；捕获条目可自动进入监测漏斗。
- **合规与可复现设计**：见下节红线。

## 设计红线（合规纪律）

- 无重试、无代理（`trust_env=False`）、验证码/频控即停（`retryable=false`）；
- `empty` 只代表合法空结果——登录页/429/结构变化绝不伪装成 empty（实测日志中存在 `source_unavailable`/`parser_changed` 如实留痕）；
- 搜索动作由人完成；工具只被动解析"平台已发给人看的数据"，不逆向签名（小红书 461 动态签名已实测确认，维持禁用）；
- Cookie/token 只从运行环境注入，不写入任何文件、不进 Git；
- 新能力一律显式 opt-in，默认参数下行为逐字节不变（离线回放为回归基线）；
- 真实验证必须双门禁 `--online --confirm-real`，输出脱敏。

## 当前状态（2026-09-16）

- `ContentItem(content.v1)` 严格 Pydantic 模型：URL 规范化（去追踪参数）、去重键、稳定 `contentId`、质量告警；
- 标准来源状态/错误码：`success/empty/rate_limited/authentication_required/parser_changed/source_unavailable/network_error`；
- 统一 `CrawlService`：内存缓存（600s TTL，暂态失败不缓存）、cache-only、去重、多来源**轮转均衡截断**（`_balanced_truncate`，防止单来源挤占名额）、会话库全文升级（实时结果按 `content_id` 用库存全文版自动升级，全文条目不被摘要捕获降级）；
- 详情页补全（显式开关）：`enrichTime` 补时间 + `enrichContent` 补正文，共享同一次 GET 与 ≤5 条/轮预算，验证码即停、失败只加条目级 warning；当前南方周末（0.3.0-ssr）与央视网（0.2.0-anonymous）支持；
- 手动捕获投喂 + 捕获队列 + 全来源共享会话视图（内存态，重启清空）+ 浏览器查看页 `/view`、捕获队列页 `/capture`；
- Agent Tool 门面：`search_content`、`search_wechat_articles`、`ingest_capture`、`get_source_health`；
- 监测配置默认 AI 主题（`config/monitoring/`：entities 13 个主体，self/competitor/industry_actor/regulator）；
- 离线测试基线：`python -m pytest -q` → **280 passed, 1 skipped**（2026-09-16）。

暂不包含：异步 Job、Redis/MySQL 等持久化数据库、MCP、常驻浏览器 Worker、Event/Agent 分析层。

## 来源矩阵

| 来源 | 状态 | 默认可见性 | 监测轮询 | 启用方式 |
|---|---|---|---|---|
| 南方周末 `south_weekend` | 真实首屏可用（0.3.0-ssr）；支持详情页补时间/正文；分页 unverified 不签发 cursor | 默认联搜成员 | ✅ | 无需配置 |
| 头条 `toutiao` | 首屏可用（0.4.0）；时间解析 9 类真实形态 + 卡片兜底；文章详情信封解析支持正文回填；分页真实两页未证 | 默认联搜成员 | ✅ | 无需配置 |
| 央视网 `cctv_news` | 官方栏目列表直读（0.2.0）；秒级时间+关键词+稳定 ID，字段质量最高；支持详情页补时间/正文 | 显式指定（灰度） | ✅ | 无需配置 |
| **搜狗微信** `sogou_wechat` | 文章索引发现；包装链接不解析、无正文、验证码即停。**实测稳定** | 默认启用 | ✅ 轮换固定成员 | 默认启用；`SOGOU_WECHAT_MODE=disabled` 显式关闭 |
| 微博 `weibo` | **匿名已停用**（2026-09 实测基本不可用，已从监测配置移除；代码保留，显式指定仍可机会性使用） | 必须显式指定 | ❌ 已停用 | 可选 `WEIBO_MODE`+`WEIBO_SESSION_COOKIE` |
| 微信 `wechat` | 授权通道受平台 200013 限制；数据走手动捕获 | 必须显式指定 | 🔗 经捕获通道 | 授权模式需 `WECHAT_MODE`+`WECHAT_SESSION_COOKIE`；默认停用 |
| 小红书 `xiaohongshu` | 接口需动态签名（461），网络禁用；走捕获投喂 | 默认 disabled | 🔗 经捕获通道 | `XHS_MODE`+`XHS_SESSION_COOKIE` |
| 抖音 `douyin` | Parser-only，不做网络自动化 | registry 可见 | 🔗 经捕获通道 | 投喂即可用 |

## 快速上手

以下命令均在**仓库根目录**（`pyproject.toml` 所在层）执行。

```powershell
# 安装（Python 3.11+）
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"

# 离线测试（不联网）
python -m pytest -q          # 基线：280 passed, 1 skipped

# ① 启动采集服务（127.0.0.1:8301，仅本机回环）
python -m crawler_tool.interfaces.app --serve
# 可选：--wayback-fallback 启用 Wayback 时间兜底（默认关；国内通常不可达，失败 2 次熔断）

# ② 运行一轮采集监测
python -m crawler_tool.monitoring.scheduled_tick --config-dir config/monitoring --data-dir data/monitoring

# ③ 告警摘要与人工判读
python -m crawler_tool.monitoring.run_tick --data-dir data/monitoring --report
python -m crawler_tool.monitoring.run_tick --data-dir data/monitoring --set-status "告警ID" accepted --note "备注"
```

**监测配置**（业务可改，每轮自动生效）：

- 查询清单/平台/触发阈值：`config/monitoring/queries.json`、`base_rules.json`
- 主体档案（我方/竞品/别名/产品名）：`config/monitoring/entities.json`

Windows 定时脚本 `scripts/run_monitor_tick.cmd` 只做采集监测（不执行任何 agent）：优先使用同级 `signalx-agent/.venv`（若存在），否则使用 PATH 中的 `python`；日志在 `data\monitoring\tick_log.txt`。定期执行需自行配置调度器，不会自动注册任务。

**数据都在哪**：

| 想看什么 | 位置 |
|---|---|
| 告警 + 观察箱摘要（最常用） | `data\reports\daily_report_日期.md` 或 `run_tick --report` |
| 告警完整记录 / 观察箱 | `data\monitoring\alerts.jsonl` / `observation.jsonl` |
| 爬到的原始内容 | 浏览器 `http://127.0.0.1:8301/view`；请求级日志 `data\crawl\` |
| 捕获关键词队列 | `data\capture_queue.json`（或 `/capture` 页） |
| 下游 agent 的发现结果/时间线 | 存于 agent 仓库的 `runs/`，不在本仓库 |

## API

```powershell
$env:CRAWLER_CURSOR_SECRET="replace-with-a-random-secret"   # 分页 cursor 需要
python -m crawler_tool.interfaces.app --serve   # 仅绑定 127.0.0.1:8301
```

端点全表：

```text
页面：
GET  /view                                # 会话视图浏览器页（平台筛选/搜索/导出）
GET  /capture                             # 捕获任务队列页（队列存服务端，agent 可推送）

系统：
GET  /api/v1/health
GET  /api/v1/tool/source-health
GET  /api/v1/tool/capture-status

搜索：
POST /api/v1/tool/search-content          # 统一搜索；显式 wechat/xiaohongshu/douyin
                                          # 时改读本会话历史；body 加 "enrichTime":true
                                          # 或 "enrichContent":true 启用详情页补全
                                          # （一次 GET 同时取时间与正文，共享 ≤5 条/轮预算；南周/央视支持）
POST /api/v1/tool/search-wechat-articles  # 微信关键词实时检索（当前 200013 受限）

捕获队列（服务端持久化 data/capture_queue.json）：
GET    /api/v1/tool/capture-queue         # 队列列表
POST   /api/v1/tool/capture-queue         # 追加 {"keywords":[...],"source":"agent|manual"}
DELETE /api/v1/tool/capture-queue?keyword=词

捕获投喂与视图：
POST /api/v1/tool/ingest-capture          # 手动捕获投喂（零网络，信封自动探测）
POST /api/v1/tool/ingest-xhs-capture      # 兼容别名
GET  /api/v1/tool/captured-items          # 会话视图（?platform=&keyword=&limit=）
GET  /api/v1/tool/captured-items/export   # 导出 csv|json
```

查询示例（默认联搜只有南方周末+头条；其余来源必须显式指定）：

```powershell
Invoke-RestMethod -Method POST `
  -Uri http://127.0.0.1:8301/api/v1/tool/search-content `
  -ContentType "application/json" `
  -Body '{"query":"AI客服","platforms":["south_weekend"],"limit":5,"freshness":"prefer_fresh"}'
```

开发用单次 smoke：`python -m crawler_tool.interfaces.app --smoke "AI客服" --limit 5`（会访问真实来源）。

## 各来源启用与配置要点

### 微博：匿名已停用，授权模式保留

**2026-09 决定：匿名模式停用**——实测基本不可用（`parser_changed` 频发），已从监测配置移除；代码保留，显式 `platforms:["weibo"]` 仍可机会性使用，但不要依赖。授权模式（过渡方案）：

```powershell
$env:WEIBO_MODE="legacy_authorized"
$env:WEIBO_SESSION_COOKIE="你的cookie"     # 只从运行环境注入，绝不写入文件
python -m crawler_tool.interfaces.app --serve
```

只抓首屏；不分页/详情/自动登录/代理。错误分类：未配置→`source_unavailable`；过期/验证码→`authentication_required`；429→`rate_limited`；结构变化→`parser_changed`。

### 微信：授权通道停用，搜狗发现源为主渠道

- `WECHAT_MODE` 不配置即为停用（代码保留不删）；实时通道自 2026-07-30 起被平台 200013 能力控制；
- **搜狗发现源：服务端默认启用**（`SOGOU_WECHAT_MODE=disabled` 可显式关闭）：只有文章发现（标题/公众号/时间/摘要/包装链），包装链不解析、无正文、验证码即停不重试；
- 微信内容的另一通道是**手动捕获投喂**，`platforms:["wechat"]` 的统一搜索会返回会话历史中的捕获条目。

### 小红书：签名墙，走捕获投喂

`XHS_MODE=disabled`（默认，零网络）/ `authorized_first_page`（+`XHS_SESSION_COOKIE`，显式查询时一次 POST）。**真实验证已确认接口必须浏览器动态签名**（无签名请求被网关 461 拒绝）——按红线不逆向签名，**不要期待该模式返回真实数据**。推荐用法：手动捕获投喂，`platforms:["xiaohongshu"]` 统一搜索会返回捕获历史。

### 抖音：Parser-only + 手动投喂

适配器零网络。识别三类信封：`{"aweme_list":[...]}`、`{"data":[{"aweme_info":{}},...]}`、`{"data":{"aweme_details":[...]}}`；其余一律 `parser_changed`。字段：`desc`→标题、`create_time`→时间、URL 固定 `https://www.douyin.com/video/{aweme_id}`。

### 搜狗微信：公开索引发现

服务端默认启用（2026-09 起，实测稳定）。关闭方式：启动服务的终端设 `SOGOU_WECHAT_MODE=disabled`（已有服务需重启），关闭后该来源返回 `source_unavailable`。

```powershell
$env:SOGOU_WECHAT_MODE="anonymous_best_effort"   # macOS/Linux 用 export
```

单请求首屏（`weixin.sogou.com/weixin?type=2`），每页约 10 条：标题/公众号名/发布时间（unix 秒→UTC，置信 1.0）/摘要/搜狗包装链。纪律：包装链不解析（一次解析=一次请求）；验证码页归类 `rate_limited` 且 `retryable=false` 即停。

### 央视网：官方媒体新闻列表

零配置，显式 `platforms:["cctv_news"]`。栏目列表静态 JSONP 直读（秒级时间/摘要/关键词/稳定 ARTI_ID/配图，全场质量最高）。**栏目列表不认识搜索词**——拉最新列表后本地关键词过滤，无命中如实 `empty`（`diagnostics.keywordFiltered` 有明细），不是故障；站内搜索被 robots 禁止，不得实现。

## 手动捕获投喂（通用工作流）

适用小红书/抖音/微信。原则：**请求由人触发，工具只解析平台已发给人看的数据**——零网络、无模式开关、默认可用。

```powershell
$raw = Get-Content .\xhs_capture.json -Raw
Invoke-RestMethod -Method POST -Uri http://127.0.0.1:8301/api/v1/tool/ingest-capture `
  -ContentType "application/json" -Body $raw
```

请求体三等价：裸信封（自动探测平台）/ 显式 `{"platform":"douyin","keyword":"...","capture":{...}}` / 仅 `{"keyword":"...","capture":{...}}`。条目以 `ext.captureIngest.origin=manual_capture` 留痕；会话视图最多 500 条、重启清空。浏览器助手 `scripts/xhs_douyin_capture_helper.user.js`（v0.3.0，小红书/抖音/公众号）自动转发；**只转发平台发给你本人的数据，搜索动作必须由人完成**。配合 `includeCaptured`，捕获条目自动进入监测漏斗。

## 浏览器自动化实验（2026-09，受限、未转正）

`scripts/browser_auto/` 是一次**红线例外实验**（详见 [scripts/browser_auto/README.md](scripts/browser_auto/README.md)）：有头 Playwright + 专用浏览器 profile，只转发"平台返回给登录用户的响应信封"，**无 headless、无反检测、无重试、不逆向签名**；用途为小红书搜索捕获与头条文章正文回填。

硬性围栏：单次 ≤5 个查询、间隔 ≥60s、各来源每日预算、触发 461/验证码/登录墙即停并当日自锁。**该能力为实验性质，未转正；默认不参与任何流程。** 浏览器 profile 目录含登录态，仅存本机 `data/`（已被 .gitignore 排除），严禁外传。

## 受控真实验证

双门禁：`--online --confirm-real` 缺一不可。每来源最多 2 次 GET（微博 1 次），无 Cookie/token/代理/浏览器/重试/自动重定向；输出脱敏（只有状态、条数、ID 哈希）。失败即停。

```powershell
python -m crawler_tool.interfaces.app --live-smoke-toutiao "AI客服" --online --confirm-real
python -m crawler_tool.interfaces.app --live-smoke-weibo "AI客服" --online --confirm-real
python -m crawler_tool.interfaces.app --live-smoke-xhs "人工智能" --online --confirm-real
python -m crawler_tool.interfaces.app --verify-south-weekend-pagination "AI客服" --online --confirm-real
```

常规测试默认跳过真实验证模块；需要 `CRAWLER_LIVE_SMOKE=1` + `CRAWLER_CONFIRM_REAL=1` 后 `pytest -m real_source_validation`。

## 架构与边界

- 定时任务、Tool 门面、监测层共用 `CrawlService`；下游 agent 优先 Python Typed Tool SDK，HTTP API 是跨进程边界；
- 下游 agent 不接触 Cookie、token、代理、浏览器和平台 Parser；分页只给 opaque cursor；
- 输出只能是 `ContentItem(content.v1)`；监测层的告警是消费侧产物，不回灌采集层；
- 失败不重试（`retryable=false` 服务端判定）；empty 只表示合法空结果；
- 当前不接入 MCP、异步 Job、持久化数据库、Event/Agent 分析层。

## 文档索引

| 文档 | 内容 |
|---|---|
| [浏览器自动化实验说明](scripts/browser_auto/README.md) | 实验围栏、使用方式与退出条件 |

其余设计与对接文档（监测层设计手册、采集对接指南等）属内部资料，未随本仓库分发。
