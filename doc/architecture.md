# go-rss-reader - Architecture

Last updated: 2026-10-06

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    Main[cmd/cli main] --> App[internal/app TUI Layer]
    App --> Collector[internal/util Collector]
    App --> Extractor[internal/util Extractor]
    App --> API[internal/api LLM Client]
    App --> DB[internal/database SQLite]
    Collector --> DB
    Collector --> Feeds[(RSS Feeds)]
    Extractor --> Web[(News Pages)]
    API --> OpenAI[(OpenAI Chat Completions)]
    Collector -.-> Model[internal/model Data Models]
    Extractor -.-> Model
    DB -.-> Model
```

## Module: App

Builds the tview interface, handles hotkeys and commands, and orchestrates collection, extraction, storage, and the digest.

```mermaid
graph TB
    subgraph App
        New[New] --> Frame[frame layout]
        New --> Refresh[refresh 5-minute ticker]
        Frame --> Listener[listener hotkeys]
        Listener --> Command[command parser]
        Listener --> GetList[getList fetch news]
        Refresh --> GetList
        GetList --> LoadContent[loadContent full text and digest]
        Frame --> ShowPreview[showPreview]
        ShowPreview --> ShowFull[showFull full text]
        ShowPreview --> ShowBasic[showBasicPreview RSS summary]
        Listener --> OpenBrowser[openBrowser]
    end
    Command --> Collector[Collector]
    GetList --> Collector
    GetList --> DB[SQLite]
    LoadContent --> Extractor[Extractor]
    LoadContent --> API[api.AskWithSmallModel]
    ShowPreview --> Extractor
    ShowPreview --> DB
```

## Module: Collector

Manages feeds and fetches, parses, deduplicates, and filters RSS items.

```mermaid
graph TB
    subgraph Collector
        Add[Add / Remove / List] --> FeedOps[Feed operations]
        GetNews[GetNews] --> Fetch[fetch HTTP GET + XML decode]
        Fetch --> Dedup[Deduplicate by URL]
        Dedup --> ParseDate[parseDate multi-format]
        ParseDate --> Window[Skip items older than 72h]
        Window --> Clean[clean strip HTML and unescape]
        Clean --> Sort[Sort by publish time]
    end
    FeedOps --> DB[(feeds table)]
    GetNews --> DB
    Fetch --> Feeds[(RSS Feeds)]
```

## Module: Extractor

Extracts title, author, and body from a news page and counts words.

```mermaid
graph TB
    subgraph Extractor
        Get[Get] --> Parse[goquery parse HTML]
        Parse --> Strip[Remove script / style / nav / aside / footer / ads / comments]
        Strip --> Title[Title: h1 → title]
        Strip --> Author[Author: rel=author / .author / itemprop=author]
        Strip --> Content[getContent]
        Content --> Article{article length > 128}
        Article -->|No| Main{main length > 128}
        Main -->|No| Div[Longest div with link ratio < 30%]
        Content --> Normalize[clean collapse whitespace]
        Content --> Count[count Han chars + English words]
    end
    Get --> Web[(News Pages)]
```

## Module: Database

SQLite access layer: resolves the database location, creates tables, and handles CRUD.

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
    subgraph Database Path Resolution
        Env{RSS_DB_PATH set} -->|Yes| Custom[Use given path]
        Env -->|No| Dev{go.mod in working dir or GO_ENV=development}
        Dev -->|Yes| WD[working dir/rss.db]
        Dev -->|No| Writable{Executable dir writable}
        Writable -->|Yes| ExeDir[executable dir/rss.db]
        Writable -->|No| Home[~/.rss-reader/rss.db]
    end
```

## Module: API

Calls OpenAI Chat Completions in streaming mode, parses SSE line by line, and assembles the reply.

```mermaid
graph TB
    subgraph API
        Small[AskWithSmallModel gpt-4o-mini] --> Ask[askWithChatGPT]
        Large[AskWithLargeModel gpt-4o] --> Check{ApiKey length ≥ 20}
        Check -->|Yes| Ask
        Ask --> Post[POST /v1/chat/completions stream=true]
        Post --> SSE[Read data: lines]
        SSE --> Concat[Concatenate delta.content]
    end
    Post --> OpenAI[(OpenAI API)]
```

## Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant A as App
    participant C as Collector
    participant D as SQLite
    participant E as Extractor
    participant L as OpenAI
    U->>A: Launch / Ctrl+R / 5-minute ticker
    A->>D: Get(72) (show cache first when list is empty)
    A->>C: GetNews()
    C->>D: GetFeed()
    C->>C: Fetch and parse each RSS feed
    C-->>A: News within 72h
    A->>D: GetFromURL() to detect new articles
    A-->>U: Update news list
    loop Each unextracted article (500ms apart)
        A->>E: Get(url)
        E-->>A: Title / author / body / word count
        A->>D: Insert(news, content)
    end
    A->>D: GetKey("apikey") / GetKey("summary")
    A->>L: Previous digest + all fetched articles
    L-->>A: New digest (streamed)
    A->>D: SetKey("summary")
    A-->>U: Update Summary pane
```

## State Machine

```mermaid
stateDiagram-v2
    [*] --> Loading: Run
    Loading --> Listed: Cache loaded / fetch done
    Listed --> Checking: Ctrl+R / ticker
    Checking --> Listed: No new articles
    Checking --> Extracting: New articles found
    Checking --> Failed: Fetch failed
    Extracting --> Summarizing: Extraction done
    Summarizing --> Listed: Digest updated / error shown
    Failed --> Checking: Ctrl+R / ticker
    Listed --> [*]: Ctrl+C
```

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
