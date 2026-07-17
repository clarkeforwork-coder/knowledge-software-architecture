# 分層架構與它的邊界

## 前言

「我們是三層架構。」——Controller、Service、Repository，人人會背，專案範本開出來就有這三個資料夾。但為什麼三年後 Service 還是長成了三千行的上帝類別？為什麼 Controller 裡出現了 SQL？為什麼「只是改個欄位」要動到每一層？

因為**建了三個資料夾不等於有了分層架構**。分層是一組關於「誰能依賴誰、什麼邏輯歸誰」的規則；資料夾只是規則的投影。這篇筆記把規則本體講清楚——包括它為什麼會腐化，以及它天生的軟肋。

## 技術背景

### 先破除誤解：資料夾結構不是架構

分層架構（Layers pattern，正式記載於《Pattern-Oriented Software Architecture》Vol.1, Buschmann et al., 1996；企業應用的三層劃分見 Fowler《Patterns of Enterprise Application Architecture》, 2002）的本體是兩條規則：

1. **依賴方向單向**：上層可以依賴下層，下層不得反向依賴上層。
2. **職責分配明確**：每一層只做自己的事——

| 層 | 職責 | 不該出現的東西 |
|---|---|---|
| 展示層（Controller） | 協議轉換：收 Request、驗格式、回 Response | 業務規則、SQL |
| 業務層（Service） | 業務規則與流程編排、交易邊界 | HTTP 細節、SQL 字串 |
| 持久化層（Repository/Mapper） | 資料存取，把表結構翻譯成物件 | 業務判斷 |

只有資料夾、沒有規則的專案，得到的是「有分層外觀的義大利麵」。

### 嚴格分層 vs 鬆散分層

還有一條要**明文決定**的規則：能不能跳層？

- **嚴格分層（closed layers）**：只能呼叫相鄰下層。Controller 想查資料，必須經過 Service。
- **鬆散分層（open layers）**：允許跳過某些層。Controller 可以為了單純查詢直接呼叫 Repository。

鬆散分層省掉轉發樣板碼，但代價是「這個查詢有沒有業務規則」的判斷散落在每個人心裡；嚴格分層樣板碼多，但規則可以機械化檢查。**哪種都行，不能的是沒講清楚**——多數專案的腐化正是從「這裡跳一下沒關係吧」開始的。

### 邊界靠什麼維持

規則不會自己執行，靠三個機制：

- **物件契約**：每層有自己的物件家族，Entity 不出資料存取層、Request 不進去——這正是〈[分層物件家族](layered-object-family.md)〉的邊界規則，後綴即層籍。
- **依賴方向**：透過模組／包的依賴宣告讓「下層 import 上層」直接編譯失敗，比 code review 可靠。
- **架構測試**：用架構檢查工具把「Controller 不得依賴 Repository」寫成會跑的測試，違規在 CI 就擋下，不必等人眼。

### 分層的天生軟肋：一切通往資料庫

把三層圖畫完會發現：**依賴的終點是資料庫**。業務層依賴持久化層，等於業務規則間接依賴表結構——表怎麼設計，常常反過來決定業務程式怎麼寫；測業務規則得先起資料庫。這不是實作瑕疵，是這個風格的結構性性質，也是下一篇依賴翻轉登場的原因。

## 實際案例

### 一條邊界的腐化史

需求：「下單滿千送折扣」。當天要上線，工程師把折扣算式直接寫進 Controller：

```java
// ❌ 業務規則寫在展示層——當下最快的路
@PostMapping("/orders")
public OrderResponse create(@RequestBody OrderCreateRequest req) {
    BigDecimal amount = req.getAmount();
    if (amount.compareTo(new BigDecimal("1000")) >= 0) {
        amount = amount.multiply(new BigDecimal("0.95"));  // 折扣邏輯
    }
    return OrderResponse.from(orderService.create(req.toCommand(amount)));
}
```

三個月後，排程系統要幫 VIP 自動補單——排程入口不走這支 Controller，於是折扣算式被**複製**到排程模組。再兩個月，折扣規則改成滿兩千九五折：有人只改了 Controller。從此同一筆訂單，人工下單和自動補單金額不同，客訴進來才發現。

