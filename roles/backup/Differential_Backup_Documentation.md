# Differential Backup Documentation

## Overview

The differential backup feature in the `network.backup.backup` role allows you to create backups **only when actual configuration changes are detected**, ignoring metadata and timestamp differences. This reduces storage overhead and SCM noise by preventing unnecessary commits and pull requests when only timestamps or metadata have changed.

## How It Works

### 1. Backup Type Selection

The differential backup is controlled by the `type` parameter:

```yaml
- name: Create Network Backup
  ansible.builtin.include_role:
    name: network.backup.backup
  vars:
    type: "diff"  # Options: "full", "incremental", or "diff"
    data_store:
      scm:
        origin:
          # ... SCM configuration
```

- **`type: "diff"`**: Enables differential backup - only publishes if config actually changed
- **`type: "full"`** (default): Always creates and publishes backup, regardless of changes

### 2. Backup Process Flow

When `type: "diff"` is set, the following process occurs:

```
1. Retrieve previous backup from SCM (if exists)
   ↓
2. Create current backup from device
   ↓
3. Normalize both backups (remove timestamps/metadata)
   ↓
4. Compare normalized content
   ↓
5. If different → backup_has_changes = true → Publish
   If identical → backup_has_changes = false → Skip publish
```

## Normalization Process

The normalization process removes metadata and timestamp lines that don't represent actual configuration changes. This ensures that only meaningful configuration differences trigger a new backup.

### What Gets Ignored (Removed During Normalization)

The following lines are **removed** from backup files before comparison:

#### Cisco IOS/IOS-XE
- `!Command: show running-config`
- `!Running configuration last done at: <timestamp>`
- `!Time: <timestamp>`
- `!NVRAM config last updated at: <timestamp>`
- `!No configuration change since last restart`

#### Cisco NX-OS
- `!Command: show running-config`
- `!Time: <timestamp>`
- `!Running configuration last done at: <timestamp>`
- `!NVRAM config last updated at: <timestamp>`

#### Cisco IOS-XR
- `Building configuration...`
- `!! IOS XR Configuration <version>`
- `!! Last configuration change at <timestamp> by <user>`

#### Arista EOS
- `! Command: show running-config`
- `! device: <device-info>`
- `! boot system flash:/<image>`

#### General Metadata
- Empty lines (whitespace-only lines)
- Comment lines that contain only timestamps or metadata

### What Gets Compared (Actual Configuration)

After normalization, the following configuration elements are compared:

- **Interface configurations**
- **Routing protocols**
- **Access control lists (ACLs)**
- **VLANs**
- **User accounts**
- **SNMP configuration**
- **System settings** (hostname, domain, etc.)
- **Any actual configuration commands**

## Examples

### Example 1: No Configuration Change (Backup Skipped)

**Previous Backup:**
```
!Time: Mon Dec 12 18:19:15 2025
hostname router1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
```

**Current Backup:**
```
!Time: Mon Dec 12 18:30:45 2025
hostname router1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
```

**Normalized Comparison:**
```
Previous: hostname router1\ninterface GigabitEthernet0/0\n ip address 192.168.1.1 255.255.255.0
Current:  hostname router1\ninterface GigabitEthernet0/0\n ip address 192.168.1.1 255.255.255.0
```

**Result:** ✅ **Identical** → `backup_has_changes = false` → **Publish skipped**

### Example 2: Configuration Change Detected (Backup Published)

**Previous Backup:**
```
!Time: Mon Dec 12 18:19:15 2025
hostname router1
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
```

**Current Backup:**
```
!Time: Mon Dec 12 18:30:45 2025
hostname router1
interface GigabitEthernet0/0
 ip address 192.168.1.2 255.255.255.0
```

**Normalized Comparison:**
```
Previous: hostname router1\ninterface GigabitEthernet0/0\n ip address 192.168.1.1 255.255.255.0
Current:  hostname router1\ninterface GigabitEthernet0/0\n ip address 192.168.1.2 255.255.255.0
```

