# CLAUDE.md

## What this is

`claude-cost` is a single-file, stdlib-only Python 3 tool that computes what a user
is being charged for Claude Code by parsing the local session logs
(`~/.claude/projects/**/*.jsonl`), and can hard-stop Claude Code via hooks when a
monthly spend cap is hit (the "kill switch"). There is no build system, no
dependencies, no tests directory — the entire project is the `claude-cost` script
plus this documentation.

## Setting it up for a user

```sh
# 1. Install (any PATH directory; /opt/homebrew/bin shown for Apple Silicon macOS)
curl -fsSL https://raw.githubusercontent.com/bennett-create/claude-cost/main/claude-cost \
  -o /opt/homebrew/bin/claude-cost && chmod +x /opt/homebrew/bin/claude-cost

# 2. Optional: spend cap / kill switch
claude-cost --install-gate    # wires PreToolUse + UserPromptSubmit hooks into ~/.claude/settings.json
claude-cost --limit 50        # hard-stop Claude Code at $50 charged this month
claude-cost --limit-hour 5    # optional: max $5 charged in any rolling 60 min
claude-cost --limit-prompt 2  # optional: max $2 per single prompt/turn

# 3. Optional: always-on web dashboard at http://localhost:4477
claude-cost --install-sidecar
```

Verify with `claude-cost --once` (prints a snapshot and exits). On Windows, run via
`python claude-cost ...`; `--install-sidecar` uses the Startup folder there instead
of launchd.

## Things you (a Claude agent) should know before touching this

- **The kill switch governs YOU.** If the user has the gate installed and armed,
  every one of your tool calls runs `claude-cost --gate` first. If the budget is
  exhausted, your tool calls will be blocked with a "BUDGET KILL SWITCH" message.
  Do not try to work around it — tell the user how to continue and stop. Monthly
  limit: `claude-cost --approve <N>`. Hourly limit: wait for the rolling window,
  or `claude-cost --snooze <min>`. Per-prompt limit: the user just sends a new
  message (that IS the approval; it resets the per-prompt budget). That approval
  flow is the entire point of the tool.
- **Per-prompt tracking** works by resetting a per-file `prompt_spend` counter
  whenever a real user prompt appears in the session log (`_is_user_prompt` —
  string content or text blocks; tool_result turns and sidechains don't count).
  The hourly limit sums a rolling `recent` event list in the gate cache. Snooze
  (`snooze_until` in state.json) waives hourly+prompt checks but never monthly.
- **Keep it a single stdlib-only file.** No pip dependencies, no splitting into
  modules. Friends install it with one curl; that property is load-bearing.
- **Gate performance is a hard constraint.** `--gate` runs before every tool call.
  It stays fast via an incremental offset cache (`~/.config/claude-cost/gate_cache.json`).
  Anything you add to the gate path must keep warm runs ~tens of milliseconds.
- **Pricing lives in `EXACT_PRICING`** at the top of the script ($ per 1M tokens,
  `(input, output)` tuples). Cache economics are in `cost_for_usage()`: cache reads
  0.1× input, 5m cache writes 1.25×, 1h writes 2×. Dated model IDs
  (`claude-haiku-4-5-20251001`) are matched by stripping the `-YYYYMMDD` suffix.
  If prices look stale, verify against current published pricing before editing.
- **Subscription vs API is inferred, not logged.** Session logs don't tag messages
  with billing type. The script reads `billingType` from `~/.claude.json`; while it
  contains "subscription", charged = $0. When it flips, a cutover timestamp is
  written to `~/.config/claude-cost/state.json` (`billed_since`) and only later
  usage counts. `--since` overrides the cutover manually.
- **State files:** `~/.config/claude-cost/state.json` (limit, per-month approved
  headroom, billed_since) and `gate_cache.json` (per-file offsets + monthly totals).
  Deleting `gate_cache.json` is safe (full rescan next gate run). Deleting
  `state.json` disarms the limit and forgets the billing cutover.
- **`<synthetic>` model entries** in the logs are internal artifacts and are
  skipped by `parse_record()` — keep it that way.

## Testing changes

No test suite; verify by hand, in this order:

```sh
python3 -m py_compile claude-cost                       # syntax
./claude-cost --once                                    # parses real logs, prints snapshot
echo '{}' | ./claude-cost --gate; echo $?               # 0 = pass expected
./claude-cost --serve 4478 &  curl -s localhost:4478/json | python3 -m json.tool
```

To test the block path without real spend: temporarily set `billed_since` to a past
date in `~/.config/claude-cost/state.json` with a low `limit`, run the gate (expect
exit 2 + message), then restore the state. Test `--install-gate` against a throwaway
`HOME` (it needs `$HOME/.claude/projects/` to exist) and confirm it merges rather
than duplicates hooks. If you change hook behavior, prove the hook actually fires:
temporarily prefix the hook command in settings.json with
`echo fired >> /tmp/hook-check;`, trigger a tool call, check the file, revert.

## Layout

```
claude-cost   # the whole program (~400 lines): pricing → billing state →
              # budget gate → log scanning → rendering → modes (tui/serve/install)
README.md     # user-facing docs
CLAUDE.md     # this file
```
