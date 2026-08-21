# Distribution wiki audit — 2026-08-21

## Result

A second research pass compared the skill with ArchWiki, Debian Wiki and manuals, Ubuntu documentation and Community Help, Fedora Docs, Gentoo Wiki, and openSUSE documentation/wiki. The skill was already stricter than the unsafe or stale cleanup shortcuts found in several wiki pages. Seven narrow improvements were warranted: Btrfs-native accounting, APT `autoclean`, APT autoremove-policy review, traditional-log coverage, Snap inventories, explicit pacman orphan inventory, and an explicit fallback for unsupported package managers.

## Source hierarchy

Sources were not treated as equally authoritative:

1. the owning command's current upstream manual/source and installed-version help;
2. maintained first-party distribution manuals and security documentation;
3. project-hosted, community-edited distribution wikis;
4. historical/community help pages and recipes.

This matters because [ArchWiki is maintained by an official team and community contributors](https://wiki.archlinux.org/title/ArchWiki:About), [Debian Wiki is freely editable](https://wiki.debian.org/DebianWiki), and [Ubuntu Community Help is explicitly user-created and maintained](https://help.ubuntu.com/community/). They are valuable operational sources, but a wiki recipe cannot redefine a command's current semantics.

## Material reviewed

- Local `arch-wiki-docs 20260702-1`: System maintenance, Pacman, Pacman tips, systemd journal, Docker, Podman, Flatpak, Core utilities, Snap, Btrfs, and Snapper.
- Current ArchWiki counterparts where accessible through indexed content.
- Debian Reference, Debian Administrator's Handbook, Debian Policy/manpages, and Debian Wiki.
- Maintained Ubuntu Server/security documentation and the older Ubuntu Community Help Wiki.
- Fedora Quick Docs and current DNF/DNF5 references.
- Gentoo Wiki plus the upstream Portage manual.
- openSUSE Wiki, openSUSE/SUSE manuals, Zypper manpages, and Snapper documentation.

## Findings and applied decisions

| Area | Evidence | Decision |
| --- | --- | --- |
| Btrfs accounting | [ArchWiki Btrfs](https://wiki.archlinux.org/title/Btrfs#Displaying_used/free_space), the upstream [`btrfs filesystem` manual](https://btrfs.readthedocs.io/en/latest/btrfs-filesystem.html), and [SUSE Snapper documentation](https://documentation.suse.com/sles/15-SP7/html/SLES-all/cha-snapper.html) agree that generic `df`/`du` cannot express allocation, shared extents, snapshots, and physical reclaim accurately. | Add `btrfs filesystem usage` and optional `btrfs filesystem du`, inventory snapshots/subvolumes, and mark unavailable snapshot-aware estimates unknown. Snapshot deletion remains outside normal cleanup. |
| APT cache | The [APT manual](https://manpages.debian.org/unstable/apt/apt-get.8.en.html) defines `autoclean` as removing only artifacts no longer downloadable, while `clean` empties the package archive. | Offer `apt-get autoclean` as the lower-impact operation and keep `clean` separate. |
| APT autoremove | [Ubuntu security guidance](https://documentation.ubuntu.com/security/common-mistakes/unnecessary-packages/) documents that `APT::AutoRemove::RecommendsImportant` and `APT::AutoRemove::SuggestsImportant` change the removal set; APT warns that non-root simulation may not read all configuration. | Record effective policy and simulation coverage, then review the exact removal list before execution. |
| Traditional logs | The [`logrotate` manual](https://man7.org/linux/man-pages/man5/logrotate.conf.5.html) says `--debug` changes neither logs nor state and that normal rotation can run configured scripts. | Add a separate logrotate inventory/preview; disclose scripts and never force, directly delete, or truncate active logs. |
| Snap | Canonical separates retained [revisions](https://snapcraft.io/docs/revisions/) from [data snapshots](https://snapcraft.io/docs/how-to-guides/manage-snaps/create-data-snapshots/); both preserve recovery material and removal can itself create a snapshot. | Inventory `snap list --all` and `snap saved` separately; keep revision/snapshot deletion outside normal cleanup. |
| pacman orphans | [Arch system maintenance](https://wiki.archlinux.org/title/System_maintenance) recommends `pacman -Qdt` for inventory. [ArchWiki's removal recipe](https://wiki.archlinux.org/title/Pacman/Tips_and_tricks#Removing_unused_packages_(orphans)) warns that recursive removal can also remove optional dependencies. | Add `pacman -Qdt` as inventory only. Never pipe it to `pacman -Rns`; any removal needs explicit reviewed targets and a separate proposal. |
| Unsupported managers | Portage separates [`emerge --depclean`](https://dev.gentoo.org/~zmedico/portage/doc/man/emerge.1.html) from [eclean](https://wiki.gentoo.org/wiki/Eclean); [Zypper](https://doc.opensuse.org/documentation/tumbleweed/zypper/) has different cache, dependency, and prompt semantics. | Keep the skill small: do not invent Portage/zypper commands. Use separately verified non-mutating inventories and explanation until support is deliberately added. |

## Wiki advice deliberately rejected

- The packaged ArchWiki Docker page describes `docker system prune` defaults incorrectly and suggests direct removal of `/var/lib/docker`. Current [Docker documentation](https://docs.docker.com/reference/cli/docker/system/prune/) remains authoritative: default prune already includes all stopped containers, excludes volumes without `--volumes`, and `-a` broadens image removal.
- ArchWiki's journal page permits direct removal from `/var/log/journal`; the skill retains native, scoped [`journalctl` vacuum](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) instead.
- Older Ubuntu Community Help pages recommend chained commands, direct log deletion/truncation, broad unmounting, purge flags, or fixed cleanup percentages. They are not imported.
- openSUSE community SDB pages contain fixed journal ages and `rm /tmp/* -rf`; neither is a safe universal default. Their Btrfs/Snapper diagnostic insight was retained, not the destructive recipe.
- Gentoo Wiki includes raw deletion examples for distfiles alongside the native `eclean` tool. Because Portage is not a supported operation row, neither recipe is inferred.

## Conclusion

The distribution documentation strengthens the skill's existing principle: wiki guidance is useful for discovering distro-specific storage and recovery behavior, but mutations must remain scoped to the installed tool's current, owning documentation. No wiki shortcut was allowed to weaken approval, scope, prompt, recovery, or native-cleaner boundaries.

## Validation

- The revised skill passed `quick_validate.py` and Git whitespace/error checks.
- Installed read-only help confirmed the documented Btrfs, logrotate, and pacman query syntax; Snap was not installed locally, so its commands were verified against Canonical documentation.
- Three Luna-max subagents forward-tested Arch/Btrfs/pacman, Debian/Ubuntu/APT/logrotate/Snap, and openSUSE/Gentoo fallback scenarios without running cleanup commands. All three passed with no remaining actionable defect.
