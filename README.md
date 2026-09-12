# dsh-agent-sandbox

A small Docker-based adapter for running **DeepSeek Harness (DSH)** coding agents against a disposable copy of a Git workspace while keeping the real checkout out of the agent's writable filesystem.

> **Status:** experimental MVP. This project is designed primarily to contain destructive filesystem mistakes by autonomous coding agents. It is **not** a hardened anti-exfiltration or hostile-code sandbox.

## Why

Coding agents need broad freedom inside a project: editing files, rebuilding dependencies, switching test fixtures, running Playwright, resetting test repositories, and occasionally making destructive mistakes. Mounting a real repository read/write into a container does not protect it from `rm -rf`, `git clean -xfd`, a bad worktree cleanup, or an incorrect path expansion.

`dsh-agent-sandbox` uses a different boundary:

- the **real development repository is never mounted into the container**;
- the agent works in a persistent, self-contained Git copy;
- a separate trusted DSH source tree runs the control WebUI;
- the control DSH home is staged on the WSL/Linux host so conversation history survives container stop/restart;
- finishing a session can export a Git patch and selectively merge only canonical DSH session history and attachments back to the real `DSH_HOME`.

The intended failure mode is: **a bad agent run loses or corrupts the staged session, not the real project or global DSH home.**

## Architecture

```text
Windows browser
    |
    | http://127.0.0.1:3181
    v
Docker published port
    |
    v
+----------------------------------------------------+
| agent container                                    |
|                                                    |
| /control-dsh    <- trusted DSH source, read-only   |
| /control-home   <- persistent staged DSH_HOME, RW  |
|                                                    |
| <real repo path> <- persistent staged Git copy, RW |
|   tests/                                           |
|     deepseek-harness/  <- test DSH, fully RW       |
|     homes/             <- test DSH_HOME(s), RW     |
|     test-credentials/  <- optional RO overlay      |
|                                                    |
| no Docker socket                                   |
| no host HOME mount                                 |
| no Windows drive mount                             |
| non-root user, cap-drop=ALL, no-new-privileges     |
+----------------------------------------------------+

WSL/Linux host
  real project                     (never mounted)
  ~/deepseek-harness               (control source, RO mount)
  ~/.dsh                           (snapshot source; never RW-mounted)
  ~/.local/state/agent-sandbox/
    sessions/<id>/
      workspace/                   (persistent staged project)
      control-home/                (persistent staged DSH_HOME)
      durable-baseline.json
      changes.patch
      history-merge-report.json
      meta.env
```

The staged workspace is mounted inside the container at the **same absolute path** as the real repository. This is intentional: DSH's `workspace-write` policy derives the workspace root from `process.cwd()`, and persisted session metadata should continue to refer to the real project path even though the container actually writes only to the staged copy.

## What this protects

The default mode is aimed at accidental/destructive filesystem operations. The agent can freely modify or delete the staged workspace, including `tests/deepseek-harness` and `tests/homes`, without writing to the real repository.

The container intentionally does **not** receive:

- the real repository as a bind mount;
- the real host home directory;
- `/mnt/c`, `/mnt/d`, etc.;
- `/var/run/docker.sock`;
- Linux capabilities or privilege escalation.

## What this does **not** protect

This MVP uses normal Docker outbound networking. It is **not** designed to contain a malicious agent, prompt-injected code, or deliberate data exfiltration.

The staged `control-home` is copied from the real DSH home (except `cache/`) so the control instance can use existing profiles, settings, credentials, sessions, attachments, and plugin state. Therefore the agent runs as the same container user that can read those staged credentials. Treat this as a **filesystem-loss boundary**, not a secret-isolation boundary.

If you need network/credential isolation, add a separate hardened egress/secret-proxy layer rather than assuming this MVP provides one.

## Requirements

Developed for **WSL2 + Docker Desktop**. A normal Linux Docker host may work but is not the primary tested environment.

Host tools:

- Bash
- Docker
- Git
- `rsync`
- Python 3

The trusted control DSH source defaults to:

```text
~/deepseek-harness
```

