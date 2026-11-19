# 產品文件搜尋系統設計

> 目標：使用者可以用一句話搜尋公司的產品文件，例如
> - 「2026 年 CPU 發展」→ 依類別、年份篩選
> - 「Core Ultra 200 的組態文件」→ 依產品樹篩選
> - 「帶小螢幕的筆電」→ 依語意找到「雙螢幕筆電」的資料

[TOC]

---

## 一、全貌：從上傳到搜尋

```mermaid
flowchart TB
    subgraph S0["① 一次性準備"]
        P0["公司產品清單<br/>(ERP / PM 系統 / Excel)"] --> P1[("product 表<br/>產品樹")]
    end

    subgraph S1["② 上傳與處理（每份文件自動跑）"]
        U["使用者上傳<br/>ZenBook_Duo_規格書.pdf"] --> F[("檔案儲存<br/>NAS / S3")]
        U --> X["抽出文字<br/>PDF / Word / PPT"]
        X --> L["LLM 判斷基本資料<br/>產品、類型、年份、tags"]
        L --> R{"信心夠高？"}
        R -- "是" --> D[("document 表<br/>document_product 表")]
        R -- "否" --> H["人工確認畫面"] --> D
        X --> C["切成段落 chunk"]
        C --> E["Embedding 模型<br/>文字 → 向量"]
        E --> V[("chunk 表<br/>含向量")]
    end

    subgraph S2["③ 搜尋"]
        Q["使用者輸入<br/>「2026 年帶小螢幕的筆電」"] --> QL["LLM 解析查詢"]
        QL --> QF["結構化條件<br/>類別=筆電、年份=2026"]
        QL --> QS["語意部分<br/>「帶小螢幕」"]
        QS --> QE["Embedding 模型<br/>查詢 → 向量"]
        QF --> SQL["Hybrid Search<br/>WHERE 篩選 + 向量相似度排序"]
        QE --> SQL
        SQL --> RES["結果清單<br/>文件標題 + 命中段落 + 下載連結"]
    end

    P1 -.-> L
    P1 -.-> SQL
    D -.-> SQL
    V -.-> SQL
    F -.-> RES
```

| 階段 | 頻率 | 要人工嗎 |
|---|---|---|
| ① 建產品樹 | 一次性，新產品上市時補 | 要，但只是匯入現有清單 |
| ② 上傳處理 | 每份文件 | 不用，LLM 信心低時才需要人看 |
| ③ 搜尋 | 每次查詢 | 不用 |

---

## 二、資料結構

```mermaid
erDiagram
    product ||--o{ product : "parent_id（上下層）"
    product ||--o{ document_product : "被掛載"
    document ||--o{ document_product : "掛在"
    document ||--o{ chunk : "切成多段"

    product {
        int id PK
        int parent_id FK "上一層產品，最上層為空"
        text name "Intel / CPU / Core Ultra 9 285K"
        text kind "vendor / category / series / sku"
    }
    document {
        int id PK
        text title
        text file_uri "檔案在 NAS / S3 的位置"
        text doc_type "config / datasheet / roadmap"
        date published
        text_array tags "LLM 自動產生"
    }
    document_product {
        int document_id FK
        int product_id FK
    }
    chunk {
        int id PK
        int document_id FK
        text content "一段原文"
        vector embedding "1024 維向量"
    }
```

- **檔案本體不進資料庫**，資料庫只記 `file_uri`。
- **一份文件可以掛多個產品**（`document_product` 多對多），所以不用資料夾分類。
- **產品樹只記上一層**（`parent_id`），查詢時沿著樹往下展開，不需要在文件上重複貼「Intel」「CPU」。

### 產品樹範例

```mermaid
flowchart LR
    I["Intel"] --> CPU["CPU"] --> CU["Core Ultra 200"] --> K["Core Ultra 9 285K"]
    I --> GPU["GPU"] --> ARC["Arc B580"]
    A["ASUS"] --> NB["筆電"] --> DUO["ZenBook Duo"]
    NB --> ROG["ROG Zephyrus"]

    DOC1["📄 285K 組態文件"] -.-> K
    DOC2["📄 2026 CPU Roadmap"] -.-> CPU
    DOC3["📄 ZenBook Duo 規格書"] -.-> DUO
```

---

## 三、上傳處理：一份文件的流程

```mermaid
sequenceDiagram
    actor U as 使用者
    participant API as 上傳服務
    participant FS as NAS / S3
    participant LLM as LLM
    participant EMB as Embedding 模型
    participant DB as Postgres + pgvector

    U->>API: 上傳 ZenBook_Duo_規格書.pdf
    API->>FS: 存檔案本體
    FS-->>API: file_uri
    API->>API: 抽出全文
    API->>DB: 讀產品清單（給 LLM 參考）
    API->>LLM: 這份文件屬於哪個產品？類型？年份？
    LLM-->>API: {產品: ZenBook Duo, 類型: 規格書, 年份: 2025}
    API->>DB: 寫入 document、document_product
    API->>API: 全文切成段落
    loop 每個段落
        API->>EMB: 段落文字
        EMB-->>API: 向量
    end
    API->>DB: 寫入 chunk（文字 + 向量）
    API-->>U: 完成，請確認自動判斷的分類
```

