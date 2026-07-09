# claude-cost

See how much money you're actually being charged for Claude Code, in real time — straight from the session logs on your own machine. No API keys, no dependencies, just Python 3 (preinstalled on macOS).

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/bennett-create/claude-cost/main/claude-cost \
  -o /opt/homebrew/bin/claude-cost && chmod +x /opt/homebrew/bin/claude-cost
```

(Or drop it anywhere on your PATH — it's a single file.)

**Windows** (needs [Python 3](https://python.org)): download
[`claude-cost`](https://raw.githubusercontent.com/bennett-create/claude-cost/main/claude-cost),
then run `python claude-cost` from a terminal. `--install-sidecar` works too —
it adds a Startup-folder entry that launches the web dashboard at login.
Works in WSL as well (Linux-style, no extra steps).

## Use

```sh
claude-cost                   # live terminal dashboard
claude-cost --once            # one snapshot, then exit
claude-cost --serve           # web dashboard at http://localhost:4477 (+ /json)
claude-cost --install-sidecar # macOS: auto-start the web dashboard at login
claude-cost --since 2026-07-15  # manually set the date per-token billing started
```

## Subscription-aware

Claude Code logs don't tag individual messages as subscription vs API-billed, so
claude-cost reads your account's `billingType` from `~/.claude.json`:

- While you're on a subscription, everything counts as **$0 charged** (shown
  separately as "covered by subscription" for reference).
- The moment your billing flips to per-token, the cutover timestamp is saved to
  `~/.config/claude-cost/state.json` and only usage after it counts as charged.
- Moving to per-token on a known date? Set it explicitly with `--since`.

## How it computes cost

Parses the `usage` block of every assistant message in `~/.claude/projects/**/*.jsonl`
and applies published per-model pricing, including cache economics: cache reads at
0.1× input price, 5-minute cache writes at 1.25×, 1-hour cache writes at 2×.
Sonnet 5's introductory pricing is applied automatically through 2026-08-31.

## Kill switch (spend limit)

Hard-stop Claude Code when monthly charged spend hits a cap — it can't make
another API call until you approve more:

```sh
claude-cost --limit 50      # stop everything at $50/month
claude-cost --approve 10    # you decide to continue: +$10 this month
claude-cost --limit off     # disarm
```

Enforcement is a Claude Code hook: add this to `~/.claude/settings.json`
(claude-cost's `--gate` exits 2 when over budget, which blocks the action):

```json
"hooks": {
  "PreToolUse":       [{ "hooks": [{ "type": "command", "command": "claude-cost --gate", "timeout": 15 }] }],
  "UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "claude-cost --gate", "timeout": 15 }] }]
}
```

Use the full path to claude-cost in the command. Over budget, every tool call
and new prompt is blocked with a message telling you how to approve more.
Warm-cache gate checks take ~40ms. Approvals reset each calendar month.

## Caveats

- This reads local logs, not Anthropic's billing system. It should match your bill,
  but it isn't your bill.
- Models without published pricing fall back to a same-tier estimate (flagged with *).
- Everything stays on your machine; the web dashboard binds to 127.0.0.1 only.

## License

MIT