It must be a clean Git worktree with dependencies already installed (`node_modules` present). The launcher pins its commit for the lifetime of a sandbox session and refuses to resume if that control source has moved or become dirty.

The Docker image is built automatically on first use and includes Node 24, Corepack/pnpm support, Git, GitHub CLI, Python, build tools, `socat`, `rsync`, and Playwright's Chromium system dependencies.

## Install

Clone this repository and put the launcher on your PATH:

```bash
git clone https://github.com/ArmourPiercer1/dsh-agent-sandbox.git
cd dsh-agent-sandbox
install -Dm755 agent-sandbox ~/.local/bin/agent-sandbox
```

Make sure `~/.local/bin` is on your PATH, then verify:

```bash
agent-sandbox --version
agent-sandbox --help
```

## Quick start

Prepare the trusted control DSH once outside the sandbox:

```bash
cd ~/deepseek-harness
pnpm install
git status --short   # should be empty
```

Then start a sandbox from a development repository:

```bash
cd ~/src/dsh-agent-team
agent-sandbox --port 3181 .
```

Open the URL/token printed in the logs, normally based on:

```text
http://127.0.0.1:3181
```

The control DSH itself still listens only on container localhost. A small `socat` relay exposes it through Docker while preserving DSH's localhost-only bind behavior.

## Typical DSH plugin test layout

The launcher is designed for repositories that keep mutable integration-test assets under an ignored `tests/` tree, for example:

```text
my-dsh-plugin/
  src/
  package.json
  tests/
    deepseek-harness/       # mutable DSH source used only for testing
    homes/                  # disposable test DSH_HOME directories
    test-credentials/       # optional credentials; mounted read-only
```

These directories may be ignored by Git:

```gitignore
/tests/deepseek-harness/
/tests/homes/
/tests/test-credentials/
```

The workspace copy is made from the **actual filesystem state**, not only `HEAD`, so ignored test fixtures and existing dirty files are preserved in the staged session. Linked Git worktrees are supported: the staged workspace gets a self-contained `.git` directory rather than retaining the source worktree's `.git` pointer.

Inside the sandbox, an agent can run a separate test DSH instance without affecting the control DSH:

```bash
cd tests/deepseek-harness
export DSH_HOME="$PWD/../homes/.dsh-test-1"
pnpm dsh web --port 3180
```

Playwright can then test it at:

```text
http://127.0.0.1:3180
```

The test harness can be rebuilt, reset, switched to another branch/version, or deleted and recloned. It is independent from `/control-dsh`.

## Lifecycle

### Start

```bash
agent-sandbox --port 3181 .
```

Equivalent explicit form:

```bash
agent-sandbox start --port 3181 .
```

Useful options:

```text
-p, --port PORT             Main DSH WebUI port (default: 3181)
-n, --name NAME             Friendly session name
    --control-source PATH   Trusted DSH source (default: ~/deepseek-harness)
    --dsh-home PATH         Real DSH_HOME snapshot source (default: ~/.dsh)
    --credentials PATH      Test credentials path inside project
    --no-credentials        Do not mount project test credentials
    --image IMAGE           Use an existing custom image
    --rebuild-image         Rebuild the default image
```

### Stop without finishing

```bash
agent-sandbox stop
```

`stop` only stops the container. It does **not** delete the session workspace or control-home staging.

### Resume

```bash
agent-sandbox resume
```

The same staged workspace and staged DSH home are reused, so code changes and DSH conversation data survive normal stop/restart.

### Inspect

```bash
agent-sandbox status
agent-sandbox list
agent-sandbox logs
```

### Finish

```bash
agent-sandbox finish
```

`finish`:

1. stops the control DSH so its persistence layer can flush;
2. exports `changes.patch` relative to the launch-time filesystem baseline;
3. validates DSH durable-state invariants;
4. merges canonical session log data back into the real `DSH_HOME` using add/append-only rules;
5. adds immutable attachment objects;
6. leaves settings, credentials, profiles, storages, and cache untouched;
7. keeps the full staged session archive until you explicitly clean it.

