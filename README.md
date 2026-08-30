# lorereef

> 故事設定聖經(story bible)+ **一致性稽核**。
> 人物 / 物件 / 地點是卡片;關聯帶型別、事實帶時效區間;規則引擎抓硬矛盾、LLM 抓軟矛盾。
> 讓長篇世界觀不自打嘴巴、時間線不錯亂 —— 而且是**能被程式證明**的那種不錯亂。

**狀態:📋 Spec 階段**(只有規格,尚無實作)。規格見 [`doc/SPEC.md`](doc/SPEC.md)。

---

## 為什麼

AI 讓寫小說變便宜,長篇 / 多線 / 多作者(含 AI 協作)的設定會爆量。
爆量之後最先壞掉的不是文筆,是**一致性**:死掉的人又出場、A 在學會 X 之前就用了 X、同一天出現在兩個城市。

既有工具(Novelcrafter / Sudowrite Story Bible / Campfire / World Anvil / Aeon Timeline)都在做「幫你寫、幫你查」,
**沒有一個真的在做一致性稽核**。lorereef 只做這一件事,而且分兩層做:

| 矛盾類型 | 例子 | 誰抓 | 可靠度 |
|---|---|---|---|
| **硬矛盾** | 死了又出場、時間線倒錯、地點衝突、能力先用後學 | **規則引擎**(確定性) | 抓到就是真的,不會漏 |
| **軟矛盾** | 動機不一致、語氣走鐘、性格前後不符 | **LLM**(`claude -p`) | 建議性質,人裁決 |

硬矛盾能確定性地抓,是因為資料模型逼你把兩件事寫清楚:**關聯有型別**(父子 / 師徒 / 擁有 / 位於)、**事實有時效區間**(t1~t2 存活、t3 之後才會 X)。
UI 誰都做得出來;**這個資料模型 + 混合稽核才是本體**。

## 前提修正(對自己誠實)

- 這不是「掌握故事製造核心就控制流量」—— 流量在作者手上,這是賣鏟子的 SaaS 生意。好生意,但別排錯優先序。
- 設定一致是**衛生條件**,不是勝負手。它讓作者不丟臉,不讓作者爆紅。

## 定位(一句話)

**先做給自己用**:第一個使用者是 Kirin 自己的 `Soulmon設定集` 與末日園丁世界。
連自己的設定集都查不出矛盾,這產品的核心價值就不成立;查得出來,就有了 demo。

## 形狀草圖(目標,尚未實作)

```
lorereef/
├── cards/            # 一張卡一個 .md(YAML frontmatter = 屬性,body = 自由描述)
│   ├── char/…        # 人物
│   ├── item/…        # 物件
│   ├── place/…       # 地點
│   └── event/…       # 事件(時間軸的錨點)
├── schemas/          # 卡片 / 關聯 / 事實的 JSON Schema(pre-commit 守門)
└── tools/
    ├── validate.py   # schema + 參照完整性
    ├── audit.py      # 硬矛盾規則引擎 → 矛盾報告
    └── soft_audit.py # 軟矛盾:組 context → claude -p → 建議清單
```

```bash
lorereef validate            # 卡片 schema、FK、id 唯一
lorereef audit               # 硬矛盾:確定性,exit 1 = 有矛盾
lorereef audit --soft        # 追加 LLM 軟矛盾建議
lorereef timeline <char-id>  # 印一個角色的時間線(除錯用)
```

## 設計原則

1. **Deterministic-first** —— 能用規則抓的矛盾不問 LLM;LLM 只碰規則抓不到的。
2. **卡片進 git、file-based** —— markdown + frontmatter,無 DB;diff 就是設定的版本史(沿用 otter / seamount 慣例)。
3. **schema 守門** —— 關聯型別、時效區間是**必填結構**,不是自由文字;寫不清楚就 validate 不過。
4. **稽核可解釋** —— 每條矛盾要指出哪兩張卡、哪兩個事實、哪條規則;不出「AI 覺得怪怪的」。
5. **UI 最後做** —— grid / mind-map 是 view,等資料模型和稽核在真實設定集上證明有用再做。

## 非目標(v0)

- 不做寫作輔助(續寫、潤稿)
- 不做 UI(grid / mind-map)
- 不做多人協作 / 帳號 / 雲端
- 不決定「InRay 垂直應用 vs 獨立產品」—— 列為待拍板,v0 用 file-based 方式兩邊都不排除

## 關聯

- 起源:coral `doc/chat/story-bible-card-tool.md`(2026-08-08)
- 第一批 dogfood 資料:coral `idea/Soulmon故事小說/Soulmon設定集.md`、`idea/末日溫馨怪物園丁.md`
- 慣例血緣:[otter](https://github.com/kirinchen/otter) / [seamount](https://github.com/kirinchen/seamount) 的 file-based entity + schema 守門
- 調度 / SA:[kelp](https://github.com/kirinchen/kelp)

## License

MIT
