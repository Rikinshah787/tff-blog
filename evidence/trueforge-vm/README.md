# Fresh-VM run evidence

This directory records a new Linux VM created on November 19, 2026. It is separate from the earlier macOS startup check in `setting-up-trueforge.md`.

The user configured OpenAI directly in the local UI. A saved agent then completed two model turns using `openai/gpt-5-4-mini`. The original prompt and answer survived a full page reload; a follow-up quoted the original input, and the conversation reopened from Sessions. No model key is included in these evidence files.

This is an AI-assisted article draft backed by an actual VM run. A human author should review the prose and technical conclusions before publishing it under their name. Ten minutes is an onboarding target, not a measured end-to-end result.

## Reproduce the VM

Host requirements for this exact recipe: macOS with hardware virtualization, Lima 2.3.0, and enough free resources for a 2-CPU, 4-GiB guest with a 20-GiB sparse disk. The host installation command was `brew install lima`.

From the project directory:

```bash
export LIMA_HOME=/private/tmp/trueforge-blog-vms
limactl start --tty=false --name=trueforge-clean --vm-type=vz \
  --cpus=2 --memory=4 --disk=20 --mount-none --containerd=none \
  --port-forward=18792:8790,static=true template:ubuntu-24.04

limactl shell --workdir=/tmp trueforge-clean -- bash -s \
  < examples/trueforge-vm-bootstrap.sh
```

The exact image URL and digest resolved by Lima are in `run.json`; the template's choice can change in a later Lima release. The bootstrap script verifies its pinned Node archive and installs TrueForge in the guest user's `~/trueforge-starter` directory. It does not install a model key.

For an interactive server:

```bash
limactl shell --workdir=/tmp trueforge-clean
source "$HOME/.profile"
cd "$HOME/trueforge-starter"
mkdir -p data
HOST=127.0.0.1 SQLITE_PATH="$PWD/data/db.sqlite" npx --no-install trueforge
```

Open `http://127.0.0.1:18792` on the host. Guest loopback port 8790 is explicitly forwarded to host loopback port 18792. The VM has no host directory mounts.

During this evidence run the same binary was launched with `nohup` to keep it available during the browser handoff. Its PID is in `~/trueforge-starter/server.pid`, and its log is in `~/trueforge-starter/server.log` inside the guest. The server must already be stopped before running a second copy on port 8790.

## Files

- `run.json`: environment, versions, measured phase durations, and explicit checkpoint status.
- `vm-create.log`: actual VM creation, verified image digest, boot, and port-forward setup.
- `install.log`: actual checksum verification, package installation, versions, warnings, and timing.
- `startup.log`: startup captured before model credentials were entered.
- `cli-check.log`: the exact local-package launch command's help output, lockfile digest, and host-mount check.
- `package-lock.json`: the dependency tree resolved by the guest installation.
- `01-fresh-provider-catalog.png`: the rendered provider catalog before credentials were entered.
- `create-agent.json`: exact credential-free request submitted to `POST /api/v1/agents`.
- `saved-agent.json`: server readback of the saved agent, including the exact instruction and model.
- `02-saved-agent.png`: the saved agent in the UI registry.
- `first-prompt.txt`: the three actual startup lines and question sent in the browser.
- `first-turn.json`: completed first turn, output, usage, and timestamps.
- `completed-turns.json`: both completed turns, including their `previous_turn_id` relationship.
- `03-first-response.png`: the first model response in the browser.
- `04-follow-up-after-reload.png`: the follow-up response after a page reload.
- `05-persisted-session.png`: the Sessions entry showing two turns.
- `article-check.json`: current article hashes and final link, markup, exact inline request/prompt/response, credential-pattern, and responsive-browser checks. Each article Bash block and the HTML's JavaScript passed syntax checks; the Copy button was exercised with a clipboard stub.

## What was actually exercised

The agent was created using the local HTTP API, after browser automation failed to enter its instruction reliably in the code editor. The exact payload was checked before submission and read back from the server afterward. No partially entered editor draft was saved. The UI agent Save path remains unverified; selecting the persisted agent and chatting with it were exercised in the browser.

The first turn reached `done` in 3.009 seconds (289 input tokens, 288 output tokens). A full browser reload restored its prompt and answer. The follow-up reached `done` in 1.058 seconds (593 input tokens, 28 output tokens), with its `previous_turn_id` pointing to the first turn. Navigating to Sessions showed two turns, and reopening the session restored the conversation. These durations are server turn timestamps from one run, not latency benchmarks.

The test did not restart the server or VM, exercise tools or code execution, test other models, or validate multiuser access. The x86-64 bootstrap branch has a verified Node archive checksum but was not executed; the actual guest was ARM64.

The install printed deprecation notices for transitive dependencies. It exited successfully. The server reported a separate missing-dependency warning for the optional local sandbox fallback, but served the UI and API docs. Model calls were verified by the completed turns above. Code execution remains untested.

## Stop or resume

```bash
export LIMA_HOME=/private/tmp/trueforge-blog-vms
limactl stop trueforge-clean
limactl start trueforge-clean
```

Stopping the VM retains its disk. Starting it does not automatically relaunch the manually started TrueForge process. Do not publish a VM disk or a copy of its SQLite database after a provider key has been configured.
