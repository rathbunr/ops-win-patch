# ops-win-patch

Windows patching automation using `ansible.windows.win_updates`. Searches, installs, reboots, validates, and reports — no MECM server access required.

This role talks directly to the Windows Update Agent (WUA) on each host. It works with whatever update source the host is configured for: WSUS, Windows Update, or Microsoft Update.

## Requirements

- `ansible.windows` collection
- WinRM over HTTPS with Kerberos authentication
- Target hosts must have the Windows Update service running
- Ansible user must be a member of local Administrators

## Repository Structure

```
ops-win-patch/
├── requirements.yml
├── playbooks/
│   └── win_patch_and_reboot.yml
└── roles/
    └── win_patch_and_reboot/
        ├── defaults/
        │   └── main.yml
        └── tasks/
            ├── main.yml
            ├── pre_patch.yml
            ├── search.yml
            ├── install.yml
            ├── validate.yml
            └── summary.yml
```

## Workflow

1. **Pre-patch** — Checks available disk space on C: (fails if below threshold), detects pending reboots, captures a baseline of all running auto-start services for post-patch comparison.
2. **Search** — Queries available updates by category. Reports the count before installing so you see what is pending even if nothing installs.
3. **Install** — Installs updates with controlled reboot handling. A rescue block catches failures without killing the play for remaining hosts.
4. **Validate** — Waits for the system to stabilize after reboot, compares currently running services against the pre-patch baseline, flags anything that did not come back up, and searches again for remaining updates to confirm the system is clean.
5. **Summary** — Structured output for AAP job results including host, status, disk, counts, reboot state, remaining updates, and any stopped services.

## Role Variables

All variables are defined in `roles/win_patch_and_reboot/defaults/main.yml` and can be overridden via AAP survey, extra vars, or group/host vars.

| Variable | Default | Description |
|---|---|---|
| `win_patch_categories` | `SecurityUpdates,CriticalUpdates,DefinitionUpdates` | Comma-separated update categories |
| `win_patch_accept_list` | `""` | Comma-separated KB list to include |
| `win_patch_reject_list` | `""` | Comma-separated KB list to exclude |
| `win_patch_reboot` | `true` | Allow reboots when required |
| `win_patch_reboot_timeout` | `900` | Seconds to wait for reboot to complete |
| `win_patch_min_disk_gb` | `5` | Minimum free disk space on C: in GB |
| `win_patch_log_path` | `C:\Windows\Temp\ansible_wu.txt` | Remote log path for WUA |
| `win_patch_stabilize_delay` | `30` | Seconds to wait before post-reboot checks |
| `win_patch_stabilize_timeout` | `300` | Max seconds to wait for system to respond after reboot |
| `win_patch_serial` | `0` | Batch size for rolling updates (0 = all at once) |

### Available Update Categories

`SecurityUpdates`, `CriticalUpdates`, `DefinitionUpdates`, `UpdateRollups`, `Updates`, `FeaturePacks`, `ServicePacks`, `Tools`, `Drivers`, `Connectors`, `Application`

## AAP Configuration

### Project

- **SCM Type:** Git
- **SCM URL:** https://github.com/rathbunr/ops-win-patch.git
- **SCM Branch:** main

### Job Template

- **Playbook:** `playbooks/win_patch_and_reboot.yml`
- **Credential:** Windows credential (WinRM/Kerberos)
- **Limit:** Target hosts or groups

### Recommended Survey Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `win_patch_categories` | Text | `SecurityUpdates,CriticalUpdates,DefinitionUpdates` | Update categories |
| `win_patch_accept_list` | Text | *(empty)* | Specific KBs to install |
| `win_patch_reject_list` | Text | *(empty)* | Specific KBs to skip |
| `win_patch_reboot` | Boolean | `true` | Allow reboots |
| `win_patch_serial` | Integer | `0` | Rolling batch size |

## Examples

### Install security and definition updates with reboot

Default behavior — no survey overrides needed.

### Emergency KB deployment

Set `win_patch_accept_list` to `KB5034441,KB5034439` in the survey to install only those specific updates.

### Scan only (no install)

Change `state: installed` to `state: searched` in `roles/win_patch_and_reboot/tasks/install.yml` or create a separate scan-only playbook referencing the search task directly.

### Rolling updates in batches of 5

Set `win_patch_serial` to `5` in the survey. Hosts are patched in groups of 5 — if one batch fails, remaining batches are skipped.

## Notes

- This role does not depend on `microsoft.mecm` or MECM server access. It is client-side only and works in any environment where `ansible.windows.win_updates` is available.
- The update source is whatever the host is configured to use. If the host points to WSUS, updates come from WSUS. If it points to Microsoft Update, updates come from the internet.
- The service baseline comparison captures all services that are both running and set to automatic start before patching. After patching and reboot, any service from that baseline that is no longer running is flagged in the summary. This is an advisory warning, not a failure.
