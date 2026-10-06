# go-rss-reader - Documentation

Last updated: 2026-10-06

> Back to [README](../README.md)

## Prerequisites

- Go 1.24.3 or higher
- A C compiler (gcc/clang) with CGO enabled: `github.com/mattn/go-sqlite3` is a CGO package
- A terminal with color support
- An OpenAI API key (only for the LLM news digest; reading works without it)
- A system command to open links: `open` on macOS, `xdg-open` on Linux, `cmd /c start` on Windows

## Installation

### Build from Source

```bash
git clone https://github.com/pardnchiu/go-rss-reader.git
cd go-rss-reader
CGO_ENABLED=1 go build -o RSSReader ./cmd/cli
```

### Run Directly

```bash
git clone https://github.com/pardnchiu/go-rss-reader.git
cd go-rss-reader
go run ./cmd/cli
```

The module name in `go.mod` is `rss-reader` (not the GitHub path), so `go install github.com/pardnchiu/go-rss-reader/...` is not supported.

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RSS_DB_PATH` | No | See Database Location | Full path to the SQLite database file; overrides all automatic resolution |
| `GO_ENV` | No | — | Set to `development` to treat the run as development and create the database in the current working directory |

### Database Location

Without `RSS_DB_PATH`, the location of `rss.db` resolves in this order:

| Condition | Path |
|-----------|------|
| `go.mod` exists in the current working directory, or `GO_ENV=development` | `<working dir>/rss.db` |
| The executable's directory is writable | `<executable dir>/rss.db` |
| Otherwise | `~/.rss-reader/rss.db` (directory created automatically) |

### API Key

The API key is not read from the environment. Enter `apikey <KEY>` in the command pane to store it in the `data` table; it loads the next time a digest is generated.

## Usage

### Basic

```bash
./RSSReader
```

The first launch has no feeds. Press `Tab` to focus the **Command** pane, add a feed, then press `Ctrl+R` to fetch:

```text
add https://feeds.bbci.co.uk/news/rss.xml
```

Move through **News List** with `↑`/`↓`; **Preview** shows the extracted full text as you go. Press `Ctrl+O` to open the original article in your default browser.

### Manage Feeds and Key

```text
add https://www.theguardian.com/world/rss
rm https://www.theguardian.com/world/rss
apikey sk-xxxxxxxxxxxxxxxxxxxxxxxx
config
```

`config` lists the current API key and all feeds in Preview. `rm`/`remove` is a soft delete (`dismiss = 1`) that keeps stored articles.

### Advanced: LLM News Digest

1. Set the OpenAI key with `apikey`
2. Press `Ctrl+R` or wait for the 5-minute auto check; when new articles appear, the app extracts full text for each in the background (500ms apart) and stores it
3. After extraction, the previous digest plus every article from this fetch (title, source, publish time, and RSS description within the 72-hour window) go to `gpt-4o-mini`; the result shows in the **Summary** pane and is saved back to the database
4. On the first digest (no previous digest), articles from the last 24 hours in the database are included as well

The digest is always written in Traditional Chinese, in four sections: major news, tech & finance, lifestyle, and trend analysis. No new articles means no digest update.

### Recommended Feeds

#### BBC News (English)

| Category | Link |
|----------|------|
| General News | https://feeds.bbci.co.uk/news/rss.xml |
| Business | https://feeds.bbci.co.uk/news/business/rss.xml |
| Entertainment & Arts | https://feeds.bbci.co.uk/news/entertainment_and_arts/rss.xml |
| Health | https://feeds.bbci.co.uk/news/health/rss.xml |
| Science & Environment | https://feeds.bbci.co.uk/news/science_and_environment/rss.xml |
| Technology | https://feeds.bbci.co.uk/news/technology/rss.xml |
| World News | https://feeds.bbci.co.uk/news/world/rss.xml |
| BBC Chinese | https://feeds.bbci.co.uk/zhongwen/trad/rss.xml |

#### The Guardian (English)

| Category | Link |
|----------|------|
| World News | https://www.theguardian.com/world/rss |
| Science | https://www.theguardian.com/science/rss |
| Politics | https://www.theguardian.com/politics/rss |
| Business | https://www.theguardian.com/uk/business/rss |
| Technology | https://www.theguardian.com/uk/technology/rss |
| Environment | https://www.theguardian.com/uk/environment/rss |
| Money | https://www.theguardian.com/uk/money/rss |

#### Taiwan Media (Traditional Chinese)

| Media | Link |
|-------|------|
| Liberty Times | https://news.ltn.com.tw/rss/all.xml |
| United Daily News | https://udn.com/rssfeed/news/2/6638?ch=news |
| ETtoday | https://feeds.feedburner.com/ettoday/news |

## CLI Reference

### Launch

| Command | Description |
|---------|-------------|
| `./RSSReader` | Start the TUI; takes no flags or subcommands |

### Hotkeys

| Key | Description |
|-----|-------------|
| `Tab` | Cycle focus: News List → Preview → Command → Summary |
| `Ctrl+R` | Check all feeds now |
| `Ctrl+O` | Open the selected article in the default browser |
| `↑`/`↓` | Move in News List; scroll in Preview/Summary |
| `Enter` | Run the command in the Command pane |
| `Ctrl+C` | Quit |

### In-App Commands

| Command | Syntax | Description |
|---------|--------|-------------|
| `add` | `add <URL>` | Add an RSS feed (re-enables a removed one) |
| `rm`/`remove` | `rm <URL>` | Disable an RSS feed |
| `apikey` | `apikey <API_KEY>` | Store the OpenAI API key |
| `config` | `config` | Show the API key and feed list |

### Panes

| Pane | Content |
|------|---------|
| Status bar | Current status, next auto-check time, hotkey hints |
| Summary | LLM news digest (last result loads on start) |
| News List | News from the last 72 hours, newest first, subtitle `MM/DD HH:mm \| source` |
| Preview | Title, author, source, publish time, word count, link, and full text; falls back to the RSS description when extraction fails |
| Command | Command input |

### Behavior Parameters

| Item | Value |
|------|-------|
| Auto-check interval | 5 minutes |
| News window | 72 hours (older RSS items are skipped) |
| HTTP timeout | 30 seconds (RSS and full-text extraction) |
| Extraction interval | 500ms per article |
| Digest model | `gpt-4o-mini` (streaming) |

### Tables

| Table | Columns | Purpose |
|-------|---------|---------|
| `news` | `title`, `url` (UNIQUE), `content`, `full_content`, `source`, `author`, `word_count`, `published_at`, `created_at` | Articles and extracted text |
| `feeds` | `url` (UNIQUE), `dismiss`, `created_at`, `updated_at` | Feeds (`dismiss = 1` means removed) |
| `data` | `key` (UNIQUE), `value` | `apikey`, `summary` |

***

©️ 2025 [邱敬幃 Pardn Chiu](https://www.linkedin.com/in/pardnchiu)