> 把產品清單交給 LLM，讓它從清單裡挑選，而不是自由發揮，這樣產品名稱才會一致（不會出現「ZenBook Duo」和「Zenbook DUO」兩種寫法）。

---

## 四、搜尋：一次查詢的流程

```mermaid
sequenceDiagram
    actor U as 使用者
    participant API as 搜尋服務
    participant LLM as LLM
    participant EMB as Embedding 模型
    participant DB as Postgres + pgvector

    U->>API: 「2026 年帶小螢幕的筆電」
    API->>LLM: 拆成結構化條件 + 語意部分
    LLM-->>API: {類別: 筆電, 年份: 2026, 語意: "帶小螢幕"}
    API->>EMB: "帶小螢幕"
    EMB-->>API: 查詢向量
    API->>DB: 展開產品樹「筆電」以下所有節點
    API->>DB: WHERE 產品∈子樹 AND 年份=2026<br/>ORDER BY 向量距離
    DB-->>API: 命中段落「鍵盤上方有一塊副螢幕…」
    API-->>U: ZenBook Duo 規格書（附命中段落與下載連結）
```

### 為什麼「小螢幕」找得到「雙螢幕」？

```mermaid
flowchart LR
    Q["查詢：帶小螢幕的筆電"] --> QV["[0.11, -0.51, 0.89, …]"]
    A["鍵盤上方有一塊副螢幕"] --> AV["[0.12, -0.53, 0.88, …]"]
    B["雙螢幕設計提升多工效率"] --> BV["[0.10, -0.49, 0.91, …]"]
    C["電池續航 12 小時"] --> CV["[-0.71, 0.33, 0.02, …]"]

    QV -- "很近 ✅" --> AV
    QV -- "很近 ✅" --> BV
    QV -- "很遠 ❌" --> CV
```

Embedding 模型把「意思」轉成數字，意思相近的句子數字也相近。**沒有人貼過「雙螢幕」標籤**，靠模型本身理解。

---

## 五、三種查詢對應的 SQL

### 1. 某產品底下的組態文件（產品樹）

```sql
WITH RECURSIVE sub AS (
  SELECT id FROM product WHERE name = 'Core Ultra 200'
  UNION ALL
  SELECT p.id FROM product p JOIN sub ON p.parent_id = sub.id
)
SELECT DISTINCT d.*
FROM document d
JOIN document_product dp ON dp.document_id = d.id
WHERE dp.product_id IN (SELECT id FROM sub)
  AND d.doc_type = 'config';
```

### 2. 2026 年 CPU 發展（產品樹 + 年份 + 類型）

```sql
WITH RECURSIVE sub AS (
  SELECT id FROM product WHERE name = 'CPU'
  UNION ALL
  SELECT p.id FROM product p JOIN sub ON p.parent_id = sub.id
)
SELECT DISTINCT d.*
FROM document d
JOIN document_product dp ON dp.document_id = d.id
WHERE dp.product_id IN (SELECT id FROM sub)
  AND d.published BETWEEN '2026-01-01' AND '2026-12-31'
  AND d.doc_type IN ('roadmap', 'whitepaper');
```

### 3. 帶小螢幕的筆電（語意搜尋，可再疊加篩選）

```sql
SELECT d.title, c.content, c.embedding <=> :query_vec AS distance
FROM chunk c
JOIN document d ON d.id = c.document_id
ORDER BY c.embedding <=> :query_vec
LIMIT 20;
```

---

## 六、技術選型

| 元件 | 建議 | 何時升級 |
|---|---|---|
| 檔案儲存 | 現有 NAS 或 S3 / MinIO | — |
| 資料庫 | Postgres + pgvector（一套搞定） | chunk 破千萬筆再考慮專用向量資料庫 |
| 文字抽取 | PDF / Office 解析套件 | 掃描檔需要加 OCR |
| LLM | 分類與查詢解析 | — |
| Embedding | 支援中文的多語模型 | — |
| 產品關係 | `parent_id` 一層層往上記 | 需要「競品」「相容於」時加 `product_relation` 表 |

## 七、人工介入點

```mermaid
flowchart LR
    A["建產品樹<br/>一次性"] --> B["LLM 自動分類"]
    B --> C{"信心低 / 抽查"}
    C -- "需要" --> D["人工修正"]
    C -- "不需要" --> E["直接上線"]
    D --> E
    F["內部機密等級<br/>LLM 無法判斷"] --> G["上傳時選欄位<br/>或依來源系統自動帶入"]
    G --> E
```
