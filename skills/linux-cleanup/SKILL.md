---
name: linux-cleanup
description: Audit Linux disk usage and safely reclaim space from package caches, logs, unused runtimes, and container data. Use for a full filesystem or a cleanup request. Audit first; run only an approved operation.
license: MIT
---

# Linux cleanup

Reclaim space without treating "unused" as permission to delete. Work in five stages: **audit → propose → approve → execute one → verify**.

## Safety boundary

- A broad request such as "clean this Linux machine" allows an audit and proposal, not changes.
- If the user names an operation and its scope and effects are clear, revalidate it and proceed. Do not ask the same question again.
- Keep the approved scope. Do not cross from user to system, rootless to rootful, one filesystem to another, local to remote, or one installation or daemon to another.
- Preserve personal files, configuration, databases, backups, snapshots, package rollback material, stopped containers, writable layers, volumes, and application data by default.
- Use the subsystem's cleanup command. Do not delete files directly from package-manager, journal, Flatpak, Docker, or Podman stores.
- Keep the audit read-only. Do not use `sudo`, access the network, refresh package metadata, or run commands known to write state. Report what the audit could not inspect.
- Use existing resolved targets. Do not pass `/`, a home directory, a workspace root, unresolved variables, globs, or `..` to a cleanup command.
- Treat approval as the safeguard. A tool prompt is not approval. Keep prompts enabled and do not add force, purge, all-resources, automatic-confirmation, or volume-removal options.

This skill does not grant permission beyond the user's request or the host agent's approval policy.

## 1. Establish scope

1. Parse `/etc/os-release` as data when present. Otherwise parse `/usr/lib/os-release`. Do not source or combine them.
2. Resolve the target to an existing canonical path. Record its mount namespace, mount point, filesystem type, free blocks, and free inodes.
3. Inspect the target's mount tree. Record nested mounts, including same-filesystem bind mounts, before traversing it.
4. Detect installed tools and versions. Distinguish DNF 4 from DNF 5. Record in-scope Flatpak user, system, and named installations.
5. Resolve the active Docker or Podman endpoint, including context, connection, environment, and CLI overrides. Distinguish local rootless and rootful stores. Stop on a remote target unless the user requested it.
6. Ask when the filesystem or execution context remains ambiguous.

Do not continue until the target filesystem and every inspected context are clear.

## 2. Audit without changes

These examples assume GNU/Linux. Check options against the installed implementation and use consistent units.

- Measure the target with `df -hT` and `df -i`, using a path on that filesystem.
- On Btrfs, add `btrfs filesystem usage MOUNT`. Use `btrfs filesystem du PATH` when shared and exclusive usage matters.
- Report missing Btrfs detail when permissions limit it. Keep estimated free space, `statfs` or `df`, and shared or exclusive usage distinct.
- Inventory Btrfs subvolumes and snapshots with the installed manager. Generic `du` and a snapshot's logical size do not show physical reclaimable space.
- Rank large directories with `du` on the canonical target. Exclude each nested or unrelated mount recorded above.
- Treat `-x` or `--one-file-system` as a same-device guard. It does not block same-filesystem bind mounts.
- If inodes are scarce, rank inode-heavy directories with `du --inodes --one-file-system` or an installed equivalent. Apply the same mount exclusions.
- Inspect only installed subsystems that matter to the target:
  - Journal: run `journalctl --disk-usage` for each selected `--system`, `--user`, or `--directory` scope. Report access limits. One view may not be the total.
  - Traditional logs: map relevant files to their effective logrotate configuration. Preview it with `logrotate --debug CONFIG`. Keep this result separate from journald.
  - Docker: after resolving the local daemon, use `docker system df -v`. Add `docker buildx du` when the selected builder matters.
  - Podman: after resolving the local connection and store, use `podman system df -v`. Its documented output omits some build-cache and external-storage candidates.
  - Flatpak: inventory each selected installation separately. `uninstall --unused` has no transaction-equivalent dry run.
  - Snap: list retained revisions with `snap list --all` and data snapshots with `snap saved`. Keep the two inventories separate.
  - Package managers: measure caches and use a native read-only orphan preview when one exists.
- If `du` is materially below `df`, inspect deleted-but-open files with `lsof` or readable `/proc/*/fd`.
  Report anything hidden by permissions or namespaces. Do not manipulate `/proc` or restart a service without separate approval.
- Treat reclaimable sizes as estimates. Shared layers, hard links, reflinks, snapshots, copy-on-write, compression, active journals, and deleted-but-open files can make estimates overlap.
- Add estimates only when their scopes and methods do not overlap.

Finish the audit with before measurements, unreadable scopes, and candidates reported by the relevant subsystem tools.

## 3. Propose operations

Present a numbered table. For each operation, show:

- scope and target
- current size and estimated reclaimable space, with confidence
- command
- data removed and preserved
- required privilege and recovery cost

Keep package caches separate from package removal. Keep user and system scopes separate. Split container cleanup by object type and keep volumes separate.

