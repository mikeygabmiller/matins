# Matins — what this repo is, and how to switch it off

This repo is only the **published output**. Nothing here generates anything.

Every morning a Cloudflare Worker called **`matins`** builds the day's issue,
writes it to KV, and then commits the result into this repo (that's why the same
commit message appears five times a day — one commit per file it publishes).
GitHub Pages serves what lands here.

- Site: <https://mikeygabmiller.github.io/matins/>
- Generator: Cloudflare Worker `matins` (account: Mikey's) — **the Worker's source
  code is not in any GitHub repo.** It was deployed straight from a machine. Read
  the deployed bundle with the Cloudflare dashboard, or `wrangler download`.
- Publish target: `SITE_REPO=mikeygabmiller/matins`, `SITE_BRANCH=main`.

## Switching it off (paused 2026-08-25)

Two different switches — pick by how much you want stopped.

**Stop the whole daily run, AI calls included** — Cloudflare dashboard →
Workers & Pages → **`matins`** → Settings → **Triggers** → delete the cron.
The Worker stays deployed and every URL keeps serving; it just stops waking up.
This is the one that ends the spending. A later `wrangler deploy` from wherever
the source lives will recreate the trigger from its `wrangler.toml`, so if the
pause is meant to last, delete the trigger *after* any redeploy.

**Stop only the email, keep building the issue** — Settings → **Variables** →
set `SEND_PAUSED = 1`. (`1` is already the built-in default, so unless a
`SEND_PAUSED=0` var is set on the Worker, subscriber email is *already* off and
the daily cost is entirely the build.)

To resume: re-add a cron trigger. The handler re-checks the local hour against
`SEND_HOUR` (5, Pacific) on every firing and returns immediately when it isn't the
right one, so an hourly `0 * * * *` is the form that can't drift — a fixed UTC
hour would move an hour off target every daylight-saving change.

## What a run costs, and what it has actually been producing

One build asks the model for **three drafts of the headline, two of the
reflection and two of the saint story, then a judging call for each block** — ten
generation calls on a good day, more when a draft gets rejected and the round is
retried. `LLM_MODEL` is set to `gemini-3.6-flash`, which is several times the
price of the Flash model the texting dashboard uses, and each call carries a long
system prompt plus exemplars. Per call it is the most expensive AI in the whole
setup.

And it has not worked once. Every issue in `issues/` from 2026-08-06 to
2026-08-24 is `"status": "partial"` with `"reflection": null` — the reflection,
saint story and headline calls come back either

```
gemini 400: { "error": { "code": 400, "message": "Request contains an invalid argument." } }
```

or, on the days they do return, blocked by the safety pass as truncated
(`"The text is incomplete, cutting off mid-sentence"`). What has been publishing
every morning is the free half: readings, the verse, a prayer and a Q&A from the
local rotation.

So the daily run has been paying for a model that rejects it and shipping a
half-finished issue. If Matins comes back, fix that first — the 400 almost
certainly means the request body carries a field `gemini-3.6-flash` won't accept
(`GEMINI_THINKING_BUDGET` and the `thinking` field it controls are the first
place to look), and it is worth confirming a build produces a complete issue
before the cron goes back on.
