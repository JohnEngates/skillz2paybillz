# Build a Mac Updater Skill for Claude Code

> **What this is:** Copy-paste this entire file into Claude Code. It will build a
> comprehensive `/mac-updater` skill that works on any Mac. The skill detects what's
> installed at runtime and adapts accordingly.

> **WARNING: Personal Macs only.** This skill is designed for Macs you own and
> administer yourself. Do NOT run this on a work-managed or MDM-enrolled computer
> without checking with your IT department first. Managed Macs often have security
> policies, restricted package managers, and monitoring agents that this skill may
> flag, conflict with, or inadvertently disrupt. Running `brew upgrade`, modifying
> firewall settings, or changing system defaults on a managed machine could violate
> your employer's security policies or trigger compliance alerts.

---

Build me a Claude Code skill called "mac-updater" that acts as an expert Mac system
administrator and security auditor. It should work on any Mac by detecting what's
installed at runtime, then checking for outdated packages, security vulnerabilities,
system health issues, and misconfigurations.

**IMPORTANT: Follow this specification exactly.** Create the files below with the
EXACT content specified. Do not summarize, abbreviate, paraphrase, or omit any
sections. Every command, table, gotcha, and operating principle listed here must
appear in the generated files verbatim. Do not add extra sections or features
beyond what is specified. The content below has been carefully reviewed for
security correctness -- do not "improve" it by changing commands, loosening
permission patterns, or rewriting security guidance.

Create these files:

```
~/.claude/skills/mac-updater/
├── SKILL.md
├── .claude/
│   └── settings.local.json
└── references/
    ├── security-baselines.md
    └── known-gotchas.md
```

---

## SKILL.md

### Frontmatter

```yaml
---
name: mac-updater
version: 1.0.0
description: |
  Expert Mac system administrator and security auditor. Checks for outdated
  packages, security vulnerabilities, system health issues, and misconfigurations.
  Invoke with /mac-updater or when the user asks to "update my mac", "check for
  updates", "security audit", "system health check", "what needs updating",
  "mac maintenance", "brew upgrade", "check my system", or discusses package
  updates, system hardening, or macOS security posture.
  Supports modes: full, update, security, quick.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - AskUserQuestion
---
```

### Core Principle

Read-only first. Gather all data before proposing any changes. Never execute
upgrades, installs, or destructive commands without showing the user exactly
what will happen and getting explicit confirmation.

### Mode Dispatch

Parse the argument after /mac-updater:

| Argument | Scope |
|----------|-------|
| (none) or `full` | All phases: updates + security + health |
| `update` | Package manager updates only (Phase 1) |
| `security` | Security audit only (Phase 2) |
| `quick` | Fast summary: outdated counts, security posture, disk space |

For `quick` mode: limit output to one summary table and the top 3 issues only.
Skip detailed command output. Aim for under 15 seconds.

### Progress Tracking

At the start of every run (except `quick` mode), create a task checklist using
TaskCreate so the user can see real-time progress. The checklist should reflect
the mode being run:

**For `full` mode, create these tasks:**

1. System identification
2. Package manager discovery
3. Homebrew
4. npm global
5. Python / pip
6. Mac App Store
7. macOS system updates
8. Other package managers
9. Core protections
10. Network security
11. SSH keys & config
12. Launch agents & persistence
13. Browser extensions
14. TCC permissions
15. Disk & storage
16. Time Machine
17. Memory & swap
18. Kernel extensions
19. Summary & recommendations

**For `update` mode:** tasks 1-8 + summary.
**For `security` mode:** tasks 1, 2, 9-14 + summary.

As each section starts, mark its task as `in_progress` (this shows a spinner
with the activeForm text). When it completes, mark it `completed`. If a section
is skipped (tool not installed, not applicable), mark it `completed` and note
"skipped" in the output.

Use short `activeForm` labels for the spinner, e.g.:
- "Checking Homebrew packages"
- "Auditing SSH keys"
- "Scanning browser extensions"

