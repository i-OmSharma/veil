# Contributing to Veil

Thanks for your interest in Veil, a container runtime written from scratch in Go. It talks straight to the Linux kernel through namespaces, cgroups v2, OverlayFS, `pivot_root`, veth pairs and iptables. That makes it a good way to learn how containers really work. It also means a bug can damage the machine it runs on.

This guide covers what is specific to this repository. Please read the safety section before anything else.

## Topics

* [⚠️ Safety Warning](#️-safety-warning)
* [Development Environment Setup](#development-environment-setup)
* [Safe Testing Workflow](#safe-testing-workflow)
* [Code Style](#code-style)
* [Architecture Overview](#architecture-overview)
* [Testing](#testing)
* [Dependencies](#dependencies)
* [Documentation](#documentation)
* [Pull Request Guidelines](#pull-request-guidelines)
* [How to Add New Features](#how-to-add-new-features)
* [Reporting Bugs](#reporting-bugs)

---

## ⚠️ Safety Warning

> [!CAUTION]
> **Only develop and test Veil inside a virtual machine or another disposable environment. Never run it on your main workstation, a laptop with data you care about, or any shared or production host.**

Veil runs as **root** and changes global host state. A bug in a branch you're working on, or in Veil as it stands today, can:

| What Veil touches | What can go wrong on the host |
|-------------------|-------------------------------|
| **Mount namespaces, `pivot_root`, bind mounts** | If mount propagation isn't set to `MS_PRIVATE` correctly, mounts and unmounts leak back to the host. A bad bind mount or a failed `pivot_root` can hide or **unmount host filesystems**, including `/dev`. |
| **OverlayFS** (`/tmp/veil-<id>/`) | Overlay mounts left behind after a crash. A cleanup bug that runs `RemoveAll` on the wrong path can **delete host files**. |
| **cgroups v2** (`/sys/fs/cgroup/`) | Veil writes to the **root** `cgroup.subtree_control`. Mistakes there change how resources are controlled for the whole system and can break systemd's resource management. |
| **Networking** (`veil0` bridge, veth pairs, iptables, firewalld) | Veil turns on `net.ipv4.ip_forward`, adds `MASQUERADE`, `DNAT` and `FORWARD` rules, and puts `veil0` in firewalld's **trusted** zone. A bad rule can **cut off host networking**, open ports to the network, or conflict with Docker, Podman or your firewall setup. |
| **Process control** (`SIGSTOP`, `SIGKILL`, PID tracking) | A wrong PID, or a stale entry in `state.json`, can **signal or kill an unrelated host process**. |
| **State** (`/var/lib/veil/`) | Root-owned files that stay behind across reboots. |

**Rules for contributors:**

1. **Use a VM.** A throwaway VM is the only setup we support for development. Take a snapshot before you test, and roll back when something looks wrong.
2. **Don't run Veil on a host that also runs Docker, Podman, Kubernetes or a firewall you care about.** The iptables and bridge changes can collide with theirs.
3. **Don't test on cloud instances that hold real workloads or credentials.**
4. **Read the code before you run it as root**, including code in PRs you review. `sudo ./veil` runs whatever is in that branch with full privileges.
5. **When something goes wrong, restore the snapshot.** Don't try to clean up a broken mount table by hand on a machine you depend on.

> [!WARNING]
> **A privileged container (`docker run --privileged`) is not a safe place to test.** It gives the container close to full access to the host kernel, so mistakes with cgroups and iptables made inside it still change the real host. Use a VM.

When you're unsure whether something is safe to run, assume it isn't and use the VM.

---

## Development Environment Setup

### Requirements

| Requirement | Version | Why |
|-------------|---------|-----|
| **Linux kernel** | 5.x or newer | cgroups v2 unified hierarchy, modern OverlayFS, `memory.oom_group` |
| **cgroups v2** | unified mode | Veil only supports `/sys/fs/cgroup` as cgroup2. cgroups v1 and hybrid mode are not supported. |
| **Go** | see [`go.mod`](go.mod) (currently 1.25) | `go.mod` is the source of truth for the minimum Go version |
| **root / sudo** | — | Namespaces, mounts, cgroups and netlink all need `CAP_SYS_ADMIN` / `CAP_NET_ADMIN` |
| **iptables**, **iproute2** (`ip`, `nsenter`) | any | NAT, port forwarding and network setup |

### Create a disposable VM

Any hypervisor will do. Here's an example with [Multipass](https://multipass.run/):

```shell
# On your host
multipass launch 24.04 --name veil-dev --cpus 2 --memory 4G --disk 20G
multipass shell veil-dev
```

Lima, Vagrant, virt-manager/QEMU, GNOME Boxes, or a cloud VM you can delete all work too. Pick whichever makes snapshots easy for you.

### Prepare the VM

```shell
# cgroups v2 must be active (should print "cgroup2fs")
stat -fc %T /sys/fs/cgroup

# Kernel 5.x or newer
uname -r

# Runtime tools
sudo apt-get install -y iptables iproute2

# Go: install the version from go.mod or newer (https://go.dev/dl/), then check it
go version
```

### Fork and clone

Fork the project on GitHub, then clone your fork **inside the VM**:

```shell
git clone git@github.com:<you>/veil.git
cd veil
git remote add upstream https://github.com/i-OmSharma/veil.git
go build -o veil ./cmd/veil
```

Keep your fork up to date with `git fetch upstream && git rebase upstream/main`.

### Take a snapshot

Multipass can only snapshot a stopped instance:

```shell
# On your host
multipass stop veil-dev
multipass snapshot veil-dev --name clean
multipass start veil-dev
```

This clean, working state is what you'll roll back to.

---

## Safe Testing Workflow

### The development loop

Run all of this **inside the VM**:

```shell
# 1. Static checks. gofmt -l always exits 0, so test its output instead.
test -z "$(gofmt -l .)" && go vet ./... && golangci-lint run

# 2. Build
go build -o veil ./cmd/veil

# 3. Unit tests: no root needed, no changes to the host
go test ./...

# 4. Privileged integration tests. -exec sudo builds as your user and runs
#    only the test binary as root, which keeps root-owned files out of your Go cache.
go test -exec sudo -tags integration ./...

# 5. Manual smoke test
sudo ./veil run alpine:latest /bin/sh -c 'echo ok; hostname; cat /proc/1/cmdline'
sudo ./veil ps
```

### Check for leftovers

A correct run leaves nothing behind once the container exits. Run this after any change to the lifecycle, mount, cgroup or network code, and include its output when you report a bug:

```shell
findmnt | grep veil                       # overlay mounts
ls -d /tmp/veil-* /sys/fs/cgroup/veil-* 2>/dev/null   # overlay dirs and cgroups
ip link show | grep -E 'veil0|veth-'      # bridge and veth pairs
sudo iptables -t nat -S | grep -iE 'dnat|masquerade'  # NAT rules
```

The `veil0` bridge and its `-s 10.88.0.0/16 -j MASQUERADE` rule are created once and kept on purpose. Anything else in this output that's left over from a container which has exited is a bug. Please report it, even if it wasn't what you were working on.

### Using a pre-built binary

Pre-built binaries (`veil-linux-amd64`) are published on the [Releases page](https://github.com/i-OmSharma/veil/releases). They are **not** in the git tree. A release binary helps you tell whether a bug comes from your changes or was already there. Download it **inside the VM**, never on your host:

```shell
curl -LO https://github.com/i-OmSharma/veil/releases/download/<tag>/veil-linux-amd64
chmod +x veil-linux-amd64
sudo ./veil-linux-amd64 run alpine:latest /bin/sh
```

If the release lists checksums, compare `sha256sum veil-linux-amd64` against them before you run it.

### When something goes wrong

**Restore the snapshot:**

```shell
multipass stop veil-dev && multipass restore veil-dev.clean
```

Don't spend time repairing a VM whose mount table, cgroup tree or iptables rules may now be in an unknown state.

---

## Code Style

### Formatting and linting

All of these must pass:

```shell
test -z "$(gofmt -l .)"
go vet ./...
golangci-lint run    # install: https://golangci-lint.run/welcome/install/
```

### Conventions in this codebase

Read a couple of files in `internal/` before you start, and match what's there:

- **Explain the kernel, not only the Go.** Every syscall, flag, or write to a cgroup or `/proc` file gets a comment saying *what the kernel does* with it and *why Veil needs it*. [`internal/cgroup/cgroup.go`](internal/cgroup/cgroup.go) is the reference for this.
- **Wrap errors with context**: `fmt.Errorf("mkdir cgroup: %w", err)`.
- **Prefix log lines with the subsystem**: `[cgroup]`, `[network]`, `[container]`.
- **Separate best-effort steps from required ones.** If a failure shouldn't stop the container from starting (for example `memory.oom_group` on older kernels), log a warning and carry on. If it would weaken isolation or limits, return an error.
- **Clean up in reverse order of setup**, and make sure every setup path, including error paths, undoes what it already did.

### Commit messages

We use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

<optional body: why this change, not what>
```

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`. Scopes match package names: `cgroup`, `container`, `network`, `overlayfs`, `image`, `state`, `cli`.

```
feat(cgroup): add pids.max limit via --pids-limit
fix(network): remove DNAT rule when container start fails
```

**Every commit must build on its own**, so that `git bisect` keeps working.

**Never commit binaries** (`veil`, `veil-linux-amd64`, `*.test`). Release binaries belong on the Releases page.

---

## Architecture Overview

The README's [How It Works](README.md#how-it-works) section has the full lifecycle diagram, and [Internals Reference](README.md#internals-reference) describes each package. In short:

```
veil run ──► image (pull/cache) ──► overlayfs (CoW mount)
         ──► clone(NEW{PID,UTS,NS,IPC,NET}) re-exec /proc/self/exe child
         ──► SIGSTOP child ──► cgroup.Apply + network.SetupVeth ──► SIGCONT
         ──► [child] MS_PRIVATE → hostname → mounts → pivot_root → /proc → exec
         ──► Wait ──► cleanup (cgroup → port rules → veth → overlay)
```

Keep these invariants intact:

1. **The child is frozen before any limits are applied.** cgroups and networking are set up while the child is `SIGSTOP`'d, so the container never runs a single instruction without its limits in place.
2. **The re-exec boundary.** `main.go` catches `child` before cobra parses anything. Code in `container.Child()` runs *inside* the new namespaces. Code in `container.Run()` runs on the *host*.
3. **The image rootfs is read-only.** All writes go to the overlay upper dir.
4. **Cleanup is complete.** Everything created during `Run()` is removed when the container exits.

---

## Testing

> [!NOTE]
> Veil doesn't have a test suite or CI yet. That makes tests one of the most useful things you can contribute. Until CI exists, the maintainer runs the checks from the [development loop](#the-development-loop) by hand before merging, so run them yourself before you open a PR.

All PRs should include tests that cover their changes. PRs that add a syscall, kernel interface, flag or feature without tests will get closer review and may be held until tests are added. The maintainer decides during review when tests aren't needed (for example docs-only changes).

Veil has two kinds of tests. Keep them separate so that `go test ./...` stays safe to run anywhere.

### Unit tests

Unit tests must not need root and must not change the host. Point the code at a temp directory instead of real kernel paths:

```go
func TestApplyWritesMemoryMax(t *testing.T) {
	root := t.TempDir() // fake /sys/fs/cgroup
	// ... exercise logic against root, assert file contents
}
```

Some packages still hard-code paths such as `cgroupRoot = "/sys/fs/cgroup"`. A PR that refactors them so the path can be injected is welcome on its own.

### Integration tests

Integration tests touch the real kernel. They **must** carry the `integration` build tag, skip when not running as root, and clean up through `t.Cleanup` even when they fail:

```go
//go:build integration

package cgroup

func TestApplyRealCgroup(t *testing.T) {
	if os.Geteuid() != 0 {
		t.Skip("requires root; run inside a disposable VM")
	}
	// ... create the real cgroup and assert
	t.Cleanup(func() { _ = cfg.Remove() })
}
```

Run them **only in your VM**: `go test -exec sudo -tags integration ./...`

---

## Dependencies

Veil is meant to stay small and readable, so every new dependency has to earn its place.

- **Prefer the standard library** and `golang.org/x/sys/unix` for syscalls. Implementing a primitive yourself is part of what this project is for.
- **Explain any new dependency in the PR**: why it's needed, and why the standard library isn't enough.
- **Run `go mod tidy`** and commit `go.mod` and `go.sum` together.
- **Put dependency bumps in their own PR**, separate from feature work.

---

## Documentation

Update the docs in the same PR as the code:

- **New or changed flags** go in the command tables in [README.md](README.md).
- **New kernel interactions** (namespaces, cgroup files, mounts, iptables rules) go in the README's [How It Works](README.md#how-it-works) and [Internals Reference](README.md#internals-reference) sections.
- **New host paths** go in the README's [Storage Paths](README.md#storage-paths) table.
- **Package doc comments** start with `// Package <name> ...` and include a diagram of the kernel state the package manages, as in `internal/cgroup`.

---

## Pull Request Guidelines

- **One concern per PR.** A new cgroup controller and a networking refactor are two PRs. Small PRs are faster to review and safer to merge.
- **Open an issue first for anything non-trivial**, especially changes to kernel-facing behaviour.
- **Describe the host impact.** Every PR description must say which host state the change touches (mounts, cgroups, iptables, sysctls, files outside the overlay) and how that state is cleaned up.
- **Say how you tested**: distro, kernel version, and the commands you ran in your VM.
- **Include tests and docs**, as described in [Testing](#testing) and [Documentation](#documentation).

### PR checklist

Copy this into your PR description:

```markdown
- [ ] One concern only
- [ ] gofmt / go vet / golangci-lint pass
- [ ] Unit tests added (no root required)
- [ ] Integration tests added for new kernel interactions (`-tags integration`)
- [ ] Tested inside a VM: distro ___, kernel ___
- [ ] Leftover check is clean after exit
- [ ] Host impact described above
- [ ] README updated (flags / internals / storage paths)
- [ ] No binaries committed
```

### Review and merging

- The maintainer reviews PRs with GitHub's review feature. Reply to every comment, either with a change or with your reasoning.
- **Push follow-up commits while the PR is in review** instead of force-pushing, so reviewers can see what changed. Squash or tidy them up only when asked, just before merge.
- When the PR is approved and the checks pass, the maintainer merges it.

---

## How to Add New Features

Two worked examples follow. Open an issue describing the design before you start.

### Example A: adding a namespace type (e.g. `CLONE_NEWCGROUP`)

1. **Read the man page first.** `man 7 namespaces` and `man 7 cgroup_namespaces`. Make sure you know what becomes isolated, what the minimum kernel version is, and how the new namespace interacts with the ones Veil already creates.
2. **Add the flag** in [`internal/container/run.go`](internal/container/run.go), next to the other `CLONE_*` flags. Add a comment line in the same style as the existing ones:
   ```go
   // CLONE_NEWCGROUP — container sees its own cgroup as the root of /sys/fs/cgroup
   ```
3. **Check the ordering.** A cgroup namespace's root is fixed at `clone()` time, which is before `cgroup.Apply()` moves the PID. Work out what that means for what the container sees, and write the reasoning down in a comment.
4. **Set up anything needed inside the container** in [`internal/container/child.go`](internal/container/child.go) (e.g. mounting `cgroup2` at `/sys/fs/cgroup` after `pivot_root`). Remember that this code runs inside the new namespaces.
5. **If the namespace is optional, make it opt-in** with a CLI flag in [`cmd/veil/main.go`](cmd/veil/main.go), following `--no-net`.
6. **Tests:**
   - Integration: start a container and compare `readlink /proc/self/ns/cgroup` inside it with the host's value. They must differ.
   - The leftover check is clean after the container exits.
7. **Docs:** update the README Features table and the How It Works diagram.

### Example B: adding a cgroup controller (e.g. `pids.max`)

1. **Read** the [cgroup v2 docs](https://docs.kernel.org/admin-guide/cgroup-v2.html) section for the controller. Note the file names, value format, and what `max` / `0` mean.
2. **Add a field** to `CgroupConfig` in [`internal/cgroup/cgroup.go`](internal/cgroup/cgroup.go), with a comment that gives the unit:
   ```go
   PidsMax int64 // Maximum number of processes in the cgroup. 0 = unlimited.
   ```
3. **Enable the controller** by adding `"pids"` to the `enableControllers(...)` call. This writes to the **root** `cgroup.subtree_control`, so be careful: on some distros systemd manages that file.
4. **Write the limit** inside `Apply()`, guarded by `if c.PidsMax > 0`. Decide whether a failure is fatal (the limit is a security boundary) or best-effort, and justify the choice in a comment.
5. **Add the CLI flag** in `cmd/veil/main.go` (`--pids-limit`) and pass it through `container.Run()`.
6. **Update the hierarchy diagram** in the package doc comment and in the README.
7. **Tests:**
   - Unit: given a fake cgroup root (`t.TempDir()`), `Apply()` writes the right value to `pids.max`, and writes nothing when the value is 0.
   - Integration: run a container with `--pids-limit 10` and start processes in a loop inside it (e.g. `sh -c 'while :; do sleep 60 & done'`). Check that forks start failing at the limit while the host stays responsive. **This is exactly the kind of test that must never run outside a VM.**

---

## Reporting Bugs

Open an issue with the following information. Most runtime bugs depend on the kernel and distro, so without these details we usually can't reproduce them.

### Required information

```shell
# Veil version: the release tag or commit you built from
git describe --tags --always    # or say "veil-linux-amd64 from release vX.Y.Z"

# Kernel and distro
uname -a
cat /etc/os-release

# cgroups setup
stat -fc %T /sys/fs/cgroup              # should be cgroup2fs
cat /sys/fs/cgroup/cgroup.controllers
cat /sys/fs/cgroup/cgroup.subtree_control

# Go version (if you built from source)
go version
```

### Also include

- **Environment:** the VM or hypervisor, SELinux/AppArmor status (`getenforce` or `aa-status`), and any other container runtimes running on the machine.
- **The exact command** you ran, with all flags.
- **The full output**, including the `[container]`, `[cgroup]` and `[network]` log lines.
- **Expected and actual behaviour.**
- **The output of the [leftover check](#check-for-leftovers)**, plus `sudo cat /var/lib/veil/state.json`.
- **Kernel messages**, if relevant: `sudo dmesg | tail -50` (look for OOM kills, overlay errors, AppArmor or SELinux denials).

### Security issues

If you find a way to **escape a container** or break isolation (namespace escape, cgroup bypass, writes outside the overlay that land on the host), **don't open a public issue.** Report it privately through GitHub's [private vulnerability reporting](https://github.com/i-OmSharma/veil/security/advisories/new) for this repository.

---

## Thank you

Every fixed leak, clearer comment and new test makes Veil a better way to learn how containers work. We're glad to help with your first kernel-level PR, so open a draft early and ask questions.

And one last time: **VM first, snapshot before you test, never on your main machine.** 🛡️
