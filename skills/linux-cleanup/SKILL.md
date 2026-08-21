---
name: linux-cleanup
description: Audit and safely reclaim disk space on Linux with native package-manager, journal, Flatpak, Docker, and Podman tools. Use when a Linux filesystem is full or the user asks to inspect disk usage, clean caches or logs, remove unused packages or runtimes, or prune container data. Keep broad requests non-mutating until the user selects an exact cleanup operation.
license: MIT
---

# Linux cleanup

Reclaim space without treating “unused” as permission to delete. Follow: **audit → propose → approve → execute one → verify**.

## Authority boundary

- Treat a broad request such as “clean this Linux machine” as authority to audit and propose, not to mutate.
- If the user names an exact operation and its scope and effects are clear, revalidate it and proceed without asking the same question again.
- Never broaden scope: user to system, rootless to rootful, one filesystem to another, local to remote, or one installation/daemon to another.
- Preserve personal files, configuration, databases, backups, snapshots, package rollback material, stopped containers and their writable layers, volumes, and application data by default.
- Use the subsystem's native cleanup command. Never delete directly from package-manager, journal, Flatpak, Docker, or Podman stores.
- During audit, do not use `sudo`, refresh package metadata, access the network, or run a command known to write state. Report permission and coverage gaps instead.
- Use explicit existing targets. Never pass `/`, a home directory, a workspace root, unresolved variables, globs, or `..` to a cleanup command.
- Approval is the safeguard; a tool prompt is not approval. Retain prompts and check whether configuration or environment disables them. Never add automatic-confirmation, force, purge, all-resources, or volume-removal options.

The skill never grants permission beyond the user's request or the host agent's approval policy.

## 1. Establish scope

1. Parse `/etc/os-release` as data when present; otherwise use `/usr/lib/os-release`. Never source or combine them.
2. Resolve the requested target to an existing canonical path. Record its current mount namespace, mount point, filesystem type, free blocks, and free inodes.
3. Inspect the target's mount tree. Identify nested mounts, including same-filesystem bind mounts, before traversing it.
4. Detect installed tools and versions. Distinguish DNF 4 from DNF 5 and Flatpak user, system, and named installations.
5. Resolve the effective local Docker or Podman endpoint, including context/connection and environment or CLI overrides. Distinguish local rootless from local rootful stores; stop on a remote target unless the user explicitly requested it.
6. Stop and ask when the target filesystem or execution context is ambiguous.

Complete this step only when the target filesystem and every inspected context are explicit.

## 2. Build a non-mutating baseline

The examples assume GNU/Linux; confirm options against the installed implementation and use consistent explicit units when comparing measurements.

- Measure the target with `df -hT` and `df -i` using a path on that filesystem.
- Rank large directories with `du` on the canonical target. Exclude every nested or unrelated mount found above. Treat `-x`/`--one-file-system` only as a same-device guard: it does not block same-filesystem bind mounts.
- If inodes are scarce, rank inode-heavy directories with `du --inodes --one-file-system` or an installed equivalent, applying the same recorded mount exclusions; `--one-file-system` still does not exclude same-filesystem bind mounts.
- Inspect only relevant installed subsystems:
  - journal: inventory every in-scope `--system`, explicitly selected `--user`, or directory context separately with `journalctl --disk-usage`; never treat one or an access-limited view as the total
  - Docker: after resolving the effective local daemon, use `docker system df -v`; add `docker buildx du` for the selected builder when BuildKit/buildx is relevant
  - Podman: after resolving the effective local connection/store, use `podman system df -v` and state that its documented view does not inventory every build-cache or external-storage candidate
  - Flatpak: inventory each selected installation separately; there is no transaction-equivalent dry run for `uninstall --unused`
  - package managers: measure caches and preview orphan candidates with the matching native tool where that preview is non-mutating
- If visible `du` usage is materially below `df`, or space remains allocated after cleanup, inspect deleted-but-open files with an installed read-only diagnostic such as `lsof` or readable `/proc/*/fd`. Report permission/namespace gaps; never manipulate `/proc` entries or restart a service without separate approval.
- Treat every reclaimable size as an estimate. Shared layers, hard links, copy-on-write, compression, active journal files, and deleted-but-open files make totals non-additive. Never present a combined total unless the scopes and methods are demonstrably disjoint and additive.

Complete this step with a before snapshot, coverage gaps, and candidates from authoritative subsystem inventories.

