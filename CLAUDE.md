# Readout：美股生技股解盲時間預測 app

> 本檔是從 claude.ai 規劃階段交接到 Claude Code 的專案說明。每次開啟專案時請先讀完本檔。
> 使用者：Danny（統計／生醫資料分析背景，熟 Python、R、SQL；PhD in Oncology）。溝通用繁體中文，技術名詞可用英文。

## 1. 專案目標

個人投資研究工具，追蹤美股生技、藥廠、工具服務類公司，核心功能是**預測事件何時發生**（尤其是臨床試驗 topline 公布時間），而不是預測股價漲跌。

- 前身是 BioCatalyst v1.2（以人為權重的「事件強度分數」），本專案以「經驗證的機率」取代它
- 必須能隨時新增公司：輸入股票代號，後端自動抓試驗、公告、財報、股價
- 手機優先：以 375px 寬度為基準，必須能用手機瀏覽器直接開啟
- 非投資建議，所有介面都要保留這個聲明

### 明確不做

- 不做股價方向預測（ARIMA、LSTM 等）。生技股短期價格由二元事件主導，這類模型預測力弱且會造成錯誤信心
- 不用推測或編造的數字填補資料缺口。缺就標示缺，推算的數字一律標 `derived: true`

## 2. 目前狀態（第一階段完成）

- `web/index.html`：手機原型，**已改為 fetch `../data/snapshot.json`**（內嵌 `DATA` 已移除；移除前已比對兩者內容完全相同），含載入中／失敗＋重試狀態；`TODAY` 改取自 `snapshot_date`
- repo 結構已建立（`backend/`、`tests/` 目前為空）；`.github/workflows/pages.yml` 會把 `web/` 與 `data/snapshot.json` 部署到 GitHub Pages（網址為 `<pages-url>/web/`，根目錄自動導向）
- 本機預覽：在 repo 根目錄執行 `python3 -m http.server 8765`，開 `http://localhost:8765/web/`（直接開 file:// 會因 fetch 被擋而失敗）
- `data/snapshot.json`：從原型抽出的資料快照，就是後端要產生的格式（schema v0）
- 四個分頁：近期（股價卡＋事件列表）、公司、帳本、說明；點事件進入詳情頁（累積機率曲線）
- 首頁最上方是股價卡，跟著公司篩選切換；目前只有 TWST 有股價
- 機率為**示範分布**，尚未訓練（見第 5 節）

### 測試公司與分工

| 代號 | 類型 | 用途 |
|---|---|---|
| TWST | 工具服務（無自有臨床試驗） | 測公司頁通用區塊：股價、財報、現金、8-K；測「沒有試驗的公司」顯示方式 |
| LLY | 大型藥廠 | 解盲時間模型的主要訓練資料來源（試驗多、歷史紀錄多） |
| VKTX | 單一產品生技 | 預告時間窗、股價反應、現金跑道 vs. 解盲；2026/9/22 維持劑量試驗公布是回測案例 |

三者關係：Twist 為 Lilly TuneLab AI 平台提供抗體數據（2026/9/16）；Lilly 與 Viking 在肥胖藥市場直接競爭。

## 3. 目標架構

```
readout/
├── CLAUDE.md
├── web/
│   └── index.html            # 前端，改為 fetch('../data/snapshot.json')
├── data/
│   ├── snapshot.json         # 後端每日產生，前端唯一資料來源
│   ├── watchlist.yaml        # 追蹤清單：新增公司只改這裡
│   └── raw/                  # 原始抓取結果（gitignore 或另存）
├── backend/
│   ├── fetch_trials.py       # ClinicalTrials.gov API v2
│   ├── fetch_sec.py          # SEC EDGAR：8-K、XBRL companyfacts（現金、股數）
│   ├── fetch_prices.py       # 每日收盤價
│   ├── build_events.py       # 合併成 events / ledger
│   ├── model/                # 解盲時間模型（見第 5 節）
│   ├── build_snapshot.py     # 輸出 data/snapshot.json
│   └── notify.py             # 每日推播
├── tests/
└── .github/workflows/daily.yml
```

部署：前端放 GitHub Pages 或 Vercel（claude.ai 發布的頁面無法連外部資料，正式版必須自架）。

## 4. 資料來源與注意事項

- **ClinicalTrials.gov API v2**：依 sponsor 抓試驗；重要欄位包含 phase、overall status、primary completion date（及其 actual/anticipated 類型）、completion date、results first posted date、conditions。注意 primary completion date 會被改寫，要保留每次抓取的歷史版本才能做回測
- **SEC EDGAR**：`data.sec.gov` 的 submissions 與 companyfacts。必須設定含聯絡 email 的 User-Agent，並遵守官方的請求頻率限制。8-K 依 Item 分級（舊版 BioCatalyst 規則：7.01、8.01 為高訊號；1.01、1.02、2.01、5.02 為中訊號；2.02、9.01 為例行）
- **股價**：第一版只做每日收盤價（GitHub Actions 每日台灣早上執行）。資料源到時候再比較 Finnhub、Alpha Vantage、Polygon 等的免費方案；yfinance 非官方 API，只適合研究用。API 金鑰放 GitHub Secrets，不可出現在前端
- **公司預告時間窗**（例如「預計 3Q26 公布」）：來自財報新聞稿與 8-K 附件，這是模型最重要的共變數，初期可能需要半自動擷取＋人工確認

