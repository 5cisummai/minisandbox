# minisandbox

A tiny, fast Linux sandbox runtime for AI agents.

`minisandbox` creates disposable shell environments with minimal startup overhead and a very small filesystem. The initial goal is not to build another general-purpose container platform. It is to provide the smallest useful execution environment for agents that need to run shell commands and code safely.

## MVP Goal

Make this work:

```bash
minisandbox run "echo hello"
```

The command should execute inside an isolated environment with:

- Its own process namespace
- Its own mount namespace
- Its own hostname
- A minimal BusyBox root filesystem
- CPU and memory limits
- Process limits
- An ephemeral `/tmp`
- A `/workspace` directory
- No network access by default

The sandbox should disappear completely after the command exits.

## Non-Goals

For the MVP, do not build:

- Docker compatibility
- OCI image support
- Kubernetes integration
- A package manager
- Persistent containers
- VM isolation
- Networking configuration
- Agent orchestration
- HTTP APIs
- Prewarming
- Multiple runtime images

Those can come later.

## Architecture

```text
Agent / CLI
     │
     ▼
 minisandbox
     │
     ├── namespaces
     ├── cgroups v2
     ├── filesystem
     └── process execution
             │
             ▼
      minimal rootfs
        ├── /bin
        ├── /proc
        ├── /tmp
        └── /workspace
```

The runtime should be a thin wrapper around Linux primitives rather than a wrapper around Docker.

## Technology

- Rust
- Linux namespaces
- cgroups v2
- BusyBox
- `tmpfs`
- `nix`
- `libc`

Later:

- seccomp
- user namespaces
- overlayfs
- capability-based tool mounting

## Project Structure

```text
minisandbox/
├── Cargo.toml
├── README.md
├── AGENTS.md
├── rootfs/
│   ├── bin/
│   ├── proc/
│   ├── tmp/
│   └── workspace/
└── src/
    └── main.rs
```

Keep this structure until the implementation becomes large enough to justify splitting modules.

Eventually:

```text
src/
├── main.rs
├── sandbox.rs
├── namespaces.rs
├── filesystem.rs
├── cgroups.rs
└── process.rs
```

## Setup

`minisandbox` requires Linux. macOS cannot directly provide the Linux namespace and cgroup features used by the runtime.

On Ubuntu:

```bash
sudo apt update
sudo apt install build-essential busybox-static
```

Install Rust if necessary, then:

```bash
git clone <repo>
cd minisandbox
cargo build
```

Create the root filesystem:

```bash
mkdir -p rootfs/{bin,proc,tmp,workspace}

cp /usr/bin/busybox rootfs/bin/busybox

cd rootfs/bin

ln -s busybox sh
ln -s busybox ls
ln -s busybox cat
ln -s busybox echo
ln -s busybox ps
ln -s busybox hostname

cd ../..
```

Test it:

```bash
sudo chroot rootfs /bin/sh
```

## MVP Interface

Initially support only:

```bash
minisandbox run "<command>"
```

Example:

```bash
minisandbox run "echo hello"
minisandbox run "ls /"
minisandbox run "ps"
```

Later:

```bash
minisandbox run \
  --memory 64m \
  --cpus 0.5 \
  --timeout 10s \
  "some command"
```

## MVP Milestones

### 1. Minimal Rootfs

Boot into a BusyBox root filesystem.

Verify:

```bash
ls /
```

returns only the sandbox filesystem.

### 2. Namespaces

Isolate:

- Mounts
- PID tree
- Hostname
- IPC

Verify that processes and hostname are isolated from the host.

### 3. Command Execution

Implement:

```bash
minisandbox run "echo hello"
```

Capture:

- stdout
- stderr
- exit code

### 4. Resource Limits

Use cgroups v2 to enforce:

- Memory
- CPU
- Maximum process count

A sandbox must not be able to consume unlimited host resources.

### 5. Ephemeral Filesystem

Make the base filesystem read-only.

Provide:

```text
/tmp
/workspace
```

as writable sandbox-local storage.

Destroy all ephemeral state after execution.

### 6. Timeout

Commands exceeding their allowed execution time must be terminated along with their child processes.

### 7. Security Hardening

After the basic runtime works:

- Drop Linux capabilities
- Add user namespaces
- Run as an unprivileged UID
- Add seccomp filtering
- Disable networking
- Prevent unexpected host filesystem access

Do not describe the runtime as secure for hostile production workloads until these protections have been implemented and reviewed.

## Design Principles

### Small

Every feature must justify its runtime and complexity cost.

### Fast

Eventually measure:

- Sandbox creation latency
- Time to first command
- Peak RSS
- Incremental memory per sandbox
- Rootfs size

Do not optimize these until correctness and isolation work.

### Ephemeral

Sandboxes are disposable execution environments, not long-running servers.

### Agent-first

The eventual API should expose execution capabilities rather than container concepts.

Instead of:

```text
image: ubuntu:latest
```

prefer something like:

```text
tools: ["python", "git"]
```

The runtime should eventually construct the smallest environment capable of satisfying those requirements.

## Long-Term Direction

A future API might look like:

```rust
let sandbox = Sandbox::new()
    .memory_mb(64)
    .pids(32)
    .network(false);

let result = sandbox.run("echo hello")?;
```

Eventually, an external agent API could expose:

```ts
const sandbox = await createSandbox({
  tools: ["python", "git"],
  memory: "128mb",
  network: false,
});

const result = await sandbox.exec("python main.py");
```

But the first version should remain a small Linux runtime with one job:

> Execute a command in the smallest practical isolated environment.
