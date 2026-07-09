# claude-cost

See how much money you're actually being charged for Claude Code, in real time —
and **hard-stop Claude when spending hits a cap you set**. Reads the session logs
on your own machine. No API keys, no dependencies, no telemetry; single Python 3
file (Python is preinstalled on macOS).

```
claude-cost — what you're being charged for Claude Code
billing: api   charged since: 2026-07-15T00:00:00
----------------------------------------------------------
Model                       Charged today    Charged total
claude-opus-4-8                     $1.62           $14.20
claude-sonnet-5                     $0.31            $2.88
----------------------------------------------------------
CHARGED                             $1.93           $17.08
covered by subscription (reference)               $635.60
```

## Install

**macOS / Linux:**

```sh
curl -fsSL https://raw.githubusercontent.com/bennett-create/claude-cost/main/claude-cost \
  -o /opt/homebrew/bin/claude-cost && chmod +x /opt/homebrew/bin/claude-cost
```

(Any directory on your PATH works — it's a single file.)

**Windows** (needs [Python 3](https://python.org)): download
[`claude-cost`](https://raw.githubusercontent.com/bennett-create/claude-cost/main/claude-cost),
then run `python claude-cost` from a terminal. WSL works Linux-style with no extra steps.

## Commands

| Command | What it does |
|---|---|
| `claude-cost` | Live terminal dashboard, refreshes every 2s |
| `claude-cost --once` | One snapshot, then exit |
| `claude-cost --serve [PORT]` | Web dashboard at `http://localhost:4477` (JSON at `/json`) |
| `claude-cost --install-sidecar` | Auto-start the web dashboard at login (macOS launchd / Windows Startup folder) |
| `claude-cost --limit 50` | **Kill switch**: hard-stop Claude Code at $50 charged this month |
| `claude-cost --limit-hour 5` | Hard-stop at $5 charged in any rolling 60 minutes |
| `claude-cost --limit-prompt 2` | Hard-stop a single prompt/turn that exceeds $2 (runaway-loop guard) |
| `claude-cost --limit[-hour\|-prompt] off` | Disarm that limit |
| `claude-cost --approve 10` | Approve $10 extra headroom for the current month |
| `claude-cost --snooze 15` | Waive hourly + per-prompt limits for 15 minutes (monthly still applies) |
| `claude-cost --install-gate` | Wire kill-switch enforcement into Claude Code (one time) |
| `claude-cost --since 2026-07-15` | Manually set the date per-token billing started |
| `claude-cost --gate` | The budget check itself (used by the hooks; exit 2 = block) |

## The kill switch

For anyone moving from a Claude subscription to per-token API billing and worried
about runaway spend on commercial work:

```sh
claude-cost --install-gate     # once
claude-cost --limit 50         # cap: $50/month
claude-cost --limit-hour 5     # optional: max $5 in any rolling hour
claude-cost --limit-prompt 2   # optional: no single prompt may burn more than $2
```

The three limits are independent — set any combination.

`--install-gate` adds two hooks to `~/.claude/settings.json`: a `PreToolUse` hook
(runs before **every tool call** Claude makes) and a `UserPromptSubmit` hook (runs
on every prompt you send). Each runs `claude-cost --gate`, which keeps an
incremental tally of the month's charged spend — warm checks take ~40ms, so
there's no perceptible slowdown.

The moment spend crosses your cap, the gate exits with a blocking error:
Claude can't execute another tool (it's stopped **mid-task**, not just between
sessions) and new prompts are rejected, with a message telling you how to
continue. You then decide, from any terminal:

```sh
claude-cost --approve 10   # yes, $10 more this month
claude-cost --limit 100    # or raise the cap
claude-cost --limit off    # or disarm
```

Approvals reset each calendar month, so a one-off "fine, $20 more in July"
doesn't silently loosen August. Existing hooks in settings.json are preserved
(the installer merges, and won't add duplicates). Review or disable anytime via
`/hooks` inside Claude Code.

**How each limit recovers:**

| Limit | Trips when | Unblocks when |
|---|---|---|
| `--limit` (monthly) | Month's charged spend ≥ cap | You run `--approve N` or raise/remove the cap. Snooze does **not** bypass it. |
| `--limit-hour` | Last 60 min of spend ≥ cap | Automatically as spend rolls out of the window, or `--snooze` |
| `--limit-prompt` | One prompt's turn costs ≥ cap | You send Claude a new message (that's the approval), or `--snooze` |

The per-prompt limit is the runaway-agentic-loop guard: it stops Claude
mid-task if a single turn spirals, and simply replying continues the work with
a fresh per-prompt budget.

**Overshoot bound:** the gate counts completed API responses, so the response
that crosses the line finishes before the block lands — overshoot is one
response, typically pennies. It governs Claude Code on this machine only;
direct API scripts and other machines aren't covered.

## Subscription-aware

Claude Code logs don't tag individual messages as subscription vs API-billed, so
claude-cost reads your account's `billingType` from `~/.claude.json`:

- On a subscription, everything counts as **$0 charged** (shown separately as
  "covered by subscription" for reference), and the kill switch never trips.
- When billing flips to per-token, the cutover timestamp is saved to
  `~/.config/claude-cost/state.json` and only usage after it counts as charged.
- Know the switch date in advance? Pin it exactly with `--since`.

## How it computes cost

Parses the `usage` block of every assistant message in `~/.claude/projects/**/*.jsonl`
and applies published per-model pricing, including cache economics: cache reads at
0.1× input price, 5-minute cache writes at 1.25×, 1-hour cache writes at 2×.
Sonnet 5's introductory pricing is applied automatically through 2026-08-31.

## Caveats

- This reads local logs, not Anthropic's billing system. It should match your bill,
  but it isn't your bill — trust the invoice for anything that matters.
- Models without published pricing fall back to a same-tier estimate (flagged with *).
- If Anthropic changes prices, update the `EXACT_PRICING` table at the top of the
  script (or pull the latest version).
- Everything stays on your machine; the web dashboard binds to 127.0.0.1 only.

## License

MIT
