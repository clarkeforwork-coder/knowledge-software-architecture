# 分層物件家族：Entity、DO、DTO 的邊界在哪

## 前言

「同一筆訂單資料，專案裡竟然有 `OrderEntity`、`OrderRankDO`、`OrderDTO`、`OrderResponse` 四個類別——這不是重複造輪子嗎？」幾乎每個剛加入分層專案的工程師都問過這個問題；然後在 code review 被擋下「直接把 Entity 回給前端」的提交時，第一次真正感受到這條規則的存在。

這篇筆記把這套「物件家族」講清楚：每個後綴代表什麼、邊界規則為什麼存在、以及什麼時候該懷疑自己過度設計。層是靜態的圖，物件是資料流過層界時的形狀——本文談後者（層本身見〈[分層架構與它的邊界](layered-architecture-and-boundaries.md)〉）。

## 技術背景

### 先破除誤解：這些類別不是「同一份資料的複本」

欄位看似雷同，但三個軸線完全不同：**欄位由誰決定、跟著什麼變動、誰看得到**。

| 物件 | 欄位由誰決定 | 跟著什麼變動 | 可見範圍 |
|---|---|---|---|
| `*Entity` | 資料表結構 | 表結構異動 | 資料存取層 |
| `*DO` | 某支查詢的輸出 | 查詢需求 | 資料存取層 |
| `*DTO` | 消費方需要什麼 | 模組間協議 | 跨模組 |
| `*Response` | API 規格 | 對外承諾 | 所有 API 消費者 |

這正是單一職責原則的物件版——「一個模組應該只對一個角色負責」（Robert C. Martin, *Clean Architecture*, 2017）。強行共用一個類別，等於讓資料表、查詢需求、API 規格三個變動來源綁在同一個檔案上，改任何一邊都牽動全部。

### 另一個誤解：這些縮寫有全宇宙統一的定義

沒有。以 DO 為例：阿里巴巴《Java 開發手冊》的分層領域模型規約中，DO（Data Object）指「與資料庫表結構一一對應」的物件——相當於下表的 Entity；而在許多專案（包括本文採用的定義）裡，DO 指客製查詢的輸出載體。同一個縮寫、兩種意義。

所以重點從來不是背誦「標準答案」，而是**專案內自洽**：一張命名定義表、全員遵守、新類型必須先過架構討論。

### 家族成員與命名定義

一套經過實務驗證的定義（Mapper 在此泛指資料存取層的查詢介面，不綁定特定 ORM 框架）：

| 類型 | 用途 |
|---|---|
| `*Entity` | 資料表完整持久化映射，同時承擔 PO 的角色 |
| `*DO` | Mapper 客製查詢輸出：JOIN、聚合、排名、欄位轉換 |
| `*Query` | Mapper 客製查詢輸入條件 |
| `*BO` | Service 層業務處理的中間物件 |
| `*DTO` | 跨模組或跨服務傳輸物件 |
| `*Request`／`*Response` | HTTP contract |

判斷速查：欄位等於整張表 → Entity；查詢輸出但不等於任何一張表 → DO；查詢輸入 → Query；業務運算中間產物 → BO；要交給別的模組 → DTO；HTTP 進出 → Request／Response。

### 邊界規則：物件不跨層

```
HTTP 層          資料存取層                資料庫
*Request ──轉換──> *Query  ──> Mapper ──┐
                                        ├──> 資料表
*Response <─轉換── *DO / *Entity <──────┘
```

- `*Query`、`*DO`、`*Entity` 是資料存取層的契約，不得出現在 HTTP contract 中。
- `*Request`／`*Response` 不得直接傳入 Mapper。
- 兩側之間必須經過明確的轉換點；轉換是邊界存在的證據，不是浪費。

### 那 PO 呢？

PO（Persistent Object）的傳統定義是「與資料表一對一的持久化映射」——與上表 Entity 的職責完全重疊。兩者並存只有代價：每張表兩個欄位相同的類別、表結構異動要同步兩處、層間多一段不承載邏輯的欄位搬運、新成員不知道該用哪個。

PO 與 Entity 的區分只在一種情境下有意義：**領域模型與持久化模型分離**時——如 DDD 的充血領域物件搭配獨立的持久化物件（Eric Evans, *Domain-Driven Design*, 2003），Entity 承載業務行為、PO 負責映射，欄位與生命週期各自獨立。在 Entity 定位為貧血表映射的專案裡，這種分離不存在，就不必預留空殼；未來若演進為領域模型分離，屆時再依當時的設計引入。

## 實際案例

### 情境一：排行榜需求的誘惑

需求：訂單金額排行榜，需要 JOIN 會員表、算排名。最省事的寫法是在 Entity 上直接加欄位：

```java
// ❌ 在 Entity 上加表中不存在的欄位承接查詢結果
public class OrderEntity {
    private Long id;
    private BigDecimal amount;
    private Integer rankNo;     // 表上沒有這個欄位
    private String memberName;  // 這是別張表的欄位
}
```

