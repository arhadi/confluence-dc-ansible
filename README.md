# Confluence Data Center Ansible Automation

Production-oriented Ansible automation for installing, configuring, validating, bootstrapping, and uninstalling Atlassian Confluence Data Center on RHEL 9 with PostgreSQL.

The role supports **two installation methods**:

- **Archive installation** using the Atlassian `.tar.gz` distribution.
- **Linux binary installer** using the Atlassian `-x64.bin` installer in unattended Install4j mode.

Both installation methods converge on the same Ansible-managed configuration, systemd service, validation, setup-wizard lifecycle, cluster bootstrap, and uninstall workflow.

> Current tested Confluence version: **10.2.15**

> Current platform used during testing: **RHEL 9**, **OpenJDK 21**, **PostgreSQL on port 15432**.

---

## Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Current Repository Structure](#current-repository-structure)
- [Lifecycle Model](#lifecycle-model)
- [Requirements](#requirements)
- [Installation Methods](#installation-methods)
  - [Archive Method](#archive-method)
  - [Linux Binary Installer Method](#linux-binary-installer-method)
- [Preparing Installation Media](#preparing-installation-media)
- [Inventory](#inventory)
- [Configuration Variables](#configuration-variables)
- [Usage](#usage)
  - [Syntax Check](#syntax-check)
  - [Precheck](#precheck)
  - [Install Confluence](#install-confluence)
  - [Validate Installation](#validate-installation)
  - [Complete Web Setup](#complete-web-setup)
  - [Bootstrap Data Center Cluster](#bootstrap-data-center-cluster)
  - [Validate Cluster](#validate-cluster)
  - [Uninstall Confluence](#uninstall-confluence)
- [Role Task Flow](#role-task-flow)
- [Binary Installer Response File](#binary-installer-response-file)
- [Installer Runtime User Behavior](#installer-runtime-user-behavior)
- [Confluence Home Ownership](#confluence-home-ownership)
- [Configuration Behavior](#configuration-behavior)
- [PostgreSQL JDBC Driver](#postgresql-jdbc-driver)
- [Systemd Ownership Model](#systemd-ownership-model)
- [Initial Setup and License Boundary](#initial-setup-and-license-boundary)
- [Cluster Bootstrap Model](#cluster-bootstrap-model)
- [Shared Home](#shared-home)
- [Idempotency](#idempotency)
- [Uninstall Behavior](#uninstall-behavior)
- [Tags Reference](#tags-reference)
- [Validation](#validation)
- [Troubleshooting](#troubleshooting)
- [Security Recommendations](#security-recommendations)
- [Operational Recommendations](#operational-recommendations)
- [Tested Lifecycle](#tested-lifecycle)

---

## Overview

The `confluence` role manages the Confluence Data Center installation lifecycle while deliberately separating **software installation** from **application bootstrap**.

The main installation workflow is:

```text
precheck
   |
   v
prerequisites
   |
   v
installation method dispatcher
   |
   +-- archive   -> install_archive.yml
   |
   +-- installer -> install_installer.yml
   |
   v
configure
   |
   v
systemd
   |
   v
validate
```

After the initial installation is running, Confluence must complete its browser-based initial setup before cluster configuration is applied:

```text
Initial Ansible installation
        |
        v
Confluence running on :8090
        |
        v
Setup Wizard / License
        |
        v
confluence.cfg.xml becomes initialized
        |
        v
bootstrap_confluence.yml
        |
        v
cluster.properties
        |
        v
Cluster validation
```

This separation is intentional.

The initial installation playbook does **not** force Confluence into a completed application state. It installs and starts the software, verifies that the application is reachable, and leaves the Atlassian setup wizard available for the license and initial application configuration.

The installation method is selected using:

```yaml
confluence_install_method: "archive"
```

or:

```yaml
confluence_install_method: "installer"
```

---

## Key Features

- RHEL 9 platform validation.
- Java 21/JDK validation.
- PostgreSQL connectivity precheck.
- Dual installation methods:
  - `.tar.gz`
  - Atlassian Linux `.bin`
- SHA256 verification for installation media.
- Custom installation directory under `/app`.
- Custom Confluence Home.
- Custom HTTP and control ports for binary installer.
- Install4j unattended response-file support.
- Prevention of installer-controlled startup.
- Prevention of installer-controlled systemd lifecycle.
- Runtime-user normalization after `.bin` installation.
- Recursive Confluence Home ownership normalization.
- PostgreSQL JDBC driver deployment.
- JVM heap configuration.
- Ansible-managed systemd service.
- PID-file support.
- HTTP and setup-page validation.
- Safe separation of installation and Data Center cluster bootstrap.
- Detection of incomplete Setup Wizard state.
- Protection against premature `cluster.properties` deployment.
- Shared-home configuration.
- Data Center node ID and cluster name configuration.
- Separate cluster validation task.
- Common uninstall workflow for archive and installer methods.
- PostgreSQL database preservation during uninstall.
- Repeatable Ansible execution.

---

## Current Repository Structure

Current project layout:

```text
confluence-dc-ansible/
├── 1
├── ansible.cfg
├── inventory
│   ├── group_vars
│   │   └── confluence.yml
│   └── hosts.yml
├── playbooks
│   ├── bootstrap_confluence.yml
│   ├── install_confluence.yml
│   └── uninstall_confluence.yml
├── roles
│   └── confluence
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── tasks
│       │   ├── cluster.yml
│       │   ├── configure.yml
│       │   ├── install.yml
│       │   ├── install_archive.yml
│       │   ├── install_installer.yml
│       │   ├── main.yml
│       │   ├── precheck.yml
│       │   ├── prerequisites.yml
│       │   ├── systemd.yml
│       │   ├── uninstall.yml
│       │   ├── validate.yml
│       │   └── validate_cluster.yml
│       ├── templates
│       │   ├── cluster.properties.j2
│       │   ├── confluence-init.properties.j2
│       │   ├── confluence.service.j2
│       │   ├── response.varfile.j2
│       │   └── setenv.sh.j2
│       └── vars
│           └── main.yml
└── ~
```

The files under `roles/confluence/files/` are installation media and are normally not committed to public source control.

During testing, the directory contained:

```text
atlassian-confluence-10.2.15.tar.gz
atlassian-confluence-10.2.15-x64.bin
postgresql-42.7.4.jar
```

---

## Lifecycle Model

The project has three operational playbooks.

### Installation

```text
playbooks/install_confluence.yml
```

Responsible for:

```text
precheck
prerequisites
install
configure
systemd
validate
```

### Cluster Bootstrap

```text
playbooks/bootstrap_confluence.yml
```

Responsible for applying cluster configuration **only after Confluence initial setup is complete**.

### Uninstall

```text
playbooks/uninstall_confluence.yml
```

Responsible for stopping and removing the Ansible-managed Confluence installation according to configured removal flags.

This produces a safer lifecycle than attempting to configure every aspect of Confluence during the first startup.

---

## Requirements

### Control Node

Recommended:

- Ansible 2.15 or later.
- Python 3.
- SSH access to managed nodes, or `ansible_connection: local` for local testing.
- Installation media available under `roles/confluence/files/`.

### Managed Node

Current automation targets:

- RHEL 9.
- Java 21.
- At least 10 GB free space on the installation filesystem.
- Network access to PostgreSQL.
- Root or privilege-escalation access.
- systemd.
- Required font packages.

### Java

The precheck detects the active Java installation:

```bash
java -version
```

The tested environment reported:

```text
openjdk version "21.0.12" 2026-07-21 LTS
OpenJDK Runtime Environment (Red_Hat-21.0.12.0.8-1)
OpenJDK 64-Bit Server VM
```

The role additionally checks for:

```text
$JAVA_HOME/bin/javac
```

to ensure the detected Java Home represents a JDK rather than only a JRE.

### PostgreSQL

The database must already exist and be reachable.

Current tested values:

```yaml
postgres_host: "localhost"
postgres_port: 15432
postgres_database: "confluence"
postgres_user: "confluence"
```

The current installation automation validates network reachability but does not create the PostgreSQL database.

For production, database credentials should be stored in Ansible Vault.

---

## Installation Methods

## Archive Method

Select:

```yaml
confluence_install_method: "archive"
```

The archive is configured using:

```yaml
confluence_archive: "atlassian-confluence-{{ confluence_version }}.tar.gz"
confluence_archive_sha256: "<SHA256>"
```

For the tested Confluence 10.2.15 archive:

```text
47689cb9cda55a00d34ead5dffa62f09cc74911db1718289ac78117e9da21479
```

The archive workflow is implemented in:

```text
roles/confluence/tasks/install_archive.yml
```

The archive workflow:

1. checks the installation archive;
2. fails if the media is missing;
3. extracts the distribution;
4. detects the extracted Atlassian directory;
5. renames it to the configured final installation directory;
6. normalizes ownership;
7. creates required application directories.

Example:

```yaml
confluence_install_parent: "/app"
confluence_install_dir: "/app/confluence-10.2.15"
confluence_home: "/app/confluence-data"
```

---

## Linux Binary Installer Method

Select:

```yaml
confluence_install_method: "installer"
```

The installer is configured using:

```yaml
confluence_installer: "atlassian-confluence-{{ confluence_version }}-x64.bin"
confluence_installer_sha256: "<SHA256>"
```

For the tested Confluence 10.2.15 binary installer:

```text
ec0f36564847920355aef0a67d467e5de2229ea23eb4b378240d703fdbb9aacb
```

The binary was verified as:

```text
POSIX shell script executable (binary data)
```

The installer workflow is implemented in:

```text
roles/confluence/tasks/install_installer.yml
```

The binary installer is run unattended using an Install4j response file generated from:

```text
roles/confluence/templates/response.varfile.j2
```

The desired binary-installer lifecycle is:

```text
Atlassian .bin
      |
      v
Install application files only
      |
      v
Do NOT start Confluence
      |
      v
Do NOT install Atlassian-managed service
      |
      v
Normalize runtime user
      |
      v
Normalize ownership
      |
      v
Ansible configure.yml
      |
      v
Ansible systemd
      |
      v
Ansible validation
```

This keeps Ansible as the lifecycle authority.

---

## Preparing Installation Media

Place required files under:

```text
roles/confluence/files/
```

For Confluence 10.2.15:

```text
roles/confluence/files/atlassian-confluence-10.2.15.tar.gz
roles/confluence/files/atlassian-confluence-10.2.15-x64.bin
roles/confluence/files/postgresql-42.7.4.jar
```

Verify:

```bash
ls -lh roles/confluence/files/
```

Calculate checksums:

```bash
sha256sum \
  roles/confluence/files/atlassian-confluence-10.2.15.tar.gz

sha256sum \
  roles/confluence/files/atlassian-confluence-10.2.15-x64.bin
```

Tested values:

```text
Archive:
47689cb9cda55a00d34ead5dffa62f09cc74911db1718289ac78117e9da21479

Binary:
ec0f36564847920355aef0a67d467e5de2229ea23eb4b378240d703fdbb9aacb
```

The precheck validates only the installation media selected by:

```yaml
confluence_install_method
```

When:

```yaml
confluence_install_method: "installer"
```

archive checks are skipped.

When:

```yaml
confluence_install_method: "archive"
```

binary-installer checks are skipped.

---

## Inventory

Example local inventory:

```yaml
all:
  children:
    confluence:
      hosts:
        localhost:
          ansible_connection: local
```

For remote deployment:

```yaml
all:
  children:
    confluence:
      hosts:
        confluence01.example.com:
          ansible_host: 10.1.20.31
          ansible_user: ansible
```

For a multi-node Data Center deployment, each node should receive a unique:

```yaml
confluence_cluster_node_id
```

Prefer `host_vars` or host-level inventory values for node-specific IDs.

---

## Configuration Variables

The primary configuration is:

```text
inventory/group_vars/confluence.yml
```

### Product Version

```yaml
confluence_version: "10.2.15"
```

### Installation Method

```yaml
confluence_install_method: "installer"
```

Supported values:

```text
archive
installer
```

### Archive

```yaml
confluence_archive: "atlassian-confluence-{{ confluence_version }}.tar.gz"

confluence_archive_sha256: "47689cb9cda55a00d34ead5dffa62f09cc74911db1718289ac78117e9da21479"
```

### Binary Installer

```yaml
confluence_installer: "atlassian-confluence-{{ confluence_version }}-x64.bin"

confluence_installer_sha256: "ec0f36564847920355aef0a67d467e5de2229ea23eb4b378240d703fdbb9aacb"
```

### Installation Directories

```yaml
confluence_install_parent: "/app"

confluence_install_dir: "{{ confluence_install_parent }}/confluence-{{ confluence_version }}"

confluence_home: "/app/confluence-data"
```

Current tested result:

```text
/app/confluence-10.2.15
/app/confluence-data
```

### Runtime Account

```yaml
confluence_user: "confluence"
confluence_group: "confluence"
```

### Java

```yaml
java_package: "java-21-openjdk"
```

The automation dynamically detects Java Home during precheck.

### PostgreSQL

```yaml
postgres_host: "localhost"
postgres_port: 15432
postgres_database: "confluence"
postgres_user: "confluence"
postgres_password: "<secret>"
```

### PostgreSQL JDBC

```yaml
postgresql_jdbc_version: "42.7.4"

postgresql_jdbc_jar: "postgresql-{{ postgresql_jdbc_version }}.jar"
```

### Network

```yaml
confluence_http_port: 8090
confluence_https_port: 8443
```

For the binary installer, the control port should also be defined according to the implementation used by `response.varfile.j2`.

The tested manual binary installation used:

```text
HTTP port    : 8090
Control port : 8000
```

### Service

```yaml
confluence_service_name: "confluence"
```

### JVM

```yaml
confluence_xms: "4096m"
confluence_xmx: "4096m"
```

The tested runtime showed:

```text
-Xms4096m
-Xmx4096m
```

### Data Center Cluster

```yaml
confluence_cluster_enabled: true

confluence_cluster_name: "confluence-cluster"

confluence_cluster_node_id: "confluence-node01"

confluence_shared_home: "/app/confluence-shared-home"
```

The inventory values override defaults.

The defaults currently provide:

```yaml
confluence_cluster_name: ConfluenceDC
confluence_cluster_node_id: "{{ inventory_hostname }}"
confluence_shared_home: /app/confluence-shared
```

For predictable production deployment, explicitly define cluster settings in inventory.

---

## Usage

Run commands from the repository root.

Example:

```bash
cd /home/ansible/confluence-dc-ansible
```

---

## Syntax Check

Installation:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --syntax-check
```

Cluster bootstrap:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/bootstrap_confluence.yml \
  --syntax-check
```

Uninstall:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_confluence.yml \
  --syntax-check
```

Always perform syntax validation after modifying task files, templates, inventory, or playbooks.

---

## Precheck

Run:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags precheck
```

The tested installer-method precheck produced:

```text
Version           : 10.2.15
Install Method    : installer
Archive           : atlassian-confluence-10.2.15.tar.gz
Binary Installer  : atlassian-confluence-10.2.15-x64.bin
Install Directory : /app/confluence-10.2.15
Home Directory    : /app/confluence-data
Database          : confluence
DB Host           : localhost
HTTP Port         : 8090
Cluster Enabled   : True
```

Precheck validates:

- RHEL 9.
- Installation method.
- Selected installation media.
- SHA256.
- Java installation.
- Java Home/JDK.
- Existing Confluence user state.
- Disk capacity.
- PostgreSQL reachability.

Example successful installer precheck:

```text
ok=18
changed=1
unreachable=0
failed=0
skipped=3
```

---

## Install Confluence

Choose the installation method in:

```text
inventory/group_vars/confluence.yml
```

Archive:

```yaml
confluence_install_method: "archive"
```

Binary installer:

```yaml
confluence_install_method: "installer"
```

Then execute:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml
```

A successful tested installer execution completed with:

```text
ok=64
changed=17
unreachable=0
failed=0
skipped=5
```

The role then started the Ansible-managed Confluence service.

---

## Validate Installation

The installation playbook already executes `validate.yml`.

Validation can also be run using the role's validation tag when appropriate:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags validate
```

The tested validation reported:

```text
HTTP Status  : 302
Setup Status : 200
```

and:

```text
========================================
Confluence installation completed
Version     : 10.2.15
Install Dir : /app/confluence-10.2.15
Home Dir    : /app/confluence-data
HTTP Port   : 8090
Database    : confluence
Service     : confluence
========================================
Next: Complete setup at http://10.148.0.2:8090
Then run: ansible-playbook playbooks/bootstrap_confluence.yml
```

A redirect to the Setup Wizard is a valid initial-installation condition.

---

## Complete Web Setup

After the software installation completes, open:

```text
http://<confluence-host>:8090
```

In the tested environment:

```text
http://10.148.0.2:8090
```

Before initial setup is complete, the HTTP response redirects to:

```text
/bootstrap/selectsetupstep.action
```

Example:

```bash
curl -sS -I http://localhost:8090/ | head
```

Observed:

```text
HTTP/1.1 302
Location: /bootstrap/selectsetupstep.action
```

At this stage, the software installation is healthy but the Confluence application is not yet initialized.

The Setup Wizard requires the appropriate Atlassian licensing/configuration steps.

If the license is not yet available, it is acceptable to stop at this point.

Do **not** run cluster bootstrap until the initial application setup is complete.

---

## Bootstrap Data Center Cluster

After the Setup Wizard is complete:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/bootstrap_confluence.yml
```

The bootstrap playbook imports:

```text
roles/confluence/tasks/cluster.yml
```

The bootstrap logic checks the actual Confluence initialization state before deploying cluster configuration.

This is important because the presence of:

```text
/app/confluence-data/confluence.cfg.xml
```

alone does **not** mean setup is complete.

During the tested initial state, the file already existed but contained:

```xml
<setupStep>setupstart</setupStep>
<setupType>initial</setupType>
<buildNumber>0</buildNumber>
```

The bootstrap logic correctly determined:

```text
Confluence configuration exists : True
Confluence initialized          : False
Cluster enabled                 : True
```

and stopped with a controlled failure.

This prevents premature cluster configuration.

---

## Validate Cluster

Cluster-specific validation is implemented in:

```text
roles/confluence/tasks/validate_cluster.yml
```

It checks:

```text
{{ confluence_home }}/cluster.properties
```

and reports:

```text
Cluster Enabled
Node ID
Shared Home
```

Example expected summary:

```text
===========================================
Confluence Cluster Configuration
Cluster Enabled : True
Node ID         : confluence-node01
Shared Home     : /app/confluence-shared-home
===========================================
```

Cluster validation should only be expected to succeed after initial setup and bootstrap have completed.

---

## Uninstall Confluence

Run:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/uninstall_confluence.yml
```

Review destructive removal variables before execution.

The uninstall workflow should be treated as independent of whether Confluence was originally installed from `.tar.gz` or `.bin`.

Ansible owns the resulting installation lifecycle.

---

## Role Task Flow

`roles/confluence/tasks/main.yml` currently imports:

| Tag | Task file | Purpose |
|---|---|---|
| `always`, `precheck` | `precheck.yml` | Platform, media, Java, DB and disk checks |
| `prerequisites` | `prerequisites.yml` | Packages, account and directory preparation |
| `install` | `install.yml` | Installation method dispatcher |
| `configure` | `configure.yml` | Home, JVM, JDBC and ownership |
| `systemd` | `systemd.yml` | Ansible-managed systemd lifecycle |
| `validate` | `validate.yml` | Post-install validation |

Installation dispatch:

```text
confluence_install_method=archive
             |
             +--> install_archive.yml

confluence_install_method=installer
             |
             +--> install_installer.yml
```

Cluster configuration is intentionally **not** part of the initial `main.yml` sequence.

Instead:

```text
bootstrap_confluence.yml
        |
        +--> cluster.yml
```

This separation prevents the first installation from trying to configure Data Center clustering before the application is initialized.

---

## Binary Installer Response File

The template is:

```text
roles/confluence/templates/response.varfile.j2
```

It was derived from a response file generated by a successful manual Confluence 10.2.15 binary installation.

The successful test generated:

```text
# install4j response file for Confluence 10.2.15
app.confHome=/var/tmp/confluence-bin-test-home
app.install.service$Boolean=false
existingInstallationDir=/opt/Confluence
httpPort$Long=8090
launch.application$Boolean=false
portChoice=custom
rmiPort$Long=8000
sys.adminRights$Boolean=true
sys.adminRightsUiRootUnix$Boolean=false
sys.installForAllUsers$Boolean=true
sys.installationDir=/var/tmp/confluence-bin-test
sys.languageId=en
sys.streamlinedInstallationString=false
```

The automation substitutes production Ansible variables for the test directories.

Two settings are particularly important:

```text
app.install.service$Boolean=false
launch.application$Boolean=false
```

These prevent Install4j from:

- installing its own service;
- starting Confluence before Ansible configuration is complete.

The binary installer should therefore be viewed as a **software extraction/install mechanism**, not the final service-management authority.

---

## Installer Runtime User Behavior

Manual testing demonstrated that the Atlassian binary installer can create additional runtime accounts.

During testing, repeated manual installer runs resulted in accounts such as:

```text
confluence
confluence1
confluence2
```

One generated `user.sh` contained:

```bash
CONF_USER="confluence1"
```

A later run produced:

```bash
CONF_USER="confluence"
```

The desired Ansible-managed state is:

```bash
CONF_USER="confluence"
```

matching:

```yaml
confluence_user: "confluence"
```

Check the runtime user with:

```bash
grep '^CONF_USER=' \
  /app/confluence-10.2.15/bin/user.sh
```

Expected:

```text
CONF_USER="confluence"
```

The service should not be changed to accommodate accidental installer-generated users such as `confluence1` or `confluence2`.

Ansible should remain authoritative.

---

## Confluence Home Ownership

A binary installer test exposed an important ownership issue.

The installation directory was correctly owned by:

```text
confluence:confluence
```

but the Home directory had become:

```text
confluence2:confluence
```

This caused:

```text
/app/confluence-10.2.15/bin/catalina.sh:
... /app/confluence-data/confluence.pid: Permission denied
```

The corrected `configure.yml` therefore contains:

```yaml
- name: Ensure Confluence home ownership
  ansible.builtin.file:
    path: "{{ confluence_home }}"
    state: directory
    owner: "{{ confluence_user }}"
    group: "{{ confluence_group }}"
    recurse: true
```

After normalization:

```bash
stat -c '%U:%G %a %n' \
  /app/confluence-10.2.15 \
  /app/confluence-data
```

reported:

```text
confluence:confluence 755 /app/confluence-10.2.15
confluence:confluence 700 /app/confluence-data
```

After restart, the PID file was created successfully:

```text
-rw-r----- confluence confluence /app/confluence-data/confluence.pid
```

This ownership enforcement is an important part of binary-installer convergence.

---

## Configuration Behavior

`roles/confluence/tasks/configure.yml` manages the Ansible-owned application configuration.

Current responsibilities include:

### Confluence Home Pointer

```text
confluence-init.properties.j2
```

deployed to:

```text
{{ confluence_install_dir }}/confluence/WEB-INF/classes/confluence-init.properties
```

### JVM Configuration

```text
setenv.sh.j2
```

deployed to:

```text
{{ confluence_install_dir }}/bin/setenv.sh
```

### PostgreSQL JDBC

The configured JDBC driver is copied into:

```text
{{ confluence_install_dir }}/confluence/WEB-INF/lib/
```

### Ownership

The role recursively normalizes:

```text
{{ confluence_home }}
```

to:

```text
{{ confluence_user }}:{{ confluence_group }}
```

### Important Safety Behavior

The current corrected configuration workflow should **not remove**:

```text
{{ confluence_home }}/confluence.cfg.xml
```

on every run.

Confluence creates and manages this file during application bootstrap.

Removing it after Confluence has started would reset or damage the application setup lifecycle.

---

## PostgreSQL JDBC Driver

Current tested JDBC driver:

```text
postgresql-42.7.4.jar
```

Configured by:

```yaml
postgresql_jdbc_version: "42.7.4"

postgresql_jdbc_jar: "postgresql-{{ postgresql_jdbc_version }}.jar"
```

The configuration task uses:

```yaml
src: "{{ postgresql_jdbc_jar }}"
dest: "{{ confluence_install_dir }}/confluence/WEB-INF/lib/{{ postgresql_jdbc_jar }}"
```

The source file must therefore exist under:

```text
roles/confluence/files/
```

unless the role is later changed to obtain it from another controlled source.

---

## Systemd Ownership Model

Both installation methods converge on:

```text
roles/confluence/templates/confluence.service.j2
```

and:

```text
roles/confluence/tasks/systemd.yml
```

The model is:

```text
Archive
   |
   +--> Ansible configuration
   |
   +--> Ansible systemd

Binary installer
   |
   +--> Install4j installs files only
   |
   +--> Ansible configuration
   |
   +--> Ansible systemd
```

The Atlassian installer is explicitly configured not to own service creation.

This prevents mixed ownership between:

```text
Install4j service management
```

and:

```text
Ansible systemd management
```

The tested service state:

```text
● confluence.service - Atlassian Confluence Data Center
Loaded: loaded
Active: active (running)
```

with the Java process running from:

```text
/usr/lib/jvm/java-21-openjdk...
```

and application paths under:

```text
/app/confluence-10.2.15
```

---

## Initial Setup and License Boundary

Confluence installation and Confluence application initialization are different states.

Immediately after successful installation, the generated configuration contained:

```xml
<setupStep>setupstart</setupStep>
<setupType>initial</setupType>
<buildNumber>0</buildNumber>
```

and HTTP redirected to:

```text
/bootstrap/selectsetupstep.action
```

This means:

```text
OS installation        = complete
Confluence service     = running
HTTP listener          = running
Application bootstrap  = pending
License/setup wizard   = pending
Cluster bootstrap      = not yet allowed
```

The role correctly treats this as a valid post-install state.

This is especially useful when licensing cannot be completed during the automation window.

The deployment can stop safely at:

```text
Confluence installation completed
Next: Complete setup at http://<host>:8090
Then run: ansible-playbook playbooks/bootstrap_confluence.yml
```

---

## Cluster Bootstrap Model

The cluster template is:

```text
roles/confluence/templates/cluster.properties.j2
```

Current structure:

```properties
confluence.cluster=true

confluence.cluster.name={{ confluence_cluster_name }}

confluence.cluster.node.name={{ confluence_cluster_node_id }}

confluence.cluster.peers=

confluence.cluster.join.type=tcp_ip

confluence.cluster.interface=0.0.0.0

confluence.cluster.home={{ confluence_shared_home }}
```

The bootstrap task must not rely solely on:

```text
confluence.cfg.xml exists
```

because Confluence creates this file before setup completion.

The initialization logic must inspect its content.

An initial state such as:

```xml
<setupStep>setupstart</setupStep>
<setupType>initial</setupType>
<buildNumber>0</buildNumber>
```

must be treated as:

```text
initialized = false
```

The tested bootstrap correctly failed with a controlled message indicating that initial setup must be completed first and that `cluster.properties` had not been deployed.

This is the expected safety behavior.

---

## Shared Home

Configured shared home:

```yaml
confluence_shared_home: "/app/confluence-shared-home"
```

During initial startup, Confluence itself also created:

```text
/app/confluence-data/shared-home
```

with content including runtime security files.

These paths should not automatically be assumed to be interchangeable.

The Ansible-configured Data Center shared-home path is:

```text
/app/confluence-shared-home
```

and is the path referenced by the cluster template.

For a real multi-node Data Center deployment, ensure the configured shared home is backed by storage suitable for all Confluence nodes and mounted consistently on each node.

Do not delete Confluence-generated runtime security content without understanding its purpose.

---

## Idempotency

The intended steady-state behavior is:

- installation media is not repeatedly reinstalled;
- installation directories are preserved when complete;
- configuration templates change only when required;
- Confluence Home ownership remains correct;
- systemd remains enabled and active;
- cluster bootstrap is not repeatedly forced before setup;
- runtime-managed Confluence configuration is preserved.

A key idempotency rule is:

```text
Do not delete confluence.cfg.xml on every configure run.
```

Another is:

```text
Do not let the .bin installer remain authoritative for runtime user or service management.
```

The role should converge both installation methods into the same managed state.

---

## Uninstall Behavior

The uninstall workflow should remove the Ansible-managed application according to configured safety flags.

Typical removal scope:

```text
systemd service
installation directory
Confluence Home
shared home
OS user/group
temporary installation files
```

The PostgreSQL database should remain untouched unless a separate explicitly controlled database-removal operation is implemented.

This mirrors the safety model used for the Jira automation.

Before uninstalling production systems, back up:

```text
Confluence Home
Data Center shared home
PostgreSQL database
application attachments
configuration files
```

Review all deletion flags before running the playbook.

---

## Tags Reference

### Precheck

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags precheck
```

### Prerequisites

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags prerequisites
```

### Install

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags install
```

### Configure

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags configure
```

### Systemd

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags systemd
```

### Validate

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  --tags validate
```

For a clean host, prefer the complete installation playbook rather than starting with a later-stage tag.

---

## Validation

Useful post-install checks:

```bash
echo "=== SERVICE ==="
systemctl status confluence --no-pager -l

echo
echo "=== PROCESS ==="
ps -ef | grep '[c]onfluence'

echo
echo "=== PORT ==="
ss -lntp | grep ':8090'

echo
echo "=== HTTP ==="
curl -sS -I --max-time 10 \
  http://localhost:8090/ | head

echo
echo "=== INSTALL ==="
ls -ld /app/confluence-10.2.15

echo
echo "=== HOME ==="
ls -ld /app/confluence-data

echo
echo "=== RUNTIME USER ==="
cat /app/confluence-10.2.15/bin/user.sh
```

The tested healthy runtime showed:

```text
service     : active (running)
port        : 8090 listening
runtime user: confluence
install     : confluence:confluence
home        : confluence:confluence
```

Before setup completion, HTTP is expected to redirect to the bootstrap wizard.

---

## Troubleshooting

### Binary installer checksum failure

Run:

```bash
sha256sum \
  roles/confluence/files/atlassian-confluence-10.2.15-x64.bin
```

Expected tested value:

```text
ec0f36564847920355aef0a67d467e5de2229ea23eb4b378240d703fdbb9aacb
```

Compare with:

```yaml
confluence_installer_sha256
```

---

### Archive checksum failure

Run:

```bash
sha256sum \
  roles/confluence/files/atlassian-confluence-10.2.15.tar.gz
```

Expected tested value:

```text
47689cb9cda55a00d34ead5dffa62f09cc74911db1718289ac78117e9da21479
```

---

### Installer asks interactive questions

The binary installer must be executed with the generated Install4j response file.

A manually typed input sequence is useful only for discovering the correct response variables.

Production Ansible execution should use unattended mode.

---

### Installer creates `confluence1` or `confluence2`

Check:

```bash
getent passwd | grep '^confluence'
getent group | grep '^confluence'
```

and:

```bash
cat /app/confluence-10.2.15/bin/user.sh
```

The desired runtime user is the configured:

```yaml
confluence_user: confluence
```

Expected:

```bash
CONF_USER="confluence"
```

Do not redesign the service around an accidental installer-created account.

Normalize the installer result.

---

### PID permission denied

Symptom:

```text
/app/confluence-10.2.15/bin/catalina.sh:
... /app/confluence-data/confluence.pid: Permission denied
```

Check:

```bash
stat -c '%U:%G %a %n' \
  /app/confluence-10.2.15 \
  /app/confluence-data
```

Expected:

```text
confluence:confluence
```

Fix ownership through Ansible rather than manual-only correction.

The current `configure.yml` recursively enforces Confluence Home ownership.

---

### PID file missing

Check:

```bash
ls -l /app/confluence-data/confluence.pid
```

If ownership is correct and Confluence is running, the PID file should exist.

Tested:

```text
-rw-r----- confluence confluence /app/confluence-data/confluence.pid
```

---

### Service starts but HTTP returns 302

A `302` is not automatically an error.

Before application setup completion:

```text
Location: /bootstrap/selectsetupstep.action
```

is expected.

Use:

```bash
curl -sS -I http://localhost:8090/ | head -15
```

---

### HTTP redirects to `/errors.jsp`

During early startup, Confluence may temporarily return:

```text
Location: /errors.jsp
```

Check service logs and wait for application initialization.

Useful commands:

```bash
systemctl status confluence --no-pager -l

journalctl \
  -u confluence \
  -n 100 \
  --no-pager \
  -o cat
```

Application logs:

```bash
tail -100 \
  /app/confluence-data/logs/atlassian-confluence.log
```

---

### Bootstrap fails saying setup is incomplete

This is expected when:

```xml
<setupStep>setupstart</setupStep>
<setupType>initial</setupType>
<buildNumber>0</buildNumber>
```

Complete the browser Setup Wizard and license step first.

Then rerun:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/bootstrap_confluence.yml
```

Do not bypass the guard by manually creating `cluster.properties`.

---

### `cluster.properties` is missing

Before setup completion, this is expected.

Check:

```bash
ls -l \
  /app/confluence-data/cluster.properties
```

The bootstrap workflow intentionally does not deploy it while Confluence is uninitialized.

---

### PostgreSQL is unreachable

Check:

```bash
nc -vz localhost 15432
```

or:

```bash
psql \
  -h localhost \
  -p 15432 \
  -U confluence \
  -d confluence
```

Review:

- PostgreSQL listener;
- firewall;
- `pg_hba.conf`;
- credentials;
- database existence.

---

### Java Home validation fails

Check:

```bash
which java
readlink -f "$(which java)"
java -version
```

Then:

```bash
JAVA_BIN=$(readlink -f "$(which java)")
JAVA_HOME=$(dirname "$(dirname "$JAVA_BIN")")

echo "$JAVA_HOME"

ls -l "$JAVA_HOME/bin/javac"
```

The role expects a JDK.

---

### Check selected installation method

```bash
grep -E \
  'confluence_install_method|confluence_archive:|confluence_installer:' \
  inventory/group_vars/confluence.yml
```

Example:

```text
confluence_install_method: "installer"
confluence_archive: "atlassian-confluence-{{ confluence_version }}.tar.gz"
confluence_installer: "atlassian-confluence-{{ confluence_version }}-x64.bin"
```

---

### Check Confluence initialization state

```bash
grep -E \
  '<setupStep>|<setupType>|<buildNumber>|hibernate.connection.url|hibernate.connection.username' \
  /app/confluence-data/confluence.cfg.xml
```

Initial state observed during testing:

```text
<setupStep>setupstart</setupStep>
<setupType>initial</setupType>
<buildNumber>0</buildNumber>
```

This is not yet cluster-bootstrap-ready.

---

### Service troubleshooting

```bash
systemctl status \
  confluence \
  --no-pager \
  -l
```

```bash
journalctl \
  -u confluence \
  -n 100 \
  --no-pager \
  -o cat
```

```bash
ps -ef | grep '[c]onfluence'
```

```bash
ss -lntp | grep ':8090'
```

---

### Verbose Ansible troubleshooting

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/install_confluence.yml \
  -vvv
```

---

## Security Recommendations

- Store PostgreSQL passwords in Ansible Vault.
- Do not commit production credentials.
- Verify Atlassian installation media checksums.
- Restrict access to inventory files containing secrets.
- Do not commit licensed installation media to public Git repositories.
- Back up Confluence Home before destructive operations.
- Back up Data Center shared home.
- Back up PostgreSQL before uninstall or upgrade.
- Preserve Confluence-generated secure configuration.
- Do not casually remove `secrets-config.yaml`.
- Do not overwrite runtime security material generated under Confluence Home.
- Use unique cluster node IDs.
- Restrict database access to required Confluence nodes.
- Review firewall rules for HTTP, database, and cluster traffic.
- Test new Confluence releases and installer changes in non-production first.

---

## Operational Recommendations

### Keep installation and bootstrap separate

Recommended:

```text
install_confluence.yml
        |
        v
manual/application setup boundary
        |
        v
bootstrap_confluence.yml
```

Do not merge cluster bootstrap into the first installation unless the entire Setup Wizard and licensing process has also been safely automated.

### Keep Ansible authoritative

For both installation methods:

```text
Ansible owns:
- user
- group
- directories
- configuration
- JVM
- JDBC
- systemd
- validation
- uninstall
```

The Atlassian binary installer should only install the product files.

### Treat runtime files carefully

Confluence modifies and generates application state under:

```text
/app/confluence-data
```

Not every runtime-generated file should be continuously replaced by Ansible.

### Review shared storage before multi-node deployment

A local path is sufficient for single-node testing, but a production Data Center cluster requires a properly designed shared-home implementation accessible from every participating node.

---

## Tested Lifecycle

The Confluence 10.2.15 binary-installer implementation has been exercised through the following workflow:

```text
Clean application state
        |
        v
Verify .bin media
        |
        +--> SHA256 verified
        |
        v
Manual installer discovery
        |
        +--> response.varfile captured
        |
        v
Configure unattended Install4j response
        |
        v
Ansible precheck
        |
        +--> RHEL 9 OK
        +--> installer checksum OK
        +--> Java 21 OK
        +--> JDK OK
        +--> disk OK
        +--> PostgreSQL reachable
        |
        v
Ansible binary installation
        |
        v
Normalize runtime user
        |
        v
Normalize Confluence Home ownership
        |
        v
Deploy JVM/JDBC/configuration
        |
        v
Deploy Ansible-managed systemd
        |
        v
Start Confluence
        |
        +--> confluence.service active
        +--> Java process active
        +--> port 8090 listening
        +--> PID file created
        |
        v
HTTP validation
        |
        +--> HTTP 302
        +--> Setup endpoint available
        |
        v
Initial application state
        |
        +--> setupStep=setupstart
        +--> setupType=initial
        +--> buildNumber=0
        |
        v
Bootstrap attempted
        |
        +--> correctly blocked
        +--> cluster.properties NOT deployed
        |
        v
License / Setup Wizard boundary
        |
        +--> pending
        |
        v
After setup completion
        |
        +--> run bootstrap_confluence.yml
        |
        v
Deploy cluster.properties
        |
        v
Validate cluster
```

The archive installation remains available by changing only:

```yaml
confluence_install_method: "archive"
```

The architecture therefore provides one Confluence role supporting both installation approaches while maintaining a common:

```text
configuration
systemd
validation
setup boundary
cluster bootstrap
uninstall
```

model.

---

## Current Tested Status

At the end of the current test cycle:

```text
Confluence version     : 10.2.15
RHEL                   : 9
Java                   : OpenJDK 21
Install method         : installer
Install directory      : /app/confluence-10.2.15
Confluence Home        : /app/confluence-data
HTTP port              : 8090
PostgreSQL port        : 15432
Service                : active
Java process           : active
PID file               : present
HTTP setup endpoint    : reachable
Initial setup          : pending
License                : pending
Cluster bootstrap      : intentionally pending
cluster.properties     : not deployed yet
```

This is a valid stopping point until the Confluence license and browser Setup Wizard can be completed.