If a safe merge cannot be proven, `finish` fails closed and retains the staging data.

### Apply the generated patch locally

```bash
agent-sandbox apply
```

The launcher runs `git apply --check` first. If it does not apply cleanly, the real repository is not modified.

### Clean

After a successful finish:

```bash
agent-sandbox clean
```

Before a successful finish, cleanup is refused unless you explicitly accept data loss:

```bash
agent-sandbox clean --force
```

## Persistence model

### Code and test work

The staged workspace lives on the host under:

```text
~/.local/state/agent-sandbox/sessions/<session>/workspace
```

It is not an ephemeral Docker writable layer, so normal container stop/restart does not discard work.

### Control DSH state

The staged DSH home lives under:

```text
~/.local/state/agent-sandbox/sessions/<session>/control-home
```

This is also host-side persistent staging. A browser close, container stop, or Docker restart does not by itself delete the staged conversation history.

### Selective write-back

`finish` writes back only:

- canonical files under `sessions/`, using new-file or append-only behavior;
- files under `attachments/v1/`, using add-only/content-equality behavior.

It deliberately does **not** write back:

- `settings.yaml`;
- `.credentials.yaml`;
- profiles;
- storages/plugin state;
- cache;
- other DSH-home state.

The complete staged `control-home` remains available until `clean`, so state not promoted to the global DSH home is still recoverable from the session archive.

### Conflict behavior

The launcher records a durable baseline when the sandbox is created. During `finish`, existing session files must still match a safe prefix relationship. If the same global DSH session has also been advanced outside the sandbox, or an unknown session-owned artifact changed, the merge is refused.

Rule of thumb:

```text
cannot prove safe -> do not modify the real DSH home -> retain staging
```

## Git and pull requests

Each staged workspace has an independent Git repository and gets an `agent/<session-id>` branch. If the source repository has an `origin`, that remote URL is restored in the staged copy.

This makes a PR-based workflow possible without modifying the real local checkout:

```text
sandbox branch -> push -> PR -> review/fix in sandbox -> merge on GitHub -> host fetch/pull
```

`gh` is installed in the image, but **GitHub authentication is not automatically inherited from the host**. This is deliberate. If the agent must push or open PRs, explicitly provide narrowly scoped GitHub credentials through your project/test credential mechanism and configure `git`/`gh` inside the sandbox. Do not mount your whole host `~/.ssh` or GitHub config just for convenience.

The built-in `changes.patch` path remains a fallback even when you primarily use PRs.

## Security notes

This project intentionally favors a small, usable safety boundary over a large zero-trust platform.

The main invariants are:

1. the real development repository is not writable or visible as a mount inside the container;
2. the real DSH home is never mounted read/write;
3. Docker control is not delegated to the agent;
4. Windows drives and the host home are not mounted;
5. normal session data lives in persistent staging, not only the container layer;
6. promotion back to global state is explicit and conservative.

This does **not** mean the agent is hostile-code-safe. It has normal network access and can read secrets that are intentionally staged for DSH or tests.

## State location

Override the session root with:

```bash
export AGENT_SANDBOX_STATE_ROOT=/path/to/state
```

Other environment overrides:

```bash
AGENT_SANDBOX_CONTROL_SOURCE=/path/to/deepseek-harness
AGENT_SANDBOX_DSH_HOME=/path/to/.dsh
AGENT_SANDBOX_IMAGE=my-image:tag
```

## Current limitations

- MVP/experimental; destructive edge cases still deserve backups.
- Designed and tested around DSH's current append-only session persistence layout. Breaking DSH persistence changes may require updating the guarded merge logic.
- The control source must remain at the same clean commit while a session is resumable.
- GitHub PR authentication is intentionally not inherited automatically.
- Global settings/profile/plugin-state changes made inside `control-home` are not promoted by `finish`.
- Normal outbound networking is allowed.
- If the agent destroys the staged Git metadata, the real repository is still safe, but automatic `changes.patch` export may no longer be possible.

## License

MIT. See [LICENSE](LICENSE).
