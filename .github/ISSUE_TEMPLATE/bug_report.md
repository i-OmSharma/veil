---
name: Bug report
about: Report something that broke or behaved unexpectedly
title: "bug(<scope>): <short summary>"
labels: bug
assignees: ""
---

<!--
⚠️ SECURITY: If this is a container escape or isolation bypass (namespace escape,
cgroup bypass, writes landing on the host outside the overlay), DO NOT file a
public issue. Report it privately instead:
https://github.com/i-OmSharma/veil/security/advisories/new

Hidden hint comments like this one won't show up in the issue. Replace the
placeholders below and remove any section that doesn't apply.
-->

## Checklist

- [ ] I tested this in a VM or other disposable environment, not on my main machine
- [ ] I checked existing issues and this hasn't been reported yet
- [ ] I can reproduce this on the latest `main` or the latest release

## Environment

| Field | Value |
|-------|-------|
| **Kernel Version** | <!-- uname -r --> |
| **Distro** | <!-- grep PRETTY_NAME /etc/os-release --> |
| **Veil Version** | <!-- git describe --tags --always, or "veil-linux-amd64 from release vX.Y.Z" --> |
| **Go Version** | <!-- go version (only if you built from source) --> |
| **cgroup filesystem** | <!-- stat -fc %T /sys/fs/cgroup (should be cgroup2fs) --> |
| **Where it ran** | <!-- e.g. Multipass VM, Lima, QEMU, cloud VM --> |
| **SELinux / AppArmor** | <!-- getenforce, or aa-status --> |
| **Other runtimes on the machine** | <!-- e.g. Docker, Podman, none --> |

## Command Used

<!-- The exact command, with every flag. -->

```shell
sudo ./veil run ...
```

## Expected Behavior

<!-- What you expected to happen. -->

## Actual Behavior

<!-- What happened instead. Does it happen every time, or only sometimes? -->

## Steps to Reproduce

<!-- Only needed if the single command above isn't enough to trigger the bug. -->

1.
2.
3.

## Logs/Error Output

<!-- Full output from veil, including the [container], [cgroup] and [network] lines. -->

```text

```

<details>
<summary><strong>Host state after the failure</strong> (please fill in for lifecycle, mount, cgroup or network bugs)</summary>

<!-- Run these inside the VM and paste the output. -->

```shell
findmnt | grep veil
ls -d /tmp/veil-* /sys/fs/cgroup/veil-* 2>/dev/null
ip link show | grep -E 'veil0|veth-'
sudo iptables -t nat -S | grep -iE 'dnat|masquerade'
sudo cat /var/lib/veil/state.json
cat /sys/fs/cgroup/cgroup.subtree_control
```

```text

```

</details>

<details>
<summary><strong>Kernel messages</strong> (OOM kills, overlay errors, SELinux/AppArmor denials)</summary>

```shell
sudo dmesg | tail -50
```

```text

```

</details>

## Additional Context

<!-- Anything else: what you've already tried, a workaround, a suspected cause. -->