Ask the user to select operation numbers unless the original request already approved one listed operation with the same scope and effects. A category such as "packages" or "Docker" is not an operation.

Do not invent retention limits, ages, version counts, filters, or thresholds.
Use a documented default or ask the user to choose.
If no exact dry run exists, mark the candidate list and recovery estimate as unknown. Review the normal transaction prompt when the tool provides one.

## 4. Use native cleanup commands

Check each command and option against the installed version's `--help` or man page. Each item below is a separate operation.

- **APT.** Resolve `Dir::Cache::archives` from the effective APT configuration. Do not assume `/var/cache/apt/archives`.
  Measure the resolved cache for cache operations. Use `apt-get -s autoremove` only to inspect package-removal candidates.
  After approval, choose one command.
  `apt-get autoclean` removes cached archive files that can no longer be downloaded.
  `apt-get clean` removes all cached archive files.
  `apt-get autoremove` removes installed packages that APT marks as no longer needed.
- **DNF 4.** Use `dnf -C list --autoremove` with cached metadata to inspect candidates.
  After approval, use either `dnf clean packages` or `dnf autoremove`.
- **DNF 5.** `dnf5 -C info --autoremove` prevents downloads, but a non-root run may create `~/.cache/libdnf5`. Skip it unless the user accepts that write.
  After approval, use either `dnf5 clean packages` or `dnf5 autoremove`.
- **pacman.** Use `paccache -d` to inspect its configured cache and `pacman -Qdt` to list true orphans.
  After approval, `paccache -r` keeps its documented default of three versions. It has no confirmation and may invoke `sudo` when the system cache is not writable, so disclose both facts first.
- **systemd journal.** Use scoped `journalctl --disk-usage` to measure active and archived files.
  After approval, use one scoped `journalctl --vacuum-time=...` or `--vacuum-size=...` command. Vacuuming removes archived files only.
  Add `--rotate` in the same command only when the user also approved moving active data into scope.
- **logrotate.** Use `logrotate --debug CONFIG` to preview eligible logs without changing logs or state.
  After approval, use `logrotate CONFIG`. Disclose configured pre- and post-rotate scripts and state changes. Do not add `--force`, delete logs directly, or truncate active logs.
- **Flatpak.** Inventory the selected installation. There is no uninstall dry run.
  After approval, use one of `flatpak --user uninstall --unused`, `flatpak --system uninstall --unused`, or `flatpak --installation=NAME uninstall --unused`. Review the transaction prompt and do not add `--delete-data`.
- **Docker.** Resolve the local daemon and use `docker system df -v` plus category-specific inventories.
  After approval, use one matching component command such as `docker image prune`, `docker builder prune`, or `docker network prune`.
- **Podman.** Resolve the local connection and store, then use `podman system df -v`.
  After approval, use one matching component command such as `podman image prune` or `podman network prune`.

Package-cache cleanup may remove offline reinstall or downgrade material. Protect required rollback packages or state that recovery may require a repository and network access.

Before APT autoremove, record `APT::AutoRemove::RecommendsImportant` and `APT::AutoRemove::SuggestsImportant`. Note when the simulation could not read root-only configuration, then review the removal list.

Before changing packages, check whether configuration disables normal prompts.
Check APT assume-yes and quiet settings, DNF `assumeyes`, and DNF 5 `DNF5_FORCE_INTERACTIVE`. Stop if prompt behavior differs from the proposal.

`pacman -Qdt` is inventory only. Do not pipe it to `pacman -Rns`. Package removal needs a separate proposal with reviewed package names.

For an unlisted package manager such as Portage or zypper, do not guess cleanup commands. Use only native read-only inventories that you verified on the installed version.

For application caches, accept `$XDG_CACHE_HOME` only when it is non-empty and absolute. Otherwise use `$HOME/.cache`. Identify the owner and use its documented cleaner. Do not empty the cache root.

### Require a separate request

Do not propose these after a broad cleanup request:

- Snap revisions or snapshots
- Flatpak application data
- package purges
- stopped containers and their writable layers, including `docker container prune` and `podman container prune`
- Docker or Podman `system prune`
- container volumes
- Podman `--build` or `--external`

Audit one only when the user names that target and understands the recovery cost. Files found by `du` are report-only. This skill does not delete them.

## 5. Execute one operation

Immediately before execution, recheck the filesystem and mount identity, command version, user, daemon or installation, candidate list, privilege, and prompt behavior.
Resolve path targets again. If any result differs from the proposal, discard the approval and return to the audit.

Run one approved command. Do not chain cleanup commands. Stop on an error, a changed candidate list, wider scope, or an undisclosed privilege request or warning.

After an interruption, rebuild the audit. Do not reuse an old candidate list.

## 6. Verify and report

After each operation:

1. Run the same subsystem inventory and `df` measurements.
2. Report the observed change separately from the estimate.
3. List what ran, what was preserved, what failed, and what remains eligible.
4. Continue only when the next operation is still approved and its preconditions have not changed. Otherwise ask.

Finish only when every executed operation has before and after evidence and no unapproved operation ran.