This gives the user a clear sense of progress during what can be a 2-3 minute
full audit. Do NOT create tasks for `quick` mode -- it should finish in under
15 seconds and the overhead isn't worth it.

---

### Pre-Flight: System Identification

Before any checks, capture and display the macOS version:

```bash
sw_vers 2>&1
uname -m 2>&1
```

Display: "Running on macOS [ProductName] [ProductVersion] ([BuildVersion]) on [arch]"

This matters because security checks, TCC behavior, and available tools differ
across macOS versions. If the version is older than macOS Ventura (13.0), warn
that some checks may behave differently.

---

### Phase 1: Package Manager Updates

#### Section 1: Runtime Discovery

Detect which package managers are installed:

```bash
command -v brew npm pip3 pip-audit conda mas cargo gem go rustup composer 2>/dev/null
```

Note: use `command -v` instead of `which` -- it is POSIX-compliant and behaves
consistently across bash and zsh. `which` can produce inconsistent results
depending on shell configuration.

Build a list of available managers. SKIP sections for managers that aren't
installed. Never fail on a missing tool. Report which managers were detected.

#### Section 2: Homebrew

Pre-flight gotcha checks (run both before any brew operations):

1. Verify Xcode developer path:
```bash
xcode-select -p 2>&1
```
If it points to a nonexistent directory (common with CCC/Time Machine backups
of old Xcode on external drives), STOP and warn. Suggest:
```bash
sudo xcode-select -s /Library/Developer/CommandLineTools
```

2. Check if Xcode license needs acceptance:
```bash
xcodebuild -license check 2>&1
```
If not accepted, warn that brew operations will fail until the user runs:
```bash
sudo xcodebuild -license accept
```

Do NOT proceed with brew operations until both checks pass.

Then:
```bash
brew update 2>&1
brew outdated 2>&1
brew outdated --cask 2>&1
brew outdated --cask --greedy 2>&1
brew list --pinned 2>&1
brew cleanup --dry-run 2>&1 | tail -1
```

Present as a table: Package | Current | Latest | Type.
Show greedy casks separately from regular casks (these auto-update themselves,
so Homebrew may show a version mismatch that the app has already resolved).
Note pinned formulae. Show reclaimable disk space.

When user approves: run `brew upgrade` for formulae and `brew upgrade --cask`
for casks separately. Verify with `brew outdated` after.

#### Section 3: npm Global Packages

Pre-flight: scan for corrupted entries:
```bash
ls -la "$(npm root -g)" 2>&1 | grep -E '^\.'
```
If dot-prefixed directories found (e.g., `.package-hEnxM1Mc`), warn that
`npm update -g` will fail. Recommend removing corrupted entries first.

Then: `npm outdated -g --depth=0 2>&1`

#### Section 4: Python (pip)

Detect environment first:
```bash
echo "Python: $(command -v python3)"
echo "Conda env: ${CONDA_DEFAULT_ENV:-none}"
pip3 --version 2>&1
```

If conda env is active, display prominent warning:
> WARNING: Active conda environment detected. pip upgrades can break
> conda-managed dependencies. Use `conda update` for conda-managed packages.

Security scan (if pip-audit installed): `pip-audit 2>&1`
Present CVEs as: Package | Version | CVE | Fix Version | Severity

If in a conda env, cross-reference CVE findings with `conda list` before
recommending pip-based fixes.

Outdated: `pip3 list --outdated 2>&1`

When upgrading: classify packages as conda-managed vs pip-managed first
(check `conda list --json` and compare channels). Use `conda update` for
conda packages, `pip install --upgrade` for pip packages. Flag major version
bumps separately for user review.

#### Section 5: Mac App Store

If mas installed: `mas outdated 2>&1`
If not: note "Install mas via `brew install mas` for App Store management."

#### Section 6: macOS System Updates

```bash
softwareupdate -l 2>&1
```

Parse and highlight:
- macOS version updates
- Security updates and Rapid Security Responses (RSR) -- flag RSR as HIGH
  PRIORITY since they patch actively exploited vulnerabilities
- Command Line Tools updates
- Anything marked "Recommended: YES"

