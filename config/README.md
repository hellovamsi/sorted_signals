# Sorted Signals — config seeds

CSV seeds for n8n **Data tables**. Live workflows read the **tables**, not these files.

**`master : first_setup`** (`7zvLKqdHQcjSzFky`) creates a missing table from the matching CSV and seeds it. If the table already exists, it leaves it alone. After install, edit the **data table** in n8n — not the CSV, and not the canvas.

Daily Digest and Monday Roundup look up enabled `rss_sources` / `scrape_sources` rows and call `component : fetch standard rss` / `component : scrape page` directly.

Do not put these CSVs under `/files/…`.

## `rss_sources.csv` → table `rss_sources`

Enabled rows only, ordered by `sort_order`. Split by `category`: `mainstream` vs `non_mainstream`.

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
| `wait_for_subworkflow` | boolean | Reserved; execute nodes keep a fixed wait setting |
| `sort_order` | number | Lookup order |

## `scrape_sources.csv` → table `scrape_sources`

Enabled rows only, ordered by `sort_order`. Keep `categories` and `examples` as JSON array **strings** (the workflow `JSON.parse`s them).

| Column | Type | Notes |
|--------|------|--------|
| `publication` | string | Shown as publication when saving articles |
| `url` | string | Listing page URL |
| `enabled` | boolean | `true` / `false` |
| `date_format` | string | Human hint for the scraper (e.g. `Month DD, YYYY`) |
| `date_pattern` | string | Regex string for dates on the page |
| `structure` | string | e.g. `Date-Category-Title` |
| `categories` | string | JSON array of category labels |
| `separators` | string | How fields are separated on the page |
| `extraction_hints` | string | Free-text hints for the LLM scraper |
| `examples` | string | JSON array of few-shot example lines |
| `wait_for_subworkflow` | boolean | Reserved |
| `sort_order` | number | Lookup order |

## `user_config.example.csv` → table `user_config`

Clone must-fill: Telegram chat ID, digest email, GitHub user. One row.

`master : setup` (`3enCZHWCsA3shIJi`) copies these three columns into `global_variables`. Keep both tables — other workflows read globals.

Do **not** commit real chat/email IDs. After first setup seeds placeholders, replace them in the live table, then run **master : setup**.

## `global_variables.example.csv` → table `global_variables`

Paths, lookbacks, and launch copy live as **columns on one row**.

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
| `error_report_model` | string | OpenRouter model for the daily node-error email |
| `supabase_articles_table` | string | Postgres table name (usually `articles`) |

Also: `tg_notifs_chat_id`, `g_email_id`, `backup_gh_owner`, `backup_gh_repo`, `backup_gh_path`. Those GitHub fields are the **backup destination** only. Install always pulls from https://github.com/hellovamsi/sorted_signals (`workflows/`). If the three backup fields are blank, `backup : github` skips the push and sends a Telegram error instead.

Prompt **file contents** stay on disk; only paths are table-driven.

## `llm_config.csv` → table `llm_config`

Model + temperature per AI job. Change a row to swap models without editing the canvas.

Used by:

- `component : fetch standard rss` — `rss_ai_gate`, `rss_summarizer`
- `component : scrape page` — `scrape_extractor`, `scrape_classifier`
- `component : generate` — `article_writer`, `digest_writer`

| Column | Type | Notes |
|--------|------|--------|
| `purpose` | string | Unique job key |
| `provider` | string | e.g. `openrouter` |
| `model` | string | Full OpenRouter model slug |
| `temperature` | number | Leave blank for provider default; `0.8` for writers |
| `response_format` | string | Optional; writers use `text` |
| `notes` | string | Human hint |

API keys stay in the n8n credential vault.

## `text_classifier.csv` → table `text_classifier`

RSS “is this AI news?” gate in `component : fetch standard rss`. Port **names** stay hardcoded on the node.

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