```java
// ✅ 規則歸位業務層，所有入口共用同一份
public class OrderService {
    public Order create(OrderCreateCommand cmd) {
        BigDecimal payable = discountPolicy.apply(cmd.getAmount());
        ...
    }
}
```

事故的根因不是那位工程師，而是**邊界沒有被機制守住**：規則寫進 Controller 的那天，沒有任何東西會叫。分層的價值不在圖好看，在「業務規則只有一個家」這件事被結構保證。

### 跳層的雪球

另一條常見路徑：「只是查個下拉選單，走 Service 太小題大作了吧」——Controller 直接呼叫 Repository。第一次真的無害；但這條先例一開，「哪些查詢算單純」沒有標準，半年後三成的 Controller 都在直連 Repository，其中幾支悄悄長出了過濾邏輯——業務規則開始定居在展示層。如果團隊選擇鬆散分層，就把「允許跳層的條件」寫進規範並用架構測試圈住；選擇嚴格分層，就接受轉發樣板碼作為守邊的保費。

## 技術優缺點

**優勢**

- **心智模型便宜**：職責問答「這段邏輯歸哪層」九成有直覺答案，新人第一天就能安全提交。
- **與組織相容**：層的分工貼合常見的技能分佈，也貼合多數團隊的溝通結構（〈[康威定律](../01-architecture-thinking/conways-law.md)〉的順風面）。
- **生態成熟**：框架、範本、檢查工具全都內建這套假設，起步成本近乎零。

**代價**

- **資料庫在依賴終點**：業務規則被表結構牽動，領域邏輯難以獨立測試。
- **樣板碼**：嚴格分層下大量「一行轉發」的 Service 方法。
- **邊界靠紀律**：規則不寫進依賴宣告與架構測試，就只是文件裡的一句話。
- **橫向擴張的極限**：層內沒有進一步的模組邊界時，每一層都會隨業務長成自己的巨石。

**這改變了什麼實務決策**：中小型、以資料操作為主的系統，分層仍是最划算的預設解——前提是把「跳層規則、物件契約、依賴檢查」三件事在第一天定下來。當領域規則複雜到「測試必須擺脫資料庫」「表結構不該再主導業務程式」時，就是考慮翻轉依賴方向的時機。

## 小結

- 分層架構的本體是規則不是資料夾：依賴單向、職責分層，缺規則的分層只是外觀。
- 嚴格 vs 鬆散分層沒有對錯，但必須明文選擇——腐化多半始於未經決定的「跳一下」。
- 邊界靠機制不靠自覺：物件契約（後綴即層籍）、依賴宣告、架構測試。
- 分層的結構性軟肋是依賴終點在資料庫，領域邏輯間接被表結構綁架。
- 想讓依賴掉頭、讓領域成為圓心？下一篇：〈[六角架構與 Clean Architecture：依賴方向的翻轉](hexagonal-and-clean-architecture.md)〉。

## 常見面試題

1. **三層架構中，一段「滿額折扣」邏輯該放哪層？怎麼說服想放 Controller 的同事？**（提示：多入口共用、規則只有一個家；用「第二個入口出現時」推演）
2. **什麼是嚴格分層與鬆散分層？你的專案怎麼選？**（提示：closed/open layers；重點是「明文決定＋機制守住」而不是選哪邊）
3. **分層架構最大的結構性弱點是什麼？**（提示：依賴終點是資料庫；答得出「所以才有依賴反轉系的風格」就到位）

## 延伸閱讀

- Frank Buschmann et al., *Pattern-Oriented Software Architecture, Vol.1*（1996）——Layers pattern 的正式記載
- Martin Fowler, *Patterns of Enterprise Application Architecture*（2002）——三層劃分與 Service Layer、Repository 等模式的出處
- Mark Richards & Neal Ford, *Fundamentals of Software Architecture*（O'Reilly, 2020）——Layered Architecture 章，含 open/closed layers 與「架構淤泥」（architecture sinkhole）反模式
