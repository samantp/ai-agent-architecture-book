# Chapter 6 — Execution and Sandboxing

Bash is the agent's most powerful tool and its entire attack surface. The engineering splits into two problems that are often conflated: **statefulness** (§6.1 — making a sequence of commands behave like one coherent shell) and **containment** (§6.2–6.4 — bounding what those commands can do). The projects here cover the full spectrum of containment: none (trust + approvals), OS-level sandboxing of individual processes, and full container isolation of the *agent itself*.

## 6.1 Persistent shells: statefulness is not optional

A fresh shell per command breaks `cd`, exports, activated virtualenvs, and background servers — and models *assume* shell state persists, because humans' shells do. Every mature runtime therefore maintains long-lived sessions:

- **SWE-agent** routes every command into a persistent session on a remote "deployment" through the SWE-ReX abstraction — `SWE-agent@1132b3e/sweagent/environment/swe_env.py:197` **[Verified]**:

```python
def communicate(self, input: str, timeout=25, *, check="ignore", ...) -> str:
    r = asyncio.run(self.deployment.runtime.run_in_session(
        BashAction(command=input, timeout=timeout, check=rex_check)))
    ...
```

  `AbstractDeployment` (imported at `swe_env.py:8-17`) has Docker/local/remote implementations — the agent code is identical whether the shell lives in a container on the laptop or a fleet in the cloud (how SWE-bench runs thousands of parallel instances). `read_file`/`write_file` are separate RPCs on the same runtime rather than `cat`-over-shell — structured operations avoid quoting/encoding hazards.
- **Codex** has a dedicated `unified_exec` subsystem (`open-interpreter@764a96e/codex-rs/core/src/unified_exec/`) for persistent interactive sessions, plus `shell_snapshot.rs` — capturing the user's shell environment (PATH, aliases-adjacent config) so agent commands see the same world the user's terminal does **[Verified structure]**.
- **opencode** ships its shell tool with a persistent PTY layer (`opencode@34e5809/packages/opencode/src/tool/shell/`, `shell.ts`) and the server owns the processes, so TUI restarts don't kill running builds **[Verified structure]**.

Universal complications, solved in all three: *output framing* (knowing when a command finished — sentinel markers/exit-code capture inside the session), *timeouts as first-class results* (a timeout is an observation for the model, not an exception), and *ANSI stripping* before results enter context.

## 6.2 OS-level sandboxing: the Codex depth chart

The `codex-rs` tree contains the most serious open-source implementation of per-process sandboxing for agents — one backend per OS, unified behind a policy layer (`open-interpreter@764a96e/codex-rs/sandboxing/` crate, adapted into the core at `open-interpreter@764a96e/codex-rs/core/src/sandboxing/mod.rs`) **[Verified]**.

**macOS — Seatbelt.** Commands run under `sandbox-exec` with a generated profile. The checked-in base policy (`open-interpreter@764a96e/codex-rs/sandboxing/src/seatbelt_base_policy.sbpl` **[Verified]**) opens with the two lines that matter:

```scheme
; inspired by Chrome's sandbox policy
(deny default)              ; closed by default — everything below is allowlist
(allow process-exec)
(allow process-fork)
(allow signal (target same-sandbox))
(allow file-write-data (require-all (path "/dev/null") (vnode-type CHARACTER-DEVICE)))
(allow sysctl-read (sysctl-name "hw.activecpu") ...)   ; enumerated, not wildcarded
```

Deny-default with enumerated allowances (down to individual sysctls), writable roots parameterized per invocation (the workspace directory), and a separate network policy file (`seatbelt_network_policy.sbpl`). The Chrome-sandbox citation is the tell: this is browser-grade sandboxing technique applied to agent tool calls.

**Linux — Landlock + seccomp via re-exec.** `open-interpreter@764a96e/codex-rs/sandboxing/src/landlock.rs` builds *command-line arguments for a helper binary* (`create_linux_sandbox_command_args`, `:23-65` **[Verified]**) — the agent re-executes itself (`codex-linux-sandbox`, the `open-interpreter@764a96e/codex-rs/linux-sandbox/` crate) which installs Landlock filesystem rules + seccomp filters and then `exec`s the target. Sandboxing-by-wrapper keeps the policy out-of-process and composable. **Fallback tiers**: `bwrap.rs` (bubblewrap) and `windows-sandbox-rs` cover other environments; `SandboxType` selects per platform.

**Beyond syscalls — policy on commands and network:**