Also check if automatic security updates are enabled:
```bash
defaults read /Library/Preferences/com.apple.SoftwareUpdate AutomaticallyInstallMacOSUpdates 2>/dev/null
defaults read /Library/Preferences/com.apple.SoftwareUpdate CriticalUpdateInstall 2>/dev/null
```
If auto-install for security updates is off, flag as Important and recommend enabling.

Note that macOS updates require restart. Do not attempt to install -- just report.

#### Section 7: Other Package Managers

For each detected: run a lightweight read-only status check.
- cargo: `cargo install --list 2>&1 | head -20`
- gem: `gem outdated 2>&1 | head -20`
- go: `go version 2>&1`
- rustup: `rustup check 2>&1`
- composer: `composer global outdated 2>&1`

Report only. Do not auto-update.

---

### Phase 2: Security Audit

#### Section 8: Core Protections

Run in parallel:
```bash
csrutil status 2>&1                                                    # SIP
fdesetup status 2>&1                                                   # FileVault
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate 2>&1  # Firewall
spctl --status 2>&1                                                    # Gatekeeper
```

**Firewall depth check** (if firewall is enabled):
```bash
/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode 2>&1
/usr/libexec/ApplicationFirewall/socketfilterfw --getblockall 2>&1
/usr/libexec/ApplicationFirewall/socketfilterfw --listapps 2>&1
```
Stealth mode should be enabled (prevents responses to probing requests like
ping and port scans). Report which apps are allowed through the firewall.
Block-all-incoming is the most secure but most users won't tolerate it -- just
note its status.

**FileVault recovery key validation:**
```bash
fdesetup validaterecovery 2>&1
```
This requires sudo. Explain: "This verifies your FileVault recovery key is
still valid. Without a working recovery key, a forgotten password means
permanent data loss." Only run if user approves. If they decline, note it as
unchecked rather than failing.

**Passwordless sudo check:**
```bash
sudo -n true 2>&1
```
If this succeeds (exit code 0), flag as **Critical**: the user has NOPASSWD
sudo configured, meaning any process running as their user is effectively root.
This is a major privilege escalation risk.

**XProtect version and freshness:**
```bash
defaults read /Library/Apple/System/Library/CoreServices/XProtect.bundle/Contents/Info.plist CFBundleShortVersionString 2>/dev/null
system_profiler SPInstallHistoryDataType 2>/dev/null | grep -A 5 "XProtect" | tail -6
```

Compare against references/security-baselines.md. Present as:
Protection | Status | Expected | Assessment

#### Section 9: Network Security

Listening ports:
```bash
lsof -iTCP -sTCP:LISTEN -P -n 2>&1
```
Present as: Process | PID | Port | Binding. Flag non-standard listeners
(anything not mDNSResponder, rapportd, ControlCenter, or other known macOS
system services).

**DNS** -- check all active network interfaces, not just Wi-Fi:
```bash
networksetup -listallnetworkservices 2>&1
```
For each active service (skip lines starting with "An asterisk"):
```bash
networksetup -getdnsservers "<service name>" 2>&1
```
Note non-standard DNS. Well-known resolvers (1.1.1.1, 8.8.8.8, 9.9.9.9) are fine.
Note: DNS may also be configured per-network in advanced settings or overridden
by DNS-over-HTTPS at the browser level, which bypasses system DNS entirely.

**Remote access services:**
```bash
systemsetup -getremotelogin 2>&1
defaults read /Library/Preferences/com.apple.RemoteManagement.plist 2>/dev/null
```
Also check Screen Sharing:
```bash
defaults read /Library/Preferences/com.apple.screensharing 2>/dev/null
```
Flag any enabled remote access service as **Important**. These are legitimate
features but should be intentionally enabled, not left on by accident. Remote
Login means SSH is open. Remote Management is Apple Remote Desktop (ARD).

#### Section 10: SSH Keys

```bash
ls -la ~/.ssh/ 2>&1
for f in ~/.ssh/id_*; do [ -f "$f" ] && ssh-keygen -l -f "$f" 2>&1; done
ssh-add -l 2>&1
```

