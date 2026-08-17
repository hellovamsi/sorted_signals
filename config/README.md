# Sorted Signals — config seeds

CSV seeds for n8n **Data tables**. These are for **fresh installs** (import into a table once). Live workflows read the data table at runtime — they do **not** read these files from disk.

## `rss_sources.csv` → table `rss_sources`

Feeds for:

- `sub : Run Mainstream Media RSS` (`category = mainstream`)
- `sub : Run Non-Mainstream Media RSS` (`category = non_mainstream`)

Both loop `component : fetch standard rss` once per enabled row.

### Columns

| Column | Type | Notes |
|--------|------|--------|
| `publication_name` | string | Shown as publication when saving articles |
| `rss_url` | string | Feed URL |
| `enabled` | boolean | `true` / `false` — toggle without editing the canvas |
| `category` | string | `mainstream` or `non_mainstream` |
| `content_url_field` | string | RSS item path for article URL (usually `link`) |
| `content_id_field` | string | Optional id path (e.g. `guid._`) |
| `title_field` | string | Usually `title` |
| `description_field` | string | Optional; leave blank if unused |
| `snippet_field` | string | Optional |
| `pub_date_field` | string | Usually `pubDate` (cutoff still uses RSS `isoDate`) |
| `image_url_field` | string | Optional |
| `wait_for_subworkflow` | boolean | Reserved / informational; execute nodes keep a fixed wait setting |
| `sort_order` | number | Lookup order |

### Load on a fresh n8n instance

1. Create a Data table named **`rss_sources`** with the columns above (types as in the table).
2. Open the table → import / paste rows from `config/rss_sources.csv` (or add rows manually to match the CSV).
3. Confirm booleans imported as booleans (`true`/`false`), not the strings `"true"`/`"false"`, if your import path is picky.
4. Publish the Sorted Signals RSS workflows (or import them from the public backup). They look up this table by name/id — after import, point the **Lookup rss_sources** nodes at your new table if the id differs.
5. Smoke-test: run a master (or either RSS sub) with a recent `cutOffDate` and confirm articles still land in Supabase.

### Not for runtime

Do **not** put this CSV under `/files/…` and wire a Read File node. That old pattern is retired from the canvas. If `/files/batchupdate/rss-sources.csv` still exists on the host, delete it manually — workflows no longer read it. Edit the **data table** to add, pause, or change a feed.

## `scrape_sources.csv` → table `scrape_sources`

Sites for `sub : Scrape Pages`. The sub loops `component : scrape page` once per enabled row.

### Columns

| Column | Type | Notes |
|--------|------|--------|
| `publication` | string | Shown as publication when saving articles |
| `url` | string | Listing page URL |
| `enabled` | boolean | `true` / `false` — toggle without editing the canvas |
| `date_format` | string | Human hint for the scraper (e.g. `Month DD, YYYY`) |
| `date_pattern` | string | Regex string for dates on the page |
| `structure` | string | e.g. `Date-Category-Title` |
| `categories` | string | JSON array of category labels |
| `separators` | string | How fields are separated on the page |
| `extraction_hints` | string | Free-text hints for the LLM scraper |
| `examples` | string | JSON array of few-shot example lines |
| `wait_for_subworkflow` | boolean | Reserved / informational; execute node keeps a fixed wait setting |
| `sort_order` | number | Lookup order |

### Load on a fresh n8n instance

1. Create a Data table named **`scrape_sources`** with the columns above.
2. Import / paste rows from `config/scrape_sources.csv`.
3. Keep `categories` and `examples` as JSON array **strings** (the workflow `JSON.parse`s them).
4. Point **Lookup scrape_sources** at your table if the id differs after import.
5. Smoke-test: run Scrape Pages (or a master that calls it) with a recent `cutOffDate`.

### Not for runtime

Edit the **data table** to add, pause, or change a site — not the canvas.

## `user_config.example.csv` → table `user_config`

What a clone **must** fill: Telegram chat ID, digest email, GitHub user. One row.

`master : setup` (`3enCZHWCsA3shIJi`) copies these three columns onto the matching fields in `global_variables`. Keep both tables — other workflows still read globals.

Do **not** commit real chat/email IDs. Live table `user_config` (`FsQNE0iq4GWOPJkd`) holds the real row.

### Load on a fresh n8n instance

1. Create table **`user_config`** with `tg_notifs_chat_id`, `g_email_id`, `backup_gh_owner` (all strings).
2. Paste one row from `config/user_config.example.csv` and replace the placeholders.
3. Run **master : setup** so globals pick up the values.

## `global_variables.example.csv` → table `global_variables`

Scalar ops settings (paths, lookbacks, launch copy) live as **columns on the existing `global_variables` table** — same table as Telegram/email/GitHub. No separate path/runtime tables.

