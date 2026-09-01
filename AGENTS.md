# AGENTS.md

## Project

`minisandbox` is an experimental lightweight Linux sandbox runtime designed primarily for AI agent execution.

The current priority is building the smallest correct MVP, not designing a complete container platform.

## Primary Goal

Implement:

```bash
minisandbox run "<command>"
```

The command must execute inside an ephemeral Linux environment isolated from the host.

## MVP Requirements

The sandbox should eventually provide:

1. PID namespace isolation
2. Mount namespace isolation
3. UTS/hostname isolation
4. IPC isolation
5. Minimal BusyBox root filesystem
6. Read-only base filesystem
7. Writable ephemeral `/tmp`
8. Writable `/workspace`
9. cgroups v2 memory limits
10. cgroups v2 CPU limits
11. process count limits
12. execution timeout
13. stdout/stderr capture
14. exit-code reporting
15. cleanup after execution
16. networking disabled by default

Security hardening such as user namespaces, capability dropping, and seccomp should follow immediately after the basic execution path works.

## Development Philosophy

Keep the implementation extremely small.

Prefer direct Linux primitives over large abstractions.

Good:

```text
Rust
 ↓
Linux syscall/interface
 ↓
kernel
```

Avoid introducing:

```text
Rust
 ↓
large container SDK
 ↓
Docker
 ↓
containerd
 ↓
runc
 ↓
kernel
```

unless there is a concrete reason.

## Scope Discipline

Do not add features simply because conventional container runtimes have them.

Do not implement during the initial MVP:

- OCI compatibility
- Dockerfiles
- Registries
- Kubernetes
- Persistent containers
- VM management
- Package installation
- HTTP server
- Authentication
- Multi-node execution
- Agent orchestration
- Web UI
- Database
- Prewarming

Build the execution primitive first.

## Implementation Order

Work approximately in this order:

```text
BusyBox rootfs
      ↓
namespaces
      ↓
correct PID isolation
      ↓
command execution
      ↓
stdout/stderr/exit code
      ↓
cgroups
      ↓
filesystem isolation
      ↓
timeouts + cleanup
      ↓
privilege reduction
      ↓
seccomp
      ↓
network isolation
      ↓
benchmarks
```

Do not prematurely optimize startup performance.

## Rust Guidelines

Prefer simple Rust.

Use `nix` where it provides a clean interface to Linux functionality.

Use `libc` or direct syscalls only when:

- `nix` does not expose the required functionality
- the abstraction introduces meaningful overhead
- direct control is required

Avoid unnecessary:

- async
- macros
- traits
- generics
- dependency injection
- framework-like abstractions

This is systems software. Make control flow obvious.

## Error Handling

Do not silently ignore failures involving isolation or resource limits.

If any security-critical setup operation fails, sandbox creation should fail.

Examples:

- namespace creation failure
- mount failure
- cgroup configuration failure
- privilege dropping failure
- seccomp installation failure

Fail closed rather than continuing with reduced isolation.

Use descriptive errors that identify which sandbox setup stage failed.

## Security

Treat executed commands as potentially hostile.

Never intentionally expose:

- host filesystem
- host process namespace
- environment secrets
- cloud credentials
- SSH keys
- Docker socket
- runtime sockets
- arbitrary host devices

Do not assume `chroot` alone is a security boundary.

Do not assume namespaces alone make arbitrary hostile execution safe.

Never weaken isolation simply to make a test pass.

## Filesystem

The sandbox should eventually see approximately:

```text
/
├── bin/
├── proc/
├── tmp/
└── workspace/
```

Keep the base root filesystem minimal.

Do not copy host libraries or binaries into the sandbox unless required.

Prefer statically linked tools where practical.

## Processes

PID namespace behavior must be implemented correctly.

Remember that creating a PID namespace affects subsequently created child processes. Do not assume calling `unshare(CLONE_NEWPID)` makes the current process PID 1.

The sandbox should have an appropriate PID 1 process responsible for child-process cleanup.

Avoid zombie processes.

When terminating a sandbox, terminate its entire process tree.

## Resource Limits

Use cgroups v2.

At minimum enforce:

```text
memory.max
pids.max
cpu.max
```

Never rely solely on application-level checks for resource enforcement.

## Networking

Default:

```text
network = false
```

Do not give a sandbox host networking by default.

Networking can become an explicit capability later.

## Performance

Correctness and isolation come first.

Once the MVP works, benchmark:

```text
create latency
time to first exec
total command latency
peak RSS
incremental RSS
rootfs size
cleanup latency
```

Measure before optimizing.

Do not claim performance improvements without benchmarks.

## Testing

Tests should actively attempt to violate sandbox boundaries.

Examples:

```bash
# Process isolation
ps

# Filesystem isolation
ls /
ls /home
ls /root

# Hostname isolation
hostname

# Memory limit
allocate memory until limit

# PID limit
fork repeatedly

# Timeout
sleep 1000

# Filesystem writes
touch /should-fail
touch /tmp/should-work
```

Add regression tests whenever an isolation bug is discovered.

## Dependencies

Be conservative about adding dependencies.

Before adding one, ask:

1. Can the standard library do this?
2. Can `nix` already do this?
3. Is the dependency significantly simplifying security-sensitive code?
4. What does it add to compile time and attack surface?

Small dependencies are preferable to large frameworks.

## Future Architecture

Once the core runtime is stable, the intended abstraction is capability-based sandboxes.

Example:

```text
Sandbox
├── shell
├── python
└── git
```

rather than conventional distro images.

Possible future interface:

```rust
Sandbox::new()
    .tool("python")
    .tool("git")
    .memory_mb(128)
    .network(false)
    .run(command)
```

Do not implement this abstraction until the basic runtime is correct.

## Definition of MVP Done

The MVP is complete when this works reliably:

```bash
minisandbox run "echo hello"
```

while providing real filesystem/process/resource isolation, deterministic cleanup, and a minimal BusyBox environment.

Everything else is secondary.
