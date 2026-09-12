# Crawler Tool

多平台内容**统一采集**与**定时监测告警**服务。为上层 Agent / 分析系统提供标准化的内容流：
8 个平台适配器 → 统一数据模型 → 规则监测漏斗 → 告警落盘与日报。

```text
平台适配器（南方周末 / 头条 / 央视网 / 微博 / 搜狗微信 / 微信 / 小红书 / 抖音）
        │  统一输出 ContentItem(content.v1)
        ▼
   CrawlService —— 内存缓存 · 去重 · 标准来源状态
   ├── HTTP API        Agent Tool 门面（127.0.0.1:8301）
   ├── 浏览器视图      /view 会话视图 · /capture 捕获队列
   └── 定时监测漏斗    查询轮询 → 规则打分 → 告警 / 观察箱 → 每日日报
                              └─ score 达标 → DiscoveryRunner（外部 Agent 接入点）
```

## 特性

- **统一数据模型**：`ContentItem(content.v1)` 严格 Pydantic 模型；URL 规范化（去追踪参数）、去重键、稳定 `contentId`、质量告警；
- **标准来源状态**：`success / empty / rate_limited / authentication_required / parser_changed / source_unavailable / network_error` —— 失败不重试、空结果不伪装，每种异常都有明确归类；
- **统一采集服务**：内存缓存（600s TTL，暂态失败不缓存）、cache-only 模式、跨平台去重；
- **安全分页**：opaque cursor 协议（HMAC + TTL + 请求指纹绑定），下游拿不到原始查询参数；
- **定时监测系统**：多平台查询轮询 → 规则漏斗（主体通道 + 行业通道 + 高危旁路）→ 事件组聚合与共振打分 → 三级分级 → 告警落盘 + 每日 Markdown 日报；**score 达标的告警可自动转交外部 Agent 做事件扩展**（幂等 + 每日上限）；
- **手动捕获投喂**：小红书 / 抖音 / 微信等难采集平台走"人工浏览 + 一键投喂"通道，零网络自动化。

## 支持的来源

| 来源 | 接入方式 | 默认联搜 | 监测轮询 |
|---|---|:---:|:---:|
| 南方周末 `south_weekend` | 网页搜索（SSR 首屏） | ✅ | ✅ |
| 头条 `toutiao` | 网页搜索 | ✅ | ✅ |
| 央视网 `cctv_news` | 官方栏目列表直读（字段质量最高） | 需显式指定 | ✅ |
| 微博 `weibo` | 匿名尽力而为；可选会话授权 | 需显式指定 | ✅ |
| 搜狗微信 `sogou_wechat` | 公开索引发现（微信文章主渠道） | 需显式开启 | ✅ |
| 微信 `wechat` | 手动捕获投喂为主 | 需显式指定 | 经捕获通道 |
| 小红书 `xiaohongshu` | 手动捕获投喂（接口需浏览器签名） | 默认关闭 | 经捕获通道 |
| 抖音 `douyin` | Parser-only + 手动投喂 | — | 经捕获通道 |

各来源的启用开关均通过环境变量注入（如 `WEIBO_MODE` / `SOGOU_WECHAT_MODE` / `XHS_MODE` + 对应 `*_SESSION_COOKIE`）。**Cookie 只从运行环境读取，绝不写入文件。**

## 快速开始

```bash
pip install -e .            # 或 pip install -e ".[dev]"（含测试依赖）
```

启动服务（仅绑定本机 8301 端口）：

```powershell
$env:CRAWLER_CURSOR_SECRET="replace-with-a-random-secret"   # 分页 cursor 签名密钥
python -m crawler_tool.interfaces.app --serve
```

单次搜索冒烟（不起服务）：

```powershell
python -m crawler_tool.interfaces.app --smoke "AI客服" --limit 5
```

浏览器打开 `http://127.0.0.1:8301/view` 可查看会话内容（按平台筛选 / 关键词过滤 / 导出 CSV、JSON）。

## HTTP API

| 方法 | 端点 | 说明 |
|---|---|---|
| GET | `/view` · `/capture` | 会话视图页 · 捕获任务队列页 |
| GET | `/api/v1/health` · `/api/v1/tool/source-health` | 健康检查 · 各来源状态 |
| POST | `/api/v1/tool/search-content` | 统一搜索（多平台联搜或显式指定） |
| POST | `/api/v1/tool/ingest-capture` | 手动捕获投喂（信封自动探测平台） |
| GET | `/api/v1/tool/captured-items` · `/export` | 会话视图查询 · 导出 csv/json |

搜索示例（默认联搜仅含南方周末 + 头条，其余来源需显式指定）：

```powershell
Invoke-RestMethod -Method POST `
  -Uri http://127.0.0.1:8301/api/v1/tool/search-content `
  -ContentType "application/json" `
  -Body '{"query":"AI客服","platforms":["south_weekend"],"limit":5,"freshness":"prefer_fresh"}'
```

## 定时监测

监测的业务配置全部是 JSON、每轮热生效：

| 文件 | 内容 |
|---|---|
| `config/monitoring/entities.json` | 监测主体档案（我方 / 竞品 / 监管机构，别名与产品名） |
| `config/monitoring/queries.json` | 查询词清单、平台列表、每轮查询数 |
| `config/monitoring/base_rules.json` | 规则漏斗（动作词 / 宾语词 / 排除词 / 权重）与告警触发线 |

执行一轮监测并生成日报：

```powershell
python -m crawler_tool.monitoring.run_tick --data-dir data/monitoring
python -m crawler_tool.monitoring.run_tick --data-dir data/monitoring --report   # 仅查看日报
python -m crawler_tool.monitoring.run_tick --set-status <告警ID> accepted --note "备注"  # 人工判读
```

产物位置：告警 `data/monitoring/alerts.jsonl`，观察箱 `observation.jsonl`，日报 `data/reports/daily_report_*.md`。
score ≥ `agentTriggerScore` 的待判读告警会自动转为种子事件，经 `DiscoveryRunner` 协议交由外部 Agent
执行事件扩展，结果写入 `data/monitoring/discoveries/`（同一告警只触发一次，每日有上限）。

## 手动捕获投喂

适用于接口有签名墙的平台（小红书 / 抖音 / 微信）。原则：**请求由人触发，工具只解析平台发给本人看的数据**——零网络自动化、默认可用：

```powershell
$raw = Get-Content .\capture.json -Raw
Invoke-RestMethod -Method POST -Uri http://127.0.0.1:8301/api/v1/tool/ingest-capture `
  -ContentType "application/json" -Body $raw
```

配套浏览器助手脚本 `scripts/xhs_douyin_capture_helper.user.js` 可在浏览小红书 / 抖音 / 公众号时一键转发内容。

## 测试

```bash
pytest -q        # 全部离线，不访问公网
```

真实验证模块默认跳过；启用需同时设置 `CRAWLER_LIVE_SMOKE=1` 与 `CRAWLER_CONFIRM_REAL=1`，
并以 `--online --confirm-real` 双门禁运行（限次、脱敏输出、失败即停）。

## 设计边界

- 采集层只产出 `ContentItem(content.v1)`；监测告警与 Agent 发现结果是消费侧产物，不回灌采集层；
- 下游 Agent 不接触 Cookie、token、代理与平台解析器，分页只交付 opaque cursor；
- `empty` 只表示合法空结果；登录页 / 验证码 / 限流 / 结构变化绝不伪装成空结果；
- 当前不包含：详情抓取、异步 Job、数据库持久化、浏览器自动化 Worker、MCP 接入。
