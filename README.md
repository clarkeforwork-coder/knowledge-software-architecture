# knowledge-software-architecture

![notes](https://img.shields.io/badge/notes-19-blue)
![deep dives](https://img.shields.io/badge/🔬_deep_dives-0-purple)
![status](https://img.shields.io/badge/roadmap-in_progress-yellow)

軟體架構筆記：從架構思維與品質屬性取捨出發，經架構風格、DDD、分散式系統基礎，
到可靠性與擴展性模式、架構決策與文件。談**模式與取捨**，不談特定產品的操作細節。

每篇的關鍵主張（數據、定理、案例）都標注可信出處——經典書、論文、官方文件、
原廠事故報告，不憑印象；與 [knowledge-java](../knowledge-java)、
[knowledge-spring](../knowledge-spring) 的實作篇互相連結，架構層與實作層前後呼應。

> Part of my [portfolio](https://github.com/clarkeforwork-coder/portfolio) —
> naming convention: `knowledge-*` = technical notes.

**特色**

- **雙軌制**：🔰 訓練主軌（廣而穩）＋ 🔬 深入選修軌（論文追讀、事故報告分析、實驗模擬）
- **有出處的論述**：關鍵主張標注來源，可執行的程式碼經實測——不憑印象
- **跨 repo 接續**：實作細節連回 knowledge-java／knowledge-spring，架構筆記專注在取捨

**範圍**：專注在軟體架構層級。特定產品的操作與調校（Redis 指令、Kafka 設定、
Docker/K8s 部署）與語言框架實作細節不在此 repo，由其他 `knowledge-*` repo 承接。

## 筆記深度：雙軌制

| 標記 | 軌道 | 說明 |
|---|---|---|
| 🔰 | 基礎（訓練主軌） | 概念說明、實務情境、常見陷阱。每章以寫完 🔰 軌為「完整」 |
| 🔬 | 深入（選修軌） | 論文與一手資料、事故報告分析、實驗模擬。檔名以 `deep-` 開頭，從對應的 🔰 筆記連結過去。寧缺勿濫 |

## 目錄

（章節資料夾在該章第一篇筆記落地時才建立；規劃中的筆記先列標題、狀態 📝。）

### 01 - 架構思維與權衡

| 筆記 | 深度 | 狀態 |
|---|---|---|
| [什麼是軟體架構？架構師在決定什麼](01-architecture-thinking/what-is-software-architecture.md) | 🔰 | ✅ |
| [品質屬性與取捨：沒有最好的架構，只有取捨](01-architecture-thinking/quality-attributes-and-tradeoffs.md) | 🔰 | ✅ |
| [康威定律：組織結構如何形塑系統](01-architecture-thinking/conways-law.md) | 🔰 | ✅ |

### 02 - 架構風格

| 筆記 | 深度 | 狀態 |
|---|---|---|
| [分層架構與它的邊界](02-architecture-styles/layered-architecture-and-boundaries.md) | 🔰 | ✅ |
| [分層物件家族：Entity、DO、DTO 的邊界在哪](02-architecture-styles/layered-object-family.md) | 🔰 | ✅ |
| [六角架構與 Clean Architecture：依賴方向的翻轉](02-architecture-styles/hexagonal-and-clean-architecture.md) | 🔰 | ✅ |
| [事件驅動架構：解耦的代價](02-architecture-styles/event-driven-architecture.md) | 🔰 | ✅ |
| [單體 vs 微服務：什麼時候該拆](02-architecture-styles/monolith-vs-microservices.md) | 🔰 | ✅ |

### 03 - DDD

| 筆記 | 深度 | 狀態 |
|---|---|---|
| [戰略設計：Bounded Context 與 Context Map](03-ddd/strategic-design-bounded-context.md) | 🔰 | ✅ |
| [戰術設計：Aggregate、Entity 與 Value Object](03-ddd/tactical-design-aggregate.md) | 🔰 | ✅ |
| [防腐層：與遺留系統共處](03-ddd/anti-corruption-layer.md) | 🔰 | ✅ |

### 04 - 分散式系統基礎

| 筆記 | 深度 | 狀態 |
|---|---|---|
| [CAP 與 PACELC：被誤讀最多的定理](04-distributed-basics/cap-and-pacelc.md) | 🔰 | ✅ |
| [一致性模型：從強一致到最終一致](04-distributed-basics/consistency-models.md) | 🔰 | ✅ |
| [冪等性：重試安全的前提](04-distributed-basics/idempotency.md) | 🔰 | ✅ |
| [分散式交易：2PC、Saga 與 Outbox](04-distributed-basics/distributed-transactions.md) | 🔰 | ✅ |

### 05 - 可靠性與擴展性模式

| 筆記 | 深度 | 狀態 |
|---|---|---|
| [快取策略：Cache-Aside、雪崩、穿透與擊穿](05-reliability-patterns/cache-strategies.md) | 🔰 | ✅ |
| [訊息佇列模式：削峰、解耦與順序性](05-reliability-patterns/message-queue-patterns.md) | 🔰 | ✅ |
| [限流與熔斷：保護系統的兩道閘門](05-reliability-patterns/rate-limiting-and-circuit-breaker.md) | 🔰 | ✅ |
| [Back-pressure：當下游跟不上](05-reliability-patterns/backpressure.md) | 🔰 | ✅ |

### 06 - 架構決策與文件

| 筆記 | 深度 | 狀態 |
|---|---|---|
| ADR：把「為什麼」留下來 | 🔰 | 📝 |
| C4 模型：畫給人看的架構圖 | 🔰 | 📝 |

## Roadmap

- **近期**：01 架構思維（✅ 完成）→ 02 架構風格（✅ 完成）
- **中期**：03 DDD（✅ 完成）→ 04 分散式系統基礎（✅ 完成）
- **長期**：05 可靠性與擴展性模式（✅ 完成）→ 06 架構決策與文件（⬅ 下一步）

## 慣例

- 每篇筆記一個主題，結構依 [TEMPLATE.md](TEMPLATE.md)：
  - 🔰 基礎：情境開場 → 概念段落（先程式碼或圖、後解釋）→ 重點 → 常見面試題 → 延伸閱讀
  - 🔬 深入：問題起點 → 追蹤過程 → 結論 → 回到實務
- 深度：🔰 基礎（訓練主軌）/ 🔬 深入（選修軌，檔名 `deep-` 開頭）
- 🔰 筆記結尾以「🔬 想深入：[標題](deep-xxx.md)」連到對應的深入筆記
- 可執行範例採混合制：預設文內 snippet 與 ASCII 圖；「不跑看不出結果」的主題
  才在該章 `examples/` 附單檔 .java（JDK 11+ 單檔執行）
- 狀態：✅ 完成（符合模板）/ 🔧 待翻新（內容完成，尚未符合 [TEMPLATE.md](TEMPLATE.md)）/
  🚧 進行中 / 📝 待補
