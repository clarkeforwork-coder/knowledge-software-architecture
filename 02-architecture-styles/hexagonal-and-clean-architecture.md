# 六角架構與 Clean Architecture：依賴方向的翻轉

## 前言

「想幫核心業務規則寫測試，結果要先起資料庫、塞十張表的假資料。」「想把簡訊供應商換掉，發現它的 SDK 型別散在四十個檔案裡。」——這兩個痛有同一個根因：**依賴全部指向基礎設施**。上一篇〈[分層架構與它的邊界](layered-architecture-and-boundaries.md)〉的結尾指出了這個結構性軟肋：分層圖的最下方永遠是資料庫，業務規則活在別人的地基上。

六角架構、Clean Architecture 這些名字你大概都聽過，也可能被它們的同心圓圖嚇退過。這篇筆記把它們還原成一個非常小的核心動作：**把依賴的方向翻過來，讓領域變成圓心**。

## 技術背景

### 先破除誤解：這不是三種要分別學會的架構

- **六角架構**（Hexagonal / Ports and Adapters，Alistair Cockburn，2005）
- **洋蔥架構**（Onion Architecture，Jeffrey Palermo，2008）
- **Clean Architecture**（Robert C. Martin，2012 部落格文，2017 成書）

三者的圖不同、名詞不同，核心主張完全相同，Clean Architecture 把它濃縮成一條**依賴規則（The Dependency Rule）**：原始碼的依賴只能指向內圈——業務規則不知道資料庫、不知道 Web 框架、不知道任何外部世界的存在。六角形有幾個邊、洋蔥畫幾圈都不重要；挑一套名詞用即可，本文用六角架構的 port／adapter。

### 翻轉的機制：介面放在圓心

分層架構裡，Service 依賴 Repository 的**實作**；翻轉之後，介面（port）由領域核心**自己定義**，實作（adapter)在外圈實現它:

```java
// 圓心：領域核心定義「我需要什麼」——這是 port
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
}

public class PlaceOrderService {          // 業務規則只認識自己定義的介面
    private final OrderRepository orders; // 不知道 JDBC、不知道任何框架
}
```

```java
// 外圈：基礎設施實現「怎麼做到」——這是 adapter
public class JdbcOrderRepository implements OrderRepository {
    // SQL、連線池、表結構映射都被關在這裡
}
```

呼叫的方向沒變（業務仍然在存取資料庫），**編譯期的依賴方向反了**：現在是基礎設施依賴領域（implements 領域的介面），領域不依賴任何人。這就是依賴反轉原則（DIP）從類別層級放大到架構層級。

### 兩種 port：驅動與被驅動

```
        （驅動端 adapter）                （被驅動端 adapter）
   HTTP Controller ─┐               ┌─> JdbcOrderRepository ──> DB
   排程觸發器      ─┼─> ┌────────┐ ─┼─> EmailNotifier ────────> SMTP
   訊息消費者      ─┘   │ 領域核心 │  └─> PaymentGatewayClient ─> 金流商
                        └────────┘
        入站 port（use case 介面）        出站 port（領域定義的需求介面）
```

- **驅動端（driving）**：誰來叫我——HTTP、排程、訊息，都只是觸發 use case 的不同皮。
- **被驅動端（driven）**：我需要誰——持久化、通知、外部金流，都只是領域所宣告介面的可替換實作。

於是資料庫從「地基」降級成「外掛之一」，和簡訊供應商同一個等級。

### 與分層架構的關係

不是推翻，是把「上下」改成「內外」：分層架構中位於最底層的持久化，在這裡被移到與展示層同一圈。展示層物件不進核心、持久化物件不出核心的契約規則依然成立（見〈[分層物件家族](layered-object-family.md)〉），只是守護的圓心從「層」變成「領域」。

## 實際案例

### 一次替換需求照出的差距

需求：簡訊供應商合約到期，換新供應商；順帶要求「重要通知同時發 Email」。

翻轉前的典型現場——供應商 SDK 直接滲透業務層：

```java
// ❌ 業務規則直接抱著供應商 SDK
public class OrderService {
    private final AcmeSmsClient acmeClient;          // 供應商型別
    public void completeOrder(Order order) {
        ...
        acmeClient.send(new AcmeSmsPayload(...));    // 供應商的參數格式
    }
}
```