三個後果會陸續浮現：

1. **Entity 與表結構失去一對一**——下一個人看到 `rankNo` 會去資料表找欄位，找不到；再下一個人有樣學樣，Entity 逐漸變成「所有查詢欄位的聯集」。
2. **一般查詢揹著幽靈欄位**——單表查詢回來的 Entity 裡 `rankNo` 永遠是 null，呼叫端無從分辨「沒有排名」與「這條路徑不填排名」。
3. **寫入路徑被污染**——insert／update 時這些欄位是什麼語意？沒有答案，只能靠「大家記得別碰」。

```java
// ✅ 客製查詢輸出另建 DO，Entity 與表結構保持一對一
public class OrderRankDO {
    private Long orderId;
    private BigDecimal amount;
    private Integer rankNo;
    private String memberName;
}
```

### 情境二：把 Entity 直接回給前端

```java
// ❌ Entity 直接當 HTTP 回應
return orderMapper.selectById(id);   // OrderEntity 直接序列化出去
```

短期沒事，直到：內部欄位（成本價、風控狀態）跟著序列化洩漏出去；DBA 把欄位改名，API 就地 breaking change，前端在毫無預警下壞掉；反過來，前端要求多回一個欄位，壓力直接打到資料表設計上。資料表結構是對內的實作細節，API 是對外的承諾——兩者的變動節奏天差地遠，共用一個類別等於把承諾綁在實作細節上。

```java
// ✅ Response 是獨立契約，轉換點集中一處
OrderEntity entity = orderMapper.selectById(id);
return OrderDetailResponse.from(entity);   // 要回什麼欄位，在這裡一目了然
```

轉換方法同時是天然的 review 檢查點：這支 API 到底吐了哪些欄位出去，看 `from()` 一眼就知道，不用去翻表結構。

## 技術優缺點

**優勢**

- **變動隔離**：表結構、查詢需求、API 規格各自演進，互不牽動。
- **資訊安全**：對外欄位是白名單（Response 有什麼才出什麼），不是黑名單。
- **review 可機械化**：後綴即契約，「Entity 出現在 Controller」這種違規肉眼可判，不需要理解業務。

**代價**

- 類別數量成倍增加，小需求也要建齊一組物件。
- 層間轉換是樣板碼，欄位多時寫起來乏味（可用映射工具緩解，但工具本身也是複雜度）。
- 對生命週期短的小工具、內部腳本，這套家族是純負擔。

**這改變了什麼實務決策**：物件家族的粒度是架構決策，跟系統壽命與團隊規模成正比——一人維護的內部小工具，Entity 直出沒什麼不行；多人長壽的業務系統，邊界必須立在第一天，因為事後補救等於全面翻修每一層的簽名。真正要避免的是中間態：規範寫了但沒人守，家族淪為命名玄學。

## 小結

- 後綴即契約：看到類別名就知道它屬於哪一層、能出現在哪裡。
- Entity、DO、DTO 不是同一份資料的複本，是三個變動來源各自的投影——欄位由誰決定、跟著什麼變動、誰看得到，三軸皆不同。
- 邊界規則一句話：存取層物件不出層、HTTP 物件不進層、轉換點唯一且明確。
- DO 這類縮寫沒有跨專案的統一定義，重點是專案內自洽並寫進規範。
- PO 與 Entity 分離只在「領域模型與持久化模型分離」時有意義，在那之前不預留空殼。

物件家族管的是「資料以什麼形狀流過層界」；但層與層之間**誰依賴誰**、依賴方向能不能翻轉，是更根本的架構決策——見〈[六角架構與 Clean Architecture：依賴方向的翻轉](hexagonal-and-clean-architecture.md)〉。

## 常見面試題

1. **DTO 和 Entity 為什麼不建議共用一個類別？**（提示：從變動來源與可見範圍切入，說到「API 承諾被綁在表結構上」就到位了）
2. **什麼時候該把 PO 從 Entity 獨立出來？**（提示：關鍵詞是領域模型與持久化模型分離，反面是貧血映射下的同義重複）
3. **Query 物件和 Request 物件差在哪？不都是查詢條件嗎？**（提示：所屬的層不同、契約對象不同，Controller 收 Request 轉成 Query 再下資料存取層）

## 延伸閱讀

- Martin Fowler, *Patterns of Enterprise Application Architecture*（2002）——DTO 模式的原始定義：為減少遠端呼叫次數而生的資料載體
- Martin Fowler, [LocalDTO](https://martinfowler.com/bliki/LocalDTO.html)——行程內該不該用 DTO 的取捨論戰，正反意見都在
- 阿里巴巴《Java 開發手冊》分層領域模型規約——另一套 DO／DTO／VO 定義，對照可見縮寫歧義
- Robert C. Martin, *Clean Architecture*（2017）——單一職責原則與 Boundaries 各章
- Eric Evans, *Domain-Driven Design*（2003）——領域模型與持久化分離的源頭
