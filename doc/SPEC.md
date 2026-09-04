> ⚠️ **已歸檔(2026-09-04)** —— 本檔已併入另一個私有專案的 SPEC「故事設定」章節。
> 以下是**合併前的歷史快照**,一字未改,保留作為規格出處(provenance)。
> **現行規格以該專案為準**,不要以本檔為真相源。

---

# lorereef SPEC(v0 草案)

> 真相源:本檔定義 (a) 設定集的中繼資料格式 (b) 稽核引擎的規則 (c) IDE 的介面範圍。`schemas/*.json` 是 (a) 的機器版,不一致以本檔為準並修 schema。
> 狀態:📋 草案,2026-08-30。尚無實作。

---

## 0. 一句話

**故事設定的 IDE**:打開使用者自己的設定集資料夾,以卡片 / 心智圖 / 時間線呈現,並用規則引擎 + LLM 稽核一致性。lorereef 只定義格式、提供引擎與介面,**不保存資料**。

## 1. 範圍

### 做(v0)
- **中繼資料格式**:四類卡片(`char` / `item` / `place` / `event`)、帶型別關聯、帶時效區間事實、`world.yaml`
- **引擎**:`validate`(schema、參照完整性)、`audit`(硬矛盾規則 H01~H09)、`audit --soft`(LLM 軟矛盾)
- **IDE**:card grid、mind-map、timeline、audit panel 四個 view;從 IDE 召喚 agent(diff 核准制)
- **CLI**:與 IDE 共用引擎,可進 pre-commit

### 不做(v0)
- 保存 / 同步 / 託管使用者資料;帳號;多人即時協作
- 寫作輔助、匯出排版
- 行動版

## 2. 分層

| 層 | 誰擁有 | 內容 |
|---|---|---|
| 設定集(world) | **使用者**(自己的資料夾 / repo) | `world.yaml` + `cards/**/*.md` |
| 中繼資料格式 | lorereef 定義 | 本檔 §3~§4 + `schemas/` |
| 引擎 | lorereef | 解析 → 時間軸 → 規則 → 報告;`claude -p` 軟稽核 |
| IDE / CLI | lorereef | 兩個 client 共用引擎 |

本 repo 只放 `examples/tiny-world/` 一個虛構範例世界(數十張卡),供測試與 demo;**不放任何真實作品的設定**。

## 3. 核心模型

```
Card(卡片)        一個實體:人物 / 物件 / 地點 / 事件
├── attrs           靜態屬性(不隨時間變的:名字、種族、類別)
├── facts[]         時效性事實:某個屬性 / 狀態在 [from, to) 區間成立
└── relations[]     帶型別的關聯:本卡 → 另一張卡,可帶時效區間

Timeline(時間軸)   故事內時間;單位由 world.yaml 定義(章節 / 日 / 年 / 自訂紀元)
```

三個關鍵設計:
1. **事實帶區間,不是快照**:「X 活著」寫成 `alive: [t0, t_death)`。矛盾檢查才有東西可算。
2. **關聯帶型別 + 方向 + 區間**:型別決定哪些規則適用(如 `located_in` 才觸發地點衝突)。
3. **事件是時間軸的錨點**:區間端點可以寫**事件 id** 而不是絕對時間;事件之間用 `before` / `after` 排偏序,稽核時先解成拓撲序再比。

## 4. 中繼資料格式

### 4.1 卡片

一張卡一個 markdown 檔。YAML frontmatter 是結構化資料(引擎只讀 frontmatter),body 是自由描述(給人、給軟稽核的 LLM 看)。

```markdown
---
id: char-ryo               # 必填,全域唯一,^(char|item|place|event)-[a-z0-9-]+$
type: char                 # 必填,四選一
name: 阿龍
aliases: [龍哥]            # 選填
tags: [四人幫]             # 選填,自由標籤
attrs:                     # 選填,靜態屬性(自由 key;值限純量 / 純量陣列)
  species: human
  birthplace: place-ruins
facts:                     # 選填,時效性事實
  - key: alive
    value: true
    from: event-story-start
    to: event-ryo-death     # 省略 = 到永遠;from 省略 = 從時間軸起點
  - key: skill.soulbind
    value: 3
    from: event-ryo-awakens
relations:                 # 選填,帶型別關聯
  - type: mentor_of
    target: char-ai
    from: event-meet-ai
  - type: located_in
    target: place-underground-city
    from: ch-12
    to: ch-15
  - type: owns
    target: item-black-blade
    from: event-ryo-awakens
---

阿龍,四人幫的老大。粗人,但對小孩極溫柔……
```

