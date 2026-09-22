# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概觀

單檔案的 DUPR 匹克球（Pickleball）8 人制記分板。整個應用（HTML、CSS、JavaScript）都在 `index.html` 裡，沒有建置流程、沒有依賴、沒有測試框架。

## 開發與執行

沒有 build / lint / test 指令。直接在瀏覽器開啟檔案即可：

```powershell
start index.html
```

如需驗證 `localStorage`（`file://` 下各檔案共用同一個 origin，行為可能與部署環境不同），起一個本機 server：

```powershell
python -m http.server 8000   # 然後開 http://localhost:8000
```

部署方式是把 `index.html` 推上 `main`（GitHub Pages / 靜態託管）。

## 架構

**渲染模型**：四個 panel（`#matches`、`#ranking`、`#players`、`#archives`）各由一個 `renderXxx()` 函式用字串模板整批寫入 `innerHTML`，沒有 diff、沒有框架。唯一的例外是 `updateWinner()`——記分時的局部更新（見下方「已知取捨」）。事件全靠 inline `onclick` / `oninput` / `onchange` 呼叫全域函式，所以**所有被 HTML 引用的函式都必須留在全域作用域**。隱藏的 `#importFile` 與 `#toast` 刻意放在 `.container` 之外，避免被任何 panel 的重繪吃掉。

**狀態**：四個模組層級變數即全部狀態。

| 變數 | 內容 | localStorage key |
|---|---|---|
| `players` | 代號 `A`–`H` → 選手名稱 | `dupr_players` |
| `scores` | `` `${matchIdx}_${1\|2}` `` → 分數字串 | `dupr_scores` |
| `archives` | 存檔陣列（新的在前），每筆 `{ id, name, date, players, scores }` | `dupr_archives` |
| `matches` | 固定賽程表（硬編碼常數，不持久化） | — |

分數以**字串**存放，空字串／`undefined` 代表「未輸入」；`calcStats()` 會跳過任一方未填的場次。要新增持久化狀態就照 `saveScores()` / `savePlayers()` / `saveArchives()` 的模式（`try/catch` 包起來，失敗時靜默忽略）。

**存檔與分享**（「紀錄」分頁 + 排名分頁底部的分享列）：

- `saveArchive()` 把當下的 `players` + `scores` 快照成一筆存檔；`loadArchive()` / `deleteArchive()` 負責還原與刪除。存檔只是快照，載入會整份覆蓋目前狀態。
- `exportBackup()` / `importBackup()` 走 JSON 檔（`{ app, version, exportedAt, players, scores, archives }`）。**匯入的資料是外部輸入**，一律經過 `normalizeArchives()` 與 `sanitizeScores()`：分數只收 `` `${n}_${1|2}` `` → 兩位數以內的數字字串，存檔 `id` 只留 `[A-Za-z0-9_-]`（因為 id 會被寫進 inline `onclick`）。新增欄位時務必一併擴充這兩個函式。
- `copyRanking()` 輸出純文字（`navigator.clipboard`，失敗時退回 `execCommand`）；`downloadRankingImage()` 用 canvas 手繪排名表輸出 PNG，**刻意寫死淺色配色**，讓圖片分享出去在任何裝置都好讀，不要改成讀 CSS 變數。
- 兩者的並列名次都走 `rankedStats()`，與 `renderRanking()` 用同一套 `${wins}_${diff}_${pf}` 判定；改排名規則時三處要一起改。

**賽程表不可隨意改動**：`matches` 的 14 場是一組平衡輪轉（social round-robin）——每位選手出賽 7 場，且與其他 7 人各搭配一次為隊友。每列格式為 `[隊1員1, 隊1員2, 隊2員1, 隊2員2]`。若要修改場次組合，必須同時維持這個不變條件，否則排名會失去公平性。

**排名規則**（`calcStats()` 的排序 + `renderRanking()` 的並列處理）：勝場 → 得失分差 → 總得分。個人的得分／失分是該選手所有場次中「所在隊伍得分」與「對手隊伍得分」的加總（雙打成績雙方隊員共享）。`renderRanking()` 用 `${wins}_${diff}_${pf}` 當 key 判斷並列，三項全同才給同名次。排名表只在切到「排名」tab 時重算。

**主題**：CSS 變數定義在 `:root`，深色模式由 `prefers-color-scheme` 自動套用，另有 `:root[data-theme="dark"|"light"]` 可覆寫（目前沒有 UI 切換器，變數已備好）。新增顏色請一律走 CSS 變數，並同時補上深色值。

## 已知取捨

- `s.code === 'E'` 被寫死成「我」的高亮列，出現在 `renderRanking()`（`tr.me`）與 `downloadRankingImage()`（淺綠底色）兩處。這是 repo 擁有者的固定代號，改動選手對應時要一併留意。
- **輸入分數時不可重繪整份賽程**。`updateScore()` 刻意只呼叫 `updateWinner(matchIdx)`，透過 `data-match` / `data-team` 定位並 toggle `.winner` class。早期版本在 `oninput` 裡呼叫 `renderMatches()`，導致輸入框被換掉、打第一個字就失去焦點（兩位數分數無法輸入）。之後若要在記分時反映其他狀態，也請走同樣的局部更新路徑。
- `showPanel()` 依賴隱含的全域 `event` 物件取得被點擊的 tab。在 Chrome / Safari 可用，Firefox 嚴格模式下會失效。
- 自從加入 JSON 匯入後，資料不再只來自本機，所有插進 `innerHTML` 的文字（選手名稱、存檔名稱）**一律要包 `esc()`**。新增任何把資料寫進模板字串的地方，記得比照辦理。
- `saveArchive()` / `loadArchive()` / `deleteArchive()` / `importBackup()` 用的是原生 `confirm()`。行為上沒問題，但會阻斷瀏覽器自動化——要用工具實測這些流程時，得先把 `window.confirm` 換掉。

## 語言

UI 文案、註解、commit message 一律使用繁體中文；程式碼識別字用英文。
