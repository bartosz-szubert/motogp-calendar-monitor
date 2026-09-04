# MotoGP 2027 Calendar Monitor

An n8n agent that watches for changes in the MotoGP 2027 calendar and messages me on Telegram **only when something actually changed**.

Running since August 2026.

## The problem

A race calendar is not published once. Rounds get added, dates move, venues are confirmed months apart, and most of what circulates is a single rumour repeated by three aggregators until it looks like news.

Checking five sites every day is not worth the time. Checking once a week means finding out late. What I wanted was the opposite of a news feed: silence by default, and a message only when a fact had genuinely moved.

## What it does

Every day at 13:00 it establishes the current state of the calendar, compares it against what it already knew, and sends a Telegram message only if something changed. Once a week, on Wednesday, it sends the full picture regardless.

## How it works

1. **Schedule trigger** — daily, 13:00 Europe/Warsaw
2. **Read state** — previous known calendar from a JSON file on disk
3. **Query** — Perplexity Sonar, restricted to a fixed list of racing sources, with a forced JSON schema
4. **Parse and merge** — normalise round keys, merge with the previous state under the ratchet rule below
5. **Write state** — overwrite the JSON file
6. **Send** — Telegram, only when the merge step sets `should_send: true`

## The rule that makes it useful

Every round carries a status, and **the status can only move upwards**: rumour → unconfirmed → confirmed. Once a date is confirmed it is frozen.

This was the fix for the failure that nearly killed the agent. Without the ratchet it flip-flopped — one source would treat a round as confirmed, the next day another would not, and every flip produced a "change" notification. The agent was technically working and practically useless, because a notification that fires on noise trains you to ignore it.

The verification rules behind the status:

| Status | Requires |
|---|---|
| **confirmed** | motogp.com, Dorna or FIM — nothing else counts |
| **unconfirmed** | at least two independent sources |
| **rumour** | a single source |

Source quality is enforced by `search_domain_filter` at the API level, not by asking the model nicely in the prompt.

## Decisions and trade-offs

**Why n8n.** Before starting I compared n8n, GitHub Actions with my own code, and Make. n8n won on being a natural step up from Make, low-code with an escape hatch into real code, and a predictable cost — a VPS plus tokens, no subscription — with one server able to host every agent I build later.

**Three models, in order.**

1. **Claude Sonnet** — worked well, too expensive to justify running daily
2. **Claude Haiku** — cheaper, less disciplined. Sometimes returned prose instead of JSON. Sometimes explained why a task was difficult instead of doing it
3. **Perplexity Sonar** — the current one. `response_format: json_schema` enforces the output structure **at the API level** instead of requesting it in the prompt, and `search_domain_filter` removes junk sources before the model ever sees them

The general lesson, which I now apply to everything I build: **prefer an API that enforces a schema over one you have to ask politely.** It is the difference between a bug you can reason about and an evening of debugging.

## What went wrong

The useful half of this project. In order of how much time each one cost.

**Google Sheets as the state store — abandoned.** Long JSON strings in a cell produced `#ERROR!`. Moved the state to a file on disk. The obvious storage was the wrong storage.

**Matching rounds by name — unreliable.** The model would return a Grand Prix name in Polish one day and English the next, so the same round looked like two different rounds and the diff went haywire. Fixed with a `normalizeKey()` function computed in code from a keyword dictionary. The lesson generalises: **do not rely on a language model to produce a stable identifier — derive the key yourself.**

**`max_tokens` too low with Claude.** Raised 1024 → 4096 → 8192. With web search the model produces long reasoning blocks, and the JSON gets truncated mid-structure. The failure looked like malformed output; the cause was a budget.

**SSH and the Execute Command node — dead end.** n8n expressions are not evaluated inside SSH commands, and the `fs` module is blocked in the Code node. Two hours to establish that the approach could not work at all.

**Writing to the wrong directory.** `/home/node/.n8n/` is blocked; the writable path is `/home/node/.n8n-files/`, set by `N8N_RESTRICT_FILE_ACCESS_TO`.

**Reading binary files.** Needs `await this.helpers.binaryToBuffer(...)`. Manual `Buffer.from(x, 'base64')` fails *silently* when the instance runs in `binaryDataMode: filesystem` — the worst kind of failure, because nothing errors.

**HTTP method resetting.** Switching to the native Perplexity credential quietly reset the request from `POST` back to `GET`.

**`Invalid expression`.** Almost always a typo in a node reference like `$('Exact Node Name')`. A node rename and its references do not stay in sync.

## A trade-off I left in place

State is written to disk **before** the Telegram message goes out. If the send fails, the change is already recorded as known and that notification is lost.

The alternative — send first, write state afterwards — fails the other way round: a failed state write produces a duplicate message the next day. Duplicates are visible and mildly annoying. Silent loss is invisible, which is worse.

I would normally take the visible failure. I left the current order for two reasons.

Reversing it means rebuilding the connections rather than editing one line, and the payoff is small — because the failure is already bounded. **The Wednesday digest sends the complete calendar regardless of whether anything changed**, so a notification lost on a Tuesday resurfaces at the latest on the Wednesday. The weekly summary, which I originally added for convenience, turns out to be the recovery path for the daily one.

It is a deliberate choice rather than an oversight, and it is the first thing I would change if the cost of a missed message were higher.

## Result

Daily check, Telegram only on change, full digest on Wednesday. The stability test I use: two consecutive runs correctly returning `should_send: false` when nothing has changed. It passes.

The agent has been quiet for most of its life, which is exactly the point.

## Running it yourself

The workflow file in this repository is exported with my identifiers stripped, so importing it will not touch anything of mine. It arrives **inactive** on purpose — check it before you switch it on.

1. **Import** `motogp-calendar-monitor.json` into n8n (Workflows → Import from File).
2. **Create two credentials** in n8n and attach them to the nodes that ask for them:
   - *Perplexity API* → the `HTTP Request - Sonar` node
   - *Telegram Bot API* → the `Telegram - wyslij` node
   The workflow references them by name, not by key. No secret is stored in this file.
3. **Set your chat ID.** In the Telegram node, replace `YOUR_TELEGRAM_CHAT_ID` with your own. Message your bot once first, otherwise it cannot write to you.
4. **Create the state file** at `/home/node/.n8n-files/motogp_2027_state.json` containing `[]`. The read node does not bootstrap it — see the note below.
5. **Set the timezone** to `Europe/Warsaw` in the workflow settings, or change the `isWednesday` check to match yours. The weekly digest depends on it.
6. **Run it once by hand**, confirm a Telegram message arrives, then activate.

**Known rough edges**, left in deliberately so that this file matches what actually runs:

- The read node has `alwaysOutputData: false`, so a missing state file stops the run instead of starting from empty. Setting it to `true` would make the workflow self-bootstrapping — `getPreviousRounds()` already returns `[]` on any failure.
- `disable_notification` on the Telegram node references a field that no longer exists, so it always evaluates to false. Harmless leftover from an earlier design.
- No retry on the HTTP call. A failed request simply means no message that day and no state change, which is the safe direction.

## Stack

Hostinger VPS (KVM) · Docker · n8n self-hosted · Perplexity Sonar API · Telegram Bot API · state as JSON on disk

---

Built as one of several small agents I use to work out where these tools genuinely hold and where they break. Sibling agent: [ai-act-monitor](https://github.com/bartosz-szubert/ai-act-monitor) — the same ratchet idea, applied to EU regulation instead of a race calendar.
