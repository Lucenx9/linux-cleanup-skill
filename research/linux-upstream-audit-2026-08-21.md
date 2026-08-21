# Linux upstream audit — 2026-08-21

## Result

The skill's safety model was sound: broad requests remained non-mutating, native subsystem cleaners were preferred, scopes were separated, and every mutation required approval and verification. The audit found no invalid APT, DNF, pacman, journal, Flatpak, Docker, or Podman command name. It did find several scope and recovery details that needed tighter wording; those fixes are now reflected in `skills/linux-cleanup/SKILL.md`.

## Method

Three independent subagents reviewed package managers, filesystem/systemd, and runtimes/containers. Each used current primary documentation and recorded the exact affected passage, severity, supporting URL, and minimal fix. The main review then reconciled overlaps and kept only guidance that changes a cleanup decision or safety boundary.

Arch-specific claims were also checked against the locally installed `arch-wiki-docs 20260702-1` package. ArchWiki is treated as a strong distribution-specific operational source; upstream project manuals and the installed command's help remain authoritative for command semantics.

## Findings and applied decisions

| Area | Verified finding | Applied decision |
| --- | --- | --- |
| Filesystem identity | GNU `du -x`/`--one-file-system` is a same-device boundary, so a same-filesystem bind mount can still be traversed. | Inspect the current mount tree first and explicitly exclude nested mounts; do not describe `-x` as a complete mount guard. |
| Inodes and hidden allocation | `df -i` reports the filesystem symptom; `du --inodes` can locate inode-heavy trees. Linux retains an unlinked file until its final open descriptor closes. | Add inode drill-down and a read-only deleted-open-file diagnostic when `df` and visible `du` materially disagree. |
| Distribution detection | [`os-release(5)`](https://www.freedesktop.org/software/systemd/man/latest/os-release.html) specifies `/usr/lib/os-release` as the fallback and the files must not be combined. | Parse the preferred file as data; never source it. |
| Journal | [`journalctl --disk-usage`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) includes active and archived files, while `--vacuum-*` removes archived files only; scope and permissions affect visibility and mutation. | Require an explicit journal scope, explain the post-vacuum difference, and use rotate-plus-vacuum only when active data is explicitly included. |
| Package cache recovery | [APT clean](https://manpages.debian.org/unstable/apt/apt-get.8.en.html) and DNF package-cache cleanup delete cached artifacts; [`paccache -r`](https://man.archlinux.org/man/paccache.8) keeps three versions by default. Cached artifacts can support offline reinstall or downgrade. | Show recovery cost before approval and never describe cache deletion as preserving all rollback material. |
| DNF 5 audit behavior | [DNF 5 caching](https://dnf5.readthedocs.io/en/latest/misc/caching.7.html) documents that a non-root query can clone root metadata into the user cache even though `--cacheonly` prevents downloads. | Omit that query unless the cache write was explicitly accepted, and report the missing candidate list. |
| pacman privilege | The upstream [`paccache` implementation](https://github.com/archlinux/pacman-contrib/blob/master/src/paccache.sh.in) can invoke `sudo` when removing from an unwritable configured cache. | Treat privilege as part of the proposed scope and stop on an unannounced elevation request. |
| Interactive safeguards | APT and DNF can suppress prompts through configuration or environment, not only CLI flags; `paccache -r` has no confirmation prompt. | Make explicit approval primary and verify effective prompt behavior before a mutation. |
| Flatpak scope and preview | [Flatpak uninstall](https://docs.flatpak.org/en/latest/flatpak-command-reference.html#flatpak-uninstall) provides user/system/named-installation selectors and no `uninstall --dry-run`; `--unused` covers unused refs such as runtimes and extensions. | Put the selector in the exact command, describe inventory as advisory, review the normal prompt, and never add `--delete-data`. |
| Docker/Podman target | Docker context and Podman storage identity alone do not exclude remote endpoint overrides. | Resolve the effective endpoint/connection and stop on remote scope unless explicitly requested. |
| Container prune granularity | Current [Docker pruning documentation](https://docs.docker.com/engine/manage-resources/pruning/) says aggregate system prune includes all stopped containers, unused networks, dangling images, and build cache; [Podman system prune](https://docs.podman.io/en/stable/markdown/podman-system-prune.1.html) is also multi-category. Stopped containers can retain writable-layer data. | Prefer one component-specific prune per approval. Keep aggregate system prune, volumes, and broad Podman storage cleanup outside the normal path. |
| Cache inventory coverage | [`docker system df`](https://docs.docker.com/reference/cli/docker/system/df/) does not replace the selected builder's [`docker buildx du`](https://docs.docker.com/reference/cli/docker/buildx/du/) view; Podman's documented system-df scope also does not cover every external/build candidate. | Report build-cache coverage separately and do not manufacture a complete reclaim estimate. |
| XDG caches | The [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/) accepts only absolute XDG paths and defaults an unset/empty cache home to `$HOME/.cache`. | Resolve the effective root, use an owner-specific documented cleaner, and never empty the root wholesale. |

## ArchWiki cross-check and source conflict

The local ArchWiki Pacman guidance agrees with upstream `paccache`: the default retains three versions, while more aggressive cache deletion sacrifices locally available downgrade/reinstall artifacts.

The local ArchWiki Docker page did not agree with the current Docker reference. Its cleanup paragraph described unflagged `docker system prune` as including dangling volumes and implied `-a` was needed for stopped containers. The current [Docker `system prune` reference](https://docs.docker.com/reference/cli/docker/system/prune/) says the opposite on those points: unflagged prune removes all stopped containers, volumes are excluded unless `--volumes` is supplied, and `-a` expands image removal. The skill therefore follows Docker's official reference and the installed client's help, not that ArchWiki paragraph.

## Validation

- The Agent Skill structure passed the repository-independent `quick_validate.py` validator.
- The Markdown diff passed Git whitespace/error checks.
- Three Luna-max subagents forward-tested the revised skill without running host cleanup commands: filesystem/journal scope, DNF5/pacman privilege and rollback behavior, and Flatpak/Docker/Podman scope. The first pass exposed two ambiguous stop/scope rules; both were corrected and the affected scenarios then passed on re-test.

## Sources reviewed

- [GNU Coreutils: `df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) and [`du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [Linux `unlink(2)`](https://man7.org/linux/man-pages/man2/unlink.2.html), [`mount(8)`](https://man7.org/linux/man-pages/man8/mount.8.html), and [`findmnt(8)`](https://man7.org/linux/man-pages/man8/findmnt.8.html)
- [systemd `os-release(5)`](https://www.freedesktop.org/software/systemd/man/latest/os-release.html) and [`journalctl(1)`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Debian APT](https://manpages.debian.org/unstable/apt/apt-get.8.en.html), [DNF 4](https://dnf.readthedocs.io/en/stable/command_ref.html), [DNF 5](https://dnf5.readthedocs.io/en/stable/), and [Arch `paccache`](https://man.archlinux.org/man/paccache.8)
- [Flatpak command reference](https://docs.flatpak.org/en/latest/flatpak-command-reference.html)
- [Docker contexts](https://docs.docker.com/engine/manage-resources/contexts/), [disk usage](https://docs.docker.com/reference/cli/docker/system/df/), and [pruning](https://docs.docker.com/engine/manage-resources/pruning/)
- [Podman global options](https://docs.podman.io/en/latest/markdown/podman.1.html), [disk usage](https://docs.podman.io/en/stable/markdown/podman-system-df.1.html), and [pruning](https://docs.podman.io/en/stable/markdown/podman-system-prune.1.html)
- [Snap revisions](https://snapcraft.io/docs/explanation/how-snaps-work/revisions/) and [snapshots](https://snapcraft.io/docs/how-to-guides/manage-snaps/create-data-snapshots/)
- Local `arch-wiki-docs 20260702-1`: System maintenance, Pacman, systemd journal, Docker, Podman, and Flatpak pages
