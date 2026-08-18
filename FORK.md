# This is a fork, and here is what differs

`blizzardbase/codex-router` forks `duolahypercho/codex-router`.

**Why it exists.** This deployment needs GLM and DeepSeek API keys usable inside
the Codex harness. That is the whole purpose. Nothing here is meant to be a
contribution back upstream, and no pull request from this fork goes to the
author.

**Why a fork rather than a pin.** A pin alone cannot change a default. Two
upstream defaults have blast radius that this use case gains nothing from, so
the fork changes them and asserts the change in the test suite. An upstream
merge that reintroduces either one fails a test rather than landing quietly.

## Audited and pinned

**Audit:** [`AUDIT-2026-08-18.md`](AUDIT-2026-08-18.md), a read only pass over
the source. Nothing was installed and nothing was run during it.

**Audited upstream commit:** `9995c77278608640759982c98ec5bdaeb371c174`,
2026-08-17.

**Verdict: safe to fork and pin, with four named changes.** Clean on telemetry,
analytics, author controlled domains, obfuscation, `eval`, dynamic code loading
and committed secrets across 584 commits of history.

**PIN THIS FORK'S `main`, NOT the audited upstream commit.** `main` was fast
forwarded from `471eca5` to `9995c77`, then the hardening was merged into it as
`467e710`. **Pinning `9995c77` would pin a tree that does NOT contain either
code change**, because the audited commit is upstream's and the hardening sits
on top of it.

**This is not a nicety, and it nearly shipped as one.** `bin/update` and
`install.sh` both converge on `origin/main`: `src/update.mjs` fetches
`origin main`, refuses any checkout not on `main`, and merges `origin/main` fast
forward. **`CODEX_ROUTER_REPOSITORY_URL` only widens an allowlist** at
`src/update.mjs:31-39`; it is added to a set that already contains the upstream
URLs and **it never steers the pull**. So while the hardening sat on a branch,
every sanctioned path, a fresh clone of the stable checkout included, would have
installed plain upstream: fallback ON and `GH_TOKEN` accepted, with nothing
erroring. Found by a cross model challenge of the adoption plan rather than of
the code, and verified against `src/update.mjs` before acting.

## The three changes carried in the code

### 1. The ChatGPT session fallback defaults to OFF

`src/codex-native-session.mjs`, and the three service generators.

Upstream reads `$CODEX_HOME/auth.json`, takes `access_token` and `account_id`,
and attaches them when a caller presents no credential of its own. So a local
process holding the caller key can spend the ChatGPT subscription, and the
DeepSeek Harness and Gemini CLI integrations can be pointed at that subscription
from a client that is not Codex.

Upstream documents `CODEX_ROUTER_NATIVE_SESSION_FALLBACK=0` as the way out.
**That switch does not survive an install.** `src/service-macos.mjs` builds the
launchd plist from a fixed list of variables and this name is not in the list, so
the service starts without it and every install and update restores the ON
default. The author reached the opposite conclusion one file away, in
`src/discovery-mode.mjs`, where the mode is persisted to a file precisely because
an environment variable dies with the setup run.

**The fork inverts the default**, so the fallback is off unless something sets
`CODEX_ROUTER_NATIVE_SESSION_FALLBACK=1`, and **writes the setting into all
three generated service files** so the posture is visible in the service
environment rather than inherited from a default nobody can read.

A caller that presents its own credential is relayed unchanged, which is every
route this fork exists to serve.

### 2. The Copilot provider no longer reads general purpose GitHub tokens

`config/github-copilot/github-copilot.json`.

Upstream accepts `COPILOT_GITHUB_TOKEN`, `GH_TOKEN` and `GITHUB_TOKEN`. The last
two are general purpose credentials that `gh` puts in the environment for
reasons that have nothing to do with Copilot, and `gh` is installed on both
machines here. The fork accepts only the dedicated name.

### 3. The injected computer use skill names Brave, not Safari and Chrome

`skills/codex-computer-use/SKILL.md`.

The installer copies five skill packs into `~/.codex/skills`, where **Codex reads
them into every eligible session**. That makes them standing instructions to an
agent, not documentation, and on this machine that agent runs with
`sandbox_mode = "danger-full-access"` and `approval_policy = "never"`.

