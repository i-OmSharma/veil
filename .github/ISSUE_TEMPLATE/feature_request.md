---
name: Feature request
about: Propose a new capability or an improvement to Veil
title: "feat(<scope>): <short summary>"
labels: enhancement
assignees: ""
---

> [!IMPORTANT]
> Please review the README [Internals Reference](https://github.com/i-OmSharma/veil#internals-reference) before proposing features that conflict with current architecture.
>
> In particular, proposals must keep these invariants intact:
> - The child is `SIGSTOP`'d until its cgroups and networking are in place
> - `container.Run()` runs on the host and `container.Child()` runs inside the new namespaces
> - The image rootfs is read-only (all writes go to the overlay)
> - Everything created during a run is cleaned up when the container exits

<!--
Hidden hint comments like this one won't show up in the issue.
Remove any optional section you don't need.
-->

## Checklist

- [ ] I read the README [Internals Reference](https://github.com/i-OmSharma/veil#internals-reference) and [Limitations](https://github.com/i-OmSharma/veil#limitations-v01)
- [ ] I checked existing issues and this hasn't been proposed yet
- [ ] This is one feature only (open separate issues for separate ideas)

## Feature Description

<!-- What should Veil do that it doesn't do now? One or two sentences. -->

## Use Case

<!--
Why do you need this? What are you trying to run or learn, and what blocks you today?
A concrete example helps a lot, e.g. "I want to run two containers with networking at the same time, but both get 10.88.0.2."
-->

## Related Linux Primitives

<!-- Tick everything this feature touches. -->

- [ ] Namespaces: <!-- which ones? e.g. CLONE_NEWUSER, CLONE_NEWCGROUP, CLONE_NEWTIME -->
- [ ] cgroups v2: <!-- which controller or file? e.g. pids.max, io.max, cpuset.cpus -->
- [ ] OverlayFS / mounts / `pivot_root`
- [ ] Networking: <!-- bridge, veth, iptables, routes, DNS -->
- [ ] OCI images / registries
- [ ] Process lifecycle / signals / state (`/var/lib/veil`)
- [ ] Security: <!-- seccomp, capabilities, LSMs, no_new_privs -->
- [ ] CLI / UX only (no kernel interaction)
- [ ] Other:

**Relevant man pages or kernel docs:**
<!-- e.g. man 7 user_namespaces, https://docs.kernel.org/admin-guide/cgroup-v2.html#pid -->

**Minimum kernel version needed (if known):**
<!-- e.g. 5.6 for time namespaces. Veil currently targets 5.x+. -->

## Host Impact

<!--
What host state would this create or change (mounts, cgroups, iptables rules, sysctls, files outside the overlay)?
How would that state be cleaned up when the container exits?
Write "none" if it's CLI-only.
-->

## Proposed Implementation Approach (optional)

<!--
If you have an idea of how to build this, sketch it out:
- Which packages change? (cmd/veil, internal/container, internal/cgroup, internal/network, internal/overlayfs, internal/image, internal/state)
- Where does it fit in the run lifecycle (before clone, while the child is SIGSTOP'd, inside Child(), during cleanup)?
- New CLI flags?
- How would you test it (unit tests with no root, integration tests that need root)?
-->

## Alternatives Considered (optional)

<!-- Other approaches, or how you work around this today. -->

## Contribution

- [ ] I'd like to implement this myself (see [CONTRIBUTING.md](https://github.com/i-OmSharma/veil/blob/main/CONTRIBUTING.md))
