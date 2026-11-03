# 憑證攻擊（釣魚）網域偵測器 — 使用指南與教學

**版本：** 1.0（2026-11-02）・**語言：** Python 3.10+ ・**需要雲服務：** 無

這是一個完全在本地運行的兩層偵測系統，用來辨識冒用銀行、郵件服務與其他品牌的網域。第一層是「品牌白名單」規則判斷，第二層是基於 15 項網域特徵的邏輯斯特回歸（logistic regression）分類器。所有操作都在 `127.0.0.1` 本機完成——不需要帳號、不需要 API key、不上傳任何資料。

---

## 1. 你需要的環境

| 項目 | 需求 |
|---|---|
| 作業系統 | Linux、macOS 或 Windows（能裝 Python 即可） |
| Python | 3.10 以上（用 `python3 --version` 確認） |
| 網路 | **僅第一次執行**需要（下載約 20 MB 的公開資料集，之後都會快取） |
| 硬體 | 任何筆電都夠——訓練只要約 4 秒 |

## 2. 環境安裝（做一次，約 1 分鐘）

```bash
# 在專案資料夾內執行
python3 -m venv .venv
# 啟用虛擬環境（依你的 shell 挑選）
source .venv/bin/activate        # bash/zsh
.venv\Scripts\activate           # Windows PowerShell/cmd
pip install scikit-learn         # 僅訓練需要——執行期完全不需要任何第三方套件
```

## 3. 快速上手（三條指令跑完整個專案）

```bash
python train.py            # 1. 下載資料（僅首次）→ 訓練 → 儲存模型與評估指標
python test_detector.py    # 2. 自我檢測：成功會印出 ALL CHECKS PASSED
python app.py              # 3. 啟動網頁 UI + API，位址 http://127.0.0.1:8321
```

用瀏覽器開啟 **http://127.0.0.1:8321**，試著輸入這兩個，觀察差異：

- `paypal-verify-secure.cf` → **PHISHING（釣魚）**（可疑頂級域名＋品牌冒充＋密碼提示詞）
- `accounts.wellsfargo.com` → **BENIGN（安全）**（白名單層：真正銀行自己的子網域是被授權的）

在終端機按 `Ctrl+C` 即可停止伺服器。

## 4. 判定是怎麼做出的

每個 `/api/check` 的回答都經過兩層：

1. **白名單檢查**——該網域是不是某個品牌「自己的」網域或其子網域（定義在 `features.py` 的 `BRAND_REAL`）？是的話直接判定 BENIGN 結束。
   理由：攻擊者不可能使用 `accounts.google.com`——除非 Google 本身被入侵。
2. **模型檢查**——15 個數字特徵（長度、點號／連字號／數字數量、字元熵、可疑 TLD 旗標、品牌冒充旗標、密碼提示詞旗標，以及兩個「TLD 交互項」特徵）代入 `models/model.json` 的邏輯斯特回歸權重向量。輸出為機率，≥ 0.5 即判定 PHISHING。

網頁介面會顯示**前 5 名特徵信號**（特徵值＋加權貢獻，正數＝往 PHISHING 方向推）。這就等於模型在自我解釋——讀它、信它，或用它來 DEBUG。

## 5. 使用 JSON API

```bash
curl -X POST http://127.0.0.1:8321/api/check \
  -H 'Content-Type: application/json' \
  -d '{"url":"chase-secure-update.xyz"}'
```

回應範例：

```json
{
  "input": "chase-secure-update.xyz",
  "label": "PHISHING",
  "prob": 1.0,
  "features": [
    {"name": "tld_suspicious", "label": "suspicious TLD", "value": 1, "signal": 7.3},
    {"name": "brand_x_tld",    "label": "brand impersonation on suspicious TLD", "value": 1, "signal": 2.1},
    ...
  ]
}
```

| 端點 | 方法 | 用途 |
|---|---|---|
| `/api/check` | POST `{"url": ...}` | 判定一個網域／URL |
| `/api/health` | GET | 存活檢查 → `{"ok": true}` |
| `/` | GET | 網頁 UI |

