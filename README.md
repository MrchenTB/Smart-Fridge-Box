# 智慧冰箱收納盒 MVP

> 技職盃黑客松參賽作品 — 視覺辨識模組 + APP + Google Gemini，讓冰箱收納盒變得「看得懂食材」。

## 專案簡介

這個專案的核心想法很簡單：把一支手機放進冰箱收納盒裡充當視覺辨識模組，另一支手機當作使用者介面。按下掃描後，盒內手機自動拍照，Gemini 辨識食材，並追蹤每樣食材放了幾天、還剩幾天、現在該先用哪個。

**主要展示頁面：**

| 頁面                       | 路徑       | 原始碼                                           |
| -------------------------- | ---------- | ------------------------------------------------ |
| 使用者 Dashboard（B 手機） | `/control` | [templates/control.html](templates/control.html) |
| 盒內攝影端（A 手機）       | `/device`  | [templates/device.html](templates/device.html)   |

---

## 介面功能一覽

### `/control` — 使用者 Dashboard

按下掃描後，這個頁面會顯示 Gemini 分析完的結果：

**優先提醒卡**
快速看到哪些食材快過期（黃色 = 剩 1 天 / 紅色 = 已超過建議保存期），附上建議行動。

**食材庫存列表**
每項食材顯示：名稱、類別、信心度、外觀狀況、已存放天數、剩餘天數。新鮮度以三色圓形狀態球呈現（綠 / 黃 / 紅）。

**盒子概覽**
依食材類別分成彩色盒子（綠色盒 = 蔬菜、紅色盒 = 肉類 ...），顯示各盒狀態與容量條。

**食譜推薦**
根據目前庫存食材推薦可做的料理，優先推薦能消耗即將過期食材的食譜。

**採購提醒**
提示冰箱內已有哪些食材，避免重複購買或應先消耗後再補貨。

**食物浪費估算（ESG）**
估算若優先使用黃 / 紅食材可節省的金額（NT$）與減少的廚餘重量（kg）。

**掃描歷史（側邊欄）**
保留最近 20 筆掃描記錄，可點擊回顧歷史分析結果。

---

### `/device` — 盒內攝影端

放在收納盒內的手機開啟此頁面後：

1. 點擊「啟動相機」，開啟後鏡頭即時預覽
2. 自動每秒輪詢後端，收到掃描請求後立刻截圖並上傳
3. 若手機瀏覽器不支援相機 API（非 HTTPS 環境），提供手動選圖上傳作為 fallback

---

## 系統架構

```
B 手機（/control）                    A 手機（/device）
     │                                      │
     │ POST /scan-request                   │ GET /scan-status（每秒輪詢）
     │                                      │
     │           FastAPI 後端（main.py）     │
     │                  │                   │
     │           POST /upload-image ◄───────┘
     │                  │
     │           Google Gemini API（圖片辨識）
     │                  │
     │           庫存追蹤（data/current_inventory.json）
     │                  │
     └── GET /latest-result ◄── 顯示 Dashboard
```

---

## 使用技術

| 類別     | 技術                                                                           |
| -------- | ------------------------------------------------------------------------------ |
| 後端     | [FastAPI](https://fastapi.tiangolo.com/) + [Uvicorn](https://www.uvicorn.org/) |
| AI 辨識  | [Google Gemini 2.5 Flash](https://ai.google.dev/)                              |
| 前端     | 原生 HTML / CSS / JavaScript（無框架）                                         |
| 模板引擎 | [Jinja2](https://jinja.palletsprojects.com/)                                   |
| 資料儲存 | 本地 JSON 檔案（無資料庫）                                                     |

---

## 安裝與啟動

### 1. 複製專案

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/smart-fridge-box.git
cd smart-fridge-box
```

### 2. 建立虛擬環境並安裝套件

```bash
python3 -m venv .venv
source .venv/bin/activate      # macOS / Linux
# .venv\Scripts\activate       # Windows

pip install -r requirements.txt
```

### 3. 設定 Gemini API Key

```bash
cp .env.example .env
# 編輯 .env，填入你的 Gemini API Key
```

> 取得 API Key：[Google AI Studio](https://aistudio.google.com/apikey)
>
> 沒有 API Key 也能執行 — 系統會自動切換為 fallback 假資料模式，介面功能正常展示。

### 4. 啟動伺服器

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

---

## 使用方式

### 單機測試（同一台電腦）

| 頁面                | 網址                          |
| ------------------- | ----------------------------- |
| 使用者 Dashboard    | http://localhost:8000/control |
| 攝影端              | http://localhost:8000/device  |
| API 文件（Swagger） | http://localhost:8000/docs    |

### 雙手機展示（同一 Wi-Fi）

1. 找到伺服器的區網 IP（例如 `192.168.1.100`）
2. A 手機開啟 `http://192.168.1.100:8000/device`，放入收納盒，按啟動相機
3. B 手機開啟 `http://192.168.1.100:8000/control`，按掃描

> **注意**：手機瀏覽器要求 HTTPS 才能使用相機 API。若區網環境無法設 HTTPS，device 頁面提供手動上傳圖片作為 fallback。

---

## 專案結構

```
smart-fridge-box/
├── main.py                  # FastAPI 後端：API 路由、Gemini 整合、庫存追蹤邏輯
├── app.py                   # 早期 Streamlit 原型（已由 templates 取代）
├── requirements.txt
├── .env.example             # API Key 設定範本
├── templates/
│   ├── control.html         # ★ 使用者 Dashboard（主要展示頁面）
│   └── device.html          # ★ 盒內攝影端（主要展示頁面）
├── data/                    # 執行期資料（不納入版本控制）
│   ├── current_inventory.json
│   └── scan_history.json
└── uploads/                 # 使用者上傳的圖片（不納入版本控制）
```

---

## 環境變數

| 變數             | 必填 | 說明                                                                        |
| ---------------- | ---- | --------------------------------------------------------------------------- |
| `GEMINI_API_KEY` | 建議 | Google Gemini API Key。未設定時自動使用 fallback 假資料，不影響 Demo 展示。 |

---

## 食材分類規則

| 類別 | 盒子   | 預設保存天數 |
| ---- | ------ | ------------ |
| 蔬菜 | 綠色盒 | 5 天         |
| 肉類 | 紅色盒 | 3 天         |
| 魚類 | 灰色盒 | 2 天         |
| 蛋乳 | 白色盒 | 14 天        |
| 水果 | 黃色盒 | 7 天         |
| 熟食 | 橘色盒 | 3 天         |
| 飲品 | 白色盒 | 7 天         |
| 其他 | 透明盒 | 3 天         |

實際保存天數優先採用 Gemini 回傳的估計值；超出合理範圍（1–90 天）時回退至上表預設值。

---

## 注意事項

- 食材辨識與新鮮度判斷僅供參考，**不作為食品安全保證**，請依實際氣味、包裝效期與外觀自行判斷。
- 目前以「食材名稱 + 盒子名稱」作為庫存追蹤 ID，同名食材視為同一項。
- 伺服器重啟後記憶體狀態會清空，`data/` 下的 JSON 資料持續保留。

---

## 授權

[MIT License](LICENSE)

## 作者

**Mr.ChenTB** · 2026 技職盃黑客松
