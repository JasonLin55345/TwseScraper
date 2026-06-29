# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automated TWSE (Taiwan Stock Exchange) stock price scraper with a Chart.js frontend deployed to GitHub Pages at https://linyos.github.io/TwseScraper.

## Build

```bash
dotnet build TwseScraper.sln
dotnet build TwseScraper.sln --configuration Release
```

Run the scraper locally:
```bash
dotnet run --project src/TwseScraper.Console
```

No test projects exist in this solution.

## Architecture

Clean Architecture with 4 layers — dependencies only flow inward:

```
Console → Infrastructure → Application → Domain
```

- **Domain**: Entities, value objects, interfaces. No external dependencies.
- **Application**: Use case orchestration (`ScrapeStockPriceUseCase`). Depends on Domain interfaces only.
- **Infrastructure**: TWSE API client, JSON file persistence. Implements Domain interfaces.
- **Console**: Entry point and manual composition root (`Program.cs`). No DI framework — wire dependencies by hand here.

Do not add an IoC container (e.g. Microsoft.Extensions.DependencyInjection) without discussion. Dependencies are intentionally assembled manually in `Program.cs`.

## Key Gotchas

**Timezone**: All date recording must use `"Taipei Standard Time"` (not UTC). Scraping runs in a UTC environment (GitHub Actions on Ubuntu); the use case explicitly converts to TST before recording dates. Preserve this conversion in any new date logic.

**files.json manifest**: The frontend discovers data files through `data/files.json`. Any operation that adds, removes, or renames files in `data/` must update this manifest — the repository does this automatically in `SaveAsync()`. If you write code that directly manipulates data files, also update `files.json`.

**File splitting at 700 MB**: `JsonFileStockPriceRepository` splits output files when they exceed 700 MB. Files are named `stock_2330_001.json`, `stock_2330_002.json`, etc. Don't assume a single data file per stock.

**Value object validation**: `StockCode` and `StockPrice` (sealed records) validate inputs in their constructors and throw on invalid data. Always construct them through their public constructors, not via reflection or JSON deserialization bypass.

## CI/CD

GitHub Actions (`csharp_scrape.yml`) runs the scraper at 06:00 UTC (14:00 Taipei time) on weekdays and commits results directly to `main`. Workflow requires `contents: write` permission and uses `action@github.com` as committer.

Commit message format used by CI: `📊 更新股價資料 YYYY-MM-DD`

This repo pushes directly to `main` — no feature branches or PRs.

## Frontend

Static HTML/JS/CSS served from GitHub Pages. `index.html` + `script.js` + `styles.css` live at the repo root. Chart.js is loaded from CDN. `script.js` fetches `data/files.json` first, then loads individual data files based on user selection.

Configuration lives in `src/TwseScraper.Console/appsettings.json` — no environment variables required.
