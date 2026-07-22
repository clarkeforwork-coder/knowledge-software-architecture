# 戰術設計：Aggregate、Entity 與 Value Object

## 前言

「只是把訂單狀態改成已出貨，為什麼要把訂單連同三十筆明細、會員資料、商品資料全部撈出來？」——因為模型裡 `Order` 引用 `Member`、`Member` 引用 `MemberLevel`、每筆 `OrderLine` 又引用完整的 `Product`，物件圖連成一張蜘蛛網，碰哪裡都是牽一髮動全身。

[上一篇](strategic-design-bounded-context.md)劃出了模型的國境線；這一篇進到境內：模型自己該怎麼長。DDD 戰術設計的三個核心構件——Entity、Value Object、Aggregate——本質上在回答同一個問題：**一致性的最小單位是什麼、物件之間的引用該在哪裡斷開**。

## 技術背景

### 先破除誤解：DDD 的 Entity 不是資料表映射

這個誤解值得多花一段，因為本 repo 自己就示範過另一種用法——〈[分層物件家族](../02-architecture-styles/layered-object-family.md)〉裡的 `*Entity` 是「資料表的完整映射」，貧血、無行為、與表結構一對一。那個定義在 CRUD 分層系統裡完全正當。

DDD 語境下的 Entity 是**另一個概念**：由「識別」（identity）與「生命週期」定義的領域物件——兩筆訂單所有欄位都相同，仍然是兩筆訂單，因為它們的 ID 不同、各有各的歷史。它有行為、守規則，和持久化方式無關。

同一個詞、兩個定義、各自在自己的脈絡裡成立——這正是上一篇 Bounded Context 的日常示範：**連「Entity」這個詞本身，都需要限界上下文**。讀 DDD 文獻時遇到的多數混亂，源頭都是把兩個語境的詞混著用。

### Entity vs Value Object：識別 vs 值

| | Entity | Value Object |
|---|---|---|
| 由什麼定義 | 識別（ID）與生命週期 | 屬性的值本身 |
| 相等的判準 | ID 相同即同一個 | 所有屬性相等即相等 |
| 可變性 | 狀態隨生命週期演進 | **不可變**，要改就換一顆新的 |
| 例子 | 訂單、會員 | 金額、地址、期間 |

Value Object 是戰術設計裡投資報酬率最高的構件：把「金額」從 `BigDecimal amount + String currency` 兩個散裝欄位收攏成不可變的 `Money`，加法自動檢查幣別、四捨五入規則只寫一次、誤把台幣加美金在型別層就擋掉。不可變讓它可以放心共享與快取——不可變設計的語言層機制，見 knowledge-java 的〈[record 與不可變物件](../../knowledge-java/02-language-core/record-and-immutability.md)〉。

實務偏方：**建模時先假設每個概念都是 Value Object，證明它需要身分與歷史才升格為 Entity**——多數專案的病是 Entity 太多，不是太少。

### Aggregate：一致性的邊界

Aggregate（聚合）是戰術設計的重頭戲：**一組必須一起維持一致的物件，以一個 Entity 作為聚合根（Aggregate Root），外界只能透過根進出**。

規則只有三條，但每條都在對抗蜘蛛網：

1. **不變量決定邊界**。「訂單總額必須等於明細加總」是一條不變量（invariant），所以 `Order` 與 `OrderLine` 同屬一個聚合，任何修改都經過 `Order` 這個根，由它保證規則成立。
2. **聚合之間以 ID 引用，不以物件引用**。`Order` 記 `memberId`，不持有 `Member` 物件——會員資料的一致性不歸訂單管，蜘蛛網在這裡斷開（Vaughn Vernon〈Effective Aggregate Design〉系列的核心建議：小聚合、以 ID 引用）。
3. **一個交易只改一個聚合**。聚合內強一致；聚合之間最終一致，用領域事件同步（〈[事件驅動架構](../02-architecture-styles/event-driven-architecture.md)〉在模型層的接口）。「下單要加會員積分」不是把兩個聚合塞進一個交易，而是 `OrderPlaced` 事件讓積分聚合自己反應。

### 邊界怎麼判：問不變量，不要問資料關聯

「訂單和會員有關聯，要不要放同一個聚合？」——問錯了。資料庫裡萬物皆有關聯，照關聯建聚合就是把整個 schema 搬進一個物件圖。正確的問句是：**「有沒有一條業務規則，要求這兩個東西在同一瞬間一致？」**有，才同聚合；沒有，ID 引用加事件同步。

## 實際案例

### 一個訂單模型的兩種長法

❌ 照資料關聯長——巨型聚合：

```java
public class Order {
    private Member member;            // 整顆會員物件
    private List<OrderLine> lines;
    private BigDecimal total;

    public void setTotal(BigDecimal total) { this.total = total; }  // 規則在誰手上？
}
// 使用端：改狀態也得先把整張圖載入；總額由呼叫端自己算自己 set
order.setTotal(order.getLines().stream()...);   // 這行散落在七個地方，其中一處忘了算運費
```