**Directory permissions:** The `~/.ssh/` directory itself must be 700. If it's
755 or world-readable, every key inside is effectively exposed regardless of
its own permissions. SSH will refuse to use keys in a too-permissive directory,
but the audit should flag this explicitly.

**Key file permissions:** private keys must be 600, public 644. Flag DSA as
deprecated. Flag RSA < 3072 bits as weak. Ed25519 and ECDSA are preferred.
If authorized_keys exists, list contents and note unexpected entries.

**SSH config audit** (if `~/.ssh/config` exists):
```bash
grep -iE "ForwardAgent|StrictHostKeyChecking|ProxyCommand" ~/.ssh/config 2>&1
```
Flag these as security-relevant:
- `ForwardAgent yes` -- **Important**: enables SSH agent forwarding, a known
  lateral movement vector. An attacker on a remote host can use your forwarded
  agent to authenticate to other systems as you.
- `StrictHostKeyChecking no` -- **Important**: disables MITM protection. SSH
  will silently accept changed host keys.
- `ProxyCommand` -- **Review**: executes an arbitrary command on every connect.
  Not inherently bad (common for jump hosts) but should be reviewed for
  unexpected entries.

#### Section 11: Launch Agents & Daemons

```bash
ls -la ~/Library/LaunchAgents/ 2>&1
ls -la /Library/LaunchAgents/ 2>&1
ls -la /Library/LaunchDaemons/ 2>&1
```

List all non-Apple entries. Flag unknown publishers or suspicious names.
Cross-reference with references/known-gotchas.md.

**Login Items & Background Items** (modern macOS persistence):
```bash
sfltool dumpbtm 2>&1
```
This shows background items registered via the SMAppService API -- these don't
appear in LaunchAgents directories. May require admin privileges; fail
gracefully if unavailable or on older macOS (pre-Ventura).

**Crontab & at jobs** (legacy persistence):
```bash
crontab -l 2>&1
ls -la /var/at/jobs/ 2>&1
```
These are legacy persistence mechanisms that are often forgotten. Malware still
uses them because administrators overlook them. Flag any entries for review.

#### Section 12: TCC / Privacy Permissions

```bash
sqlite3 "$HOME/Library/Application Support/com.apple.TCC/TCC.db" \
  "SELECT client, service, auth_value FROM access WHERE auth_value = 2;" 2>&1
```

If this fails (common -- requires Full Disk Access for the terminal), note:
> TCC database is not readable from this terminal. To review privacy
> permissions, check System Settings > Privacy & Security. Grant your terminal
> Full Disk Access if you want programmatic TCC auditing.

On newer macOS versions, even Full Disk Access may not be sufficient for
some TCC queries. Fail gracefully and move on.

#### Section 13: Browser Extensions

Browser extensions are the #1 actual attack vector on consumer Macs. A
malicious or overly-permissive extension can read every page, exfiltrate
cookies, inject scripts, and access passwords.

**Chrome extensions:**
```bash
ls ~/Library/Application\ Support/Google/Chrome/Default/Extensions/ 2>&1
```
For each extension directory, read its `manifest.json` and flag:
- Extensions with `<all_urls>` or `*://*/*` in permissions (can read every page)
- Extensions with `cookies`, `webRequest`, or `tabs` permissions
- Extensions not from the Chrome Web Store (sideloaded)

**Other browsers** (best-effort, check if paths exist):
- Brave: `~/Library/Application Support/BraveSoftware/Brave-Browser/Default/Extensions/`
- Edge: `~/Library/Application Support/Microsoft Edge/Default/Extensions/`
- Firefox: `~/Library/Application Support/Firefox/Profiles/*/extensions.json`

Present as: Extension Name | Permissions | Assessment.
This is a best-effort audit -- extension paths vary by browser version and
profile. Note any extensions that couldn't be identified by name (the
extension directory name is a Chrome Web Store ID).

---

### Phase 3: System Health

#### Section 14: Disk & Storage