- The `open-interpreter@764a96e/codex-rs/execpolicy/` crate defines which *commands* are safe, in **Starlark** policy files **[Verified structure]** — a programmable allow/deny/match layer (`exec_policy.rs`, with `command_canonicalization.rs` normalizing invocations so `python3 -c ...` can't smuggle past a `python` rule) that runs before any OS sandbox is even consulted.
- Network egress is its own subsystem: a managed proxy (`codex_network_proxy`, wired through `ExecRequest.network` — `open-interpreter@764a96e/codex-rs/core/src/sandboxing/mod.rs:46-68` **[Verified]**), per-request network approvals (`open-interpreter@764a96e/codex-rs/core/src/tools/network_approval.rs`), and honest environment markers so tooling inside the sandbox can adapt: `CODEX_SANDBOX_NETWORK_DISABLED=1`, `CODEX_SANDBOX=seatbelt` (`mod.rs:140-149` **[Verified]**).

The composite is a **defense-in-depth stack**: command policy (Starlark) → OS mechanism (seatbelt/Landlock/bwrap) → network proxy → and above all of it, the approval loop: run sandboxed by default; on a sandbox denial, surface an approval request to run outside it **[Inference from `open-interpreter@764a96e/codex-rs/core/src/tools/sandboxing.rs` + approval templates; the escalation flow is Codex's documented behavior]**.

Cline's SDK, notably, is growing the same organ: `cline@6309971/sdk/packages/core/src/runtime/tools/subprocess-sandbox.ts` **[Verified structure]** — evidence that per-process sandboxing is becoming table stakes even for IDE-resident agents.

## 6.3 Container isolation: OpenHands puts the agent in the box

OpenHands V1 inverts the classic layout (Ch. 1.3D): instead of sandboxing *commands*, it sandboxes **the whole agent**. The app server's Docker sandbox service — `OpenHands@3949e1c/openhands/app_server/sandbox/docker_sandbox_service.py:385-513` **[Verified]**:

```python
async def start_sandbox(self, sandbox_spec_id=None, sandbox_id=None) -> SandboxInfo:
    await self.pause_old_sandboxes(self.max_num_sandboxes - 1)   # LRU capacity mgmt
    sandbox_id = base62.encodebytes(os.urandom(16))
    session_api_key = base62.encodebytes(os.urandom(32))          # per-sandbox credential
    env_vars[SESSION_API_KEY_VARIABLE] = session_api_key
    env_vars[WEBHOOK_CALLBACK_VARIABLE] = \
        f'http://host.docker.internal:{self.host_port}/api/v1/webhooks'
    ...
    container = self.docker_client.containers.run(
        image=sandbox_spec.id,          # an *agent server* image, not a bare runtime
        environment=env_vars,
        ports=port_mappings,            # container ports → random host ports
        volumes=volumes,
        init=True,                      # tini: signal handling + zombie reaping
        ...)
```

The architecture that falls out:

- The container runs the **agent server** (loop + tools + LLM access, from the external `openhands-agent-server` package). The app server *orchestrates*: starts/pauses sandboxes, proxies the UI, stores events.
- **Events flow back over webhooks** — the sandboxed agent POSTs to `host.docker.internal:<port>/api/v1/webhooks`; the app server persists and re-streams to browsers (`OpenHands@3949e1c/openhands/app_server/event/`, `event_callback/`). Contrast with opencode (shared DB) and Cline (in-process events): once the agent is network-isolated from its UI, your event bus becomes HTTP.
- **Credential scoping**: each sandbox gets a random session API key; a compromised sandbox can spoof only itself.
- The unglamorous details are the production story: LRU pausing to cap concurrent sandboxes, `init=True` for zombie reaping, random host ports with env-var advertisement, CORS origins injected for remote browser access (`:427-437`), optional host networking and KVM passthrough (`:470-481`).
- The same `SandboxService` interface has **process** (local subprocess — dev mode) and **remote** (delegated to a fleet API) implementations (`process_sandbox_service.py`, `remote_sandbox_service.py` **[Verified structure]**) — isolation level is a deployment decision, not an architecture change.

## 6.4 Choosing a containment model

| | Trust + approvals (Cline classic, opencode default) | Per-process OS sandbox (Codex) | Agent-in-container (OpenHands) |
|---|---|---|---|
| Blast radius of a bad command | user account | policy'd process | container |
| Prompt-injection posture | human is the firewall | policy is; human for escalations | container is |
| Local-workflow fidelity (IDE, keychain, running services) | full | high | low (mounts/ports only) |
| Infra cost | none | helper binaries per OS | images, registry, orchestration |
| Latency overhead | none | ~ms per exec | container lifecycle + HTTP hops |

Decision heuristics:

1. **The threat that matters is not the model "going rogue" — it is the *repo attacking the agent*** (malicious build scripts, poisoned READMEs steering an agent that reads them). Approvals alone don't survive that, because the human approves a command that *looks* innocent (`npm install`). Egress control (Codex's network proxy; container network policy) is the control that actually contains exfiltration.
2. Match containment to *who* triggers the agent: developer-in-the-loop on their own repo → approvals + OS sandbox; autonomous/cloud/multi-tenant → agent-in-container, no exceptions.
3. Whatever the mechanism, keep the **policy declarative and versioned** (`.sbpl` files, Starlark execpolicy, sandbox specs) — security policy embedded in imperative code cannot be audited or diffed.
4. Tell the sandboxed world it is sandboxed (`CODEX_SANDBOX_NETWORK_DISABLED=1`): tools that know degrade gracefully; tools that don't produce confusing failures the model then wastes tokens diagnosing.
