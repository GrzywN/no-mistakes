# Cloud pivot: phase 0 spike results

This fork is pivoting from a local gate with a background daemon to a runner that executes inside a cloud coding agent's own sandbox, on that sandbox's compute.
Phase 0 checks, before anything is torn down, whether that sandbox can run the pipeline at all.
Every result below was measured in a Claude Code on the web session on 2026-10-15 unless it is marked as not yet verified.

## Decisions taken so far

- Hard fork: keep the pipeline engine (`internal/pipeline`, the agent adapters, the review and Test contracts), replace the host shell (daemon, service managers, IPC, bare gate and hooks).
- The runner executes inside the cloud agent's sandbox that drives it, on that sandbox's compute; nothing is hosted by the fork's author.
- Independent agent mode only: the runner spawns its own agent CLI. A mode that delegates model turns to the driving agent is out of scope.
- Claude Code on the web is the first and only supported platform until the runner works there end to end.
- Licensing and distribution come after the runner works.

## Environment

| Property | Value |
| --- | --- |
| Session environment | `anthropic_cloud`, outbound HTTPS through a local agent proxy |
| CPU and memory | 4 vCPU, 15.7 GiB RAM |
| Free disk | about 29 GB |
| PID 1 | `process_api`, no systemd user bus (`systemctl --user` fails with "Failed to connect to bus") |
| Preinstalled | git 2.43.0, Go 1.25.0, Node 22.22.0, pnpm 11.5.2, gh 2.89.0, Claude Code CLI 2.1.300, Chromium for Playwright build 1194 |
| Checkout | shallow, depth 50 |

## Upstream blockers confirmed in this environment

- A push from the shallow checkout into a freshly initialised gate is rejected with `shallow update not allowed`; the same push succeeds once the gate holds the history from `origin`.
- `gh auth status --hostname github.com` exits 1 ("The token in GH_TOKEN is invalid") because the proxy substitutes the real credential in flight, so `internal/scm/github.Available` skips the PR and CI steps.
- GitHub GraphQL answers HTTP 403 ("GitHub GraphQL is not available from Claude Code sessions"), which breaks `gh pr list`, `gh pr create`, `gh pr view`, `gh pr edit`, `gh pr checks` and the three `gh api graphql` queries.
- The daemon unsets only five Claude Code variables at startup (`internal/daemon/daemon.go`), so the rest of the parent session's environment reaches pipeline agents.
- The release CDN on github.com answers 403, so `install.sh` and `no-mistakes update` cannot fetch a binary; `proxy.golang.org`, `registry.npmjs.org`, `pypi.org` and `nodejs.org` are reachable.

The detached daemon itself starts fine here: `make build`, `doctor` and `init` pass and the daemon is ready in 42 ms.

## S1: nested agent CLI inside the sandbox

Three one-word `claude -p --verbose --output-format stream-json --dangerously-skip-permissions` calls, each from a scratch repository.

| Variant | Environment | Result | Session id | MCP servers | Reported cost |
| --- | --- | --- | --- | --- | --- |
| V1 | parent environment minus the five variables the daemon unsets | `ok` | the parent session's id | 8 inherited | 0.18 USD |
| V2 | strict allowlist, no `CLAUDE_*` variable at all | `ok` | fresh | 0 | 0.05 USD |
| V3 | V2 plus an empty `CLAUDE_CONFIG_DIR` | `ok` | fresh | 0 | 0.02 USD |

- Authentication needs no API key: `apiKeySource` is `none` in every variant and the CLI reads the session's own credentials from the container's files.
- V1 reuses the parent session's id even with `CLAUDE_CODE_SESSION_ID` unset, and emits the parent's `active_goal` and `autocompact_state` events.
  Upstream resumes fixer sessions by that id, so every nested turn would collide on one session.
- No event of the V1 run reached the parent session's event stream; none of its event or message ids appear there.
- V2 and V3 still load 19 to 20 skills and 3 plugins, which come from the account's managed settings rather than the user directory.

Conclusion: nested agents work on the user's own login, and the runner must launch them from an explicit environment allowlist, never from the inherited environment.

## S2: the target repository's end-to-end suites in the sandbox

The target is `osiedlo/osiedlo` at `e0f7e7f`, run through its own Makefile with `CI=true`, sampling CPU and memory every 2 s.

| Step | Result | Wall time | Memory peak | CPU |
| --- | --- | --- | --- | --- |
| `make install` (cold) | pass | 48 s | 1.8 GB | 50% average |
| `make e2e`: demo, auth and a11y suites | 154 + 79 + 50 tests passed, none failed | 437 s | 4.3 GB (2.6 GB average) | 64% average, at or above 95% for a quarter of the run, load peak 12.5 |

- The repository requires Node 24.11 or newer with `engine-strict`, while the image ships Node 22.22; `nvm install 24` from `nodejs.org` takes 7 s.
- Playwright 1.62 needs Chromium build 1234 while the image ships build 1194, so `playwright install chromium` downloads about 184 MB from `cdn.playwright.dev` in every fresh container.
- `make e2e-full`, which builds the frontend and boots three servers, was not run.

Conclusion: one task's full end-to-end run fits a session's container with memory to spare, and CPU is the constraint; parallelism comes from running each task in its own session, not from packing several into one.
A setup script has to install the pinned Node and the matching Playwright browser.

## S3: GitHub over REST through the proxy

- The proxy injects the session's GitHub credential itself: a plain `curl` to `api.github.com` without any `Authorization` header gets 200 for a repository in the session's scope, and 403 for one outside it.
  A native REST client in the runner therefore needs neither `gh` nor a token in this environment.
- Every read the PR and CI steps need answers over REST: the repository with its push permission, pull request listing, check runs and combined status for a commit, and Actions workflow runs; `gh run list` also works because it is REST-based.
- A plain `git push` of a new branch through the proxy works, and the session signs the commit (`gpgsig` present).
- A history rewrite of the session's own branch pushed with `--force-with-lease=<ref>:<expected sha>`, the form the Push step uses, also works.
- Creating the draft pull request over REST was not run: Claude Code's auto-mode permission classifier refused the `gh api -X POST .../pulls` call as an external write, so the write half of S3 (create, body update, the `ccr` draft and ready routes, close) is not yet verified.
  Inferred consequence: under auto mode, a driving agent's call to the runner, which pushes and opens pull requests, may need an explicit permission rule.
- The fork has no Actions runs and only a `main` branch, so its CI checks are empty.

## Not yet verified

- S4: three cloud sessions running the pipeline at once, against session concurrency and model usage limits.
- Whether commits made by pipeline agents are signed the way the session's own commits are.
