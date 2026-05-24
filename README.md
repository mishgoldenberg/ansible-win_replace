<div align="center">

# ansible-win_replace

**Regex-based text replacement for Windows files — an Ansible module built for Windows.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Language: PowerShell](https://img.shields.io/badge/language-PowerShell-blue.svg)](plugins/modules/win_replace.ps1)
[![Platform: Windows](https://img.shields.io/badge/platform-Windows-0078D6.svg)](https://docs.ansible.com/ansible/latest/os_guide/windows_usage.html)

</div>

---

## What is this?

Ansible ships with `ansible.builtin.replace` for Unix-style text replacement, but it does not work reliably on Windows because of CRLF line endings and encoding differences. `win_replace` is a drop-in Windows-native equivalent: it reads the file with a configurable .NET encoding, temporarily normalises line endings for regex matching, then writes the result back with the original CRLF endings intact.

It is packaged as an Ansible Collection module and integrates transparently with Ansible's check mode, diff mode, and backup utilities.

---

## Features

| Feature | Details |
|---|---|
| Regex replacement | Full .NET `[regex]::Replace` with `Multiline` flag |
| CRLF-safe | Normalises to LF for matching, restores CRLF on write |
| Encoding support | Any .NET encoding (`utf8`, `utf-16`, `windows-1252`, …) |
| Backup | Optional timestamped backup via `Ansible.ModuleUtils.Backup` |
| Check mode | Dry-run support — reports what would change without writing |
| Idempotent | Reports `changed: false` when content is already correct |

---

## How it works

```
Ansible controller
       |
       |  (WinRM / SSH)
       v
Windows target host
       |
       +-- win_replace.ps1 (executed by Ansible)
              |
              1. Read file with specified encoding
              |
              2. Normalise CRLF → LF
              |
              3. [regex]::Replace(content, regexp, replace, Multiline)
              |
              4. Restore LF → CRLF
              |
              5. Write back (unless check_mode or no change)
              |
              6. Return JSON result to Ansible
```

---

## Quick Start

### 1. Clone / install the collection

```bash
git clone https://github.com/mishgoldenberg/ansible-win_replace.git
```

Or install directly from the repository in your `requirements.yml`:

```yaml
collections:
  - source: https://github.com/mishgoldenberg/ansible-win_replace.git
    type: git
    version: main
```

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Reference the module in a playbook

```yaml
- name: Replace a string in a config file
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

## Configuration

The module accepts the following parameters (all passed inline in the task):

| Parameter | Required | Default | Description |
|---|---|---|---|
| `path` | yes | — | Absolute path to the target file on the Windows host. Aliases: `dest`, `destfile`, `name`. |
| `regexp` | yes | — | .NET regular expression to match. `Multiline` mode is always enabled. |
| `replace` | no | `""` | Replacement string. Supports .NET backreferences (`$1`, `${name}`). |
| `backup` | no | `false` | If `true`, creates a timestamped backup of the file before writing. |
| `encoding` | no | `utf8` | .NET encoding name used to read and write the file (e.g. `utf-16`, `windows-1252`). |

### Return values

| Key | Type | Returned | Description |
|---|---|---|---|
| `path` | str | always | Path of the file targeted. |
| `changed` | bool | always | Whether the file was modified. |
| `msg` | str | always | Human-readable result (`"Replaced matching content"` or `"No changes"`). |
| `backup_file` | str | when `backup: true` | Full path to the backup file created. |

---

## Project structure

```
ansible-win_replace/
├── plugins/
│   └── modules/
│       ├── win_replace.py   # Ansible module stub: DOCUMENTATION, EXAMPLES, RETURN
│       └── win_replace.ps1  # PowerShell implementation executed on the Windows host
├── galaxy.yml               # Ansible Galaxy collection metadata
├── .gitignore
├── LICENSE
└── README.md
```

---

## Customisation

**Custom encoding** — if your files are not UTF-8, pass `encoding: windows-1252` (or any encoding name accepted by `[System.Text.Encoding]::GetEncoding`):

```yaml
- name: Replace in a legacy ANSI config
  mishgoldenberg.windows.win_replace:
    path: C:\Legacy\app.cfg
    regexp: 'OldValue'
    replace: 'NewValue'
    encoding: windows-1252
```

**Multiline patterns** — the `Multiline` regex flag is always active, so `^` and `$` match the start/end of each line:

```yaml
- name: Remove a full line matching a pattern
  mishgoldenberg.windows.win_replace:
    path: C:\App\config.ini
    regexp: '^debug=true\r?$'
    replace: ''
```

**Check mode** — run with `--check` to see whether the file would change without writing anything:

```bash
ansible-playbook playbook.yml --check
```

---

## License

[MIT](LICENSE) — Copyright (c) 2026 Mish Goldenberg
