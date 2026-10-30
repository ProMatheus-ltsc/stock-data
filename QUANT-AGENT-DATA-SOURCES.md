# quant-trading-agent 数据获取与存储说明

> 本文件说明 `quant-trading-agent` 的数据获取方式与存储布局，覆盖 **A 股**与**美股**两条相互独立的轨道。
> 依据 `scripts/quant_agent/data/*`、`scripts/quant_agent/us/*`、`config/quant_agent.yaml`、`config/us_equity.yaml`、`config/source_registry.yaml` 实现整理。

---

## 0. 总体架构

A 股与美股是**两套完全分离的取数与存储体系**：

| 维度 | A 股 | 美股 |
|------|------|------|
| 配置 | `config/quant_agent.yaml` | `config/us_equity.yaml` |
| 代码 | `scripts/quant_agent/data/`（manager/sources/cache/quality/replay） | `scripts/quant_agent/us/`（fetch/config/panel/strategy/engine） |
| 行情源 | 腾讯 / 新浪 / 同花顺 / AKShare（降级链） | 腾讯行情（报价 / 前复权K线 / 分钟） |
| 数据落点 | **仓库外缓存** `~/.quant_agent/cache`（不入库） | **随仓库分发** `data/us/*.parquet`（入库） |
| 基准 | 沪深 300（`000300.SH`） | 标普 500 ETF（`SPY`） |

---

## 1. A 股数据获取

### 1.1 行情源路由（OHLCV 日线）

`DataManager` 按 `preferred = [ths, tencent, sina, akshare]` 顺序路由，单源超时 15s、重试 2 次，全源失败抛 `DataUnavailableError`（退出码 2）。

| 优先级 | 源 | 接口 / 方式 | 复权 | 说明 |
|:--:|------|-------------|------|------|
| 1 | 同花顺 iFinD `ths` | REST（`X-api-key`）/ Python SDK / MCP / CLI | qfq | 需 `HITHINK_FINANCE_API_KEY`；未配置则 `not_configured` 且**零网络请求** |
| 2 | 腾讯财经 `tencent` | `https://web.ifzq.gtimg.cn/appstock/app/fqkline/get`，640 行/页 | qfq | 前复权日线，代码 `sh600519`/`sz000001` |
| 3 | 新浪财经 `sina` | `https://quotes.sina.cn/cn/api/json_v2.php/CN_MarketDataService.getKLineData`，`scale=240` | none | 备份源（不复权） |
| 4 | AKShare `akshare` | `stock_zh_a_hist(adjust="qfq")` | qfq | 最后的免费备份 |

- 在线 `research` 使用 `config/universe_cn.json` 的 24 行业生产股票池（每篮 3 只），并额外拉基准 `000300.SH`；历史窗口 = `as_of − 520 天` 至 `as_of`。
- 未使用的更高优先级源标记 `not_used/not_needed`，不伪装成 `pending`。

### 1.2 标准化与质量清洗

- 统一 OHLCV 列：`date / open / high / low / close / volume / amount / source / adjustment / is_degraded`。
- `quality.py` 过滤：负价格、`high < low`、`volume <= 0`、重复日期（留首条）、越界日期、畸形行；清洗报告记录 `total_in/total_out/dropped` 与各原因计数。

### 1.3 基本面（AKShare，可选增强）

`data/fundamentals_source.py` 默认调用 `data/akshare_fundamentals.py`（版本 `1.19.1`），与 OHLCV 路由分开：

| 接口 | 参数 | 映射字段 |
|------|------|----------|
| `stock_value_em` | 六位代码 | `PE(TTM)→pe_ttm`、`市净率→pb`、`市销率→ps_ttm`、`总市值→market_cap` |
| `stock_financial_analysis_indicator_em` | 带后缀代码 + `indicator="按报告期"` | `ROEJQ→roe`、`XSMLL→gross_margin`、`ZCFZL→debt_ratio`、`TOTALOPERATEREVETZ→revenue_yoy`、`PARENTNETPROFITTZ→profit_yoy` |

- 比例不变：五个财务百分数先 `/100` 再 `round6`（`ROEJQ=16.75 → roe=0.1675`）。
- 日期分离：估值取不晚于完整行情截止日的最新记录，保留 `valuation_as_of`；财务按实际 `report_date/notice_date/update_date` 验证。
- 主记录 `source=akshare/source_tier=T3`，另记 `upstream_source=eastmoney`；仅无 AKShare 记录的代码才整记录回退东方财富估值（T2）。
- 未安装 AKShare 可明确降级；离线 fixture 不导入 AKShare。

### 1.4 新闻 / 公告

- 新闻三窗口（当日/当周/当月，金融+时政）：东方财富 / 财联社 / 新浪，`config/news` 控制窗口与去重（Jaccard 0.85）。
- 公告：`data_notice`（巨潮资讯网 cninfo 等 T1 源）。
- 信源分层见 §4。

---

## 2. A 股数据存储

