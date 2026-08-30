# Sorted Signals

An agentic AI-news newsletter system on [n8n](https://n8n.io/). It pulls headlines from RSS feeds and blog listing pages, keeps AI-related stories, stores summaries in [Supabase](https://supabase.com/), and writes a **daily email** plus a **weekly long article file**.

This repo is a **skeleton**. Swap feeds and prompts so the output matches your taste.

Built in public on [Trello](https://trello.com/b/5ttJxOKc/sorted-signals). Demo: self-hosted n8n Community Edition.

**Three products** (each is its own master workflow):

| Product | What you get |
|---------|----------------|
| Daily Digest | HTML email, every night |
| Monday Roundup | Markdown article on disk (not an email) |
| Generate Article | On-demand markdown from stories already in the database |

## What’s in this repo

| Path | What it is |
|------|------------|
| [`docker-compose.yml`](./docker-compose.yml) + [`.env.example`](./.env.example) | n8n **and** the [task runner](https://docs.n8n.io/hosting/configuration/task-runners/) (Code nodes need the runner). No secrets. |
| [`workflows/`](./workflows) | One JSON file per workflow. Nightly backup. Includes **`master : first_setup`**. |
| [`config/`](./config) | Data-table seed CSVs. First setup creates a missing table from the matching CSV. |
| [`prompts/`](./prompts) | Writer instructions. First setup writes a missing file onto the host. |

**Not in the repo:** API keys, live chat/email, paywalled feed URLs, production table dumps. There is no `master : setup` workflow.

## Prerequisites

- n8n **1.113.1+** (Data tables) with **external task runners** on self-hosted CE. Install: [n8n Docker](https://docs.n8n.io/hosting/installation/docker/) and [task runners](https://docs.n8n.io/hosting/configuration/task-runners/).
- [Supabase](https://supabase.com/docs/guides/getting-started) project
- Accounts: [OpenRouter](https://openrouter.ai/), [Jina](https://jina.ai/), [Telegram bot](https://core.telegram.org/bots/tutorial), Gmail OAuth, [GitHub PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens), [n8n API key](https://docs.n8n.io/api/authentication/) with permission to **create, update, delete, and tag** workflows (a read-only key is not enough)

This repo only adds compose mounts, seeds, prompts, and workflow JSON. Do not follow a generic n8n tutorial for those extras.

## Self-hosted (Community Edition)

1. Clone this repo. Copy `.env.example` → `.env`. Run `docker compose up -d`.
   - HTTPS / reverse proxy: follow [n8n hosting](https://docs.n8n.io/hosting/) — we bind `127.0.0.1:5678` on purpose.
   - Host dirs under `./files` map to `/files` in the containers (`batchupdate/prompts|summaries|articles`, `sortedsignals/logs`).
2. In n8n, add [credentials](https://docs.n8n.io/integrations/builtin/credentials/): OpenRouter, Jina AI, Supabase, Telegram, Gmail, GitHub, n8n API. Names like **Telegram account** / **Gmail account** match the JSON; attach them after import.
3. Import **only** [`workflows/master-first-setup__7zvLKqdHQcjSzFky.json`](./workflows/master-first-setup__7zvLKqdHQcjSzFky.json) ([import from file](https://docs.n8n.io/workflows/export-import/)). Attach **GitHub** and **n8n API** credentials on that workflow.
   - If this n8n is **not** `https://app.easyn8n.work/`, point the installer’s create/tag HTTP nodes at **your** n8n URL before you run.
4. Run first setup once. It reads `config/`, `prompts/`, and `workflows/` from [hellovamsi/sorted_signals](https://github.com/hellovamsi/sorted_signals) on `main` — not from your fork, and not from the files on your laptop.
   - Missing tables get created and seeded. Missing prompt files get written. Missing workflows get created, tagged `Sorted_Signals`, and left **unpublished**.
   - Existing tables, prompt files, and workflow names are skipped. Lookups and “call another workflow” nodes are retargeted by name.
   - Credentials are **not** copied onto new workflows — attach them after. First setup does **not** probe GitHub/Telegram/Gmail, and it does **not** copy `user_config` into `global_variables`. Telegram is only for an optional “done” message.
   - If the n8n API key cannot create workflows, you get HTTP 403: tables and prompts still install; workflow files are skipped.
5. Put your Telegram chat ID, digest email, and GitHub owner in **`user_config` and `global_variables`**. Products read globals. [How to get a Telegram chat ID](https://core.telegram.org/bots/tutorial).
6. In Supabase, create an `articles` table ([SQL editor](https://supabase.com/docs/guides/database/overview)). Columns: `id`, `title`, `url`, `publication`, `published_date`, `description`, `snippet`, `ai_or_no`, `category`, `subcategories`, `ai_summary`, `created_at`. Set `global_variables.supabase_articles_table` if you used another name.
7. **Publish** the workflows you want to run. Set **Error Workflow** to `error : notify error over telegram` if it did not stick. Enable **1–2** RSS feeds first.
8. Run Daily Digest once. Then turn on schedules.

Do **not** hand-create the six n8n tables or copy prompt files. First setup does that. Do **not** import every JSON in `workflows/` by hand — first setup pulls those.

**GitHub PAT:** `contents:read` so first setup can download this repo; `contents:write` if you use nightly backup.

## n8n Cloud

[n8n Cloud](https://docs.n8n.io/manage-cloud/overview/) already runs Code nodes — skip compose. **Not smoke-tested.**

Same idea: credentials → import and run `master : first_setup` from `workflows/` → fill `user_config` **and** `global_variables` → Supabase `articles`.

Cloud extras:

- File nodes only allow **`/home/node/…`**. After first setup, change `prompt_*` / `dir_*` in `global_variables` off `/files/batchupdate/…`.
- Disk under `/home/node/` may vanish on redeploy — re-run first setup (it skips files that still exist).
- Hardcoded error-log paths under `/files/sortedsignals/logs/` will not work; Telegram still will.
- Prefer Pro+ RAM; keep few feeds enabled at first.

## How it works

A schedule (or you) starts one product workflow. It reads settings and source lists from n8n tables, harvests RSS and blog listings, waits until that work finishes, then asks the writer to turn stored AI summaries into text.

- Daily Digest **emails** that text.
- Monday Roundup and Generate Article **save a markdown file**. They do not email.

Change feeds and models in **tables**, not on the canvas. Hard crashes ping Telegram. Soft ingest failures go into a daily log and an evening summary email.

Nightly, `backup : github` writes tagged workflow JSON to **your** fork (`backup_gh_*`). **Install always pulls from** [hellovamsi/sorted_signals](https://github.com/hellovamsi/sorted_signals), not from your fork.

## Workflows

Tag: `Sorted_Signals`. IDs below match the JSON filenames in this repo. After first setup on *your* n8n, those IDs will be different; first setup rewires by **name**.

### Products and ops

| Product | Canvas name | ID in this repo | When | Output |
|---------|-------------|-----------------|------|--------|
| Daily Digest | `master : send daily digest on gmail` | `LuYOE5Yzbl9azUqZ` | Daily 00:30 UTC | Gmail HTML |
| Monday Roundup | `master : generate article on disk` | `TjoB5EbSxgiL7Klv` | Sunday 01:00 UTC | Markdown file, **no email** |
| Generate Article | `master : generate article manually` | `SeqfvIBhBPBi5oLL` | Manual (off by default) | Markdown file |
| First setup | `master : first_setup` | `7zvLKqdHQcjSzFky` | Manual | Missing tables, prompts, and workflows from GitHub |
| Node error report | `master : daily node error report` | `qMf0noQMQ3vPD6kg` | Daily 18:00 UTC | Gmail if the day’s log exists |

### Building blocks

| Canvas name | ID in this repo | Role |
|-------------|-----------------|------|
| `component : fetch standard rss` | `1k5LcDuD476rkM0s` | One RSS feed → AI filter → summary → Supabase |
| `component : scrape page` | `Qy8xlKqvJk27GyCe` | One listing page → articles → summary → Supabase |
| `component : generate` | `S4ap9r37QHoR730E` | Shared writer (digest or article) |
| `component : append node error` | `mGKudn2060Ph8LoW` | One ingest error line on the daily log |
| `backup : github` | `mHFCJIFLGDvshgOj` | Nightly 01:33 export to your fork |
| `error : notify error over telegram` | `goB83Sc0EKWZkYZx` | Crash → Telegram (or a log file if Telegram is down) |

Ignore old names if you see them: `master : setup`, `import : github`, `sub :` RSS/scrape wrappers, twin article writer, file-only error workflow.

## After install

Edit **tables**, not the canvas:

- `user_config` **and** `global_variables` — chat, email, GitHub owner (products read globals)
- `rss_sources` / `scrape_sources` — flip `enabled`, or add a row (that is how you add a feed)
- `llm_config` — models
- `global_variables` — lookback, paths, `backup_gh_*` for your fork

Credentials stay in the n8n vault.

To pick up a **newer** workflow JSON: delete that workflow, then re-run **`master : first_setup`** (it never overwrites an existing name). First setup itself is always skipped.

Schedules in this repo are written as UTC. Set timezone in the [n8n UI](https://docs.n8n.io/hosting/configuration/environment-variables/timezone/).

## Troubleshooting

- Empty digest → no AI rows yet, or `lookback_days_daily` is too small.
- Lookup empty / raw `={{ … }}` on classifier ports → re-run first setup so table IDs remap.
- No Telegram → bot has not talked to you, or chat ID is still a placeholder in `global_variables`.
- Code node timeouts → the **task-runners** container is not up.
- First setup created tables but no workflows → the n8n API key cannot create workflows (HTTP 403). Give it full access and re-run.

## License

[MIT](./LICENSE). Formerly “Batch Update”; some disk paths still use `/files/batchupdate/`.
