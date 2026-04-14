# mac-updater

A comprehensive Claude Code skill that audits Mac system health, package
updates, and security posture -- with detailed findings and actionable
fix commands.

## What It Does

When invoked via `/mac-updater`, it runs through three phases:

### Phase 1: Package Manager Updates
- Homebrew (with Xcode path + license pre-flight checks)
- npm global packages (with corruption detection)
- Python / pip (with conda environment awareness and CVE scanning)
- Mac App Store (via `mas`)
- macOS system updates (flags Rapid Security Responses as high priority)
- cargo, gem, go, rustup, composer

### Phase 2: Security Audit
- Core protections: SIP, FileVault, Firewall (with stealth mode), Gatekeeper, XProtect
- Passwordless sudo detection (critical)
- Firewall stealth mode and allow-list review
- FileVault recovery key validation
- Network security: listening ports, DNS on all interfaces, remote access services
- SSH keys + `~/.ssh/config` audit (ForwardAgent, StrictHostKeyChecking, ProxyCommand)
- Launch agents & daemons + modern Login Items (`sfltool dumpbtm`)
- Legacy persistence (crontab, at jobs)
- Browser extension permissions audit
- TCC privacy permissions

### Phase 3: System Health
- Disk & storage (APFS container-aware, not misleading `df -h`)
- Time Machine backups
- Memory, swap, pressure
- Kernel extensions

## Modes

```
/mac-updater              # Full audit
/mac-updater quick        # Fast summary, top 3 issues
/mac-updater update       # Package managers only
/mac-updater security     # Security audit only
```

## How to Install

1. Copy the entire contents of [`build-mac-updater-skill.md`](./build-mac-updater-skill.md)
2. Paste into Claude Code
3. Claude creates the skill at `~/.claude/skills/mac-updater/`

## Design Principles

- **Read-only first.** Never modifies anything without explicit approval.
- **Detect before using.** Skips checks for tools that aren't installed.
- **Fail gracefully.** One broken check doesn't block the rest.
- **Progress tracking.** Uses task checklists for real-time visibility during
  long audits.
- **Baseline-driven.** Compares findings against documented security baselines
  (informed by the [macOS Security Compliance Project](https://github.com/usnistgov/macos_security)).

## Warning

**Personal Macs only.** Do NOT run on work-managed or MDM-enrolled computers
without IT approval. This skill may flag, conflict with, or inadvertently
disrupt managed configurations, and running `brew upgrade` or modifying
firewall settings could violate your employer's security policies.

## What It Won't Do Without Asking

- Install or upgrade any package
- Enable or disable the firewall
- Modify any `defaults` writes
- Run `sudo` for anything
- Delete files or cleanup caches

It will always show you the exact command and ask before running anything
that changes state.
