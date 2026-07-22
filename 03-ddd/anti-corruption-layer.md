# 防腐層：與遺留系統共處

## 前言

新訂單系統上線三個月，code review 時有人搜了一下：`"03".equals(prodStsCd)` 出現在十七個地方。舊 ERP 的商品狀態碼——`'01'` 在售、`'03'` 停售、`'07'` 沒人記得是什麼——已經滲進了新系統的業務規則、測試程式、甚至前端顯示邏輯。新模型還沒長大，就先學會了遺留系統的方言。

前兩篇建好了邊界與模型；但多數系統不是活在真空裡，而是活在一個有二十年 ERP、三家外部廠商 API 的世界。防腐層（Anticorruption Layer, ACL）是 Evans 在 Context Map 模式中給出的自保手段：**在你的模型與不可控的上游模型之間，築一道翻譯的牆**。

## 技術背景

### 先破除誤解：ACL 不是「又一層物件轉換」

「這不就是 adapter 加 DTO 轉換嗎？每個專案都有，何必取個新名字。」——差別在翻譯的**層次**：

- 一般的轉換層翻譯**格式**：欄位改名、型別對齊、日期格式化。上游的概念原封不動地流進來，只是換了衣服。
- ACL 翻譯**模型與語意**：上游的概念在牆上被攔下、拆解，重組成**你的上下文自己的概念**。`PROD_STS_CD = '03'` 進來，出去的是 `ProductAvailability.DISCONTINUED`——狀態碼、它的歷史包袱、它的「'07' 沒人知道」全部留在牆外。

判斷你寫的是哪一種很簡單：**看轉換後的型別屬於誰的語言**。轉出來還是 `ErpProductDto`，那是格式轉換；轉出來是你統一語言裡的詞，才是防腐。

### 結構：三個零件

Evans 給的 ACL 構造是三種熟面孔的組合——facade（把上游混亂的介面收攏成少數幾個入口）、adapter（技術協定的對接）、translator（模型與語意的翻譯本體）。放進〈[六角架構](../02-architecture-styles/hexagonal-and-clean-architecture.md)〉的地圖看最清楚：**ACL 就是一種被驅動端 adapter，只是它的職責從「技術對接」加重為「語意翻譯」**——port 依然由你的領域定義（「我需要知道商品能不能賣」），ACL 負責把上游的方言翻譯成這個 port 的語言。

```
你的上下文                    防腐層                        上游（ERP）
┌────────────┐   ┌──────────────────────────┐   ┌──────────────┐
│ 領域模型     │<──│ translator ← facade ← ada.│<──│ PROD_STS_CD  │
│ Availability│   │ （方言在此止步）            │   │ '01'/'03'/'07'│
└────────────┘   └──────────────────────────┘   └──────────────┘
```

### 什麼時候值得築牆

ACL 貴——上游每個會流進來的概念都要翻譯、上游每次改版都要維護牆。所以它不是預設配備，而是 Context Map 上的**選項之一**，和便宜的替代方案一起擺上桌：

| 情境 | 合理選擇 |
|---|---|
| 上游模型健康、語言相近、演進可預期 | 順從者（Conformist）——直接用上游模型，省下整道牆 |
| 上游是核心域的資料來源，但模型混亂／不可控／將被汰換 | **ACL**——模型主權值得保費 |
| 整合價值低、資料可自建 | 各行其道（Separate Ways）——最被低估的選項 |

一句話版本：**牆築在「你的模型值得保護」且「上游不值得信任」同時成立的邊界上**。邊緣上下文對上游 conformist 不是懦弱，是成本意識；核心域對爛上游不設防，才是失職。

### 特別適用的兩個場合

- **遺留系統演進**：新系統透過 ACL 讀寫舊系統，新模型保持乾淨；日後舊系統汰換，只有牆要重寫——這正是絞殺者遷移（Strangler Fig，漸進替換遺留系統的模式，Fowler 2004 命名）的標準配件。
- **外部廠商 API**：廠商的模型你永遠不可控，而且合約到期會換——〈六角架構〉那個換簡訊商的案例，如果 adapter 同時做了語意翻譯（廠商的回執狀態 → 你的 `DeliveryStatus`），它就已經是一道小型 ACL。

## 實際案例

新訂單上下文需要判斷「這個商品現在能不能賣」，資料源是舊 ERP。

❌ 不設防——方言直接殖民：

```java
// 散在十七個地方的寫法
ErpProduct p = erpClient.getProduct(productId);
if ("03".equals(p.getProdStsCd()) || "07".equals(p.getProdStsCd())) {
    throw new ProductNotSellableException(...);   // '07' 到底是什麼？沒人知道，反正一起擋
}
```

三個月後 ERP 改版，`'03'` 細分成 `'03A'`（停售）與 `'03B'`（暫停補貨、可售完現貨）——十七個判斷點要逐一翻修，漏掉的那幾個在大促當天把可以賣的商品全下了架。更深的傷害是**語言**：團隊開會已經開始說「那個商品是不是 03」——上游的方言成了你們的統一語言，模型的癌變完成了。

