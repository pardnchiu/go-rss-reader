# go-rss-reader - 技術文件

最後更新：2026-10-06

> 返回 [README](./README.zh.md)

## 前置需求

- Go 1.24.3 或更高版本
- C 編譯器（gcc／clang）並啟用 CGO：`github.com/mattn/go-sqlite3` 為 CGO 套件
- 支援色彩顯示的終端機
- OpenAI API 金鑰（僅 LLM 新聞概要需要，閱讀功能不需要）
- 開啟原文所需的系統指令：macOS `open`、Linux `xdg-open`、Windows `cmd /c start`

## 安裝

### 從原始碼建置

```bash
git clone https://github.com/pardnchiu/go-rss-reader.git
cd go-rss-reader
CGO_ENABLED=1 go build -o RSSReader ./cmd/cli
```

### 直接執行

```bash
git clone https://github.com/pardnchiu/go-rss-reader.git
cd go-rss-reader
go run ./cmd/cli
```

`go.mod` 的 module 名稱為 `rss-reader`（非 GitHub 路徑），因此不支援 `go install github.com/pardnchiu/go-rss-reader/...`。

## 設定

### 環境變數

| 變數 | 必要 | 預設 | 說明 |
|------|------|------|------|
| `RSS_DB_PATH` | 否 | 見下方資料庫位置 | 指定 SQLite 資料庫檔案的完整路徑，優先於所有自動判斷 |
| `GO_ENV` | 否 | — | 設為 `development` 時視為開發環境，資料庫建立於目前工作目錄 |

### 資料庫位置

未設定 `RSS_DB_PATH` 時，依下列順序決定 `rss.db` 位置：

| 條件 | 路徑 |
|------|------|
| 目前工作目錄存在 `go.mod`，或 `GO_ENV=development` | `<工作目錄>/rss.db` |
| 執行檔所在目錄可寫入 | `<執行檔目錄>/rss.db` |
| 其他 | `~/.rss-reader/rss.db`（自動建立目錄） |

### API 金鑰

API 金鑰不讀環境變數，而是於介面指令欄輸入 `apikey <KEY>` 寫入資料庫 `data` 表，下次產生概要時載入。

## 使用方式

### 基礎

```bash
./RSSReader
```

首次啟動沒有任何訂閱源。按 `Tab` 將焦點切到 **Command** 欄，新增訂閱源後按 `Ctrl+R` 抓取：

```text
add https://feeds.bbci.co.uk/news/rss.xml
```

在 **News List** 以 `↑`／`↓` 移動，**Preview** 會即時顯示萃取後的全文；按 `Ctrl+O` 以預設瀏覽器開啟原文。

### 管理訂閱源與金鑰

```text
add https://www.theguardian.com/world/rss
rm https://www.theguardian.com/world/rss
apikey sk-xxxxxxxxxxxxxxxxxxxxxxxx
config
```

`config` 會在 Preview 列出目前 API 金鑰與所有訂閱源。`rm`／`remove` 為軟刪除（`dismiss = 1`），文章紀錄保留。

### 進階：LLM 新聞概要

1. 以 `apikey` 設定 OpenAI 金鑰
2. `Ctrl+R` 或等待 5 分鐘自動檢查；當發現新文章時，背景逐篇萃取全文（每篇間隔 500ms）並寫入資料庫
3. 萃取完成後，將前次概要與本次抓取的全部文章（72 小時內的標題、來源、發布時間與 RSS 描述）送交 `gpt-4o-mini`，結果顯示於 **Summary** 欄並存回資料庫
4. 首次產生概要（無前次概要）時，會額外帶入資料庫中近 24 小時的文章

概要固定以繁體中文輸出，分為「重大要聞」「科技與金融」「生活資訊」「趨勢分析」四段。沒有新文章時不會觸發概要更新。

### 推薦訂閱源

#### BBC News（英文）

