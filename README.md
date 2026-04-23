# Mo's Library

A personal book search engine powered by [Meilisearch](https://www.meilisearch.com/), built for the MGT 858 Database Systems full-text search assignment.

Browse, search, and filter a personal book collection imported from Goodreads — deployed on [Vercel](https://vercel.com/) with a serverless proxy that keeps the search API key out of the browser.

**Live demo:** _deployed via Vercel_

---

## Features

- **Instant full-text search** across titles, authors, descriptions, and tags
- **Faceted filtering** by genre, shelf status, rating, publication year, and tags
- **Rich result cards** with descriptions, star ratings, page counts, and Goodreads links
- **Responsive design** with a bookish Lora serif typeface
- **Search state in the URL** — shareable links to any query + filter combination

## Tech stack

| Layer | Technology |
|-------|-----------|
| Search engine | Meilisearch (class-hosted) |
| Search UI | InstantSearch.js + instant-meilisearch (CDN) |
| API proxy | Vercel Serverless Function (`api/meili.js`) |
| Data pipeline | Python + Open Library API |
| Deployment | Vercel (rewrites for SPA routing) |

## Project structure

```
├── api/
│   └── meili.js               # Vercel function — proxies search requests to Meilisearch
├── sample-data/
│   ├── books.json             # Enriched book data (uploaded to Meilisearch)
│   └── yale-som-courses-spring-2026.json
├── scripts/
│   ├── build-books-json.py    # Convert Goodreads CSV → enriched JSON via Open Library
│   ├── upload.py              # Python upload helper
│   └── upload.mjs             # Node upload helper
├── search-app/
│   ├── favicon.svg
│   └── index.html             # Single-file search application
└── vercel.json                # Rewrites: / → search-app, /api/meili/* → serverless fn
```

## Getting started

### Prerequisites

From the course dashboard you'll need:

- `hw-meilisearch-url`
- `hw-meilisearch-index`
- `hw-meilisearch-write-key`
- `hw-meilisearch-search-key`

### 1. Build book data from Goodreads

Export your Goodreads library as a CSV, then enrich it with descriptions from Open Library:

```bash
python3 scripts/build-books-json.py \
  sample-data/goodreads_library_export.csv \
  sample-data/books.json
```

The script fetches book descriptions and subject tags from [Open Library](https://openlibrary.org/), assigns genres, cleans tags, and writes a ready-to-upload JSON file.

### 2. Upload to Meilisearch

Set environment variables and upload:

```bash
export MEILI_URL='http://search.858.mba:7700'
export MEILI_INDEX='your-index-from-the-dashboard'
export MEILI_WRITE_KEY='your-write-key-from-the-dashboard'

python3 scripts/upload.py sample-data/books.json --reset
```

Or with Node:

```bash
node scripts/upload.mjs sample-data/books.json --reset
```

Optional flags for custom schemas:

```bash
python3 scripts/upload.py my-data.json --reset \
  --primary-key slug \
  --filterable category \
  --filterable tags
```

### 3. Run locally

Open `search-app/index.html` in a browser and enter:

- Meilisearch URL
- Index name
- **Search-only** API key

### 4. Deploy to Vercel

The app is configured to deploy on Vercel. Set these environment variables in the Vercel dashboard:

- `MEILI_URL` — the Meilisearch host
- `MEILI_SEARCH_KEY` — the search-only API key

The serverless function at `api/meili.js` proxies search requests so the search key never reaches the browser.

## Book data schema

Each document in the Meilisearch index:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier (`book-{goodreadsId}`) |
| `title` | string | Book title |
| `author` | string | Author name |
| `description` | string | Enriched from Open Library |
| `publisher` | string | Publisher name |
| `year` | int | Original publication year |
| `yearRange` | string | Bucketed range (e.g. `2010–2019`) |
| `pages` | int | Page count |
| `myRating` | int | Personal rating (0–5) |
| `shelf` | string | Goodreads shelf (`read`, `to-read`, `currently-reading`) |
| `genre` | string | Auto-assigned genre |
| `tags` | string[] | Cleaned, de-duplicated subject tags |
| `dateRead` | string | Date finished |
| `url` | string | Goodreads link |

## Customizing the search UI

Edit `search-app/index.html`. Key entry points:

- **`renderHit()`** — controls how each result card is displayed
- **`startSearch()`** — add InstantSearch widgets (refinement lists, range sliders, etc.)

Example — adding a genre filter:

```js
widgets.panel({ templates: { header: 'Genre' } })(
  widgets.refinementList({ container: '#custom-filters', attribute: 'genre' })
)
```

## Security

- The **write key** is only used in terminal upload scripts — never in the browser.
- The Vercel serverless function restricts proxied paths to search-only endpoints.
- The **search-only key** is stored as a server-side environment variable, not exposed to the client.
