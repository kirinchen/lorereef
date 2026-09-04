# lorereef(已歸檔)

> **本 repo 已於 2026-09-04 併入 [swellplot](https://github.com/kirinchen/swellplot)(私有),不再開發。**

**為什麼併**:寫故事需要設定,設定不需要寫故事 —— 依賴方向上由消費端(swellplot,劇本產生器,
已有完整實作、GUI 與 aura-stream 的 adapter 橋接)吸收供給端的模型;lorereef 只有規格、零程式,反過來併等於把能跑的東西搬進空殼。

**去哪找後續**:核心模型(四類卡片、帶型別關聯、帶時效區間事實)與 H01–H09 硬矛盾規則
已整章搬進 swellplot 的 `doc/SPEC.md` §19「故事設定(Bible)」,現行規格以那邊為準。
決策全文見 aura-stream 的 `doc/note/story-model-merge.md`。**兩者皆為私有 repo,連結對外會 404。**

**本 repo 保留的用途**:規格出處(provenance)。[`doc/SPEC.md`](doc/SPEC.md) 原封不動留著當歷史快照
(僅在開頭加了歸檔橫幅),要追「這條規則當初怎麼想的」時回來查。**不是刪除,是凍結**。

---

## 原始定位(存查)

故事設定聖經 + 一致性稽核:定義角色 / 物品 / 地點 / 事件四類卡片的中繼資料格式,
用可判定的規則揪出設定前後矛盾(而不是叫 LLM 「感覺一下」)。

⚠️ 從頭到尾**只有規格,沒有實作** —— README 裡曾提過的 CLI、IDE、引擎都未曾存在。