**Important:** Do NOT rely on `df -h /` for disk space. On APFS (all modern
Macs), the container shares space across multiple volumes (Data, VM, Preboot,
Recovery). `df -h` only shows the root volume's own usage (~12 GB for the OS),
which is wildly misleading.

Instead, use `diskutil` to get container-level numbers that match Finder:
```bash
diskutil info / 2>&1 | grep -E "Volume Total|Volume Available|Volume Used|Container Total|Container Free"
diskutil info disk0 2>&1 | grep -E "SMART Status|Media Name|Disk Size"
```

Report these values:
- **Container Total Space** -- this is the drive capacity
- **Container Free Space** -- actual free space available
- **Purgeable note:** Finder may show more "available" space than Container Free
  because it includes purgeable space (caches, Time Machine local snapshots)
  that macOS can reclaim on demand. Note this discrepancy if relevant.

Flag if free space < 10% of container total. SMART should be "Verified".

#### Section 15: Time Machine

```bash
tmutil status 2>&1
tmutil latestbackup 2>&1
```
Flag if last backup > 24 hours. Note if not configured.

#### Section 16: Memory & Swap

```bash
vm_stat 2>&1
sysctl vm.swapusage 2>&1
memory_pressure 2>&1
```
Report memory pressure level. Flag if swap > 2 GB.

#### Section 17: Kernel Extensions

```bash
kmutil showloaded --list-only 2>/dev/null | grep -v com.apple
```
Flag non-Apple kexts. Note that kernel extensions are deprecated in modern
macOS in favor of System Extensions, and they can cause stability issues.
If any are found, identify the software they belong to.

---

### Phase 4: Summary & Recommendations

**Summary Table** -- always present at the end, covering every section:

```
| Category | Status | Details |
|----------|--------|---------|
```

Status key: OK = good, ! = needs attention, !!! = action required, -- = not checked

For sections that were skipped (wrong mode, tool not installed), show --.

**Recommendations** sorted by severity:

**Critical** (disabled protections, active CVEs, RSR available):
- Exact fix command
- Risk of not fixing

**Important** (outdated security packages, stale backups, auto-updates disabled):
- Command and what it changes
- Dependency risks

**Routine** (version bumps, cleanup, optional):
- Command and benefit (disk savings, etc.)

End with: "Which of these would you like me to handle?"

**Export option:** After presenting recommendations, also offer:
"Would you like me to save this report to ~/mac-audit-YYYY-MM-DD.md?"

Note: Do NOT default to ~/Desktop or ~/Documents -- if iCloud Drive sync is
enabled, those directories are synced to Apple's servers, and this report
contains security-sensitive information (open ports, SSH key fingerprints,
installed software versions, security posture). Use the home directory or
ask the user where they'd like it saved.

---

### Operating Principles

1. **Read-only first.** Never modify without explicit user approval.
2. **Detect before using.** Always check if a tool exists before calling it.
3. **Fail gracefully.** One failure must not block the rest of the audit.
4. **Explain sudo.** Say why elevated privileges are needed before asking.
5. **Batch for speed.** Run independent checks in parallel.
6. **Show your work.** Include command output so the user can verify.
7. **Check known gotchas.** Reference the gotchas file before risky operations.
8. **Confirm before destructive actions.** Cleanup, removal, and dependency-conflicting upgrades need approval.

---

### Example Invocations

```
/mac-updater              # Full audit: updates + security + health
/mac-updater quick        # Fast summary: counts, posture, disk
/mac-updater update       # Package managers only
/mac-updater security     # Security audit only
```

The skill also activates when the user says things like:
- "update my mac"
- "check for updates"
- "run a security audit"
- "what needs updating?"
- "is my mac healthy?"
- "brew upgrade"

---

## references/security-baselines.md

