# stock-data

本仓库存放量化投研 Agent（`ProMatheus-ltsc/quant-trading-agent`）的**美股回测数据**，以及数据获取方式的说明文档。

## 目录

```
parquet_store/us/<TICKER>.parquet   单票前复权日线，563 个标的（含基准 SPY）
pit/sp500_membership.csv.gz         标普 500 时点成分股快照，列 date,tickers
QUANT-AGENT-DATA-SOURCES.md         数据获取与存储说明（A股 + 美股）
```

## 数据口径

- **前复权（qfq）**：已按拆股与分红调整。原始抓取用腾讯美股前复权接口
  （`web.ifzq.gtimg.cn/appstock/app/usfqkline/get`）；若用未复权数据回测，
  NFLX 这类大比例拆股会被误判为 −90% 暴跌。
- **列**：`date, open, high, low, close, volume, pct_change, price_change, amplitude, symbol`。
- **日期范围**：约 2011-04 起至 2026-10-28（不同标的上市时间不同）。
- **成分股快照**：约每月一条，装载时按交易日向前填充；某日不在最新快照的标的当日不可买，
  因此回测不含幸存者偏差。
- **代码校验**：抓取时用「最新收盘价一致（±3%）」反推真实代码，避免模糊搜索错配；
  清洗后保留 562 只可交易标的 + SPY。

## 如何被 quant-trading-agent 使用

`quant-trading-agent` 通过 **git submodule** 以外链方式引用本仓库，挂载在其 `data/us`：

```bash
# 首次克隆（连带数据）
git clone --recursive https://github.com/ProMatheus-ltsc/quant-trading-agent.git

# 已克隆但数据缺失时
git submodule update --init --recursive
```

挂载后 `data/us/parquet_store/us/*.parquet` 与 `data/us/pit/*` 即对应本仓库路径，
`config/us_equity.yaml` 的 `data.root: data/us` 无需改动。

也可不挂子模块，改用环境变量指向任意本地数据目录：

```bash
export QUANT_US_DATA_ROOT=/path/to/stock-data
```

或运行时传 `--data-root`（`quant_agent.us` 各子命令均支持）。

## 更新数据

```bash
# 在 quant-trading-agent 仓库内增量更新（用批量报价补齐当日已收盘日线）
cd scripts && python -m quant_agent.us fetch

# 用前复权 K 线做权威校验/刷新
cd scripts && python -m quant_agent.us fetch --kline AAPL,MSFT
```

更新后在本仓库重新提交 `parquet_store/us/*.parquet` 即可，子模块指针按需推进。

> 仅客观数据与说明，不构成投资建议。数据可能有延迟，以交易所官方为准。