三種病由此而生：**載入成本**（動根毫毛都要撈全圖）、**並發衝突**（兩個人改同一訂單的不同明細互相打架，鎖的粒度是整個聚合）、**不變量無主**（總額規則靠呼叫端自律，七處實作、一處出錯）。

✅ 照不變量長——小聚合、根守規則：

```java
public class Order {                              // 聚合根
    private final OrderId id;
    private final MemberId memberId;              // ID 引用，蜘蛛網在此斷開
    private final List<OrderLine> lines = new ArrayList<>();
    private Money total = Money.zero(TWD);        // Value Object

    public void addLine(ProductId productId, int qty, Money unitPrice) {
        ensureModifiable();                       // 生命週期規則：出貨後不可改
        lines.add(new OrderLine(productId, qty, unitPrice));
        this.total = recalculate();               // 不變量只在這裡維護
    }
}
```

總額規則從「七個呼叫端的默契」變成「根的職責」；會員與商品都以 ID 引用，載入與鎖的範圍縮到一張訂單；「下單送積分」交給 `OrderPlaced` 事件。模型本身變成了業務規則的文件——新人問「訂單什麼時候不能改」，答案不在 wiki，在 `ensureModifiable()`。

### 誠實的另一面

這個模型也付了學費：查詢端很痛。「訂單列表要顯示會員名稱」——聚合裡只有 `memberId`，於是列表查詢繞過聚合、直接用〈[分層物件家族](../02-architecture-styles/layered-object-family.md)〉的 `*DO` 撈 JOIN 結果。這不是作弊，是常態：**聚合為寫入與一致性而設計，查詢走自己的路**——讀寫分離到極致就是 CQRS，此處先按下不表。

## 技術優缺點

**優勢**

- **不變量有唯一的家**：業務規則收進聚合根，不再散落在呼叫端。
- **並發與載入的粒度受控**：小聚合＝小鎖＝小物件圖。
- **模型即文件**：行為與規則長在領域物件上，程式碼直接回答業務問題。

**代價**

- **建模是真功夫**：找不變量、劃聚合需要領域知識與多次迭代，比照表建類貴得多。
- **與 ORM 慣性對撞**：預設急切載入、雙向關聯、cascade 全開的用法，處處和小聚合原則打架。
- **查詢要另闢蹊徑**：跨聚合的讀取需求得靠查詢模型（DO）承接，多一套物件。
- **簡單領域用不上**：純 CRUD 的上下文套聚合是儀式感——貧血物件家族＋交易腳本更誠實。

**這改變了什麼實務決策**：在 Context Map 上標出**核心域**——業務規則密集、構成競爭力的那一兩個上下文，戰術設計的投資集中在那裡；支撐域和通用域用 CRUD 寫法，沒有人規定全系統要同一種建模風格。「哪裡值得聚合」本身就是一個架構取捨。

## 小結

- DDD 的 Entity 由識別與生命週期定義，和資料表映射的 `*Entity` 同詞異義——這個衝突本身就是 Bounded Context 的示範。
- Value Object 不可變、以值相等，是投報率最高的構件；先假設是 VO，需要身分才升格。
- Aggregate 是一致性邊界：不變量決定成員、根是唯一入口、聚合間以 ID 引用＋事件同步。
- 一個交易改一個聚合；聚合為寫入設計，查詢走 DO 自己的路。
- 戰術投資集中在核心域；支撐域用 CRUD 是成熟不是偷懶。
- 新模型建好了，但世界上還有一個上游不受你控制：那套欄位叫 `PROD_STS_CD`、狀態碼是 `'03'` 的遺留系統。新模型怎麼跟它相處而不被殖民？下一篇：〈[防腐層：與遺留系統共處](anti-corruption-layer.md)〉。

## 常見面試題

1. **Entity 和 Value Object 怎麼區分？地址該是哪一種？**（提示：問「兩個屬性完全相同的算不算同一個」；地址在多數上下文是 VO，在物流上下文可能升格）
2. **Aggregate 的邊界怎麼決定？**（提示：不變量，不是資料關聯；說出「一個交易一個聚合」與 ID 引用就到位）
3. **兩個聚合需要一起改，怎麼辦？**（提示：先質疑邊界是否劃錯；確認沒錯就是領域事件＋最終一致，並向業務確認時差可接受）

## 延伸閱讀

- Eric Evans, *Domain-Driven Design*（2003）——Part II，Entity／Value Object／Aggregate 的原典
- Vaughn Vernon, [Effective Aggregate Design](https://www.dddcommunity.org/library/vernon_2011/)（2011，三部曲論文）——小聚合、ID 引用、最終一致的完整論證
- Vaughn Vernon, *Implementing Domain-Driven Design*（2013）——第 10 章 Aggregate 實作細節
- knowledge-java〈[record 與不可變物件](../../knowledge-java/02-language-core/record-and-immutability.md)〉——Value Object 在語言層的落地機制
