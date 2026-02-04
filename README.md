# sse.sinequa

DevOps - Sinequa lifecycle automation (install/upgrade) on Windows

This role supports **install** and **upgrade** workflows for Sinequa on Windows hosts, with optional per-node configuration driven by inventory groups (e.g. `webengine`, `webapp`, `engine`, `indexer`, `queuecluster`).

## Supported platforms

- Windows Server 2022
- Windows Server 2025

(As declared in `meta/main.yml`.)

## Requirements

### Ansible

- Ansible **>= 2.14**

### Collections

Declared in `requirements.yml`:

- `ansible.windows`
- `community.windows`

> Note: `requirements.yml` currently lists `ansible.windows` twice with different version constraints. Consider keeping only one entry.

### Target host prerequisites

- The role uses **7-Zip** (`7z.exe`) to extract the Sinequa archive. Ensure 7-Zip is installed and available in PATH (the role also adds `C:\Program Files\7-Zip` to the machine PATH).

## Role behavior

At a high level, the role:

1. **Discovers** existing installation and version (`tasks/discover.yml`)
2. Applies **prerequisites** (firewall rules, env vars, PATH, optional hardening tweaks) (`tasks/prereqs.yml`)
3. **Downloads** the requested Sinequa release zip using a bearer token (`tasks/download.yml`)
4. Runs either:
   - **Install** (`tasks/install.yml`) when `sinequa_action: install`
   - **Upgrade** (`tasks/upgrade.yml`) when `sinequa_action: upgrade`
5. Applies **group-specific** configuration:
   - `webengine` → renders `sinequa.xml` (`tasks/webengine.yml`)
   - `webapp` → webapp firewall ports (`tasks/webapp.yml`)
   - `engine` → engine firewall ports (`tasks/engine.yml`)
   - `indexer` → indexer firewall ports (`tasks/indexer.yml`)
   - `queuecluster` → queue cluster firewall ports (`tasks/queuecluster.yml`)
6. **Cleans up** temporary files (`tasks/cleanup.yml`)

## Inventory groups

Add hosts to these groups to enable group-specific tasks:

- `webengine`
- `webapp`
- `engine`
- `indexer`
- `queuecluster`

Example inventory snippet:

```ini
[webengine]
win-webengine-01

[webapp]
win-webapp-01

[engine]
win-engine-01

[indexer]
win-indexer-01

[queuecluster]
win-queue-01
```

## Role variables

### Main inputs (defaults)

From `defaults/main.yml`:

| Variable | Default | Description |
|---|---:|---|
| `sinequa_action` | `install` | Action to execute: `install` or `upgrade`. |
| `sinequa_version` | `11.13.0.2133` | Target Sinequa version. Used in download URL and upgrade guardrails. |
| `sinequa_download_token` | `token` | **Required** bearer token for Sinequa download API. Override via vault/group_vars. |

### Derived / operational variables

From `vars/main.yml` (override in inventory/group_vars if needed):

| Variable | Default | Description |
|---|---:|---|
| `temp_folder` | `C:\\Temp\\` | Temporary working directory (download + staging). |
| `destination_folder` | `C:\\` | Destination root used during extraction. |
| `sinequa_install_root` | `{{ destination_folder }}Sinequa` | Sinequa installation directory. |
| `version_file` | `{{ sinequa_install_root }}\\version.txt` | File used to detect installed version. |
| `sinequa_service_name` | `sinequa.service` | Windows service name queried during discovery. |
| `service_name` | `sinequa.service` | Service name used when creating the service during install. |
| `sinequa_temp_dir` | `D:\\sinequa\\temp` | Value for machine env var `SINEQUA_TEMP`. |
| `sinequa_release_download_url` | (templated) | Download URL for the requested version zip. |
| `sinequa_download_headers` | (templated) | HTTP headers including bearer token. |

### Firewall ports

These are opened inbound (TCP) using `community.windows.win_firewall_rule`:

| Variable | Default | Applies to |
|---|---:|---|
| `sinequa_common_firewall_ports` | `[10301]` | Always |
| `sinequa_webapp_firewall_ports` | `[443]` | `webapp` group |
| `sinequa_engine_firewall_ports` | `[10300]` | `engine` group |
| `sinequa_indexer_firewall_ports` | `[13002]` | `indexer` group |
| `sinequa_queue_cluster_ports` | `[10303]` | `queuecluster` group |


## Example playbook

```yaml
- name: Install/upgrade Sinequa
  hosts: windows
  gather_facts: false
  roles:
    - role: sse.sinequa
      vars:
        sinequa_action: install          # or: upgrade
        sinequa_version: "11.13.0.2133"
        sinequa_download_token: "{{ vault_sinequa_download_token }}"
```

## Dependencies

None (declared as an empty list in `meta/main.yml`).

## Testing

A basic test playbook exists at `tests/test.yml`:

```bash
ansible-playbook -i tests/inventory tests/test.yml
```

## License

MIT-0

## Author

- Charles BORCKE (CEBIS)
