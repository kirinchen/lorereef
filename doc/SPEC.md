# lorereef SPEC(v0 草案)

> 真相源:本檔定義卡片資料模型、關聯型別、時效區間與硬矛盾規則。`schemas/*.json` 是本檔的機器版,不一致以本檔為準並修 schema。
> 狀態:📋 草案,2026-08-30。尚無實作。

---

## 0. 一句話

**把世界觀寫成可稽核的資料**:卡片 + 帶型別的關聯 + 帶時效區間的事實 → 規則引擎確定性地抓硬矛盾,LLM 補軟矛盾。

## 1. 範圍

### 做(v0)
- 四類卡片:`char` 人物 / `item` 物件 / `place` 地點 / `event` 事件
- 帶型別關聯、帶時效區間事實的資料模型 + JSON Schema
- `validate`(schema、參照完整性、id 唯一)
- `audit`(硬矛盾規則引擎 v0 清單,§5)
- `audit --soft`(LLM 軟矛盾,`claude -p`,建議性質)
- `timeline <id>`(除錯視圖,純文字)

### 不做(v0)
- UI、多人、帳號、雲端、寫作輔助
- 「InRay 垂直應用 vs 獨立產品」的決定(§9 待拍板)

## 2. 核心模型

```
Card(卡片)        一個實體:人物 / 物件 / 地點 / 事件
├── attrs           靜態屬性(不隨時間變的:名字、種族、類別)
├── facts[]         時效性事實:某個屬性 / 狀態在 [from, to) 區間成立
└── relations[]     帶型別的關聯:本卡 → 另一張卡,可帶時效區間

Timeline(時間軸)   故事內時間,單位由 world.yaml 定義(章節 / 日 / 年 / 自訂紀元)
```

三個關鍵設計:
1. **事實帶區間,不是快照**:「X 活著」寫成 `alive: [t0, t_death)`,不是某頁寫活某頁寫死。矛盾檢查才有東西可算。
2. **關聯帶型別 + 方向 + 區間**:`father_of` / `mentor_of` / `owns` / `located_in` / `member_of`;型別決定哪些規則適用(如 `located_in` 才觸發地點衝突)。
3. **事件是時間軸的錨點**:事實 / 關聯的區間端點可以寫**事件 id** 而不是絕對時間,事件之間用 `before` / `after` 排偏序;稽核時先解成拓撲序再比。

## 3. 卡片格式

一張卡一個 markdown 檔,YAML frontmatter 是結構化資料(稽核只讀 frontmatter),body 是給人 / 給 LLM 看的自由描述。

```markdown
---
id: char-ryo               # 必填,全域唯一,^(char|item|place|event)-[a-z0-9-]+$
type: char                 # 必填,四選一
name: 阿龍
aliases: [龍哥]            # 選填
tags: [四人幫]             # 選填,自由標籤
attrs:                     # 選填,靜態屬性(自由 key,schema 只限型別為純量 / 純量陣列)
  species: human
  birthplace: place-taipei-ruins
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

阿龍,四人幫的老大。粗人,但對小孩極溫柔……(自由描述,軟矛盾稽核會讀這裡)
```

### 3.1 事件卡

```markdown
---
id: event-ryo-death
type: event
name: 阿龍之死
at: ch-40                  # 選填,絕對時間(時間軸單位見 world.yaml)
after: [event-final-battle-start]   # 選填,偏序:本事件在這些事件之後
before: []                 # 選填
participants: [char-ryo, char-ai]   # 選填
location: place-tower-top  # 選填
---
```

### 3.2 `world.yaml`(每個世界一份)

```yaml
name: Soulmon
timeline:
  unit: chapter            # chapter | day | year | custom
  format: "ch-{n}"         # 絕對時間寫法;稽核時解成整數比大小
relation_types:            # 型別註冊表;未註冊的型別 validate 不過
  father_of:   { inverse: child_of, symmetric: false }
  mentor_of:   { inverse: student_of, symmetric: false }
  owns:        { exclusive: true }        # 同一物件同一時間只能有一個 owner
  located_in:  { exclusive: true }        # 同一人同一時間只能在一個地點
  member_of:   {}
  allied_with: { symmetric: true }
  enemy_of:    { symmetric: true }
```

## 4. `validate`(參照完整性)

| 檢查 | 錯誤要說 |
|---|---|
| frontmatter 符合 `schemas/card.schema.json` | 哪張卡、哪個欄位 |
| `id` 全域唯一、prefix 與 `type` 一致 | 撞名的兩個檔 |
| `relations[].target`、`facts[].from/to`、`event.after/before` 指向存在的卡 | 哪張卡引用了不存在的 id |
| `relations[].type` 在 `world.yaml` 註冊 | 型別是什麼、已註冊有哪些 |
| 時間軸可解析:絕對時間符合 format,事件偏序無環 | 環在哪幾個事件之間 |
| 區間 `from < to`(解析後) | 哪個事實、解出來的值 |