- **行情缓存**：`~/.quant_agent/cache`（仓库外；`QUANT_AGENT_CACHE_DIR` 可覆盖）。按 `symbol + 日期区间 + 复权 + 源` 写分区缓存；pyarrow 可用写 Parquet，否则降级 CSV；缓存键 sha256 前 32 位。命中即 `status=cache_hit`、`requests=0`。
- **研究产物**（`--out` 目录，受完整性保护）：`selection.json`、`timing.json`、`fundamentals-raw.json`、`events-raw.json`、`report.json/.md/.html`、`strategy.json`、`run-manifest.json`。
- **离线夹具**：`panel.csv`（长表 date,symbol,OHLCV+amount）、`benchmark.csv`、`universe.json`、新闻 `items.json`、基本面 `fundamentals.json`、公告 `announcements.json`。
- A 股行情**不入库**（避免缓存成为未跟踪产物污染仓库）。

---

## 3. 美股数据获取

美股取数由 `scripts/quant_agent/us/fetch.py` 完成，全部走**腾讯美股接口**（`User-Agent: Mozilla/5.0`）：

| 数据 | 接口 | 覆盖 | 备注 |
|------|------|------|------|
| 前复权日线 | `https://web.ifzq.gtimg.cn/appstock/app/usfqkline/get` | 常规时段 OHLCV | **T+1 发布**，当日收盘后要等次日 |
| 批量报价 | `https://qt.gtimg.cn/q=usAAPL,...` | 常规时段 OHLCV | 收盘后即可用，字段 `[5][33][34][3][6]` |
| 分钟分时 | `https://web.ifzq.gtimg.cn/appstock/app/usMinute/query` | 仅 09:30–16:00 | 不含盘前/盘后分钟序列 |
| 盘前/盘后 | `usMinute` 的 `pandata` | 仅最新快照 | `tag=before/after`，无历史序列 |
| 代码解析 | `https://smartbox.gtimg.cn/s3/` | `AAPL → usAAPL.OQ` | K 线接口需交易所后缀 |
| 夜盘（Blue Ocean 等） | — | 不支持 | 腾讯无此数据 |

两条更新路径：
- `fetch_live_bars`：用批量报价补齐「当日已收盘」的日线（快，适合每日增量；仅接受时间戳 ≥ 16:00 且 OHLC 自洽的报价）。
- `fetch_kline`：用前复权 K 线做权威刷新（慢，逐标的解析代码，适合校正；含拆股/分红调整）。

---

## 4. 美股数据存储

`data/us/` 是美股模块的唯一数据来源，**随仓库一起分发**（"克隆即可回测"）：

```
data/us/
  parquet_store/us/<TICKER>.parquet   单票前复权日线，563 个标的（含基准 SPY）
  pit/sp500_membership.csv.gz         标普 500 时点成分股快照，列 date,tickers
```

- 单票 parquet 列：`date, open, high, low, close, volume, pct_change, price_change, amplitude, symbol`。
- **前复权（qfq）**：已按拆股与分红调整（否则 NFLX 大比例拆股会被误判为 −90% 暴跌）。
- 日期范围约 2011-04 起至 2026-10-28（不同标的上市时间不同）；约 123 MB / 564 个文件。
- 成分股快照约每月一条，按交易日向前填充；不在最新快照的标的当日不可买 → 回测不含幸存者偏差。
- 代码校验：抓取时用「最新收盘价一致（±3%）」反推真实代码，避免模糊搜索错配（历史上曾错配污染 74 只，清洗后保留 562 只 + SPY）。
- 数据由上游 `quant_llm_eval` 的抓取脚本产出，本仓库只消费；也可用本仓库命令增量更新：

```bash
cd scripts
python -m quant_agent.us fetch                 # 用批量报价补齐当日已收盘的日线
python -m quant_agent.us fetch --kline AAPL,MSFT  # 用前复权 K 线做权威校验/刷新
```

---

## 5. 信源可靠性分层（`config/source_registry.yaml`）

`tier: T1 > T2 > T3`，AI 产物每条结论需绑定可解析源与 `source_tier/published_at/url`。

| 层 | 定位 | 代表源 |
|----|------|--------|
| T1 | 法定信息披露与权威媒体/机构 | 巨潮 cninfo、上交所、深交所、证监会、央行、发改委、财政部、统计局、新华社、人民日报、同花顺 iFinD |
| T2 | 主流财经媒体与数据平台 | 东方财富、腾讯财经、财联社、新浪财经、央视 |
| T3 | 开放聚合与离线夹具 | AKShare、离线 fixture |

---

## 6. 密钥与安全红线

- 同花顺 Key 仅从环境变量 `HITHINK_FINANCE_API_KEY` 读取，绝不落盘明文；`config/ths-mcp.json` 只允许 `${HITHINK_FINANCE_API_KEY}` 占位符。
- `run-manifest.json` 只记录 `key_present` 布尔。
- 未配置 Key 时 `configured=false`、零网络请求，自动降级 `tencent → sina → akshare`。

---

## 7. 数据能力边界速查

| 数据 | A 股 | 美股 |
|------|------|------|
| 日线 | 前复权（qfq），多源降级 | 前复权（qfq），T+1，当日用报价补 |
| 分时 | 分钟（`data_minute`） | 常规时段 1 分钟约 391 根，无盘前/盘后 |
| 盘前/盘后 | — | 仅最新快照，无历史序列 |
| 夜盘 | — | 不支持 |
| 基本面 | AKShare 估值+财务 | 不适用 |

> 仅客观数据获取/存储说明，不构成投资建议。
