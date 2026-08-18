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

**Audited commit:** `9995c77278608640759982c98ec5bdaeb371c174`, 2026-08-17.

**Verdict: safe to fork and pin, with four named changes.** Clean on telemetry,
analytics, author controlled domains, obfuscation, `eval`, dynamic code loading
and committed secrets across 584 commits of history.

`main` was fast forwarded from `471eca5` to the audited commit on 2026-08-18, so
the pin and the audit name the same tree. They did not before.

## The two changes carried in the code

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

## The two changes that are operating rules, not code

### 3. Decline every provider path that runs `npm install -g`

The Kimi OAuth, Grok OAuth and DeepSeek Harness paths install third party CLIs
globally with no version pin. GLM and DeepSeek are wired through their platform
API keys instead, which install nothing and use the hidden terminal prompt.

### 4. `bin/update` refuses to run on a fork

It checks the repository URL. Set `CODEX_ROUTER_REPOSITORY_URL` to this fork in
the environment used for updates, or the update refuses.

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