```markdown
# Security Baselines

Expected-good values for macOS security checks. These align with general best
practices and are informed by the macOS Security Compliance Project (mSCP) from
NIST (https://github.com/usnistgov/macos_security). For enterprise or compliance
contexts, refer to the full mSCP profiles for your macOS version.

Last updated: [skill will use creation date]

| Check | Expected Value | Severity if Not Met |
|-------|---------------|---------------------|
| SIP (System Integrity Protection) | Enabled | Critical |
| FileVault | On (Enabled) | Critical |
| Firewall | Enabled | Important |
| Gatekeeper | assessments enabled | Important |
| SSH private key permissions | 600 | Important |
| SSH public key permissions | 644 | Routine |
| SSH key type | Ed25519 or ECDSA (or RSA >= 3072) | Important |
| DSA keys present | None | Important (deprecated) |
| XProtect definitions age | < 14 days | Routine |
| Auto-install security updates | Enabled | Important |
| Time Machine last backup | < 24 hours | Important |
| Disk free space | > 10% of total | Important |
| Swap usage | < 2 GB | Routine |
| Non-Apple kernel extensions | None (or known/vetted) | Routine |
| Xcode developer path | Valid local path | Important |
| Xcode license | Accepted | Important |
| ~/.ssh directory permissions | 700 | Important |
| ~/.ssh/config ForwardAgent | no (or absent) | Important |
| ~/.ssh/config StrictHostKeyChecking | yes or ask (not no) | Important |
| Firewall stealth mode | Enabled | Routine |
| Passwordless sudo (NOPASSWD) | Disabled | Critical |
| Remote Login (SSH) | Off (unless intentional) | Important |
| Screen Sharing / ARD | Off (unless intentional) | Important |
```

---

## references/known-gotchas.md

```markdown
# Known Gotchas

Common macOS issues the skill should proactively check for. Add machine-specific
gotchas to this file as they are discovered.

## Stale Xcode Developer Path

**Symptom:** `brew upgrade` fails with "Your Xcode is too outdated" pointing to a
nonexistent path.
**Cause:** `xcode-select -p` points to an old Xcode.app on a backup drive (CCC,
Time Machine, external RAID volume).
**Fix:** `sudo xcode-select -s /Library/Developer/CommandLineTools`
**Check:** Verify `xcode-select -p` returns a valid, accessible local path before
brew operations.

## Xcode License Not Accepted

**Symptom:** Brew or build tools fail with "Agreeing to the Xcode/iOS license
requires admin privileges".
**Cause:** macOS or Xcode was updated but the license was not re-accepted.
**Fix:** `sudo xcodebuild -license accept`
**Check:** Run `xcodebuild -license check` before brew operations. If it returns
non-zero, the license needs acceptance.

## Corrupted npm Global Entries

**Symptom:** `npm update -g` fails with EINVALIDPACKAGENAME referencing a
dot-prefixed directory.
**Cause:** Broken or leftover directories in `$(npm root -g)` with names like
`.package-hEnxM1Mc`.
**Fix:** Remove the corrupted entry, then update individually:
`rm -rf "$(npm root -g)/.corrupted-name" && npm install -g package@latest`
**Check:** Scan for dot-prefixed entries in `$(npm root -g)` before running
`npm update -g`.

## Conda / pip Dependency Conflicts

**Symptom:** pip upgrade succeeds but prints dependency conflict warnings.
**Cause:** pip and conda have separate dependency solvers. Upgrading via pip can
violate conda's pinned versions.
**Fix:** Use `conda update packagename` for conda-managed packages.
**Check:** Detect active conda env via `$CONDA_DEFAULT_ENV` or `which python3`.
Cross-reference with `conda list`.

## Auto-Updating (Greedy) Casks

**Symptom:** Apps like Chrome, VS Code, Slack don't show in `brew outdated --cask`
but have newer versions available.
**Cause:** These apps auto-update themselves. Homebrew skips them unless `--greedy`
is used.
**Fix:** `brew upgrade --cask --greedy <cask-name>`
**Check:** Always show `--greedy` results separately and let the user decide.

## pip-audit in Conda Environments

**Symptom:** pip-audit suggests fix versions that break the conda environment.
**Cause:** pip-audit doesn't know about conda's version constraints.
**Fix:** Check `conda list | grep packagename` before applying pip-audit fixes.
**Check:** Cross-reference CVE findings with conda package list.

## Automatic Security Updates Disabled

**Symptom:** Rapid Security Responses (RSR) and critical patches not being applied.
**Cause:** Auto-install was disabled in System Settings or via MDM profile.
**Fix:** System Settings > General > Software Update > Automatic Updates > enable
"Install Security Responses and system files".
**Check:** `defaults read /Library/Preferences/com.apple.SoftwareUpdate CriticalUpdateInstall`
should return 1.

## TCC Database Access on Newer macOS

**Symptom:** sqlite3 query of TCC.db fails even with Full Disk Access.
**Cause:** Newer macOS versions add additional protections around TCC.db.
**Fix:** No programmatic fix. Review permissions in System Settings > Privacy & Security.
**Check:** Attempt the query, catch the error, and suggest manual review instead of
failing the entire audit.

## sfltool Not Available on Older macOS

**Symptom:** `sfltool dumpbtm` returns "command not found".
**Cause:** `sfltool` and the Background Task Management system were introduced in
macOS Ventura (13.0). Older versions don't have it.
**Fix:** On older macOS, rely on Launch Agents/Daemons checks only.
**Check:** Verify macOS version >= 13.0 before running `sfltool`. Fall back gracefully.

## Browser Extension Paths Vary

**Symptom:** Extension audit returns empty or wrong results.
**Cause:** Browser extension directories vary by browser version, profile name,
and whether the browser uses a custom profile directory. Multi-profile setups
have extensions in `Profile 1/`, `Profile 2/`, etc. instead of `Default/`.
**Fix:** Glob for `*/Extensions/` under the browser's Application Support directory.
**Check:** Use `ls` to discover available profile directories before scanning.
```

