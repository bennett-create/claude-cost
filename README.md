# claude-cost

See how much money you're actually being charged for Claude Code, in real time — straight from the session logs on your own machine. No API keys, no dependencies, just Python 3 (preinstalled on macOS).

## Install

```sh
curl -fsSL https://raw.githubusercontent.com/bennett-create/claude-cost/main/claude-cost \
  -o /opt/homebrew/bin/claude-cost && chmod +x /opt/homebrew/bin/claude-cost
```

(Or drop it anywhere on your PATH — it's a single file.)

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

## Caveats

- This reads local logs, not Anthropic's billing system. It should match your bill,
  but it isn't your bill.
- Models without published pricing fall back to a same-tier estimate (flagged with *).
- Everything stays on your machine; the web dashboard binds to 127.0.0.1 only.

## License

MIT
