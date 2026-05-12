---
name: last72hours
version: "0.1.0"
description: "Run a 72-hour viral radar across Reddit, X, TikTok, Instagram, Hacker News, YouTube, and GitHub — then auto-build an editorial-scorecard leaderboard in Paper.design via its MCP server."
argument-hint: 'last72hours AI coding agents | last72hours nvidia earnings | last72hours <topic>'
allowed-tools: Bash, Read, Write, AskUserQuestion, WebSearch
homepage: https://github.com/iharnoor/last72hours-skill
repository: https://github.com/iharnoor/last72hours-skill
author: iharnoor
license: MIT
user-invocable: true
---

# /last72hours

A thin wrapper around [`/last30days`](https://github.com/mvanhorn/last30days-skill) that pins the time window to **72 hours**, then renders the results as an editorial-scorecard leaderboard in [Paper.design](https://paper.design) via the Paper MCP server.

## When to invoke this skill

User typed `/last72hours <topic>` — they want both the scrape AND the visual leaderboard. If they only want the raw scrape, redirect to `/last30days --days=3`.

If no topic is provided, ask the user a single short question and wait. Do NOT run the engine until a topic is set.

---

## Step 1 — Pre-flight

Run all checks in parallel:

1. **Confirm the `last30days` engine is installed**
   ```bash
   test -f "$HOME/.claude/skills/last30days/scripts/last30days.py" && echo OK || echo "MISSING"
   ```
   If missing: tell the user to install `/last30days` first (link to https://github.com/mvanhorn/last30days-skill) and stop.

2. **Check `.env`** — look in this order, source the first one found:
   - `$LAST72_ENV` (if set)
   - `$PWD/.env`
   - `~/.config/last72hours/.env`

   The keys this skill cares about: `SCRAPECREATORS_API_KEY` (IG/TikTok, paid), `AUTH_TOKEN` + `CT0` (X, free), or `XAI_API_KEY` (X, paid). Reddit/HN/GitHub/YouTube/Polymarket are free.

   **Never echo key values to the user. Mask with `sed -E 's/(=).*$/\1***SET***/'` when listing.**

3. **Probe Paper MCP**
   ```bash
   curl -s -o /dev/null -w "%{http_code}" -X POST -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cc","version":"1.0"}}}' \
     http://127.0.0.1:29979/mcp
   ```
   - HTTP 200 with body containing `paper-desktop` → Paper is up
   - Any other response → Paper Desktop isn't running (or has no document open). Tell the user, then ask if they want to (a) start Paper and continue, or (b) skip the viz step and just output the raw Markdown. Do not block on this — fall through gracefully.

---

## Step 2 — Run the engine

Set `LAST72_OUTPUT_DIR` (default: `$HOME/Documents/Last72Hours`). Make sure the dir exists.

```bash
set -a; source "$LAST72_ENV_FILE"; set +a
mkdir -p "$LAST72_OUTPUT_DIR"

python3 "$HOME/.claude/skills/last30days/scripts/last30days.py" "<TOPIC>" \
  --emit=compact \
  --days 3 \
  --save-dir="$LAST72_OUTPUT_DIR" \
  --save-suffix=72h \
  --plan "$(cat /tmp/last72_plan.json)"
```

You (the model) generate `/tmp/last72_plan.json` first with 2–4 subqueries matching the topic. The primary subquery MUST include sources `[reddit, x, hackernews, youtube, tiktok, instagram, polymarket]`. Set `freshness_mode: "strict_recent"`.

For person/product topics, also resolve and pass:
- `--subreddits=` 3–5 brand-specific + category-peer subs (see `last30days` Step 0.55 category table)
- `--x-handle=` primary handle
- `--tiktok-hashtags=`, `--ig-creators=`, `--github-user=` / `--github-repo=` when applicable

Use a **300-second timeout** on the Bash call. Read the entire output — it contains 7 source sections.

---

## Step 3 — Paper.design leaderboard

**If Paper MCP probe in Step 1 failed**, skip this step entirely. Print the raw-output path and stop.

**If Paper MCP is up**, follow this build order. All calls go through JSON-RPC at `http://127.0.0.1:29979/mcp`. Get a session ID from `initialize`, pass it as `mcp-session-id` on every subsequent call.

### Design brief (post to user BEFORE first mutation)

```
Mood: editorial / saturated (pure white × cobalt). First instinct was terminal/phosphor; editorial reads more authoritative for ranked content.
Palette: #FFFFFF ground · #0A0A0A ink · #0029FF cobalt accent · #6B6B6B mute · #E5E5E5 hairline.
Type: Archivo Black (display 96–288px), Inter Tight (body 13–18px), Paper Mono (numerics).
Direction: oversized "72H" mark; cobalt thesis bar; hero card for #1; 3-column grid for #2–#6; 2×2 narrative-cluster lozenges; source-coverage footer; clickable links appendix.
```

### Build order (one visual group per `write_html` call)

1. `create_artboard` — 1600 × `fit-content`, `padding:96px`, `gap:80px`, `flex column`, name `"AI Agent Radar — 72h"` (or `"<Topic> Radar — 72h"`)
2. `write_html` header: small cobalt dot + eyebrow caps + "72H" 288px display + right-aligned date / source counts
3. `write_html` cobalt thesis bar: 8px cobalt strip + `"The week in one line"` eyebrow + 48px headline that captures the dominant narrative
4. `write_html` section divider: `"Top viral · by engagement"` + `"Ranked 01 → 06"`
5. `write_html` hero card #1 (wide, asymmetric): big rank, platform dot, big headline, snippet, 3 stats columns, attribution
6. `write_html` 3-column grid of cards #2–#6: same shape as hero but smaller, share consistent top-border
7. `write_html` section divider: `"Narrative Clusters"` + `"Four threads · cross-platform"`
8. `write_html` 2×2 cluster grid (`#F7F7F5` background panels): cluster letter, count, headline, body, source chips
9. `write_html` source-coverage footer: per-source mini-stats (Reddit/X/TikTok/IG/HN/GitHub) with platform dots
10. `write_html` "Links · go look" appendix: 3 grouped lists (top viral / cluster A / cluster D) with `<a href>` URLs in cobalt
11. `finish_working_on_nodes` on the artboard ID
12. `export` `type:"image"`, `format:"png"`, `scale:"2x"`, save the returned path to `$LAST72_OUTPUT_DIR/<topic-slug>-72h-radar.png`

### Screenshot reviews

After step 5, step 8, and step 10: `get_screenshot` and quickly self-review for spacing, alignment, contrast. Fix issues with `update_styles` (don't delete + rewrite). If the artboard clips, set `height: "fit-content"`.

### Platform color dots (8px, used sparingly)

- X: `#000000` · Reddit: `#FF4500` · HN: `#FF6600` · IG: `#E1306C` · TikTok: `#25F4EE` · GitHub: `#181717`

Never fill whole cards with platform color. The viz must read as one editorial piece, not six brand silos.

---

## Step 4 — Wrap up

End your response with:
- Path to the raw Markdown
- Path to the exported PNG (if built)
- 3–5 bullet `Top viral signals` summary with engagement numbers
- 3–4 `Narrative clusters` one-liners

Do NOT trail with a `Sources:` block — the appendix in the Paper artboard and the engine footer in the raw `.md` are the citation.

---

## Secret hygiene

- Never echo, log, or commit `.env` values.
- When showing the user what's configured, always mask: `sed -E 's/(=).*$/\1***SET***/' .env`.
- If you see a key string that looks like a leaked credential in any tool output, redact it before quoting.

---

## Credits

Wraps [`/last30days`](https://github.com/mvanhorn/last30days-skill) by Matt Van Horn. Editorial-scorecard layout authored 2026-05-11.
