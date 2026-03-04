# Polymarket 跟單交易機器人（Rust 版本）

[![Rust](https://img.shields.io/badge/rust-1.70%2B-orange.svg)](https://www.rust-lang.org/)
[![Platform](https://img.shields.io/badge/platform-Windows%20WSL2-blue.svg)](https://docs.microsoft.com/zh-tw/windows/wsl/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

一個針對 [Polymarket](https://polymarket.com/) 的高效能 Rust 跟單交易機器人。  
透過 **WebSocket newHeads 即時訂閱** + **HTTP 500ms 超高速輪詢**，自動監控目標交易員並複製交易，實現業界最低延遲的跟單執行。

---

## ⚡ 效能規格

| 指標 | 規格 |
|------|------|
| HTTP 輪詢間隔 | **500ms**（每位交易員獨立並行） |
| 鏈上偵測延遲 | **~0.1 秒**（WebSocket newHeads 即時訂閱） |
| 網路穩定性 | **100%**（WSL2 原生網路，無 Docker NAT 層） |
| 交易員並行監控 | **無限制**（每人獨立 goroutine） |
| 去重機制 | ✅ 共用時間戳防止重複下單 |

---

## 🚀 功能特性

### 超高速偵測（v2 架構）
- **WebSocket newHeads 即時訂閱**：連接 Polygon 鏈上節點，新區塊出現（~2 秒/塊）即時觸發 `eth_getLogs`，偵測延遲從 2 秒縮短至 **0.1 秒**
- **HTTP 並行輪詢**：每位交易員獨立任務，500ms 間隔同時監控多位交易員（不再串行等待）
- **雙路徑去重**：HTTP 和鏈上路徑共用時間戳狀態，防止同一交易被重複跟單

### 智慧 RPC 管理
- **5 個免費 HTTP 備用 RPC**：publicnode, llamarpc, ankr, 1rpc, polygon-rpc
- **2 個免費 WSS 備用 RPC**：publicnode WSS, ankr WSS
- **自動切換**：連續失敗 3-5 次後自動切換下一個 RPC
- **指數退避**：失敗時 2→4→8→16→32→60 秒間隔，保護 RPC 配額
- **WSS 降級**：所有 WSS 端點失敗時，自動降級為 HTTP 輪詢備用模式

### 多種跟單策略
- **固定金額（FIXED）**：每次固定金額跟單，適合風控嚴格的用戶
- **百分比（PERCENTAGE）**：複製交易員倉位的固定百分比
- **自適應（ADAPTIVE）**：根據交易規模動態調整複製比例

### WebSocket 市場資料
- 連接 Polymarket 官方 WebSocket（`wss://ws-subscriptions-clob.polymarket.com/ws/market`）
- 自動指數退避重連（1s→2s→4s→...→60s）
- 心跳機制：每 30 秒 Ping，3 次未回應觸發重連

### 訂單執行
- **EIP-712 簽名**：符合 Polymarket CLOB API 規範
- **直接 API Key 認證**：填入預配置的 API 憑證跳過 L1 認證，解決 401 問題
- 查詢市場最佳賣價後建立限價單
- 失敗自動重試機制

---

## 📋 系統需求

### Windows 用戶（推薦方式）
- **Windows 10 版本 2004** 或更高（WSL2 支援）
- **WSL2** + Ubuntu（已安裝）
- **磁碟空間**：~2 GB（Rust 工具鏈 + 編譯快取）
- **記憶體**：最低 2 GB 可用

### 所有平台通用
- **Polymarket 帳戶**：錢包有 USDC 餘額
- **Polygon RPC**：免費或付費端點均可

---

## 🛠️ 安裝教學

### 方式一：WSL2 + Ubuntu（Windows 推薦，最穩定）

> 此方式完全繞過 Docker NAT 網路層，解決了間歇性連接失敗問題。

#### 第一步：確認 WSL2 已安裝

```powershell
# 在 PowerShell 中執行
wsl --list --verbose
```

預期輸出應包含 `Ubuntu  Running  2`。

如果沒有 Ubuntu，執行以下指令安裝：
```powershell
wsl --install -d Ubuntu
```

#### 第二步：安裝 Rust 工具鏈（WSL2 內）

```powershell
# 在 PowerShell 中執行，一鍵安裝
wsl -d Ubuntu -- bash -c 'curl --proto "=https" --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y'
```

驗證安裝：
```powershell
wsl -d Ubuntu -- bash -c 'source ~/.cargo/env && rustc --version'
# 預期輸出：rustc 1.93.x (...)
```

#### 第三步：安裝 C 編譯工具

```powershell
# 需要輸入 WSL2 Ubuntu 密碼
wsl -d Ubuntu -- bash -c 'sudo apt-get update && sudo apt-get install -y build-essential pkg-config'
```

驗證：
```powershell
wsl -d Ubuntu -- bash -c 'gcc --version'
# 預期輸出：gcc (Ubuntu 13.x) ...
```

#### 第四步：配置 `.env` 文件

在 Windows 中用記事本或 VS Code 編輯 `.env`：

```env
# 必填：要跟單的交易員地址（逗號分隔）
USER_ADDRESSES=0x交易員地址1,0x交易員地址2

# 必填：你的 Polymarket 錢包地址
PROXY_WALLET=0x你的錢包地址

# 必填：私鑰（64 位十六進制，不含 0x 前綴）
PRIVATE_KEY=你的64位私鑰

# 必填：CLOB API 憑證（取得方式見下方說明）
CLOB_API_KEY=你的API_KEY
CLOB_SECRET=你的SECRET
CLOB_PASSPHRASE=你的PASSPHRASE
```

#### 第五步：啟動機器人

**方式 A：使用啟動腳本（最簡單）**
```powershell
cd C:\Users\你的用戶名\Desktop\bott\polymarket-copytrading-bot
.\run-wsl.ps1
```

**方式 B：手動執行**
```powershell
wsl -d Ubuntu -- bash -c 'source ~/.cargo/env && cd /mnt/c/Users/你的用戶名/Desktop/bott/polymarket-copytrading-bot && CARGO_TARGET_DIR=/tmp/polymarket-target cargo run'
```

> 💡 **提示**：首次編譯約需 3-5 分鐘。之後增量編譯只需 ~1 秒。

---

### 方式二：Linux / macOS 原生安裝

#### 安裝 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

#### 安裝依賴

```bash
# Ubuntu/Debian
sudo apt-get install -y build-essential pkg-config

# macOS
xcode-select --install
```

#### 編譯並運行

```bash
cd polymarket-copytrading-bot
cargo run --release
```

---

### 方式三：Docker（舊方式，有網路間歇問題）

> ⚠️ Docker 在 Windows 上有 NAT 網路間歇問題，建議改用 WSL2 方式。

```bash
# 建立 cargo 快取 volume（只需執行一次）
docker volume create polymarket-cargo-cache
docker volume create polymarket-cargo-target

# 編譯並運行
docker run --rm \
  -v "$(pwd):/app" \
  -v "polymarket-cargo-cache:/root/.cargo/registry" \
  -v "polymarket-cargo-target:/tmp/target" \
  -w /app \
  -e CARGO_TARGET_DIR=/tmp/target \
  rust:slim \
  cargo run
```

---

## ⚙️ 詳細配置說明

### 取得 CLOB API 憑證

CLOB API 憑證是機器人下單的必要認證。取得方式：

```bash
# 安裝 Python 客戶端
pip install py-clob-client

# 執行以下 Python 腳本取得憑證
python3 << 'EOF'
from py_clob_client.client import ClobClient

# 填入你的私鑰（有 0x 前綴）
PRIVATE_KEY = "0x你的私鑰"

c = ClobClient(
    host="https://clob.polymarket.com",
    key=PRIVATE_KEY,
    chain_id=137
)

creds = c.create_api_credentials()
print("CLOB_API_KEY=" + creds.api_key)
print("CLOB_SECRET=" + creds.api_secret)
print("CLOB_PASSPHRASE=" + creds.api_passphrase)
EOF
```

將輸出的三個值填入 `.env` 文件。

---

### RPC 端點配置

#### 免費端點（預設，已配置備用切換）

```env
# HTTP RPC（用於 eth_getLogs）
RPC_URL=https://polygon-bor-rpc.publicnode.com

# WSS RPC（用於 newHeads 即時訂閱）
POLYGON_WSS_URL=wss://polygon-bor-rpc.publicnode.com
```

程式內建 5 個 HTTP 備用 + 2 個 WSS 備用，失敗時自動切換。

#### 付費端點（更快更穩，建議正式使用）

**🏆 Alchemy（推薦）— 免費 300M compute units/月**
1. 前往 [alchemy.com](https://www.alchemy.com/) 免費註冊
2. 建立 **Polygon Mainnet** App
3. 複製 API Key，填入：

```env
RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/你的API_KEY
POLYGON_WSS_URL=wss://polygon-mainnet.g.alchemy.com/v2/你的API_KEY
```

**⚡ QuickNode — 免費 10M credits/月**
1. 前往 [quicknode.com](https://www.quicknode.com/) 免費註冊
2. 建立 Polygon endpoint，複製 HTTP/WSS URL

```env
RPC_URL=https://你的端點.quiknode.pro/你的TOKEN/
POLYGON_WSS_URL=wss://你的端點.quiknode.pro/你的TOKEN/
```

**🌐 Infura — 免費 100K requests/天**
1. 前往 [infura.io](https://www.infura.io/) 免費註冊
2. 建立 Polygon 專案，複製 Project ID

```env
RPC_URL=https://polygon-mainnet.infura.io/v3/你的PROJECT_ID
POLYGON_WSS_URL=wss://polygon-mainnet.infura.io/ws/v3/你的PROJECT_ID
```

---

### 跟單策略配置

#### 固定金額策略（推薦新手）

```env
COPY_STRATEGY=FIXED
COPY_SIZE=1.5           # 每次固定跟單 $1.5 USD
MAX_ORDER_SIZE_USD=20.0  # 單筆最大 $20
MIN_ORDER_SIZE_USD=1.0   # 單筆最小 $1
```

#### 百分比策略

```env
COPY_STRATEGY=PERCENTAGE
COPY_SIZE=10.0           # 複製交易員倉位的 10%
MAX_ORDER_SIZE_USD=100.0
MIN_ORDER_SIZE_USD=1.0
```

#### 自適應策略

```env
COPY_STRATEGY=ADAPTIVE
COPY_SIZE=10.0
ADAPTIVE_MIN_PERCENT=5.0     # 小額交易（<$500）用 5%
ADAPTIVE_MAX_PERCENT=20.0    # 大額交易（≥$500）用 20%
ADAPTIVE_THRESHOLD_USD=500.0  # 分界點 $500
```

---

### 完整 `.env` 參數說明

```env
# ── 必填 ─────────────────────────────────────────────────────────────────────

# 監控的交易員地址（逗號分隔，支援多個）
USER_ADDRESSES=0x...

# 你的 Polymarket 代理錢包地址
PROXY_WALLET=0x...

# 私鑰（64 位十六進制，不含 0x）
PRIVATE_KEY=...

# CLOB API 憑證
CLOB_API_KEY=...
CLOB_SECRET=...
CLOB_PASSPHRASE=...

# ── API 端點（有默認值，通常不需修改）───────────────────────────────────────

CLOB_HTTP_URL=https://clob.polymarket.com/
CLOB_WS_URL=wss://ws-subscriptions-clob.polymarket.com/ws

# ── RPC 端點 ──────────────────────────────────────────────────────────────────

# Polygon HTTP RPC（eth_getLogs 查詢）
RPC_URL=https://polygon-bor-rpc.publicnode.com

# Polygon WSS RPC（newHeads 即時訂閱）
POLYGON_WSS_URL=wss://polygon-bor-rpc.publicnode.com

# USDC 合約（通常不需修改）
USDC_CONTRACT_ADDRESS=0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174

# ── 跟單策略 ──────────────────────────────────────────────────────────────────

COPY_STRATEGY=FIXED          # FIXED | PERCENTAGE | ADAPTIVE
COPY_SIZE=1.5                # 金額（USD）或百分比
MAX_ORDER_SIZE_USD=20.0      # 單筆最大金額
MIN_ORDER_SIZE_USD=1.0       # 單筆最小金額

# 可選：交易乘數（例如 1.5 = 放大 1.5 倍）
# TRADE_MULTIPLIER=1.0

# ── 效能設定 ──────────────────────────────────────────────────────────────────

# HTTP 輪詢間隔（毫秒），推薦 500ms
POLL_INTERVAL_MS=500

# HTTP 請求超時（毫秒）
REQUEST_TIMEOUT_MS=3000

# ── 風控設定（可選）──────────────────────────────────────────────────────────

# SKIP_BALANCE_CHECK=true    # 跳過餘額檢查（RPC 無法讀取餘額時使用）
# MAX_POSITION_SIZE_USD=1000.0  # 最大持倉
# MAX_DAILY_VOLUME_USD=5000.0   # 每日最大交易量

# ── Telegram 通知（可選）─────────────────────────────────────────────────────

# TELEGRAM_BOT_TOKEN=...
# TELEGRAM_CHAT_ID=...
```

---

## 🎯 啟動方式

### 一般啟動（WSL2）

```powershell
# 在 PowerShell 中
.\run-wsl.ps1
```

### 開發模式（顯示詳細日誌）

```powershell
wsl -d Ubuntu -- bash -c 'source ~/.cargo/env && cd /mnt/c/Users/cc183/Desktop/bott/polymarket-copytrading-bot && CARGO_TARGET_DIR=/tmp/polymarket-target RUST_LOG=debug cargo run'
```

### 健康狀態檢查

```powershell
wsl -d Ubuntu -- bash -c 'source ~/.cargo/env && cd /mnt/c/Users/cc183/Desktop/bott/polymarket-copytrading-bot && CARGO_TARGET_DIR=/tmp/polymarket-target cargo run --bin health_check'
```

### 尋找活躍交易員

```powershell
wsl -d Ubuntu -- bash -c 'source ~/.cargo/env && cd /mnt/c/Users/cc183/Desktop/bott/polymarket-copytrading-bot && CARGO_TARGET_DIR=/tmp/polymarket-target cargo run --bin find_traders'
```

---

## 🏗️ 架構說明

```
polymarket-copytrading-bot/
├── src/
│   ├── main.rs           # 程式入口：任務調度、Ctrl+C 處理
│   ├── config.rs         # 配置解析：策略計算、WebSocket URL 修正
│   ├── monitor.rs        # 交易監控：HTTP 輪詢 + WSS newHeads + 去重
│   ├── executor.rs       # 訂單執行：EIP-712 簽名、CLOB API 提交
│   ├── types.rs          # 資料結構定義
│   └── bin/
│       ├── health_check.rs  # 系統健康檢查工具
│       └── find_traders.rs  # 尋找優秀交易員工具
├── Cargo.toml            # Rust 依賴配置
├── Makefile              # 構建自動化指令
├── run-wsl.ps1           # WSL2 啟動腳本（Windows 用）
├── .env                  # 你的配置（不要提交到 Git）
├── .env.example          # 配置範本
└── README.md             # 本文件
```

### 監控架構（`monitor.rs`）

```
run_monitor()
├── ① run_trader_poll()     × N 個交易員（並行）
│     └── 每 500ms 查詢 Data API，偵測新交易
│
├── ② run_onchain_monitor_ws()
│     ├── WSS 主模式：訂閱 Polygon newHeads
│     │   └── 新區塊 → 立即 eth_getLogs → 偵測 OrderFilled 事件
│     └── HTTP 備用：若 WSS 全部失敗，每 2 秒 HTTP 輪詢
│
└── ③ run_websocket_with_retry()
      └── 訂閱 Polymarket 市場資料（自動重連）

共用時間戳（SharedTimestamps）: 防止 ① 和 ② 重複下單
```

---

## 🔒 安全注意事項

> ⚠️ **重要安全提醒**

1. **私鑰安全**：`.env` 文件包含私鑰，**絕對不要**提交到 Git 或分享給他人
2. **專用錢包**：使用專門的交易錢包，不要使用存有大量資產的主錢包
3. **小額測試**：正式使用前，先將 `COPY_SIZE` 設為最小值（如 `1.0`）測試
4. **資金限制**：設置 `MAX_ORDER_SIZE_USD` 和 `MAX_DAILY_VOLUME_USD` 作為保護
5. **私有 RPC**：正式使用建議申請 Alchemy/QuickNode 私有端點，避免速率限制

---

## 🐛 故障排除

### 問題：編譯時出現「linker `cc` not found」

**原因**：WSL2 Ubuntu 未安裝 C 編譯工具。

**解決**：
```powershell
wsl -d Ubuntu -- bash -c 'sudo apt-get update && sudo apt-get install -y build-essential'
```

---

### 問題：「Permission denied」無法寫入 target 目錄

**原因**：之前用 Docker（root）建立的 `target` 目錄 WSL2 沒有寫入權限。

**解決**：使用 `/tmp` 作為 target 目錄（已在腳本中設定）：
```bash
CARGO_TARGET_DIR=/tmp/polymarket-target cargo run
```

---

### 問題：401 Unauthorized（API 認證失敗）

**原因**：CLOB API 憑證未設定或已過期。

**解決**：
1. 重新執行 Python 腳本取得新憑證
2. 確認 `.env` 中三個欄位都已填入：`CLOB_API_KEY`, `CLOB_SECRET`, `CLOB_PASSPHRASE`

---

### 問題：鏈上監控 WSS 連接失敗

**症狀**：`[On-chain WSS] Error (#1) on RPC[0]: ...`

**說明**：程式會自動切換到備用 WSS 端點（ankr），或降級為 HTTP 輪詢模式。不影響 HTTP 輪詢的主要監控功能。

**最佳解決**：申請 Alchemy 免費帳號，設定付費 WSS 端點（更穩定）。

---

### 問題：HTTP 輪詢查詢失敗（偶發）

**症狀**：`⚠️ [HTTP] 0x...查詢失敗`

**說明**：Polymarket Data API 偶發限流。機器人會在下次輪詢（500ms 後）自動重試。

---

### 問題：余額不足警告

**解決**：
1. 確認 Polymarket 帳戶有足夠 USDC
2. 或設定 `SKIP_BALANCE_CHECK=true` 跳過餘額檢查

---

## 📊 啟動後正常輸出示例

```
╔══════════════════════════════════════════╗
║   🤖  Polymarket 跟單交易機器人           ║
╚══════════════════════════════════════════╝

✅ 配置載入成功
🔄 輪詢間隔: 500ms
📡 監控交易員 (2 個): ...

📡 交易監控器啟動中（超高速模式 v2）...
⚡ 並行監控 2 位交易員，HTTP 輪詢間隔: 500ms
🔗 Polygon 鏈上監控: WebSocket newHeads 即時訂閱（~0.1s 偵測）

  🔄 [HTTP] 啟動獨立監控: 0x000d25... (500ms 間隔)
  🔄 [HTTP] 啟動獨立監控: 0x5350af... (500ms 間隔)
✅ [WS] 連接成功（HTTP 101 Switching Protocols）
  ✅ [HTTP] 0x000d25... 初始化完成（跳過 5 個歷史交易）
  ✅ [HTTP] 0x5350af... 初始化完成（跳過 5 個歷史交易）
✅ [鏈上] 初始化完成（區塊 83742405），開始 WSS 即時監控...
[On-chain WSS] ✅ 已連接 wss://polygon-bor-rpc.publicnode.com
[On-chain WSS] ✅ newHeads 訂閱成功
```

當偵測到交易時：
```
⚡ [鏈上] 0x000d25... 偵測到新交易 | tx: 0xabc123... | 區塊: 83742500
  ✅ [鏈上+API] 「誰會贏得美國大選？」 | BUY | $1.50

🚨 [HTTP] 0x5350af... | 「比特幣年底破 $100K？」 | SELL | $1.50
✅ [執行] 跟單成功！訂單 ID: abc-123-def
```

---

## 📞 支援

如有問題或建議：
- Telegram: [@Vladmeer](https://t.me/vladmeer67)
- Twitter: [@Vladmeer](https://x.com/vladmeer67)

---

**使用 Rust 強力驅動 ❤️ | WSL2 原生運行，零 Docker 網路問題**
