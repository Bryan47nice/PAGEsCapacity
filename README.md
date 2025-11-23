# PAGEs Capacity System (PAGEs產能系統)

> **版本**: v1.2 (Live)  
> **定位**: 專為 PAGEs 元件庫團隊設計的敏捷產能管理與 Roadmap 規劃工具。

本系統結合看板管理 (Kanban)、產能計算與 Google Gemini AI 輔助，協助團隊追蹤開發點數 (Points)、分配 Sprint，並平衡「國內」與「海外」的雙軌產能需求。系統採用 **Local-First** 架構，支援離線操作與 Firebase 即時雲端同步。

---

## 1. 使用者角色與權限 (User Roles)

系統採用輕量級身分驗證，分為以下三種角色：

| 角色代碼 | 顯示名稱 | 權限描述 | 預設密碼 |
| :--- | :--- | :--- | :--- |
| **PAGES** | PAGEs PM (Admin) | **超級管理員**。<br>• 擁有所有權限 (新增 Sprint、設定產能、強制刪除/還原)。<br>• 可管理所有角色的密碼。<br>• **無限制拖曳**：在任何視圖下皆可調整工單順序。 | `0324` |
| **STRATEGY_DOMESTIC** | 策略 PM (國內) | **國內需求方**。<br>• 僅能在「國內視圖」下調整優先順序。<br>• 無法修改 Sprint 結構。<br>• 預設過濾為國內視圖。 | `1111` |
| **STRATEGY_OVERSEAS** | 策略 PM (海外) | **海外需求方**。<br>• 僅能在「海外視圖」下調整優先順序。<br>• 無法修改 Sprint 結構。<br>• 預設過濾為海外視圖。 | `2222` |

---

## 2. 核心功能規格

### 2.1 看板與 Sprint 管理
- **產能可視化**：每個 Sprint 獨立顯示「國內 (Amber)」與「海外 (Purple)」兩條產能進度。
- **超載警示**：當 `已用點數 > 設定產能` 時，進度條與文字變為紅色並閃爍。
- **收折功能**：Sprint Header 可收折，僅顯示垂直進度條以節省空間。
- **Backlog (待辦)**：未分配的工單存放區，支援針對 PAGEs PM 的「顯示封存」切換功能。

### 2.2 需求工單 (Ticket)
- **點數系統**：採用費波那契變體 `[0.5, 1, 2, 3, 5]`，定義如下：
  - `0.5`: 極小 (文案/設定微調)
  - `1`: 很小 (樣式/Icon)
  - `2`: 簡單 (單一邏輯/無副作用)
  - `3`: 小型 (標準新功能)
  - `5`: 中型 (複雜邏輯)
- **日期邏輯與風險偵測**：
  - **Projected (預期)**: `Sprint End Date + 7 days` (系統自動計算)。
  - **Expected (期望)**: 使用者手動輸入。
  - **風險**: 若 `Expected < Projected`，卡片顯示紅色驚嘆號 ⚠️。
- **狀態管理**:
  - `Active`: 正常顯示。
  - `Hidden`: 軟刪除 (封存)，可還原。
  - `Deleted`: 永久刪除 (僅 Admin 或清空 Sprint 封存時觸發)。

### 2.3 拖曳與排序規則 (Drag & Drop Logic)
這是系統互動最複雜的部分，分為桌面與手機版：

#### 排序優先級
1.  **有日期 (Date Locked)**：若設定了 `Expected Date`，強制依日期排序，**禁止手動拖曳**。
2.  **無日期 (Manual Sort)**：排在有日期工單之後，可手動拖曳調整順序。

#### 互動模式
*   **Desktop**: 
    *   使用 HTML5 Native Drag。
    *   僅在滑鼠 `Hover` 時啟用 `draggable="true"` (避免手機版誤觸原生鬼影)。
*   **Mobile**: 
    *   **長按觸發 (Long Press)**：長按 600ms 後震動。
    *   **無鬼影 (No Ghost)**：不使用瀏覽器原生拖曳，改為彈出 **「移動工單選單」**，直接點選目標 Sprint。

---

## 3. AI 智能輔助 (Gemini Integration)

使用 Google Gemini 2.5 Flash 模型增強 PM 工作效率。

