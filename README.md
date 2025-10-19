# Ansible Collection – your_namespace.windows

This collection provides custom Windows modules for Ansible.

## Modules

### `win_replace`
Performs regex-based replacements on Windows files, similar to `ansible.builtin.replace` but optimized for Windows environments.

### Example
```yaml
- name: Replace string in a file
  your_namespace.windows.win_replace:
    path: C:\temp\test.txt
    regexp: 'foo'
    replace: 'bar'