**Result:** ❌ **Different** → `backup_has_changes = true` → **Publish triggered**

### Example 3: First Backup (Always Published)

**Previous Backup:** Does not exist

**Result:** ✅ **No previous backup** → `backup_has_changes = true` → **Publish triggered** (first backup always created)

## Implementation Details

### Normalization Command

The normalization is performed using `sed` and `grep` commands:

```bash
sed -E \
  -e '/^!Command:/d' \
  -e '/^!Running configuration last done at:/d' \
  -e '/^!Time:/d' \
  -e '/^!NVRAM config last updated at:/d' \
  -e '/^!No configuration change since last restart/d' \
  "backup_file.txt" | grep -v '^[[:space:]]*$'
```

This command:
1. Removes timestamp/metadata lines using `sed`
2. Removes empty lines using `grep`
3. Preserves all actual configuration content

### Comparison Logic

The comparison is performed in the `differential_scm.yaml` task file:

```yaml
- name: Compare normalized backups
  ansible.builtin.set_fact:
    backup_has_changes: "{{ normalized_previous != normalized_current }}"
```

- If `normalized_previous == normalized_current`: No changes detected
- If `normalized_previous != normalized_current`: Changes detected

### Publish Decision

The publish task only runs when `backup_has_changes` is `true`:

```yaml
- name: Include build tasks
  ansible.builtin.include_tasks: publish.yaml
  when: 
    - data_store.scm.origin is defined
    - backup_has_changes | default(true)
  run_once: true
```

## Benefits

1. **Reduced SCM Noise**: No unnecessary commits or pull requests for timestamp-only changes
2. **Storage Efficiency**: Only stores backups when configuration actually changes
3. **Idempotency**: Running the playbook multiple times with the same config won't create new PRs
4. **Clear Change History**: Git history only shows actual configuration changes

## Limitations

1. **Platform-Specific Metadata**: The normalization patterns are designed for common Cisco and Arista metadata. Custom metadata formats may not be filtered.
2. **Whitespace Changes**: Significant whitespace changes in configuration (beyond empty lines) will be detected as changes.
3. **Order-Dependent**: If device output order changes but configuration is identical, it may be detected as a change (rare).

## Troubleshooting

### Backup Always Shows Changes

If backups always show changes even when configuration hasn't changed:

1. **Check normalization patterns**: Verify that all timestamp/metadata lines are being removed
2. **Inspect normalized content**: Use debug tasks to see what's being compared:
   ```yaml
   - name: Debug normalized content
     ansible.builtin.debug:
       msg: "Previous: {{ normalized_previous }}\nCurrent: {{ normalized_current }}"
   ```

### Backup Never Shows Changes

If backups never show changes even when configuration changed:

1. **Verify backup type**: Ensure `type: "diff"` is set
2. **Check file paths**: Verify previous backup file exists and is being read correctly
3. **Check normalization**: Ensure normalization isn't removing actual configuration lines

## Best Practices

1. **Use `type: "diff"` for automated backups**: Prevents unnecessary commits in scheduled backup jobs
2. **Use `type: "full"` for manual backups**: Ensures backup is always created when manually triggered
3. **Monitor backup_has_changes**: Use the debug message to understand when backups are skipped
4. **Review normalized content**: Periodically verify that normalization is working correctly for your device types

## Related Files

- `/roles/backup/tasks/differential_scm.yaml`: Differential backup logic
- `/roles/backup/tasks/backup.yaml`: Main backup orchestration
- `/roles/backup/tasks/cli_backup.yaml`: Backup file creation
- `/roles/backup/tasks/publish.yaml`: SCM publishing logic

## See Also

- [Network Backup Role README](README.md)
- [Ansible Network Backup Collection Documentation](https://docs.ansible.com/ansible/latest/collections/network/backup/)