### Ops columns (add these if missing)

| Column | Type | Example / meaning |
|--------|------|-------------------|
| `prompt_article_path` | string | `/files/batchupdate/prompts/article_gen.md` |
| `prompt_digest_path` | string | `/files/batchupdate/prompts/digest_gen.md` |
| `dir_summaries` | string | `/files/batchupdate/summaries` |
| `dir_articles` | string | `/files/batchupdate/articles` |
| `prefix_summary` | string | `summary_` |
| `prefix_article` | string | `gen-article-` |
| `prefix_digest` | string | `gen-digest-` |
| `lookback_days_daily` | number | Daily Digest cutoff: today minus N days |
| `ingest_week_start_offset_days` | number | Monday Roundup ingest: start of week minus N days |
| `article_cutoff_hours` | number | Generate Article window: now minus N hours |
| `display_tz_offset_hours` | number | Telegram “local” clock offset from UTC |
| `launch_message_daily` | string | Daily Digest launch Telegram line |
| `launch_message_roundup` | string | Monday Roundup launch Telegram line |
| `launch_message_article` | string | Generate Article launch Telegram line |

Also keep Phase 1 columns: `tg_notifs_chat_id`, `g_email_id`, `backup_gh_owner`, `backup_gh_repo`, `backup_gh_path`. Those GitHub fields are the **backup destination** only. `master : setup` always imports from https://github.com/hellovamsi/sorted_signals (`workflows/`). If the three backup GitHub fields are blank or placeholders, `backup : github` skips the push and sends a Telegram error instead.

### Load on a fresh n8n instance

1. Open **`global_variables`** (or create it) and add any missing columns from the table above.
2. Copy values from `config/global_variables.example.csv` into **one** row — replace placeholders (`YOUR_TELEGRAM_CHAT_ID`, `you@example.com`, etc.). Do **not** commit real chat/email IDs.
3. Confirm masters + generate article/digest already look up this table (Lookup global variables). After import, retarget the table id if it differs.
4. Smoke-test: edit `launch_message_daily` or `lookback_days_daily`, run Daily Digest (or a short Generate Article), confirm Telegram copy / cutoff / file paths follow the table.

### Not for runtime

Prompt **file contents** stay on disk; only paths are table-driven. Credentials stay in the n8n vault.

## `llm_config.csv` → table `llm_config`

Which OpenRouter model (and temperature) each AI job uses. Change a row here (in the **data table**) to swap models without editing the canvas.

Used by:

- `component : fetch standard rss` — `rss_ai_gate`, `rss_summarizer`
- `component : scrape page` — `scrape_extractor`, `scrape_classifier`
- `component : generate article` — `article_writer`
- `component : generate digest` — `digest_writer`

### Columns

| Column | Type | Notes |
|--------|------|--------|
| `purpose` | string | Unique job key (see seed rows) |
| `provider` | string | e.g. `openrouter` |
| `model` | string | Full OpenRouter model slug |
| `temperature` | number | Leave blank for provider default; `0.8` for writers |
| `response_format` | string | Optional; writers use `text` |
| `notes` | string | Human hint |

### Load on a fresh n8n instance

1. Create a Data table named **`llm_config`** with the columns above.
2. Import / paste rows from `config/llm_config.csv`.
3. Point each workflow’s **Lookup llm_config** node at your table if the id differs after import.
4. Smoke-test: change `rss_ai_gate` or `article_writer` model, run the matching component once, confirm the new model is used.

### Not for runtime

API keys stay in the n8n credential vault. Prompt text stays in nodes or on disk — only model settings live here.

## `text_classifier.csv` → table `text_classifier`

Labels and batch settings for the RSS “is this AI news?” gate in `component : fetch standard rss`.

### Columns

| Column | Type | Notes |
|--------|------|--------|
| `pipeline` | string | e.g. `standard_rss` |
| `categories_json` | string | JSON array of label strings |
| `batch_size` | number | Classifier parallel batch size |
| `delay_ms` | number | Delay between batches |
| `ai_flag_yes` | number | Supabase `ai_or_no` when AI Related (`1`) |
| `ai_flag_no` | number | When Not AI Related (`-1`) |
| `ai_flag_unsure` | number | When Unsure (`0`) |
| `notes` | string | Human hint |

### Load on a fresh n8n instance

1. Create a Data table named **`text_classifier`** with the columns above.
2. Import / paste rows from `config/text_classifier.csv` (keep `categories_json` as a JSON array **string**).
3. Point **Lookup text_classifier** in `fetch standard rss` at your table if the id differs.
4. Smoke-test: run one feed item through classify and confirm labels / `ai_or_no` still look right.
