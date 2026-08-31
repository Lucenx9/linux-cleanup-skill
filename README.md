# Linux cleanup skill

A small, portable Agent Skill for auditing and safely reclaiming disk space on Linux. It is designed for Claude Code, Codex, Cursor, and other clients that support the [Agent Skills specification](https://agentskills.io/specification).

The skill is intentionally instruction-only: there is no universal cleanup script and no direct deletion of managed storage. Its operating contract is:

> audit → propose → approve → execute one → verify

Broad requests stay non-mutating until the user selects an exact operation. Personal files, configuration, backups, snapshots, application data, stopped-container writable layers, and container volumes are preserved by default.

## What it covers

- filesystem space and inode diagnosis, including snapshot-aware Btrfs accounting
- APT, DNF 4, DNF 5, and pacman package cleanup
- systemd journal and `logrotate`-managed log retention
- Snap revision and data-snapshot inventory; removal remains high-risk
- Flatpak unused runtimes and extensions, in an explicit installation
- Docker and Podman objects, one category and local daemon/store at a time
- application caches, only through an identified owner's documented cleaner

Unsupported or ambiguous tools fall back to audit and explanation; the agent must not guess a cleanup command.

## Install

Clone the repository:

```sh
git clone https://github.com/Lucenx9/linux-cleanup-skill.git
```

For Codex or Cursor, copy the skill into the user skill directory:

```sh
mkdir -p ~/.agents/skills
cp -R linux-cleanup-skill/skills/linux-cleanup ~/.agents/skills/
```

For Claude Code:

```sh
mkdir -p ~/.claude/skills
cp -R linux-cleanup-skill/skills/linux-cleanup ~/.claude/skills/
```

You can instead copy `skills/linux-cleanup` into a project's `.agents/skills/` or `.claude/skills/` directory. Cursor also supports `.cursor/skills/`.

## Example prompts

```text
My root filesystem is almost full. Audit it and show me safe cleanup options.
```

```text
Inspect Docker disk use, but do not prune anything yet.
```

```text
Run only the approved APT cache cleanup, then show the before/after disk space.
```

## Design basis

The format and authoring choices follow primary documentation:

- [Agent Skills specification](https://agentskills.io/specification)
- [OpenAI: Build skills](https://developers.openai.com/codex/skills)
- [Anthropic: Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Claude Code skills](https://code.claude.com/docs/en/skills)
- [Cursor Agent Skills](https://cursor.com/docs/skills)

The workflow adapts the strongest ideas from pstack—authoritative audit, boundary discipline, idempotent retries, small verifiable units, and evidence after mutation—without copying its worktree-specific deletion assumptions:

- [pstack skill authoring playbook](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/playbooks/authoring-a-skill.md)
- [pstack worktree cleanup playbook](https://github.com/cursor/plugins/blob/main/pstack/skills/poteto-mode/playbooks/worktree-cleanup.md)
- [pstack skills](https://github.com/cursor/plugins/tree/main/pstack/skills)

Cleanup semantics are grounded in upstream manuals for [GNU `df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html), [GNU `du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html), [Btrfs](https://btrfs.readthedocs.io/en/latest/btrfs-filesystem.html), [APT](https://manpages.debian.org/unstable/apt/apt-get.8.en.html), [DNF 4](https://dnf.readthedocs.io/en/stable/command_ref.html), [DNF 5](https://dnf5.readthedocs.io/en/stable/), [paccache](https://man.archlinux.org/man/paccache.8), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html), [`logrotate`](https://man7.org/linux/man-pages/man5/logrotate.conf.5.html), [Snap](https://snapcraft.io/docs/revisions/), [Flatpak](https://docs.flatpak.org/en/latest/flatpak-command-reference.html#flatpak-uninstall), [Docker](https://docs.docker.com/engine/manage-resources/pruning/), and [Podman](https://docs.podman.io/en/stable/markdown/podman-system-prune.1.html).

Distribution guidance was checked against a local, versioned copy of [ArchWiki](https://wiki.archlinux.org/) and current Debian, Ubuntu, Fedora, Gentoo, and openSUSE documentation. When a wiki passage conflicts with a current upstream command reference, the upstream project documentation and the installed command's help take precedence. See the [upstream audit](research/linux-upstream-audit-2026-08-21.md), the [distribution-wiki follow-up](research/distro-wiki-audit-2026-08-21.md), and the [current-practice delta audit](research/linux-cleanup-audit-2026-08-31.md).

## License

MIT
