# Confluence DC Automation

Ansible automation for deploying **Atlassian Confluence Data Center** on RHEL 9, including infrastructure installation, PostgreSQL JDBC integration, systemd service management, post-setup-wizard cluster configuration, and uninstallation.

## Repository Structure

```
confluence-dc-ansible
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── confluence.yml        # Variable overrides for the confluence group
│   └── hosts.yml                 # Inventory of Confluence nodes
├── playbooks
│   ├── bootstrap_confluence.yml  # Post-setup-wizard cluster configuration
│   ├── install_confluence.yml    # Full infrastructure install
│   └── uninstall_confluence.yml  # Removes Confluence installation
└── roles
    └── confluence
        ├── defaults/main.yml     # Default role variables
        ├── files/                # Installer archive + PostgreSQL JDBC driver
        ├── handlers/main.yml     # Restart / reload handlers
        ├── meta/main.yml         # Role metadata
        ├── tasks/                # Task files (see below)
        ├── templates/            # Jinja2 templates for config files
        └── vars/main.yml         # Role variables
```

## What This Automation Does

The `confluence` role is broken into discrete task files, orchestrated by `roles/confluence/tasks/main.yml`:

| Task file | Purpose |
|---|---|
| `precheck.yml` | Validates OS (RHEL 9 only), verifies the installer archive exists, detects/validates `JAVA_HOME` (must be a JDK), checks disk space, and tests PostgreSQL connectivity |
| `prerequisites.yml` | Installs required packages, creates the Confluence OS user/group, and creates install/home/shared-home directories |
| `install.yml` | Extracts the Confluence archive, renames it to the target install directory, sets ownership, and creates `logs`, `temp`, and `work` directories |
| `configure.yml` | Deploys `confluence-init.properties`, `setenv.sh` (JVM config), and the PostgreSQL JDBC driver; removes any stale `confluence.cfg.xml` to force first-run setup |
| `systemd.yml` | Makes startup scripts executable, deploys and enables the systemd unit, and starts the Confluence service |
| `validate.yml` | Waits for the HTTP port to come up, checks the base URL and the setup wizard endpoint, and prints next-step instructions |
| `cluster.yml` | Deploys `cluster.properties` — **only runs after** the setup wizard has completed (detected via the presence of `confluence.cfg.xml`) |
| `validate_cluster.yml` | Confirms `cluster.properties` was deployed and displays the cluster configuration |
| `uninstall.yml` | Removes the Confluence installation *(currently empty — see [Known Gaps](#known-gaps))* |

## Playbooks

### `playbooks/install_confluence.yml`
Runs the full role (`precheck` → `prerequisites` → `install` → `configure` → `systemd` → `validate`). Use this for the initial infrastructure setup.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/install_confluence.yml
```

### `playbooks/bootstrap_confluence.yml`
Runs **only** `cluster.yml`. This must be executed **after** you have manually completed the Confluence Setup Wizard in the browser and the database schema has been created. Running it before that will simply skip cluster configuration (it checks for `confluence.cfg.xml`).

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap_confluence.yml
```

### `playbooks/uninstall_confluence.yml`
Removes the Confluence installation via `tasks/uninstall.yml`.

```bash
ansible-playbook -i inventory/hosts.yml playbooks/uninstall_confluence.yml
```

## End-to-End Deployment Flow

1. Populate `inventory/hosts.yml` and `inventory/group_vars/confluence.yml` with your environment's values.
2. Place the Confluence installer archive and PostgreSQL JDBC driver in `roles/confluence/files/`.
3. Run `install_confluence.yml` to provision the OS user, install Confluence, configure the JVM/JDBC driver, and start the service.
4. Open `http://<host>:<confluence_http_port>` in a browser and complete the **Confluence Setup Wizard** (this generates `confluence.cfg.xml` and the database schema).
5. Run `bootstrap_confluence.yml` to apply cluster configuration (`cluster.properties`) now that Confluence is initialized.
6. (Optional) Run `validate_cluster.yml` tasks or re-run validation to confirm the cluster config was applied.

## Key Variables

These are referenced throughout the role and should be defined in `roles/confluence/defaults/main.yml`, `roles/confluence/vars/main.yml`, or overridden in `inventory/group_vars/confluence.yml`:

| Variable | Description |
|---|---|
| `confluence_version` | Confluence version being installed |
| `confluence_archive_path` | Path to the local installer `.tar.gz` |
| `confluence_install_parent` | Parent directory where Confluence is extracted |
| `confluence_install_dir` | Final Confluence installation directory |
| `confluence_home` | Confluence home directory (logs, temp, work, config) |
| `confluence_shared_home` | Shared home directory (required for clustering) |
| `confluence_user` / `confluence_group` | OS user/group Confluence runs as |
| `confluence_required_packages` | OS packages installed via `dnf` |
| `confluence_http_port` | HTTP port Confluence listens on |
| `confluence_service_name` | Name of the systemd service |
| `confluence_cluster_enabled` | Whether to deploy cluster configuration |
| `confluence_cluster_node_id` | Node ID for cluster configuration |
| `postgres_host` / `postgres_port` / `postgres_database` | PostgreSQL connection details |
| `java_home` | Detected automatically in `precheck.yml`; not user-set |

## Requirements

- **Target OS:** RHEL 9 (enforced by an `assert` in `precheck.yml`; the playbooks will fail on any other distribution)
- **Java:** A JDK must be present on the target host (validated by checking for `javac` under the detected `JAVA_HOME`)
- **PostgreSQL:** An external, reachable PostgreSQL instance (connectivity is checked during precheck)
- **Ansible control node:** `ansible-core` with the `community.general` collection if using modules such as `dnf`/`systemd` beyond core (adjust per your Ansible version)
- **Files:** The Confluence installer archive and the PostgreSQL JDBC driver (`postgresql-42.7.4.jar`) must be placed under `roles/confluence/files/` before running the install playbook

## Handlers

Defined in `roles/confluence/handlers/main.yml`:

- **Reload systemd** — runs `systemd daemon_reload`
- **Restart Confluence** — restarts the Confluence systemd service (also reloads the daemon)

## Known Gaps

- `roles/confluence/tasks/uninstall.yml` is currently **empty**. The `uninstall_confluence.yml` playbook and role `meta` reference it, but no removal logic (stopping/disabling the service, deleting install/home directories, removing the OS user) has been implemented yet.
- `validate_cluster.yml` exists as a task file but is not currently wired into any playbook or `main.yml` import — it must be invoked explicitly (e.g., via `import_tasks` or an ad-hoc playbook) if cluster validation is desired after bootstrap.

## Notes

- `configure.yml` deletes any existing `confluence.cfg.xml` on every run, which forces the setup wizard to re-run on subsequent installs. Avoid re-running `install_confluence.yml` against an already-initialized, in-use instance without accounting for this.
- Cluster configuration is intentionally decoupled from the initial install because Confluence must complete its setup wizard (and generate `confluence.cfg.xml`) before cluster settings can be safely applied.
