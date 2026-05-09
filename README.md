# claude-ralph

A [ralph loop](https://ghuntley.com/ralph/) runner for Claude Code, with **externally-verified success criteria** — scripts you write, that exit 0 when the work is done. Drop in a spec, drop in a verifier, run `claude-ralph`, watch the first few iterations and tune the prompt.

> **What's a ralph loop?** `while :; do cat PROMPT.md | claude-code; done`, basically. You hand the agent a goal and a punch list, it iterates, you watch. The hard part isn't the loop — it's knowing when to stop and what "done" actually means. See Huntley's [original](https://ghuntley.com/loop/) and [follow-up](https://ghuntley.com/ralph/).

## Why this exists

The canonical ralph loop has one critical weakness: **"done" is whatever the LLM says it is.** The model writes `DONE` to a sentinel file when it *thinks* it's done. That's a hallucination away from a green light on red code.

`claude-ralph` replaces self-report with **external verifiers** — exit-code-driven scripts that run after every iteration and gate the loop. The agent doesn't decide it's done; your test suite does. Or your typechecker. Or a Playwright script. Or Chrome DevTools MCP eyeballing the rendered page. Or all four.

That's the whole pitch. The tool itself is small, foreground, Unix-y — one process, one loop, ^C to stop. **Sandboxing is your call.** Run it on your laptop, in a devcontainer, in a VM, in a Codespace, behind `firejail` — `claude-ralph` doesn't care. We recommend you don't run unbounded `claude` on a host you'd cry over, but the tool isn't going to hold your hand about it. See [Sandboxing recommendations](#sandboxing-recommendations) below.

## How it works

```mermaid
flowchart TB
    prompt["1. run claude with composed prompt"]
    verify["2. run verifiers/*"]
    gate{"3. all green?"}
    report["write verifier_report.md"]
    done(["write DONE, exit"])

    prompt --> verify --> gate
    gate -- yes --> done
    gate -- no --> report --> prompt
```

Each iteration:

1. Compose a prompt from `PROMPT.md` + `specs/` + `fix_plan.md` + the previous iteration's `verifier_report.md`.
2. Pipe it to `claude`. The agent edits files, runs commands, updates `fix_plan.md`.
3. Run every executable in `.ralph/verifiers/` in lexical order. Capture stdout/stderr and exit codes.
4. **All exit 0** → write `.ralph/DONE`, exit cleanly. **Anything nonzero** → write a fresh `verifier_report.md` summarizing failures, loop.

The verifier report is the feedback signal. The agent reads it next iteration and reacts.

## Install

With curl:

```sh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/nbarlow/claude-ralphs/main/install.sh)"
```

Or wget:

```sh
sh -c "$(wget -qO- https://raw.githubusercontent.com/nbarlow/claude-ralphs/main/install.sh)"
```

Works from bash, zsh, fish, dash, or any POSIX shell — the outer `sh -c` spawns its own POSIX shell, so your interactive shell doesn't matter. The installer is POSIX `sh` itself (no bashisms) so it runs on Alpine, BusyBox, and macOS's stock `/bin/sh`.

We use `sh -c "$(... )"` instead of `... | sh` so the script is downloaded **completely** before any of it executes — a flaky connection produces a parse error, not a half-installed system. Same pattern Homebrew and oh-my-zsh use.

What it does:

- Drops `claude-ralph` into `~/.local/bin/` (override with `PREFIX=/usr/local`, will prompt for sudo if needed).
- Detects your login shell (`bash`, `zsh`, `fish`) and prints the exact one-liner to add `~/.local/bin` to `$PATH` if it isn't already. Doesn't edit your rc files without consent.
- Verifies `claude` is on `$PATH` and `ANTHROPIC_API_KEY` is set; warns but doesn't fail if not.

To upgrade, re-run the installer. To pin or roll back, install from source at the tag you want.

### Windows

Use WSL with the POSIX installer above. `claude-ralph` is bash; native Windows isn't a target.

### From source

Clone the repo, then:

```bash
make install                # symlinks bin/claude-ralph into ~/.local/bin (override with PREFIX=...)
```

Other targets: `make test`, `make check`, `make uninstall`, `make help`. `make` is only needed if you're building from source — the installed CLI itself doesn't depend on it.

### Requirements

- The `claude` CLI on `PATH` ([install instructions](https://docs.anthropic.com/en/docs/claude-code/quickstart))
- `ANTHROPIC_API_KEY` in your environment (or whatever auth `claude` is set up for)
- `bash` (you have it)

## Quick start

In any project:

```bash
claude-ralph init           # scaffolds .ralph/ — PROMPT.md, specs/, verifiers/, fix_plan.md
$EDITOR .ralph/PROMPT.md    # state the goal
$EDITOR .ralph/specs/001-*.md
chmod +x .ralph/verifiers/01-tests.sh   # already scaffolded; edit to taste
claude-ralph                # start looping. ^C to stop.
```

It's a foreground process. Background it with `&`, `nohup`, `tmux`, `systemd-run --user`, or whatever you'd reach for normally. Tail per-iteration logs with `tail -f .ralph/iterations/*/stdout.log`.

## The `.ralph/` layout

| path | written by | purpose |
|---|---|---|
| `PROMPT.md` | you | The prompt fed to claude every iteration. Tune it. |
| `AGENT.md` | agent | How-to-build, how-to-run, learnings. Agent appends. |
| `fix_plan.md` | agent | Prioritized punch list. Agent maintains. |
| `specs/NNN-*.md` | you | What to build. One file per slice. |
| `verifiers/NN-*.sh` | you | Success criteria. Exit 0 = pass. Lexical run order. |
| `verifier_report.md` | ralph | Last run's results. Read by the agent next iteration. |
| `iterations/NNNN/` | ralph | Full history per iteration: `prompt.txt`, `stdout.log`, `stderr.log`, `verifiers.json`. |
| `config.toml` | you | Loop / claude knobs. Optional. |
| `DONE` | ralph | Sentinel — written when all verifiers pass. Loop exits. |

## Verifiers: the plugin contract

**The contract is exit code + stdout.** That's it. No YAML, no plugin registry, no DSL.

A verifier is any executable in `.ralph/verifiers/`. It:

- Runs from the workspace root, in the same environment as the agent.
- Exits **0** if its criterion is satisfied, **nonzero** otherwise.
- Writes human-readable diagnostic output to stdout/stderr — this gets fed back to the agent.

Naming: lexical order = run order. Use `NN-name` (`01-tests.sh`, `02-lint.sh`) so you control sequencing.

### Examples

**Tests pass:**

```bash
#!/usr/bin/env bash
# .ralph/verifiers/01-tests.sh
set -euo pipefail
npm test
```

**Typecheck clean:**

```bash
#!/usr/bin/env bash
# .ralph/verifiers/02-typecheck.sh
set -euo pipefail
npx tsc --noEmit
```

**End-to-end UI check (Playwright, headless):**

```bash
#!/usr/bin/env bash
# .ralph/verifiers/03-e2e.sh
set -euo pipefail
npm run dev &>/dev/null &
trap "kill $!" EXIT
until curl -fsS http://localhost:3000 >/dev/null; do sleep 0.2; done

npx playwright test e2e/checkout-coupon.spec.ts
```

**Binary size budget:**

```bash
#!/usr/bin/env bash
# .ralph/verifiers/04-size.sh
set -euo pipefail
go build -o /tmp/myapp ./cmd/myapp
test "$(stat -c%s /tmp/myapp)" -lt $((20 * 1024 * 1024))    # 20MB ceiling
```

**On using `claude` itself as a verifier.** You *can* spawn `claude --print` from inside a verifier — pointing it at Chrome DevTools MCP for visual checks, for instance. Reach for it only when no deterministic check exists. It's slow, expensive, non-deterministic, and asking an LLM whether the LLM did its job is exactly the self-report problem this tool exists to fix. Playwright with a screenshot diff is almost always the right answer first.

### Optional metadata

A header comment block lets ralph render nicer reports and treat some verifiers as advisory:

```bash
#!/usr/bin/env bash
# ralph: name = "frontend tests"
# ralph: blocking = false              # default true; false = report but don't gate DONE
# ralph: timeout = 5m                  # default unlimited
```

This is parsed best-effort. No metadata is also fine — the contract is still just exit code.

**On flakes.** ralph will not retry, quarantine, or score-track flaky verifiers. A flake is a bug in the verifier (or in the thing it's testing). Fix it. Flake-tolerance is how test suites rot into uselessness.

## Configuration

`.ralph/config.toml` (all fields optional):

```toml
[loop]
max_iterations = 50          # hard stop. default 50. --unlimited overrides.
on_done = "notify"           # "notify" | "exit"

[claude]
model = "claude-opus-4-7"
mcp_config = ".ralph/mcp.json"
extra_args = ["--dangerously-skip-permissions"]     # if you know what you're doing
```

`max_iterations` is the hammer for runaway cost — calculate before launching, set the ceiling, walk away from theatrical FinOps dashboards. If you want to push to a branch when the loop succeeds, write a final verifier that does `git push` and `gh pr create`. ralph stays dumb.

## Subcommands

| command | does |
|---|---|
| (no args) | start the loop in the foreground; ^C to stop |
| `init` | scaffold `.ralph/` |
| `verify` | run verifiers once against current state, no loop |
| `step` | run one iteration, then exit |
| `status` | iteration count, last verifier results, token spend |
| `clean` | remove `.ralph/iterations/` and `DONE` |

That's the whole surface. Backgrounding, log following, killing — those are jobs your shell already does.

## Sandboxing recommendations

`claude-ralph` itself does nothing to constrain what the agent can do. The agent has whatever permissions the shell that launched `claude-ralph` has. If you point it at your `$HOME` and tell it to "fix the bugs," it can do anything you can do.

Pick whatever isolation matches your appetite:

- **Docker / Podman** — the straightforward answer. `docker run --rm -it -v "$PWD:/work" -w /work -e ANTHROPIC_API_KEY <your-image> claude-ralph`. Build an image with `claude` plus your project's toolchain.
- **VM** — heavier than a container, sometimes the right tool. Vagrant, libvirt, whatever you already have.
- **`firejail` / `bubblewrap`** — namespace-only sandboxing if you don't want a full container. Lighter, fiddlier.
- **Devcontainer** — if you already live in VS Code Dev Containers, this just wraps the Docker case.
- **Bare host** — fine for small, scoped tasks where you've read `PROMPT.md` and the verifiers carefully. Not the default for "leave it running overnight."

The verifiers run in the same environment as the loop, so they get whatever toolchain you've set up. That's the point — you don't have to tell `claude-ralph` about your stack.

## Tuning the prompt

The default `PROMPT.md` follows Huntley's structure: study the specs, work the punch list one item at a time, update `fix_plan.md`, append learnings to `AGENT.md`. **Don't expect to leave it alone.** Watch the first few iterations and tune.

> "There is no such thing as a perfect prompt." — Huntley

## What's deliberately *not* in scope

- **No container management.** `claude-ralph` is a CLI; sandboxing is your call. See above.
- **No daemon / service mode.** Foreground process. `tmux`, `nohup`, and `systemd-run --user` exist.
- **No GUI / web dashboard.** `tail -f`, `less`, and a terminal are enough.
- **No remote orchestration.** One loop, one host, one repo. If you want a fleet, wrap it.
- **No model-agnostic abstraction.** This is for Claude Code. The loop pattern is universal but the prompt format and tool wiring aren't.
- **No self-update.** `git pull && make install`, or re-run the curl installer. CLIs that update themselves are a Vercel-era flourish.
- **No cost dashboard / FinOps tracking.** Set `max_iterations`, calculate before launching.
- **No flake quarantine.** Fix your flakes.
- **No auto-spec generation.** Write your specs. An LLM splitting a task file into specs is another hallucination vector for marginal benefit.
- **No PR / branch automation.** A final verifier doing `git push && gh pr create` is the right place for that.
- **No verifier marketplace.** They're 10 lines of bash. Write yours.

## Multiple loops on one repo

`.ralph/` is per-worktree, like `.git/`. Run multiple loops by checking out multiple `git worktree` directories — one per concurrent goal. Don't try to run two loops against the same working tree; they'll fight over `fix_plan.md` and the iteration counter.

## Prior art

- [ghuntley/loop](https://ghuntley.com/loop/) — the original technique.
- [exokomodo/im-gonna-ralph](https://github.com/exokomodo/im-gonna-ralph) — bash implementation atop GitHub Copilot CLI, with iteration history and SDD spec splitting. We borrow the iteration-dir pattern.
- This project differs by: (a) Claude Code instead of Copilot, (b) external verifier gating instead of self-reported DONE.

## License

See [LICENSE](LICENSE).