## 5. `audit`:硬矛盾規則(v0 清單)

每條規則:輸入 = 全部卡片解析後的時間軸,輸出 = 矛盾清單,**每條矛盾都附:規則 id、涉及卡片 id、涉及事實 / 關聯、解析後的時間點**。

| 規則 id | 名稱 | 邏輯 |
|---|---|---|
| `H01` | 死後出場 | `alive` 為 false 的區間內,該角色出現在任何 `event.participants` 或作為 `located_in` 主體 |
| `H02` | 生前不存在 | 角色在 `alive.from` 之前出現在事件 / 關聯中 |
| `H03` | 地點衝突 | `exclusive` 型別(如 `located_in`)在同一時間點有兩個不同 target |
| `H04` | 擁有權衝突 | `owns` 為 exclusive:同一物件同一時間兩個 owner |
| `H05` | 能力先用後學 | 事件標記 `uses: [skill.x]`(v0 選填欄位)但該角色 `skill.x` 事實區間尚未開始 |
| `H06` | 事實區間重疊 | 同一卡同一 `key` 的兩個事實區間重疊且 value 不同 |
| `H07` | 關聯反向不一致 | `father_of` 有寫、對方 `child_of` 寫成別人 |
| `H08` | 事件偏序 vs 絕對時間衝突 | `after: [e1]` 但 `at` 早於 e1 的 `at` |
| `H09` | 事件地點 vs 參與者位置 | 事件 `location` 與參與者當時 `located_in` 不一致(參與者有 located_in 資料時才檢) |

原則:**寧可漏,不可誤報**。規則只在資料**明確**時觸發(缺資料 = 不檢,並在報告的「盲區」段列出「哪些角色沒有 alive 事實」之類的提示)。

## 6. `audit --soft`:軟矛盾(LLM)

- 觸發條件:`audit` 硬矛盾為 0 才跑(硬矛盾沒清乾淨,軟稽核是噪音)
- 流程:對每張 `char` 卡,組 context = 該卡 frontmatter + body + 直接關聯卡的 name/body 摘要 + 該角色參與的事件(依時間序)→ `claude -p` 問固定 prompt(「依時間序,列出動機 / 性格 / 語氣前後不一致之處,每條附涉及事件 id」)→ 解析成清單
- 輸出標 `soft`,**不影響 exit code**;人裁決
- 費用 / 頻率:手動觸發,不進 pre-commit

## 7. 報告格式

```
lorereef audit
✗ 3 hard contradictions

H01 死後出場   char-ryo
  alive: false since event-ryo-death (ch-40)
  appears in event-epilogue-reunion (ch-42) as participant
  → cards/char/ryo.md:facts[0], cards/event/epilogue-reunion.md:participants

H03 地點衝突   char-ai @ ch-13
  located_in place-underground-city [ch-12, ch-15)
  located_in place-surface-camp     [ch-13, ch-14)
  → cards/char/ai.md:relations[1], relations[2]

…
blind spots: 7 chars without `alive` fact (H01/H02 skipped): char-x, char-y, …
```

`--json` 給機器(agent 修卡用)。

## 8. 實作草圖

- Python uv inline script(PEP 723),依賴 `pyyaml`、`jsonschema`
- 解析:全部卡 → 記憶體 graph;事件偏序 + 絕對時間 → 統一整數時間(拓撲排序,絕對時間優先、偏序補洞)
- 規則 = 一個檔一條(`rules/h01_dead_appears.py` …),每條純函式 `(world) -> list[Contradiction]`,可單測
- pre-commit:`validate` 必過;`audit` 選配(設定集開發中會常有暫時矛盾)
- 第一批測試資料:把 coral `Soulmon設定集` 手拆 ~30 張卡(這是 dogfood 步驟,開卡做,不進本 repo scaffold)

## 9. 待拍板

- [ ] **InRay 垂直應用 vs 獨立產品** —— v0 用 file-based,兩邊都不排除;等 dogfood 有結果再決定
- [ ] 時間軸單位:Soulmon 用章節還是故事內年份?(影響 `world.yaml.timeline.unit` 與 H08 的解析)
- [ ] `facts[].key` 是否要註冊表(像 relation_types)—— 草案:v0 自由 key,H06 只比同 key;若 dogfood 發現同義 key 亂寫再收
- [ ] 卡片放本 repo 還是各世界一個 repo(`lorereef` = 工具,`soulmon-lore` = 資料)—— 草案:工具與資料分離,本 repo 只放 `examples/` 極小範例世界

## 10. 已知風險

- **偏序 + 絕對時間混用**的解析是最容易出 bug 的地方;要有「無法排序」的明確錯誤,不能默默猜
- 規則越多誤報越多;每條規則上線前要在真實設定集上跑一次,誤報率高的降級為 warning
- 軟稽核的 prompt 漂移:固定 prompt 進 repo、版本化;輸出格式用 JSON schema 約束