全案搜尋 `AcmeSmsClient`：42 個檔案。每一處都要改成新 SDK 的型別與參數格式，而且這 42 處散在各種業務流程中間——改壞哪個都是業務事故。測試也連坐：這些業務規則的單元測試，要嘛 mock 到天邊，要嘛真的打出簡訊。

翻轉後，領域只認識自己定義的 port：

```java
// ✅ 圓心定義需求；供應商是圓外的一個 adapter
public interface NotificationPort {
    void notify(MemberId to, NotificationMessage message);
}
```

換供應商＝新寫一個 `NewVendorSmsAdapter`，一個檔案；加 Email＝再掛一個 `EmailAdapter`（或一個組合兩者的 adapter），**業務程式碼零修改**。而測試業務規則時掛一個記在記憶體裡的 fake adapter，不必起任何外部服務——「核心可以在沒有世界的情況下被測試」，是依賴規則最直接的紅利。

值得誠實記錄的另一面：這個專案為此多了一組介面、兩個 adapter、一個 `NotificationMessage` 領域物件與它到各供應商格式的映射——**翻轉不是免費的**，它把「將來替換的成本」預付成「現在抽象的成本」。這筆保費值不值，取決於「將來」會不會來。

## 技術優缺點

**優勢**

- **核心可獨立測試**：業務規則的測試不需要資料庫、佇列或任何容器，快而穩。
- **技術可替換**：資料庫、框架、外部服務都是 adapter，替換的爆炸半徑被介面圈住。
- **領域不被綁架**：表結構、框架註解、SDK 型別都進不了圓心，業務語言保持乾淨。
- **延後決策**：Clean Architecture 反覆強調的紅利——資料庫選型這類大決策可以晚點做，先寫核心。

**代價**

- **樣板碼倍增**：每個出站依賴一組介面＋實作＋領域物件映射；小功能也要過這套儀式。
- **間接層的閱讀成本**：追一個呼叫要多跳一次「介面→實作在哪」，新人容易迷路。
- **抽象可能白付**：資料庫十年沒換過的系統大有人在；為不會發生的替換付保費，就是過度設計。

**這改變了什麼實務決策**：問題從「要不要採用 Clean Architecture」變成「**這個模組的領域規則，值不值得一個不依賴世界的圓心**」。領域複雜、規則長壽、外部依賴多變的模組——值得；CRUD 透傳、活不過兩年的內部工具——分層就夠。翻轉可以局部進行：先讓最核心的一個模組擁有圓心，不必全系統一次改宗。

## 小結

- 六角、洋蔥、Clean 是同一個思想的三種畫法，核心只有一條依賴規則：依賴只指向內圈、領域不知道外部世界。
- 機制是 DIP 的架構級放大：port（介面）由圓心定義，adapter（實作）在外圈；呼叫方向不變，編譯依賴反轉。
- 資料庫從地基降級為外掛，與簡訊供應商平起平坐——這是它與分層架構的本質差異。
- 紅利是可測試與可替換，保費是樣板與間接層；為不會發生的替換付保費就是過度設計。
- 依賴翻轉解決了「誰依賴誰」，但模組之間**何時**互動仍是同步的——呼叫方還是得等。把時間也解耦掉，是另一種風格的主場：〈[事件驅動架構：解耦的代價](event-driven-architecture.md)〉。

## 常見面試題

1. **六角架構、洋蔥架構、Clean Architecture 有什麼差別？**（提示：陷阱題——說出「同一依賴規則的不同表述」比背出三張圖的差異更對）
2. **依賴反轉原則（DIP）和六角架構是什麼關係？**（提示：類別層級的原則放大到架構層級；介面的「所有權」歸內圈是關鍵）
3. **什麼樣的系統你不會用 Clean Architecture？**（提示：答得出「不用」的場景才證明懂取捨——CRUD 透傳、短命工具、團隊尚無守介面的紀律）

## 延伸閱讀

- Alistair Cockburn, [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)（2005）——Ports and Adapters 原文
- Robert C. Martin, [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)（2012）與《Clean Architecture》（2017）——依賴規則的完整論述
- Jeffrey Palermo, [The Onion Architecture](https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/)（2008）——洋蔥版本的表述
- 實作層面的對照：Spring 生態中 port／adapter 如何落地，見 knowledge-spring 相關筆記（規劃中）