### 4.2 事件卡

```markdown
---
id: event-ryo-death
type: event
name: 阿龍之死
at: ch-40                  # 選填,絕對時間(單位見 world.yaml)
after: [event-final-battle-start]   # 選填,偏序
before: []                 # 選填
participants: [char-ryo, char-ai]   # 選填
location: place-tower-top  # 選填
uses: [skill.soulbind]     # 選填,本事件用到的能力(觸發 H05)
---
```

### 4.3 `world.yaml`

```yaml
name: tiny-world
timeline:
  unit: chapter            # chapter | day | year | custom
  format: "ch-{n}"         # 絕對時間寫法;解成整數比大小
relation_types:            # 型別註冊表;未註冊的型別 validate 不過
  father_of:   { inverse: child_of, symmetric: false }
  mentor_of:   { inverse: student_of, symmetric: false }
  owns:        { exclusive: true }        # 同一物件同一時間只能有一個 owner
  located_in:  { exclusive: true }        # 同一主體同一時間只能在一個地點
  member_of:   {}
  allied_with: { symmetric: true }
  enemy_of:    { symmetric: true }
```

### 4.4 目錄慣例(使用者資料夾)

```
my-world/
├── world.yaml
└── cards/
    ├── char/*.md
    ├── item/*.md
    ├── place/*.md
    └── event/*.md
```

子目錄只是慣例,引擎以 frontmatter `type` 為準,遞迴掃 `cards/`。

## 5. `validate`

| 檢查 | 錯誤要說 |
|---|---|
| frontmatter 符合 `schemas/card.schema.json` | 哪張卡、哪個欄位 |
| `id` 全域唯一、prefix 與 `type` 一致 | 撞名的兩個檔 |
| `relations[].target`、`facts[].from/to`、`event.after/before/participants/location` 指向存在的卡 | 哪張卡引用了不存在的 id |
| `relations[].type` 在 `world.yaml` 註冊 | 型別是什麼、已註冊有哪些 |
| 時間軸可解析:絕對時間符合 format,事件偏序無環 | 環在哪幾個事件之間 |
| 區間 `from < to`(解析後) | 哪個事實、解出來的值 |

## 6. `audit`:硬矛盾規則(v0)

每條矛盾附:規則 id、涉及卡片 id、涉及事實 / 關聯、解析後的時間點、檔案位置。

| 規則 | 名稱 | 邏輯 |
|---|---|---|
| `H01` | 死後出場 | `alive=false` 的區間內,角色出現在 `event.participants` 或作為 `located_in` 主體 |
| `H02` | 生前不存在 | 角色在 `alive.from` 之前出現在事件 / 關聯中 |
| `H03` | 地點衝突 | `exclusive` 型別(如 `located_in`)在同一時間點有兩個不同 target |
| `H04` | 擁有權衝突 | `owns` exclusive:同一物件同一時間兩個 owner |
| `H05` | 能力先用後學 | `event.uses` 含 `skill.x`,但參與者的 `skill.x` 事實區間尚未開始 |
| `H06` | 事實區間重疊 | 同一卡同一 `key` 的兩個事實區間重疊且 value 不同 |
| `H07` | 關聯反向不一致 | `father_of` 有寫、對方的 `child_of` 指向別人 |
| `H08` | 偏序 vs 絕對時間衝突 | `after: [e1]` 但 `at` 早於 e1 的 `at` |
| `H09` | 事件地點 vs 參與者位置 | 事件 `location` 與參與者當時 `located_in` 不一致(有資料時才檢) |

原則:**寧可漏,不可誤報**。缺資料 = 不檢,並在報告「盲區」段列出(例:哪些角色沒有 `alive` 事實)。

## 7. `audit --soft`:軟矛盾(LLM)