## 3. Propose exact operations

Present a numbered table containing, for each operation:

- scope and target
- current size and estimated reclaimable space, including confidence
- exact command
- what will be removed and preserved
- privilege required and recovery cost

Keep package caches separate from package removal, user scope separate from system scope, and every container-object category separate from volumes. Ask the user to select operation numbers. Classification is advice, not approval.

Do not invent retention limits, ages, version counts, filters, or thresholds. Obtain them from a documented default or explicit user choice and show the consequences. If upstream exposes no exact dry run, label the candidate set and recovery estimate as unknown until the normal transaction prompt.

## 4. Use conservative native operations

Confirm every command and option against the installed version's `--help` or man page.

| Subsystem | Non-mutating preflight | One operation after approval |
| --- | --- | --- |
| APT | measure the archive cache; `apt-get -s autoremove` for package candidates, treating simulation as an estimate | `apt-get clean` for cached packages; `apt-get autoremove` is a separate operation |
| DNF 4 | `dnf -C list --autoremove` with cached metadata | `dnf clean packages`; `dnf autoremove` is a separate operation |
| DNF 5 | `dnf5 -C info --autoremove` prevents downloads but a non-root run may create `~/.cache/libdnf5`; omit it unless that cache write was explicitly accepted | `dnf5 clean packages`; `dnf5 autoremove` is a separate operation |
| pacman | `paccache -d` using the configured cache | `paccache -r` keeps its documented default of three versions; it has no confirmation and may invoke `sudo` for an unwritable system cache, so disclose scope and privilege first |
| systemd journal | scoped `journalctl --disk-usage`, which includes active and archived files | scoped `journalctl --vacuum-time=...` or `--vacuum-size=...`, which removes archived files only; use one combined `--rotate --vacuum-*` command only if active data is explicitly included |
| Flatpak | advisory inventory of the selected installation; no uninstall dry run | exactly one of `flatpak --user uninstall --unused`, `flatpak --system uninstall --unused`, or `flatpak --installation=NAME uninstall --unused`; review the prompt for unused runtimes/extensions and never add `--delete-data` |
| Docker | effective local daemon plus `docker system df -v`; use category-specific inventories | one matching component command, such as `docker image prune`, `docker builder prune`, `docker network prune`, or `docker container prune`; state that container prune removes all stopped containers and their writable layers |
| Podman | effective local connection/store plus `podman system df -v` | one matching component command, such as `podman image prune`, `podman network prune`, or `podman container prune`; state the exact category and defaults |

Package-cache cleanup deletes local package artifacts and may remove offline reinstall or downgrade material. `apt-get clean` and DNF package-cache cleanup delete all cached packages; `paccache -r` retains only its documented default. Protect rollback artifacts or state that recovery may require a repository and network download.

Before a package mutation, check for effective auto-answer settings such as APT assume-yes/quiet configuration, DNF `assumeyes`, or DNF 5 `DNF5_FORCE_INTERACTIVE`. Stop if the interaction differs from the proposal.

For application caches, resolve the effective cache root per XDG: accept `$XDG_CACHE_HOME` only when non-empty and absolute; otherwise use `$HOME/.cache`. Identify the owner and use its documented cleaner. Never empty the cache root wholesale.

Keep outside the normal path: Snap revisions/snapshots, Flatpak application data, package purges, stopped-container removal, Docker/Podman aggregate `system prune`, container volumes, and Podman `--build` or `--external`. Audit one only when the user explicitly requests that exact high-risk target and understands the recovery cost. Arbitrary files found by `du` are report-only; this skill never deletes them.

## 5. Execute one operation

Immediately before execution, recheck the target filesystem and mount identity, command version, user, effective daemon/installation, candidate scope, privilege, and prompt behavior. Re-resolve any path target; if any precondition differs from the proposal, invalidate approval and return to audit.

Run exactly one approved command. Do not chain cleanup commands. Stop on an error, unexpected candidate set, expanded scope, or a privilege request or warning not disclosed in the proposal.

After interruption, rebuild the audit; never resume from an old candidate list.

## 6. Verify and report

After each operation:

1. Re-run the same subsystem inventory and `df` measurements.
2. Report the observed delta separately from the estimate.
3. List what ran, what was preserved, what failed, and what remains eligible.
4. Continue only if the next operation is still explicitly approved and its preconditions are unchanged; otherwise ask.

Complete only when every executed operation has before/after evidence and no unapproved operation ran.
