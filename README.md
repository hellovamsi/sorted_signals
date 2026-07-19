# Sorted Signals

n8n workflows for an AI news digest and newsletter pipeline.

This repo is a **skeleton**, not a finished newsletter. You get the workflow structure (12 workflows and sub-workflows in `sortedsignals.json`). You bring your own scraping targets and your own prompts so the output matches *your* taste, not mine.

## What’s included

[`sortedsignals.json`](./sortedsignals.json) — a single JSON export from a Docker-hosted n8n instance (12 workflows). Pipeline shape: scrape → digest → article → notify/error handling.

| Role | Workflow |
|------|----------|
| Master | Daily Digest |
| Master | Monday Roundup |
| Master | Generate Article |
| Sub | Run Mainstream Media RSS |
| Sub | Run Non-Mainstream Media RSS |
| Sub | Scrape Pages |
| Component | fetch standard rss |
| Component | scrape page |
| Component | generate digest |
| Component | generate article |
| Ops | Simple Telegram Error Notification |
| Ops | Error Fallback to File |

## What’s *not* included (on purpose)

- **Content scraping target lists** (RSS feeds, sites, URLs)
- **AI agent prompts**

You’ll need to add both. That’s intentional: shared sources and shared prompts produce the same digest everyone else has. Your list + your prompts = unique results suited to what you actually care about.

## Prerequisites

- A running [n8n](https://n8n.io/) instance (Docker is assumed below; self-hosted or cloud works too)
- Credentials for whatever AI / HTTP / notification nodes you wire up
- Your own source list and prompts (see above)

## Setup

### Option A — Import the whole export into Docker n8n

1. Clone or download this repo.
2. Copy the export into your n8n container:

```bash
docker cp ./sortedsignals.json <n8n-container-name>:/tmp/sortedsignals.json
```

3. Import it with the n8n CLI:

```bash
docker exec -u node <n8n-container-name> n8n import:workflow --input=/tmp/sortedsignals.json
```

4. Open the n8n UI, confirm the workflows appear, then:
   - Attach your credentials
   - Fill in your scraping targets
   - Write your prompts
   - Activate the workflows you want to run

Replace `<n8n-container-name>` with your container name or ID (`docker ps` if you’re unsure).

> If the import shape differs on your n8n version (single file vs. one-workflow-per-file), check `n8n import:workflow --help` inside the container and adjust the flags accordingly.

### Option B — Split the file with an AI agent

If you’d rather work with individual workflow files (easier to read, diff, and edit):

1. Give `sortedsignals.json` to an AI agent (Cursor, ChatGPT, Claude, etc.).
2. Ask it to split the export into one JSON file per workflow.
3. Import those files one by one in the n8n UI (**… → Import from File**) or via CLI.

Example prompt:

> Split sortedsignals.json into separate JSON files, one per workflow. Keep each file valid for n8n import. Name files after the workflow name.

## After import

1. **Sources** — Point the scrape/RSS nodes at *your* feeds and sites.
2. **Prompts** — Replace placeholder/empty prompt fields with instructions that match your voice, topics, and length limits.
3. **Credentials** — Reconnect AI models, Telegram (or whatever you use), and any APIs.
4. **Dry run** — Execute step by step before activating schedules.

## Status

Built and shared in the open as a learning project. Forks, issues, and suggestions welcome. No monetization.

Proper per-workflow GitHub sync is on the roadmap; for now `sortedsignals.json` is the source of truth.
