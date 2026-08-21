---
name: linux-cleanup
description: Audit and safely reclaim disk space on Linux with native package-manager, journal, Flatpak, Docker, and Podman tools. Use when a Linux filesystem is full or the user asks to inspect disk usage, clean caches or logs, remove unused packages or runtimes, or prune container data. Keep broad requests read-only until the user selects an exact cleanup operation.
license: MIT
---

# Linux cleanup

Reclaim space without treating “unused” as permission to delete. Follow this sequence: **audit → propose → approve → execute one → verify**.

## Authority boundary

- Treat a broad request such as “clean this Linux machine” as authority to audit and propose, not to mutate.
- If the user names an exact operation or command and its scope and effects are clear, revalidate it and proceed without asking the same question again.
- Never broaden scope: user to system, rootless to rootful, one filesystem to another, or local to remote.
- Preserve personal files, configuration, databases, backups, snapshots, package rollback material, container volumes, and application data by default.
- Use the subsystem's own cleanup command. Do not delete files directly from package-manager, journal, Flatpak, Docker, or Podman stores.
- During audit, do not use `sudo`, refresh package metadata, or access the network. Report permission gaps instead.
- Operate on explicit existing targets. Do not pass `/`, a home directory, a workspace root, unresolved variables, globs, or `..` to a removal command.
- Keep interactive safeguards. Do not add automatic confirmation, force, purge, all-resources, or volume-removal flags.

The skill never grants permission beyond the user's request or the host agent's approval policy.

## 1. Establish scope

1. Read `/etc/os-release` when present.
2. Identify the filesystem that needs space and record its mount point, type, free blocks, and free inodes with `df`.
3. Detect installed tools and their versions. Distinguish DNF 4 from DNF 5, Flatpak user from system installations, and Docker/Podman local rootless from local rootful contexts.
4. Stop and ask when the target filesystem or execution context is ambiguous.

Complete this step only when the target filesystem and every inspected context are explicit.

## 2. Build a read-only baseline

- Measure the target with `df -hT` and `df -i` using a path on that filesystem.
- Find large directories with `du` restricted to that filesystem (`-x` or the installed equivalent). Do not traverse pseudo-filesystems, remote mounts, or unrelated mount points.
- Inspect only subsystems that are installed and relevant:
  - systemd journal: `journalctl --disk-usage`
  - Docker: first confirm the context is local, then `docker system df -v`
  - Podman: confirm the user/storage context, then `podman system df -v`
  - Flatpak: inspect user and system installations separately
  - package managers: inspect their cache and preview orphan candidates with the matching native tool
- Treat reclaimable sizes as estimates. Shared container layers, hard links, active journal files, and deleted-but-open files can make totals non-additive. Do not present a combined total unless the scopes and measurement methods are demonstrably disjoint and additive.

Complete this step with a before snapshot, the coverage gaps, and candidates from authoritative subsystem inventories.

## 3. Propose exact operations

Present a numbered table containing, for each operation:

- scope and target
- current size and estimated reclaimable space, including confidence
- exact command
- what will be removed and what will be preserved
- privilege required and recovery cost

Keep package caches separate from package removal, user scope separate from system scope, and container objects separate from volumes. Ask the user to select operation numbers. Classification is advice, not approval.

Do not invent retention limits, ages, version counts, or other cleanup thresholds. Obtain them from documented defaults or an explicit user choice and show their consequences.

## 4. Use conservative native operations

First confirm each command and option against the installed version's `--help` or man page.

| Subsystem | Read-only preview | Conservative operation after approval |
| --- | --- | --- |
| APT | measure its archive cache; `apt-get -s autoremove` for package candidates, treating simulation as a preview rather than a transaction guarantee | `apt-get clean` for downloaded packages; `apt-get autoremove` only as a separate approved operation |
| DNF 4 | `dnf -C list --autoremove` using cached metadata only | `dnf clean packages`; `dnf autoremove` only as a separate approved operation |
| DNF 5 | `dnf5 -C info --autoremove` using cached metadata only | `dnf5 clean packages`; `dnf5 autoremove` only as a separate approved operation |
| pacman | `paccache -d` | `paccache -r`, whose default keeps the latest three package versions |
| systemd journal | `journalctl --disk-usage` | `journalctl --vacuum-time=...` or `--vacuum-size=...` using the user's chosen limit; explain that vacuum removes archived files |
| Flatpak | inspect the selected installation and unused runtimes | `flatpak uninstall --unused` in the explicitly selected user or system scope |
| Docker | local context plus `docker system df -v` | `docker system prune` only after listing its categories and retaining its confirmation prompt |
| Podman | user/storage context plus `podman system df -v` | `podman system prune` only after listing its categories and retaining its confirmation prompt |

Treat application caches as application-specific work: identify the owner and use its documented cleaner. Do not empty `$XDG_CACHE_HOME` or `~/.cache` wholesale.

Keep these outside the normal cleanup path: Snap revisions and snapshots, Flatpak application data, package purges, Docker/Podman volumes, Podman build containers or external storage, and arbitrary files found by `du`. Handle one only when the user explicitly requests that exact high-risk target and recovery consequences are understood.

## 5. Execute one operation

Immediately before execution, recheck the target filesystem, command version, user, and runtime context. If any differs from the proposal, invalidate the approval and return to audit.

Run exactly one approved command. Do not chain cleanup commands. Stop on an error, unexpected candidate set, expanded scope, or interactive warning that differs from the proposal.

After interruption, rebuild the audit; do not resume from an old candidate list.

## 6. Verify and report

After each operation:

1. Re-run the subsystem inventory.
2. Re-run the same `df` measurements.
3. Report the observed delta separately from the estimate.
4. List what ran, what was preserved, what failed, and what remains eligible.
5. Continue only when the next operation remains explicitly approved and its preconditions are unchanged; otherwise ask.

Complete the task only when every executed operation has before/after evidence and no unapproved operation ran.