- **文案潤飾 (Refine Text)**：
  - 針對 `Strategy Plan` (價值面) 與 `PAGEs Plan` (技術面) 提供潤飾。
  - AI 會根據欄位自動扮演 Product Strategy 或 Technical PM 角色。
- **覆盤分析 (Retro Analysis)**：
  - 輸入：預期計畫 + 實際結果。
  - 輸出：結構化評估 (`SUCCESS` / `FAILURE` / `NEUTRAL`)、差異分析與改進建議。

---

## 4. 數據埋點與追蹤計畫 (Analytics Plan)

本系統整合 **Firebase Analytics (GA4)** 進行使用者行為追蹤。由於資料量較小，重點在於「關鍵行為轉化」與「功能使用率」。

### 4.1 埋點事件清單 (Event Reference)

以下是目前已實作的 Event 及其參數定義：

| 事件名稱 (`event_name`) | 觸發時機 | 關鍵參數 (`params`) | 分析目的 |
| :--- | :--- | :--- | :--- |
| `login` | 使用者輸入密碼成功切換身分時 | `role` (登入的身分) | 追蹤各角色的活躍度 (DAU)。 |
| `ticket_create` | 新增一張工單並儲存時 | `role` (操作者)<br>`region` (工單區域) | 評估需求產生的頻率與來源分布。 |
| `ticket_drag_drop` | 將工單拖曳至**不同的 Sprint** 時 | `sprint_from`<br>`sprint_to`<br>`ticket_region` | 追蹤需求變更與排程調整的頻率 (Scope Creep)。 |
| `view_mode_change` | 切換上方篩選器 (Filter) 時 | `filter` (ALL/DOMESTIC/OVERSEAS) | 了解使用者習慣看全貌還是專注單一區域。 |
| `ai_refine_click` | 點擊「AI 潤飾」按鈕時 | `field` (strategy/pages/retro)<br>`ticket_region` | **關鍵指標**：AI 功能的滲透率與實用性。 |
| `ai_retro_analyze_click` | 點擊「執行 AI 覆盤分析」時 | `ticket_region` | 追蹤團隊是否落實 Sprint Retro 文化。 |
| `cloud_connect_attempt` | 在設定視窗點擊連線時 | `project_id` | 追蹤雲端同步功能的啟用狀況。 |
| `contact_support_click` | 在設定選單點擊聯絡客服時 | `role` | 監測系統是否造成使用者操作困難。 |
| `music_play` / `music_pause` | 播放或暫停背景音樂 | `track` (曲目名稱) | 趣味性指標，了解使用者偏好。 |

### 4.2 如何開始累積與查看數據？

#### 第一步：確認環境
由於 GA4 的標準報告有 **24~48 小時** 的延遲，剛開始使用時，您在儀表板上會看到「沒有資料」。

#### 第二步：使用 DebugView (即時驗證)
這是開發階段唯一能即時看到數據的方法：
1. 前往 [Firebase Console](https://console.firebase.google.com/)。
2. 進入您的專案 -> 左側選單 **Analytics (分析)** -> **DebugView**。
3. 在本系統中操作 (例如：登入、拖曳卡片)。
4. 您應該會在 DebugView 的時間軸上看到事件即時跳出。

#### 第三步：建立探索報表 (Explorations)
累積一週數據後，建議建立以下自訂報表：
1. **AI 使用率漏斗**：`ticket_create` (分母) vs `ai_refine_click` (分子)。
2. **產能調整熱圖**：以 `ticket_drag_drop` 事件分析哪些 Sprint 最常被插入或移出需求。

---

## 5. 資料儲存與同步 (Technical Architecture)

- **Level 1: LocalStorage (預設)**
  - 資料預設儲存在瀏覽器，即開即用。
  - 使用 `BroadcastChannel` 實現同一瀏覽器多分頁同步。
- **Level 2: Firebase Firestore (雲端)**
  - 使用者手動輸入 Config 連線。
  - 採用 `onSnapshot` 監聽模式，實現多裝置秒級同步。
  - **初始化邏輯**：連線時若雲端為空，**不會**自動上傳本地資料 (防止誤蓋)，需依賴正常操作寫入。

## 6. 安裝與開發

1. **安裝依賴**
   ```bash
   npm install
   ```
2. **設定環境變數**
   請確保環境變數 `API_KEY` (Gemini API) 已設定。
3. **啟動**
   ```bash
   npm start
   ```