Upstream's version named Safari and Chrome in two places, including "Common apps
such as Safari and Chrome are usually pre-approved". The operator's standing rule
is Brave only, never Chrome, and never Safari without being asked. The fork says
Brave, and adds the rule that automation never touches the window the operator is
working in.

**The fork note in that file carries no date, deliberately.** A guard test at
`test/skills-install.test.mjs:502` asserts no pack `SKILL.md` matches
`/20\d\d-\d\d-\d\d/`, because the pack is loaded into every eligible session and a
date in it rots in front of the model. The first version of this change dated the
note and broke the suite on all three platforms. The date belongs here.

**The point is not the browser. It is that this directory is an instruction
surface.** The original audit's coverage statement named neither
`src/skills-install.mjs` nor `src/codex-agent-catalog.mjs`, so the code that
writes agent visible instructions into every future Codex session was never read.
Anything added to `skills/` by a future upstream merge reaches the agent the same
way and must be read as instructions rather than diffed as text.

## The three changes that are operating rules, not code

### 4. Decline every provider path that runs `npm install -g`

The Kimi OAuth, Grok OAuth and DeepSeek Harness paths install third party CLIs
globally with no version pin. GLM and DeepSeek are wired through their platform
API keys instead, which install nothing and use the hidden terminal prompt.

### 5. `bin/update` refuses to run on a fork

It checks the repository URL. Set `CODEX_ROUTER_REPOSITORY_URL` to this fork in
the environment used for updates, or the update refuses.

### 6. Never hand write an agent file called `router-model-*.toml`

`~/.codex/agents/router-model-<anything>.toml` is a namespace this tool claims
**by filename alone**, and it deletes what it finds there.

`src/codex-agent-catalog.mjs:31` defines the namespace as the regular expression
`/^router-model-[a-z0-9-]+\.toml$/`, and `syncRoutedCodexAgents` at `:100-104`
unlinks every file matching it that is not in the current model set. There is no
marker, no token and no ownership record. Compare the skills path, which requires
a 64 hex token to match both an on disk marker and a 0600 ownership file
(`src/skills-install.mjs:181-196`) and which fails closed at every branch. The
agent path has none of that.

**So a file you write yourself at that name is deleted with no backup and no
message.** It fires on every catalog publish (`src/catalog.mjs:884`), and
`doctor --fix` re-runs the installer (`src/doctor.mjs:250-254`), so a repair
triggers it too. The unlink error path swallows failures and success prints
nothing, so you would not see it happen.

**Nothing is at risk today and that is luck, not design.** `~/.codex/agents` does
not exist on this machine, so there is nothing there to lose. The risk begins the
first time anything is put in it.

**Any other agent file name is safe.** The test suite proves only that negative
case: `test/codex-agent-catalog.test.mjs:89-97` asserts a file named
`reviewer.toml` survives. Nothing tests a user file INSIDE the namespace, because
the code cannot tell one from its own.

Found by a read only audit of the install surface on 2026-08-18. **Recorded, not
fixed**, because closing it means adding an ownership scheme to a code path this
fork otherwise leaves untouched, and that is a larger change than a fork note.

## Known and accepted, not fixed

**The Codex app tool relay has no off switch.** `src/codex-app-tools.mjs` and
`src/router.mjs` merge Codex's native app tool definitions into requests going to
routed providers, unconditionally. The set includes `automation_update`,
`plugin_management`, `uninstall_plugin`, `send_message_to_thread` and
`read_thread_terminal`. The router never executes these; Codex does. So prompt
injection reaching a routed model has the Codex automation and plugin surface in
reach, on a provider that may be trusted less than OpenAI.

**This is accepted rather than solved.** Removing the relay is a real change to
routing behaviour and it was not made under an audit finding alone. Weigh it
before pointing a third party model at a repository that matters.

## Updating this fork

The order matters, and skipping the audit is the failure this file exists to
prevent.

1. `git fetch upstream`
2. Read the diff from the pinned commit to the new upstream head.
3. Audit it. A version that has not been read has not been audited.
4. Merge into this fork on a branch, run `npm test`, and confirm the two guard
   tests still pass. They are `the fallback is OFF when nothing asks for it` and
   the Copilot environment list assertion in `test/registry.test.mjs`.
5. Update the audited commit named in this file and in `AUDIT-2026-08-18.md`.

**Never use GitHub's Sync fork button.** It bypasses every step above.