| 分類 | 連結 |
|------|------|
| 綜合新聞 | https://feeds.bbci.co.uk/news/rss.xml |
| 商業財經 | https://feeds.bbci.co.uk/news/business/rss.xml |
| 娛樂藝術 | https://feeds.bbci.co.uk/news/entertainment_and_arts/rss.xml |
| 健康醫療 | https://feeds.bbci.co.uk/news/health/rss.xml |
| 科學環境 | https://feeds.bbci.co.uk/news/science_and_environment/rss.xml |
| 科技資訊 | https://feeds.bbci.co.uk/news/technology/rss.xml |
| 國際新聞 | https://feeds.bbci.co.uk/news/world/rss.xml |
| BBC 中文 | https://feeds.bbci.co.uk/zhongwen/trad/rss.xml |

#### The Guardian（英文）

| 分類 | 連結 |
|------|------|
| 國際新聞 | https://www.theguardian.com/world/rss |
| 科學 | https://www.theguardian.com/science/rss |
| 政治 | https://www.theguardian.com/politics/rss |
| 商業財經 | https://www.theguardian.com/uk/business/rss |
| 科技 | https://www.theguardian.com/uk/technology/rss |
| 環境 | https://www.theguardian.com/uk/environment/rss |
| 理財 | https://www.theguardian.com/uk/money/rss |

#### 台灣媒體（繁體中文）

| 媒體 | 連結 |
|------|------|
| 自由時報 | https://news.ltn.com.tw/rss/all.xml |
| 聯合新聞網 | https://udn.com/rssfeed/news/2/6638?ch=news |
| ETtoday 新聞雲 | https://feeds.feedburner.com/ettoday/news |

## 命令列參考

### 啟動

| 指令 | 說明 |
|------|------|
| `./RSSReader` | 啟動 TUI；無任何 flag 或子命令 |

### 快捷鍵

| 按鍵 | 說明 |
|------|------|
| `Tab` | 依序切換焦點：News List → Preview → Command → Summary |
| `Ctrl+R` | 立即檢查所有訂閱源 |
| `Ctrl+O` | 以預設瀏覽器開啟目前選取的新聞 |
| `↑`／`↓` | 在 News List 移動；在 Preview／Summary 捲動 |
| `Enter` | 於 Command 欄執行指令 |
| `Ctrl+C` | 結束程式 |

### 介面內指令

| 指令 | 語法 | 說明 |
|------|------|------|
| `add` | `add <URL>` | 新增 RSS 訂閱源（已移除者會重新啟用） |
| `rm`／`remove` | `rm <URL>` | 停用 RSS 訂閱源 |
| `apikey` | `apikey <API_KEY>` | 儲存 OpenAI API 金鑰 |
| `config` | `config` | 顯示 API 金鑰與訂閱源列表 |

### 介面區塊

| 區塊 | 內容 |
|------|------|
| 狀態列 | 目前狀態、下次自動檢查時間、快捷鍵提示 |
| Summary | LLM 新聞概要（啟動時載入上次結果） |
| News List | 近 72 小時新聞，依發布時間由新到舊，副標為 `MM/DD HH:mm \| 來源` |
| Preview | 標題、作者、來源、發布時間、字數、連結與全文；萃取失敗時退回 RSS 描述 |
| Command | 指令輸入欄 |

### 行為參數

| 項目 | 值 |
|------|-----|
| 自動檢查間隔 | 5 分鐘 |
| 新聞保留視窗 | 72 小時（早於此時間的 RSS 項目略過） |
| HTTP 逾時 | 30 秒（RSS 與全文萃取） |
| 全文萃取間隔 | 每篇 500ms |
| 概要模型 | `gpt-4o-mini`（串流） |

### 資料表

| 資料表 | 欄位 | 用途 |
|--------|------|------|
| `news` | `title`、`url`（UNIQUE）、`content`、`full_content`、`source`、`author`、`word_count`、`published_at`、`created_at` | 文章與萃取全文 |
| `feeds` | `url`（UNIQUE）、`dismiss`、`created_at`、`updated_at` | 訂閱源（`dismiss = 1` 為已移除） |
| `data` | `key`（UNIQUE）、`value` | `apikey`、`summary` |

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