輸入格式錯誤（不像網域）→ HTTP 400 `{"error": "bad input"}`。
帶路徑或埠號的 URL 會先被正規化成主機名再評分。
**安全提示：** 伺服器只綁定本機環回位址（loopback），不會向外傳送任何資料。若要把埠號暴露到網路，請先加真實的鑑別機制。

## 6. 重新訓練與更新資料

- **隨時可重跑：** `python train.py`——`data/` 裡的資料檔只在缺漏時才會下載，因此第一次之後就能離線重訓。
- **更新資料集：** 刪掉 `data/` 內的原始清單（喜歡 `legit.json` 可保留），重新執行；`train.py` 會從 `data/SOURCES.txt` 記載的來源重新抓取。
- **每次重訓後都跑一次檢測：** `python test_detector.py`。
- 指標會寫入 `data/eval_report.json`；目前留出集（held-out）數字為 precision 0.995／recall 0.882／AUC 0.967（19,064 筆釣魚 vs 1,001 筆正規）。

## 7. 擴充它（既定的升級方向）

1. **新增品牌**——在 `features.py` 加兩行：
   `BRAND_REAL["mybank"] = ("mybank.com", "mybank.co.uk")` → 白名單與冒充邏輯都會自動套用。重訓是選做的（模型權重不會變）。
2. **降低門檻值**——想要更積極地警告，把 `app.py:score()` 裡的 0.5 門檻改成例如 0.3（召回提高，但誤報也會增加）。
3. **頁面層特徵**——抓取 Landing Page 的 HTML，對表單 action 主機與關鍵字注入做評分。這是不帶品牌的尾部案例最大的提升點。
4. **仿冒網域（Typosquatting）**——對品牌網域清單做 Levenshtein 距離＋同形字（homoglyph）檢查，捕捉未列入白名單品牌的仿冒。

## 8. 除錯對照表

| 症狀 | 解法 |
|---|---|
| `Port 8321 already in use` | 換埠號 `python app.py 8322`，或 `pkill -f "app.py 8321"` |
| `ModuleNotFoundError: sklearn` | 沒有用虛擬環境——執行 `source .venv/bin/activate` |
| 首次執行卡在下載 | 公開資料集可能失效/被擋——刪掉 `data/` 的未完成檔，檢查 `train.py` 內的 URL |
| 我信任的品牌被標記了 | 把它的真實網域加進 `BRAND_REAL`（見 §7.1），或讀特徵分解——你會看到到底是哪個信號贏了 |
| UI 顯示 `bad input` | 輸入框只收主機名／網域字串，例如 `paypal.com` 或 `https://paypal-verify.cf/x` |

## 9. 專案結構

```
credential-attack_detector/
├── features.py          # 特徵萃取＋白名單（純標準函式庫）
├── train.py             # 抓取 → 清洗 → 訓練 → models/model.json＋指標
├── app.py               # 標準函式庫 HTTP 伺服器：UI＋/api/check
├── test_detector.py     # 一個可執行的自我檢測
├── static/index.html    # 網頁 UI（單檔，無建置步驟）
├── models/model.json    # 訓練好的權重——執行期免依賴
├── data/                # 資料集＋SOURCES.txt＋eval_report.json
├── data/SOURCES.txt     # 出處紀錄：每個資料集的來源、URL、筆數、日期
└── README.md            # 本指南的精簡版
```

## 10. 誠實的限制

- **只看網域層。** 寄駐在合法平台「內部」的釣魚頁（如 `x.weebly.com`）不在範圍內（這類在訓練前已被過濾掉，不是「碰巧沒抓到」）。
- **recall ≈ 0.88**——在「不帶品牌、隨機字串」的尾部案例上較弱，這是已知弱點。
- **白名單是固定清單**；仿冒未列名單品牌的情況，只能靠通用特徵。
- **不是生產級 WAF。** 本質上是一個教學用偵測器：快速、可解釋、完全本地——用它來「學習與展示」，升級路線見 §7。