### 新增公司的流程（目標）

在 `watchlist.yaml` 加一行代號與公司類型 → 下次排程自動抓取 → 前端自動出現。公司類型（大型藥廠、臨床階段生技、醫材、工具服務）決定公司頁顯示哪些區塊。

## 5. 解盲時間模型

**問題定義**：存活分析。

- 起點：試驗 primary completion date（或公司首次給出預告時間窗的日期，兩種都試）
- 事件：首次公開 topline 結果（8-K 或新聞稿）
- 尚未公布的試驗為右設限資料
- 輸出：累積發生機率曲線 F(t)，以及以「今天尚未公布」為條件的 P(30/90/180/365 天內)、中位數、80% 區間

**共變數候選**：公司預告時間窗、該公司過去預告的準度與「落在窗內哪個位置」的習慣、Phase、適應症、公司規模、試驗規模、是否對齊主要醫學大會（ASCO、ESMO、ASH、AHA、EASD、AACR、AAN、CTAD、ObesityWeek、ADA 等）。

**方法順序建議**：Kaplan–Meier 描述 → Weibull／log-logistic AFT 或 Cox 模型 → 視資料量考慮 discrete-time hazard 模型。Python 可用 lifelines，也可在 R 用 survival 驗證。

**最大工程難點**：建立「topline 公布日」標籤。8-K Item 只代表「可能含試驗資訊」，需要關鍵字比對加人工標註一批種子資料。

**驗證**：預測帳本。每次預測要**凍結並記錄時間戳**，事件發生後比對，不可事後改寫（避免 look-ahead）。累積 Brier score 與校準圖；至少 30 筆後才顯示校準結果。

**原型的示範分布**（上線前的佔位）：時間窗內以 Kumaraswamy(a=2, b=1.6) 分配、偏向窗內後段，窗內總機率 `pWithin` 手動設定，窗外以平均 75 天的指數分布延後。這些參數都要被真實模型取代。

## 6. 每日推播

- 時間：台灣早上 7–8 點（美股收盤在台灣清晨 4 點，冬令 5 點）
- 內容：前一晚盤後摘要（股價、新 8-K、試驗狀態變化），用 Claude API 整理成中文
- 即時警報：新的高訊號 8-K、單日漲跌超過 10%、試驗狀態改變
- 管道：先做 Telegram Bot（最簡單）；LINE 需用 Messaging API（LINE Notify 已停止服務）
- 語音版每日摘要列為之後的選項

## 7. 前端需要的修改

1. 移除內嵌的 `DATA`，改為讀取 `data/snapshot.json`，並處理載入中與失敗狀態
2. 股價卡：完整每日資料、區間切換（1 月／6 月／1 年）、事件標記由資料自動產生
3. LLY、VKTX 等公司的股價卡（目前顯示「待後端串接」）
4. 模型訓練完成後移除頂部的「原型版本」提示，改顯示模型版本與訓練資料筆數
5. 保持：375px 無水平捲動、底部分頁列、觸控目標至少 44px、深色模式、safe-area、手機返回鍵可關閉詳情頁（hash routing）

## 8. 原型資料的已知問題

- TWST 股價：2026/8/11–9/22 沒有每日資料（圖上以虛線標示「資料缺口」）
- TWST 9/25 收盤 $182.83 來自 Robinhood 頁面，該頁日期標為 9/27（週日），推定為 9/25 收盤，需以正式資料確認
- TWST FY26 Q2 營收 110.9 百萬美元為推算（全年指引中值減其他季），標為 derived
- TWST Q4 財報時間窗（11/5–11/25）為依往年推估，待公司公告
- LLY TRIUMPH-5：公司說 2026 年有 7 項 Phase 3 讀出，但未逐項確認本試驗在內
- VKTX VANQUISH-1/2：「2027 年讀出」來自二手報導；時間窗起點是用收案完成＋78 週推算的結構下限
- VKTX 現金跑道為最近一季燒錢速度線性外推，未計入融資與支出變化
- LLY、VKTX 帳本的股價反應欄位尚未填入

## 9. 建議的工作順序

1. 建立 repo 結構，前端改讀 `snapshot.json`，確認手機可開（部署到 GitHub Pages）
2. `fetch_prices.py` + GitHub Actions 每日排程，補齊三家公司股價
3. `fetch_sec.py`：8-K 列表與分級、XBRL 現金與股數
4. `fetch_trials.py`：抓 LLY、VKTX 試驗並保存每次抓取的版本
5. 建立歷史「預告 vs. 實際公布」資料集（先從 LLY 開始，人工標註種子資料）
6. 訓練第一版模型，取代示範分布，接上預測帳本
7. 每日推播（Telegram）
8. 擴充追蹤清單

每一步完成後都要：手機實測、確認沒有編造數字、更新本檔的「目前狀態」。
