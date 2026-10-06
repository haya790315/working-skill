# Plane 平台層

工具前綴：`mcp__claude_ai_RAYVEN_Plane_OAuth__`（workspace 固定為 `rayven`）。
**沒有留言 API，也沒有票之間關聯的 API**，所以寫回只能改 `description_html`（整段覆蓋）。

## 找票

| 輸入 | 作法 |
| --- | --- |
| 票號 `TUMIKI-203` | `plane_workitem` `action: search`、`query: "TUMIKI-203"` → 取 `id`、`project_id` |
| 網址 | 從網址取出票號或 work item UUID；只有 UUID 時，用 `retrieve` 搭配網址中的 project UUID |
| 分支名稱 | 抓出 `<識別碼>-<數字>` 或 `<數字>`，再用 search 確認；比對不到就問使用者 |

專案的識別碼與描述（內含 GitHub repo、Outline、Slack 頻道）：`plane_project` `action: list`。

## 讀票

- 本體：`plane_workitem` `action: retrieve`、`project_id`、`work_item_id`。
- 父票：用本體的 `parent`（UUID）再 `retrieve`。
- 兄弟票：沒有依 parent 篩選的參數。改用 `action: list`、`per_page: 100`，結果會存成檔案，再用 jq 篩選：
  ```bash
  jq -r --arg p "<parent UUID>" '.result.results[] | select(.parent==$p)
    | "\(.sequence_id)\t\(.name)\t\(.state)"' <檔案>
  ```
  只看標題和狀態，需要內容時再個別 `retrieve`。
- 票內提到的其他票（`TUMIKI-199` 等）：用 search 找出來，只讀需要的部分。
- 狀態名稱：`plane_state` `action: list`（`state` 欄位是 UUID）。
- 讀 HTML 時先去掉標籤：`sed -E 's/<\/(p|li|h[1-6]|tr)>/\n/g; s/<[^>]+>//g'`。

## 寫回（流程 B）

1. **update 前一定先重新 `retrieve`**，拿最新的 `description_html`，避免覆蓋掉別人剛改的內容。
2. 找區塊的錨點：最後一個以 `対応内容` 開頭的 `<h3>`。
   - 找不到 → 把新區塊**接在原文最後**。
   - 找得到 → **從該 `<h3>` 到結尾**替換成新區塊。
   - 錨點用標題而不用 HTML 註解，因為 Plane 編輯器存檔時可能會把註解刪掉。
3. 格式要跟 Plane 編輯器一致：標題用 `<h3>`，條列用 `<ul><li><p>…</p></li></ul>`，段落用 `<p>`。
4. `plane_workitem` `action: update`、`data.description_html` = 原文 + 新區塊。

## 開新票（未対応的後續）

`plane_workitem` `action: create`、`project_id` 同原票、`data`：
- `name`：跟同專案的票命名風格一致（例如原票是 `【A-6】`，就用 `【A-6a】`）
- `parent`：原票的 `parent`（讓新票與原票成為兄弟）
- `description_html`：背景（為什麼會產生這項）＋要做什麼＋「源自 <原票票號>」

從回應取出 `sequence_id`，組成 `<識別碼>-<sequence_id>` 填回原票。