✅ 築牆——方言止步於 translator：

```java
// 你的上下文只認識自己的語言（port 由領域定義）
public enum ProductAvailability { ON_SALE, DISCONTINUED, LAST_STOCK }

public interface ProductCatalogPort {
    ProductAvailability availabilityOf(ProductId id);
}
```

```java
// 防腐層：全系統唯一認識 PROD_STS_CD 的地方
class ErpProductCatalogAcl implements ProductCatalogPort {
    public ProductAvailability availabilityOf(ProductId id) {
        String code = erpFacade.fetchProduct(id).getProdStsCd();
        return switch (code) {
            case "01"        -> ON_SALE;
            case "03", "07"  -> DISCONTINUED;    // '07' 的考古結論與出處記在這裡
            default -> throw new UnknownUpstreamStateException(code);  // 不吞未知值
        };
    }
}
```

ERP 改版時，全系統只有 `switch` 這一處要改；`'03B'` 這種上游的新概念，由翻譯層決定映射到 `LAST_STOCK` 還是新增概念——**決定權在你的模型這一側**。兩個容易漏的細節：未知狀態碼要炸出來而不是默默歸類（吞掉未知值等於把上游的下一次改版變成靜默資料污染）；翻譯的考古結論（`'07'` 是 2009 年併購時的遺留值）要註記在牆上，這道牆同時是團隊對上游知識的**唯一存放處**。

## 技術優缺點

**優勢**

- **模型主權**：你的統一語言不被上游方言污染，核心域保持可演進。
- **變更隔離**：上游改版的爆炸半徑被壓縮到翻譯層一處。
- **可測試性**：牆內的業務邏輯面對的是自己的模型，測試不需要真的 ERP；牆本身用上游的實際回應樣本測。
- **知識集中**：對上游的考古成果（狀態碼語意、暗規則）有唯一的存放處。

**代價**

- **翻譯層的維護**：上游每個新概念都要過牆，牆會隨整合面積長大。
- **雙模型的認知成本**：同一份資料存在兩種表述，除錯時要在牆兩側對帳。
- **延遲與失真**：翻譯必然取捨——上游模型裡某些細微語意（你用不到的）會被有意丟棄，日後突然需要時要回頭擴牆。
- **過度使用的儀式化**：對健康的上游也築牆，得到的是一堆透傳的假翻譯——那是 conformist 收了 ACL 的價。

**這改變了什麼實務決策**：整合評審的問題從「怎麼接這個 API」變成「**這條線上我們要不要模型主權**」。要，預算裡就編進一道牆和它的長期維護；不要，就明文選 conformist 並接受被上游牽動——兩個都是正當決策，不正當的是沒有決策、讓 `'03'` 自己滲進來。

## 小結

- ACL 翻譯的是模型與語意，不只是格式；判準是轉換後的型別屬於誰的語言。
- 結構是 facade＋adapter＋translator；在六角架構裡它是語意加重版的被驅動端 adapter。
- 牆築在「模型值得保護 × 上游不可信任」的交點；邊緣上下文用 conformist 是成本意識不是懦弱。
- 未知的上游狀態要顯式失敗，不要默默歸類；牆同時是上游考古知識的唯一存放處。
- 它是絞殺者式遷移的標準配件：新模型乾淨地長，舊系統死時只賠一道牆。
- 第三章到此收束：邊界（戰略）、模型（戰術）、邊防（ACL）。但 Bounded Context 之間的最終一致、事件同步，把我們推向了那組繞不開的分散式必答題——下一章從〈[CAP 與 PACELC：被誤讀最多的定理](../04-distributed-basics/cap-and-pacelc.md)〉開始。

## 常見面試題

1. **ACL 和一般的 adapter／DTO 轉換差在哪？**（提示：格式翻譯 vs 語意翻譯；「轉出來的型別屬於誰的語言」這個判準說出來就到位）
2. **什麼情況你會選 conformist 而不是 ACL？**（提示：上游健康＋非核心域；答得出「ACL 有維護成本、不是預設配備」才顯出取捨能力）
3. **ACL 應該由上游團隊還是下游團隊維護？**（提示：牆保護的是下游的模型，天然歸下游；若上游願意提供乾淨介面，那叫 Open Host Service，是另一個模式）

## 延伸閱讀

- Eric Evans, *Domain-Driven Design*（2003）——第 14 章，Anticorruption Layer 模式原典
- Vaughn Vernon, *Implementing Domain-Driven Design*（2013）——第 3 章 Context Mapping 的 ACL 實作範例
- Microsoft Azure Architecture Center, [Anti-corruption Layer pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer)——雲端情境下的精簡版說明
- Martin Fowler, [StranglerFigApplication](https://martinfowler.com/bliki/StranglerFigApplication.html)（2004）——ACL 最常登場的遷移模式
