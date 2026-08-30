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
| [`workflows/`](./workflows) | One JSON file per workflow (nightly backup) |
| [`config/`](./config) | Data-table seed CSVs. **`master : first_setup`** loads these. |
| [`prompts/`](./prompts) | Writer instructions. First setup copies them onto the host. |

**Not in the repo:** API keys, live chat/email, paywalled feed URLs, production table dumps.

## Prerequisites

- n8n **1.113.1+** (Data tables) with **external task runners** on self-hosted CE. Install: [n8n Docker](https://docs.n8n.io/hosting/installation/docker/) and [task runners](https://docs.n8n.io/hosting/configuration/task-runners/).
- [Supabase](https://supabase.com/docs/guides/getting-started) project
- Accounts: [OpenRouter](https://openrouter.ai/), [Jina](https://jina.ai/), [Telegram bot](https://core.telegram.org/bots/tutorial), Gmail OAuth, [GitHub PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens), [n8n API key](https://docs.n8n.io/api/authentication/)

This repo only adds compose mounts, seeds, prompts, and workflow JSON. Do not follow a generic n8n tutorial for those extras.

## Self-hosted (Community Edition)

1. Clone this repo. Copy `.env.example` → `.env`. Run `docker compose up -d`.
   - HTTPS / reverse proxy: follow [n8n hosting](https://docs.n8n.io/hosting/) — we bind `127.0.0.1:5678` on purpose.
   - Host dirs under `./files` map to `/files` in the containers (`batchupdate/prompts|summaries|articles`, `sortedsignals/logs`).
2. In n8n, add [credentials](https://docs.n8n.io/integrations/builtin/credentials/): OpenRouter, Jina AI, Supabase, Telegram, Gmail, GitHub, n8n API. Names like **Telegram account** / **Gmail account** match the JSON; re-link if import does not pick them up.
3. Import **`master : first_setup`** (`7zvLKqdHQcjSzFky`) from `workflows/` ([import from file](https://docs.n8n.io/workflows/export-import/)).
4. Run it once. It creates missing data tables from `config/`, writes missing prompt files, imports missing workflows, and rewires table/workflow IDs. Existing tables and files are left alone. Needs **GitHub** + **n8n API** (and Telegram if you want the install report).
5. Fill `user_config` (chat ID, email, GitHub owner). [How to get a Telegram chat ID](https://core.telegram.org/bots/tutorial). Then run **`master : setup`** so those values copy into `global_variables` and credentials are probed.
6. In Supabase, create an `articles` table ([SQL editor](https://supabase.com/docs/guides/database/overview)). Columns: `id`, `title`, `url`, `publication`, `published_date`, `description`, `snippet`, `ai_or_no`, `category`, `subcategories`, `ai_summary`, `created_at`. Set `global_variables.supabase_articles_table` if you used another name.
7. **Publish** the workflows. Set **Error Workflow** to `error : notify error over telegram` if it did not stick. Enable **1–2** RSS feeds first.
8. Run Daily Digest once. Then turn on schedules.

Do **not** hand-create the six n8n tables or copy prompt files. First setup does that.

**GitHub PAT:** `contents:read` for first setup / setup import; `contents:write` if you use nightly backup.

## n8n Cloud

[n8n Cloud](https://docs.n8n.io/manage-cloud/overview/) already runs Code nodes — skip compose. **Not smoke-tested.**

Same idea: credentials → import and run `master : first_setup` → fill `user_config` → `master : setup` → Supabase `articles`.

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

Nightly, `backup : github` writes workflow JSON to **your** fork (`backup_gh_*`). **Install always pulls from** [hellovamsi/sorted_signals](https://github.com/hellovamsi/sorted_signals), not from your fork.

## Workflows

Tag: `Sorted_Signals`.

### Products and ops

| Product | Canvas name | ID | When | Output |
|---------|-------------|-----|------|--------|
| Daily Digest | `master : send daily digest on gmail` | `SoIEMUBKJPynE2jV` | Daily 00:30 UTC | Gmail HTML |
| Monday Roundup | `master : generate article on disk` | `rzmkSSHMbbThfLWV` | Sunday 01:00 UTC | Markdown file, **no email** |
| Generate Article | `master : generate article manually` | `Kv362LANnXzDNbHv` | Manual (off by default) | Markdown file |
| First setup | `master : first_setup` | `7zvLKqdHQcjSzFky` | Manual | Tables, prompts, missing workflows |
| Setup | `master : setup` | `3enCZHWCsA3shIJi` | Manual | Copy `user_config` → globals, probe creds, import any still-missing workflows |
| Node error report | `master : daily node error report` | `Y41e9Eh2nmZoCTrN` | Daily 18:00 UTC | Gmail if the day’s log exists |

### Building blocks

| Canvas name | ID | Role |
|-------------|-----|------|
| `component : fetch standard rss` | `X5drlaLIiyY5VOYf` | One RSS feed → AI filter → summary → Supabase |
| `component : scrape page` | `go4FNonP2GmrCEHc` | One listing page → articles → summary → Supabase |
| `component : generate` | `S6jCKcFVnVwE0bNX` | Shared writer (digest or article) |
| `component : append node error` | `ySr3LZRexU7vXCBD` | One ingest error line on the daily log |
| `backup : github` | `qbjCpsc1xuN49iX1` | Nightly 01:33 export to your fork |
| `error : notify error over telegram` | `zTMJFr8EYWX9glAC` | Crash → Telegram (or a log file if Telegram is down) |

Ignore old names if you see them: `sub :` RSS/scrape wrappers, twin article writer, file-only error workflow, `import : github`.

## After install

Edit **tables**, not the canvas:

- `user_config` — chat, email, GitHub owner, then re-run **`master : setup`**
- `rss_sources` / `scrape_sources` — flip `enabled`, or add a row (that is how you add a feed)
- `llm_config` — models
- `global_variables` — lookback, paths, `backup_gh_*` for your fork

Credentials stay in the n8n vault.

To pick up a **newer** workflow JSON: delete that workflow, then re-run **`master : first_setup`** (it never overwrites).

Schedules in this repo are written as UTC. Set timezone in the [n8n UI](https://docs.n8n.io/hosting/configuration/environment-variables/timezone/).

## Troubleshooting

- Empty digest → no AI rows yet, or `lookback_days_daily` is too small.
- Lookup empty / raw `={{ … }}` on classifier ports → re-run first setup so table IDs remap.
- No Telegram → bot has not talked to you, or chat ID is still a placeholder.
- Code node timeouts → the **task-runners** container is not up.

## License

[MIT](./LICENSE). Formerly “Batch Update”; some disk paths still use `/files/batchupdate/`.
