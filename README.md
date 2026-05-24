<div align="center">

# 📝 win_replace

**Ansible module that performs regex-based text replacement in Windows files — handling CRLF line endings and .NET encodings that `ansible.builtin.replace` gets wrong.**

[![Ansible](https://img.shields.io/badge/Ansible-Collection-EE0000?style=for-the-badge&logo=ansible&logoColor=white)](https://docs.ansible.com/ansible/latest/collections_guide/index.html)
[![PowerShell](https://img.shields.io/badge/PowerShell-Windows-5C2D91?style=for-the-badge&logo=powershell&logoColor=white)](https://learn.microsoft.com/en-us/powershell/)
[![Python](https://img.shields.io/badge/Python-Module_Stub-3776AB?style=for-the-badge&logo=python&logoColor=white)](plugins/modules/win_replace.py)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

</div>

---

## What is this?

`ansible.builtin.replace` is built for Unix and breaks on Windows — it mishandles CRLF line endings and does not support .NET encodings, so regex anchors misbehave and files can be silently corrupted. `win_replace` is a drop-in Windows-native equivalent: it reads the file with a configurable .NET encoding, normalises line endings to LF so the regex engine sees clean lines, applies a full multiline regex replacement, then writes the result back with CRLF restored.

It runs entirely on the Windows target host via PowerShell, integrates with Ansible's check mode and backup utilities, and is idempotent — it only reports `changed` when the file content actually differs.

---

## ✨ Features

| | |
|---|---|
| 🪟 **Windows-native** | Runs as PowerShell on the target host — no POSIX assumptions, no cross-OS surprises |
| 🔄 **Regex replacement** | Full .NET `[regex]::Replace` with `Multiline` flag — `^` and `$` match per line |
| 📄 **CRLF-safe** | Normalises to LF for matching, restores CRLF on write — files stay Windows-formatted |
| 🔤 **Encoding support** | Any .NET encoding name: `utf8`, `utf-16`, `windows-1252`, and more |
| 💾 **Backup** | Optional timestamped backup before any write via `Ansible.ModuleUtils.Backup` |
| ✅ **Check mode** | Dry-run safe — reports what would change without writing anything |
| ♻️ **Idempotent** | Returns `changed: false` when content already matches — safe to run repeatedly |

---

## 🏗️ How it works

```
Ansible controller
       │
       │  (WinRM / SSH)
       ▼
Windows target host — win_replace.ps1
       │
       ├── 1. Resolve path, fail if file not found
       │
       ├── 2. Read file with specified .NET encoding
       │
       ├── 3. Normalise CRLF ──► LF
       │
       ├── 4. [regex]::Replace(content, regexp, replace, Multiline)
       │
       ├── 5. Content changed?
       │          │                         │
       │         yes                        no
       │          │                         │
       │          ├── backup: true?         └──► msg: "No changes"
       │          │     └── Backup-File         changed: false
       │          │
       │          ├── Restore LF ──► CRLF
       │          └── Write file (skipped in check mode)
       │
       ▼
    JSON result ──► Ansible (path, changed, msg, backup_file)
```

---

## 🚀 Quick Start

### 1. Clone and install the collection

```bash
git clone https://github.com/mishgoldenberg/ansible-win_replace.git
ansible-galaxy collection install ./ansible-win_replace
```

### 2. Reference the module in a playbook

```yaml
- name: Replace a value in a config file
  mishgoldenberg.windows.win_replace:
    path: C:\App\config.ini
    regexp: 'server=old-host'
    replace: 'server=new-host'
    backup: true
```

### 3. Run

```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## ⚙️ Configuration Reference

### Module parameters

| Parameter | Required | Default | Description |
|---|:---:|---|---|
| `path` | ✅ | | Absolute path to the target file. Aliases: `dest`, `destfile`, `name` |
| `regexp` | ✅ | | .NET regular expression to match. `Multiline` mode is always enabled |
| `replace` | | `""` | Replacement string. Supports .NET backreferences (`$1`, `${name}`) |
| `backup` | | `false` | Create a timestamped backup before writing |
| `encoding` | | `utf8` | .NET encoding name used to read and write the file |

### Return values

| Key | Type | Returned | Description |
|---|---|---|---|
| `path` | str | always | Path of the file targeted |
| `changed` | bool | always | Whether the file was modified |
| `msg` | str | always | `"Replaced matching content"` or `"No changes"` |
| `backup_file` | str | when `backup: true` | Full path to the backup file created |

---

## 🗂️ Project Structure

```
ansible-win_replace/
├── plugins/
│   └── modules/
│       ├── win_replace.py    # Ansible module stub: DOCUMENTATION, EXAMPLES, RETURN
│       └── win_replace.ps1   # PowerShell implementation executed on the Windows host
├── galaxy.yml                # Ansible Galaxy collection metadata
├── .gitignore
├── LICENSE
└── README.md
```

---

## 🔧 Customisation

- **Use a non-UTF-8 encoding** — pass `encoding: windows-1252` (or any name accepted by `[System.Text.Encoding]::GetEncoding`) in the task when targeting legacy ANSI files.

- **Match across multiple lines** — `Multiline` is always active so `^`/`$` are per-line anchors; use `[\s\S]` for patterns that must span line boundaries.

- **Dry-run before applying** — run the playbook with `--check`; the module reads `_ansible_check_mode` and skips the write while still reporting `changed: true` if content would differ.

- **Delete lines** — leave `replace` unset (defaults to `""`) and the matched text is removed; combine with `regexp: '^pattern\r?$'` to drop entire lines cleanly.

---

## 📄 License

[MIT](LICENSE) © 2026 Michael Goldenberg
