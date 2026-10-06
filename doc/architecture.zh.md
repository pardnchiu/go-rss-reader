# go-rss-reader - 架構

最後更新：2026-10-06

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    Main[cmd/cli main] --> App[internal/app TUI 應用層]
    App --> Collector[internal/util 收集器]
    App --> Extractor[internal/util 萃取器]
    App --> API[internal/api LLM 用戶端]
    App --> DB[internal/database SQLite]
    Collector --> DB
    Collector --> Feeds[(RSS 訂閱源)]
    Extractor --> Web[(新聞網頁)]
    API --> OpenAI[(OpenAI Chat Completions)]
    Collector -.-> Model[internal/model 資料模型]
    Extractor -.-> Model
    DB -.-> Model
```

## 模組：App

建立 tview 介面、處理鍵盤事件與指令，並協調收集、萃取、儲存與概要流程。

```mermaid
graph TB
    subgraph App
        New[New] --> Frame[frame 版面配置]
        New --> Refresh[refresh 5 分鐘 ticker]
        Frame --> Listener[listener 快捷鍵]
        Listener --> Command[command 指令解析]
        Listener --> GetList[getList 取得新聞]
        Refresh --> GetList
        GetList --> LoadContent[loadContent 全文與概要]
        Frame --> ShowPreview[showPreview 預覽]
        ShowPreview --> ShowFull[showFull 全文顯示]
        ShowPreview --> ShowBasic[showBasicPreview 摘要顯示]
        Listener --> OpenBrowser[openBrowser 開啟原文]
    end
    Command --> Collector[Collector]
    GetList --> Collector
    GetList --> DB[SQLite]
    LoadContent --> Extractor[Extractor]
    LoadContent --> API[api.AskWithSmallModel]
    ShowPreview --> Extractor
    ShowPreview --> DB
```

## 模組：Collector

管理訂閱源並抓取、解析、去重、篩選 RSS 項目。

```mermaid
graph TB
    subgraph Collector
        Add[Add / Remove / List] --> FeedOps[訂閱源操作]
        GetNews[GetNews] --> Fetch[fetch HTTP GET + XML 解析]
        Fetch --> Dedup[依 URL 去重]
        Dedup --> ParseDate[parseDate 多格式日期解析]
        ParseDate --> Window[略過 72 小時前項目]
        Window --> Clean[clean 去 HTML 標籤與跳脫字元]
        Clean --> Sort[依發布時間排序]
    end
    FeedOps --> DB[(feeds 資料表)]
    GetNews --> DB
    Fetch --> Feeds[(RSS 訂閱源)]
```

## 模組：Extractor

從新聞網頁萃取標題、作者與正文，並計算字數。

```mermaid
graph TB
    subgraph Extractor
        Get[Get] --> Parse[goquery 解析 HTML]
        Parse --> Strip[移除 script / style / nav / aside / footer / 廣告 / 留言]
        Strip --> Title[標題：h1 → title]
        Strip --> Author[作者：rel=author / .author / itemprop=author]
        Strip --> Content[getContent]
        Content --> Article{article 長度 > 128}
        Article -->|否| Main{main 長度 > 128}
        Main -->|否| Div[最長 div 且連結比例 < 30%]
        Content --> Normalize[clean 壓縮空白]
        Content --> Count[count 中文字 + 英文單字]
    end
    Get --> Web[(新聞網頁)]
```

## 模組：Database

SQLite 存取層，負責資料庫位置判斷、建表與 CRUD。

```mermaid
classDiagram
    class SQLite {
        -db *sql.DB
        +Insert(news, content) error
        +Get(hours) []News, error
        +GetFromURL(url) *News, error
        +InsertFeed(url) error
        +RemoveFeed(url) error
        +GetFeed() []string, error
        +GetKey(key) string, error
        +SetKey(key, value) error
        +Close() error
    }
    class News {
        Title string
        Content string
        Source string
        URL string
        PublishedAt time.Time
        FullContent *string
        Author *string
        WordCount *int
    }
    class NewsContent {
        Title string
        Author string
        Content string
        WordCount int
    }
    SQLite ..> News
    SQLite ..> NewsContent
```

```mermaid
graph TB
    subgraph 資料庫路徑判斷
        Env{RSS_DB_PATH 已設定} -->|是| Custom[使用指定路徑]
        Env -->|否| Dev{工作目錄有 go.mod 或 GO_ENV=development}
        Dev -->|是| WD[工作目錄/rss.db]
        Dev -->|否| Writable{執行檔目錄可寫入}
        Writable -->|是| ExeDir[執行檔目錄/rss.db]
        Writable -->|否| Home[~/.rss-reader/rss.db]
    end
```

## 模組：API

以串流方式呼叫 OpenAI Chat Completions，逐行解析 SSE 並組合回應。

```mermaid
graph TB
    subgraph API
        Small[AskWithSmallModel gpt-4o-mini] --> Ask[askWithChatGPT]
        Large[AskWithLargeModel gpt-4o] --> Check{ApiKey 長度 ≥ 20}
        Check -->|是| Ask
        Ask --> Post[POST /v1/chat/completions stream=true]
        Post --> SSE[逐行讀取 data: 區塊]
        SSE --> Concat[串接 delta.content]
    end
    Post --> OpenAI[(OpenAI API)]
```

## 資料流

```mermaid
sequenceDiagram
    participant U as 使用者
    participant A as App
    participant C as Collector
    participant D as SQLite
    participant E as Extractor
    participant L as OpenAI
    U->>A: 啟動 / Ctrl+R / 5 分鐘 ticker
    A->>D: Get(72)（列表為空時先顯示快取）
    A->>C: GetNews()
    C->>D: GetFeed()
    C->>C: 抓取並解析各 RSS
    C-->>A: 72 小時內新聞
    A->>D: GetFromURL() 判斷新文章
    A-->>U: 更新新聞列表
    loop 每篇未萃取文章（間隔 500ms）
        A->>E: Get(url)
        E-->>A: 標題 / 作者 / 正文 / 字數
        A->>D: Insert(news, content)
    end
    A->>D: GetKey("apikey") / GetKey("summary")
    A->>L: 前次概要 + 本次抓取全部文章
    L-->>A: 新概要（串流）
    A->>D: SetKey("summary")
    A-->>U: 更新 Summary 欄
```

## 狀態機

```mermaid
stateDiagram-v2
    [*] --> Loading: Run
    Loading --> Listed: 載入快取 / 抓取完成
    Listed --> Checking: Ctrl+R / ticker
    Checking --> Listed: 無新文章
    Checking --> Extracting: 發現新文章
    Checking --> Failed: 抓取失敗
    Extracting --> Summarizing: 全文萃取完成
    Summarizing --> Listed: 概要更新 / 顯示錯誤
    Failed --> Checking: Ctrl+R / ticker
    Listed --> [*]: Ctrl+C
```

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