- 觸發條件:硬矛盾為 0 才跑(硬矛盾沒清乾淨,軟稽核是噪音)
- 流程:對每張 `char` 卡,組 context = 該卡 frontmatter + body + 直接關聯卡摘要 + 該角色參與的事件(依時間序)→ `claude -p` 固定 prompt(「依時間序列出動機 / 性格 / 語氣前後不一致之處,每條附涉及事件 id」)→ 解析成清單
- 輸出標 `soft`,**不影響 exit code**;人裁決
- 手動觸發,不進 pre-commit;prompt 進 repo 版本化,輸出用 JSON schema 約束

## 8. IDE

### 8.1 四個 view

| view | 內容 | 互動 |
|---|---|---|
| **Card grid** | 卡片牆;依 type / tag / attr 篩選排序 | 點開 = 編輯器(frontmatter 表單 + body markdown);新增 / 刪除卡 |
| **Mind-map** | 節點 = 卡,邊 = 帶型別關聯(顏色 = 型別) | 拖拉、縮放;**時間滑桿**只顯示該時刻成立的關聯;拉線 = 新增關聯(跳出型別 / 區間表單) |
| **Timeline** | 橫軸 = 時間軸;事件為錨點;每張卡的事實 / 關聯區間為橫條 | 矛盾直接標在對應區間上;拖橫條端點 = 改區間 |
| **Audit panel** | 硬矛盾清單、軟矛盾建議、盲區提示 | 點一條 → 同時打開涉及的卡並高亮欄位;存檔即重跑 validate + audit |

### 8.2 Agent 召喚

- IDE 內一個輸入框,把使用者指令 + 當前 view 的 context(選中的卡 id、篩選條件)組成 prompt,shell out `claude -p`,工作目錄 = 使用者的 world 資料夾
- agent 對 markdown 的改動以 **diff** 呈現,**人核准才寫入**;核准後自動跑 validate + audit
- 不在 IDE 進程內整合任何 LLM SDK

### 8.3 檔案為真相源

- IDE 讀寫的就是使用者資料夾裡的 markdown;外部(編輯器、agent、git)改了檔,IDE 監看 fs 事件即時重載
- IDE 不存任何 world 資料在自己的設定裡;只存「最近開過的資料夾」與 view 偏好

## 9. CLI

```
lorereef open     <world-dir>              # 開 IDE
lorereef validate <world-dir>              # exit 0/1
lorereef audit    <world-dir> [--soft] [--json]
lorereef timeline <world-dir> <card-id>    # 純文字時間線(除錯)
```

## 10. 實作草圖(初判,待拍板)

| 層 | 草案 |
|---|---|
| 引擎 | **TypeScript** package(`packages/core`):frontmatter 解析、schema(JSON Schema)、時間軸解析、規則(一檔一規則,純函式,可單測)。TS 是因為 IDE 與 CLI 要共用同一顆引擎 |
| IDE | **Tauri 2 + React + TypeScript**(`packages/app`):桌面 app,fs 直讀使用者資料夾;mind-map 用現成 graph 套件;timeline 自繪 |
| CLI | `packages/cli`:薄殼包引擎 |
| LLM | 一律 subprocess `claude -p`;不在 app 內整合 SDK |
| 範例 | `examples/tiny-world/`:虛構世界,故意埋幾條矛盾當測試樣本 |

## 11. 待拍板

- [ ] IDE 殼:Tauri 桌面 vs 本機 web server + 瀏覽器(草案 Tauri:fs 直讀、免開 port)
- [ ] 時間軸單位是否允許一個 world 混用(章節 + 故事內年份)—— 草案:v0 單一單位
- [ ] `facts[].key` 是否要註冊表(像 `relation_types`)—— 草案:v0 自由 key,H06 只比同 key
- [ ] mind-map 套件選型(React Flow / Cytoscape / 自繪)

## 12. 已知風險

- **偏序 + 絕對時間混用**的解析最容易出 bug;要有「無法排序」的明確錯誤,不能默默猜
- 規則越多誤報越多;每條規則上線前在範例世界 + 真實設定集各跑一次,誤報率高的降級為 warning
- IDE 與外部編輯器同時改同一檔的衝突:v0 採「fs 事件優先、IDE 未存的變更提示重載」,不做合併
