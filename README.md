# stock-data

量化投研仓库 [`ProMatheus-ltsc/quant-trading-agent`](https://github.com/ProMatheus-ltsc/quant-trading-agent) 的**原始行情与基本面数据**。

本仓只存原始数据与取数说明：**不放策略产物、报告、交易单据或任何派生产物**。代码仓只读消费本仓，写入方向是单向的。

## 目录

```
parquet_store/cn/<代码>.parquet      A 股日线，299 只（沪深300 全成分）     103,281 行  2025-06-12 ~ 2026-11-13
parquet_store/us/<TICKER>.parquet    美股日线，563 只（含基准 SPY）        1,952,910 行  2008-05-08 ~ 2026-11-12
parquet_store/crypto/<交易对>.parquet 加密现货日线，1 只（BTCUSDT）            3,377 行  2017-08-17 ~ 2026-11-14
pit/hs300_membership.csv.gz          沪深300 时点成分股快照                    1 期  300 只
pit/sp500_membership.csv.gz          标普 500 时点成分股快照                2,720 期  487~507 只  1996-01-02 ~ 2026-08-18
fundamentals_us/annual_long.parquet  SEC XBRL 财务长表（ticker×concept×年度）  142,188 行
fundamentals_us/annual_wide.parquet  SEC XBRL 财务宽表（每行一个 ticker×财年）   9,504 行
fundamentals_us/summary.json         抓取覆盖统计
QUANT-AGENT-DATA-SOURCES.md          数据获取与存储说明（A 股 + 美股）
```

合计 872 个文件，工作树约 116 MB。

## 三种口径（务必分清）

三个分区的复权口径**不一样**，跨分区写代码前先看这张表。

| 分区 | 复权口径 | 列 | 说明 |
|---|---|---|---|
| `cn` | **不复权原始价 + 等比因子** | `date, open, high, low, close, volume, amount, adj_factor, source, adjustment` | `adjustment="none"`；复权价在读取时派生 |
| `crypto` | 不复权（加密现货无复权概念） | 同上 | `adjustment="none"`，`adj_factor` 恒为 1.0 |
| `us` | **前复权（qfq）** | `date, open, high, low, close, volume, pct_change, price_change, amplitude, symbol` | 遗留口径，**尚未迁移**；已按拆股与分红调整 |

### A 股：原始价 + 因子，不是前复权价

2026-11 迁移后的口径。`open/high/low/close` 是交易所**原始成交价**，永不因除权而改写；另一列 `adj_factor` 存等比后复权因子，读取时派生：

```
后复权 = 原始价 × adj_factor
前复权 = 原始价 × adj_factor / adj_factor[末日]        # 末日 = 你取到的最后一行，保证区间可拼接
```

为什么这么做：前复权价以最新价为基准回推，**一次除权就要重算全部历史**。只补尾巴会把新旧基准混进同一份文件，重算失败还会让错误永久无法检出——因为下一轮的重叠日已经变成新基准了。改成「原始价 + 因子」后，除权只让因子多一个跳变点，历史一行不动，增量退化成纯追加，且任何时候都能还原「当时看到的原始价」。

**因子必须单调不减**（除权只会让后复权价越来越高）。退市股的事件表末尾可能出现 `1.0` 之类的回落哨兵值，一旦照单全收会把前复权的基准日拉错，导致**整段**派生价格系统性偏移（实测 601989 偏 69%）——取数侧已 clamp 到历史峰值。

> 口径差异提示：本仓 A 股用**等比复权**（tushare/wind 口径）。国内券商软件的「前复权」多为**等差**（复权价 = 原始价 ± 累计除权差额），两者派生价格中位差约 0.14%、分红密集期可达 1.5%。腾讯与 westock 的 `qfq` 实测都是等差（`qfq - raw` 在 639 天里只变 5 次且分段恒定），**无法**用等比因子还原，因此它们不作为本仓 `cn` 分区的写入源。

### 美股：仍是前复权（待迁移）

`us` 分区沿用早期口径，直接存腾讯美股前复权接口（`usfqkline`）的调整后价格，没有 `adj_factor` 列。若改用未复权数据回测，NFLX 这类大比例拆股会被误判成 −90% 暴跌——所以当前口径是有意选择，不是遗漏。迁移到「原始价 + 因子」需要重取全历史，尚未排期。

### 单位约定

| 列 | 单位 | 备注 |
|---|---|---|
| `volume` | A 股为**手**（100 股） | 新浪接口给股，取数侧 ÷100；腾讯直接给手 |
| `amount` | **元** | ⚠️ 腾讯日线接口只返回 6 个字段、**不含成交额**，因此 `source=tencent` 的行走 `amount=0`；`source=sina` / `sina_batch` 的行才有真实成交额 |
| `adj_factor` | 无量纲，可正数 | `us` 分区无此列 |

`amount=0` 是数据源能力边界，不是抓取失败。依赖成交额的流动性约束请改用 `close × volume × 100` 估算，或只在该行 `amount > 0` 时启用。

## 如何被 quant-trading-agent 使用

**不再是 git submodule**（早期挂过 `data/us`，已取消）。四种方式任选：

```bash
# 1) 直接克隆到代码仓的 data/（.gitignore 已忽略该目录，不会进代码索引）
git clone http://192.168.1.108:3001/root/stock-data.git data

# 2) 克隆到任意位置，用环境变量指过去
export QUANT_US_DATA_ROOT=/path/to/stock-data

# 3) 运行时传 --data-root（quant_agent.us 各子命令支持）
python -m quant_agent.us backtest --data-root /path/to/stock-data

# 4) R2 为主通道（推荐，按 sha256 增量，A 股也覆盖）
python3 -m scripts.quant_agent data r2 pull
```

代码仓 `config/us_equity.yaml` 的 `data.root` 默认指向 `data`，`config/quant_agent.yaml` 的 `store.sections.cn.dir` 默认指向 `data/parquet_store/cn`——按方式 1 放置时无需改配置。

> R2 与 GitHub 的分工：`data r2 push/pull` 覆盖 `parquet_store/{us,cn}`、`pit`、`fundamentals_us`；本仓作为 R2 不可用时的降级基线，且**只有 `us`/`pit`/`fundamentals_us` 在 GitHub 覆盖范围内**。A 股分区历史上曾不入 GitHub，现已同步，但降级路径只补「缺失或 0 字节」的已跟踪文件，不覆盖本地已有内容。

## 更新数据

### A 股（沪深300 全成分）

```bash
# 日常：批量快照只补当日一根，3 次请求覆盖 300 只（约 2 秒）
python3 -m scripts.quant_agent data sync --pit hs300 --fast

# 有缺口/除权/无历史：逐只走历史接口补齐（约 5 分钟）
python3 -m scripts.quant_agent data sync --pit hs300 --force

# 首次建库或重建：整段重拉（约 20 分钟）
python3 -m scripts.quant_agent data sync --pit hs300 --days 520 --full
```

同步完把 `parquet_store/cn/*.parquet` 提交到本仓即可。**增量语义**是从本地末根往后拉、故意重叠一根做校验；`--force` 只影响新鲜度门槛（默认 4 天内不联网），不影响增量逻辑。

### 沪深300 成分快照

```bash
python3 -m scripts.quant_agent data pit build --index hs300    # 抓当前成分，同日覆盖、历史保留
python3 -m scripts.quant_agent data pit build --index hs300 --from-file history.csv  # 导入外部历史
```

> 免费源只给**当前**成分，拿不到长期成分变更史。东方财富 `BK0500` 有全量但同 IP 连续分页会被直接断连（实测），中证官网历史成分需逐条解析调整公告。所以 `hs300_membership.csv.gz` 目前只有 1 期，靠逐日累积；要补历史只能从 tushare `index_member`、聚宽等导出后 `--from-file` 导入。**不要假设它的历史深度和 `sp500_membership` 一样。**

### 加密现货

```bash
python3 -m scripts.quant_agent data binance kline --symbol BTCUSDT --save        # 自动从末根续抓
python3 -m scripts.quant_agent data binance kline --symbol BTCUSDT --start 2017-01-01 --save  # 首次全历史
```

### 美股

```bash
cd scripts
python -m quant_agent.us fetch                  # 增量补当日日线（批量报价，秒级）
python -m quant_agent.us fetch --kline AAPL,MSFT  # 走前复权 K 线权威刷新（逐标的，慢）
```

## 财务数据（SEC EDGAR 一手 XBRL）

`fundamentals_us/` 由 `scripts/quant_agent/us/sec_fundamentals.py` 抓取 **SEC 官方接口**（`data.sec.gov` companyconcept），非第三方聚合，覆盖 **519/563 只**（32 只无 CIK：退市/改名/行业 ETF）。

- **科目**：Assets、Liabilities、StockholdersEquity、NetIncomeLoss、Revenues、GrossProfit、OperatingIncomeLoss、NetCashProvidedByUsedInOperatingActivities、LongTermDebtNoncurrent、CommonStockSharesOutstanding、PaymentsOfDividends、PaymentsForRepurchaseOfCommonStock、EarningsPerShareDiluted
- **只取 10-K 年报**：以资产负债表科目（只在期末出现）的 `end` 为年度锚点，剔除 10-K 里夹带的季度期间事实
- **PIT（时点）**：每条事实带 `filed`（申报日），同一 `end` 被重述时保留**最早 filed** 的原始值，用 `filed_first` 做时点对齐，避免前视偏差
- **派生比率**：`roe / roa / gpoa(毛利/资产) / cfoa(经营现金流/资产) / leverage / ltd_ratio / gross_margin / operating_margin / payout((股息+回购)/净利润)`
- 覆盖：assets 88%、equity 97%、net_income 92%、cfo 86%、roe 91%（缺失集中在早期年份与金融类无毛利科目）

更新命令（需要 SEC 可访问，约 30–40 分钟）：

```bash
python scripts/quant_agent/us/sec_fundamentals.py --data-root data --workers 16
python scripts/quant_agent/us/sec_fundamentals.py --data-root data --convert-only  # 只重建 parquet
```

### 股数口径（重要）

`shares_dei`（封面页 `dei:EntityCommonStockSharesOutstanding`）是算市值的**唯一可信口径**：

- `us-gaap:CommonStockSharesOutstanding` 单位不统一（千股/百万股），且常只报**单一股份类别**（如 Alphabet 仅 Class A 约 3,300 万股，实际总量约 7 亿股），直接算市值会把 B/M 抬高数十倍。
- 另有一层坑：行情是**前复权价**（已按拆股调整），若配**当年原始股数**，拆股公司市值被低估数十倍（实测 NVDA 2019 差约 40 倍）。正确做法是 `equity / (qfq_price × shares_latest)`——总权益不随拆股变化，用最新股数配前复权价才自洽。
- dei 的 `end` 是**封面日**（如 2024-10-18），与财年末（2024-09-28）不同，必须按**同一份 10-K 的 `filed`** 对齐。
- 宽表已产出 `shares_final` / `shares_source`（`dei` 8522 行 / `us-gaap` 982 行），调用方应只信任 `dei`。

## 已知边界

- **A 股仅 299 只**（沪深300 全成分），缺 `600837.SH`（海通证券已并表为国泰海通），属正常缺口；退市股如 `601989.SH` 只保留到停牌日。
- **`cn` 历史深度 520 个自然日**（2025-06-12 起），不是长周期回测库。要更长历史需放宽 `data sync --days`。
- **`us` 存在幸存者偏差**：PIT 成分表覆盖 1996 年以来，但行情库缺部分退市/改名成员的历史价格，2016–2020 区间的当组成员有缺失。
- **`amount` 只在新浪来源下有效**，见上文单位约定。
- 数据可能有延迟，以交易所官方为准。

> 仅客观数据与说明，不构成投资建议。