---

## .claude/settings.local.json

Pre-approve read-only commands to reduce permission prompts during audits:

```json
{
  "permissions": {
    "allow": [
      "Bash(brew update:*)",
      "Bash(brew outdated:*)",
      "Bash(brew list:*)",
      "Bash(brew cleanup --dry-run:*)",
      "Bash(pip3 list:*)",
      "Bash(pip3 check:*)",
      "Bash(pip-audit:*)",
      "Bash(npm outdated:*)",
      "Bash(softwareupdate -l:*)",
      "Bash(mas outdated:*)",
      "Bash(conda list:*)",
      "Bash(csrutil status:*)",
      "Bash(fdesetup status:*)",
      "Bash(spctl --status:*)",
      "Bash(lsof -iTCP -sTCP:LISTEN -P -n:*)",
      "Bash(diskutil info:*)",
      "Bash(tmutil status:*)",
      "Bash(tmutil latestbackup:*)",
      "Bash(vm_stat:*)",
      "Bash(sysctl vm.swapusage:*)",
      "Bash(memory_pressure:*)",
      "Bash(sw_vers:*)",
      "Bash(uname:*)",
      "Bash(xcode-select:*)",
      "Bash(xcodebuild -license check:*)",
      "Bash(defaults read /Library/Preferences/com.apple.SoftwareUpdate:*)",
      "Bash(defaults read /Library/Apple/System/Library/CoreServices/XProtect.bundle:*)",
      "Bash(defaults read /Library/Preferences/com.apple.RemoteManagement.plist:*)",
      "Bash(defaults read /Library/Preferences/com.apple.screensharing:*)",
      "Bash(kmutil showloaded:*)",
      "Bash(networksetup -listallnetworkservices:*)",
      "Bash(networksetup -getdnsservers:*)",
      "Bash(systemsetup -getremotelogin:*)",
      "Bash(crontab -l:*)",
      "Bash(sfltool dumpbtm:*)",
      "Bash(sudo -n true:*)",
      "Bash(/usr/libexec/ApplicationFirewall/socketfilterfw:*)"
    ]
  }
}
```

Note: Destructive commands (brew upgrade, pip install --upgrade, brew cleanup,
rm, sudo) are intentionally NOT pre-approved. The skill will always ask before
running those.

---

## After creating all files:

1. Show me the file tree you created
2. Run `/mac-updater quick` to verify it works on my machine
3. Ask if I want to customize anything based on the results
