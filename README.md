
# Personal Ansible Workstation Setup

Reproducible machine configuration using Ansible. Clone this repo on a fresh machine, run a few commands, and get a fully configured workstation.


## Quick Start (Fresh Machine)

```bash
# 1. Install prerequisites (using OS package manager)
# install git ansible python3

# 2. Clone and enter repo
git clone https://github.com/driossantini/personal-ansible
cd personal-ansible

# 3. Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# 4. Run baseline setup
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags base

# 5. Run profiles as needed
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags development,aur
```

## Common Operations

### Adding packages

| To install on... | Edit this file | Add to this variable |
|-----------------|----------------|---------------------|
| **All machines** | `group_vars/all.yml` | `pacman_packages_common` |
| **Machines in a profile** (e.g., `work`, `development`) | `group_vars/<profile>.yml` | `pacman_packages_<profile>` |
| **One specific machine** | `host_vars/<hostname>.yml` | `pacman_packages_extra` |
| **From AUR** (any scope) | Same files as above | `aur_packages_*` (instead of `pacman_packages_*`) |

After editing, run:
```bash
# For official repo packages
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags base

# For AUR packages
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags aur
```

**Example:** To add `slack-desktop` (AUR) to work machines only:
1. Edit `group_vars/work.yml`
2. Add `slack-desktop` to `aur_packages_work` list
3. Run with `--tags aur`

---

### Setting up a new machine

**1. Add the machine to inventory** (`inventories/local/hosts.yml`):

```yaml
all:
  hosts:
    new-laptop:
      ansible_connection: local  # or configure SSH
      ansible_python_interpreter: /usr/bin/python3

  children:
    # Assign to profiles
    development:
      hosts:
        new-laptop:
    
    work:
      hosts:
        new-laptop:
```

**2. (Optional) Create host-specific config** (`host_vars/new-laptop.yml`):

```yaml
pacman_packages_extra:
  - steam
  - discord

aur_packages_extra:
  - spotify
```

**3. Run the playbook:**

```bash
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags base,development,aur
```

---

### Updating the system

```bash
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags upgrade
```

---

### Creating a new profile

Profiles are groups that share common packages/configuration.

**1. Define the group in inventory** (`inventories/local/hosts.yml`):

```yaml
all:
  children:
    gaming:  # new profile
      hosts:
        pc:
```

**2. Create group variables** (`group_vars/gaming.yml`):

```yaml
pacman_packages_gaming:
  - steam
  - lutris
  - gamemode

aur_packages_gaming:
  - protonup-qt
```

**3. Add tagged task to role** (`roles/base/tasks/main.yml`):

```yaml
- name: Install gaming packages
  community.general.pacman:
    name: "{{ pacman_packages_gaming | default([]) }}"
    state: present
  when: "'gaming' in group_names"
  tags:
    - gaming
```

**4. Run with the new tag:**

```bash
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags gaming
```

> **Note:** Similar tasks for `aur_packages_gaming` need to be added in `roles/aur/tasks/main.yml` following the existing pattern.

---

## Available Profiles & Tags

### Current profiles (groups)
- `development` — Programming tools, IDEs, language runtimes
- `work` — Work-specific applications
- `nvidia` — NVIDIA drivers and utilities
- `kde` — KDE Plasma desktop environment

### Available tags
- `base` — Core system packages (always run this first)
- `development` — Development toolchain
- `work` — Work applications
- `nvidia` — NVIDIA packages
- `kde` — Desktop environment
- `aur` — AUR/foreign packages
- `upgrade` — System update

**Run multiple tags:**
```bash
ansible-playbook -i inventories/local/hosts.yml playbook.yml --ask-become-pass --tags base,development,aur
```

**List all tags:**
```bash
ansible-playbook -i inventories/local/hosts.yml playbook.yml --list-tags
```

---

## How It Works

### Repository structure

```
.
├── inventories/local/hosts.yml    # Machine definitions and group membership
├── group_vars/                    # Profile-specific packages
│   ├── all.yml                    # Packages for all WS
│   ├── development.yml            # Development profile
    ├── nvidia.yml                 # Profile for WS with nVidia hardware
│   └── work.yml                   # Work profile
├── host_vars/                     # Machine-specific packages
│   ├── laptop.yml
    └── pc.yml
├── roles/
│   ├── aur/                       # AUR packages
│   ├── base/                      # Official repo packages (pacman)
    └── dotfiles/                  # Config files for packages
├── ansible.cfg                    # Ansible configuration
├── playbook.yml                   # Main playbook
├── README.md                      # Readme with this explanation
└── requirements.yml               # Required Ansible collections
```

### Variable precedence (most specific wins)

1. `host_vars/<hostname>.yml` — Host-specific overrides
2. `group_vars/<group>.yml` — Profile/group configuration
3. `roles/*/defaults/main.yml` — Safe defaults

### Package list merging

Lists are **additive**. For example, packages installed on a machine in both `development` and `work` groups:

```
pacman_packages_common      (from group_vars/all.yml)
+ pacman_packages_development  (from group_vars/development.yml)
+ pacman_packages_work         (from group_vars/work.yml)
+ pacman_packages_extra        (from host_vars/<hostname>.yml)
= final unique package list
```

---

## Troubleshooting

### Package conflicts

  - Symptom: pacman reports conflicting packages (e.g., `tldr` vs `tealdeer`)
  - Solution: Remove one from the package lists so the desired state is unambiguous.


### AUR build failures

  - Symptom: AUR package fails to build
  - Cause: Upstream URL changes, build dependency issues
  - Solution:
    1. Remove the failing package from `aur_packages_*` lists
    2. Find an alternative or fix the PKGBUILD
    3. Re-run with `--tags aur`


### Connectivity test

Verify Ansible can reach all hosts:

```bash
ansible -i inventories/local/hosts.yml all -m ping
```

Expected output: `"ping": "pong"` for each reachable host.

---

## Future Enhancements to Consider

- **Dotfile management** — Symlink configuration files
- **Service management** — Enable/start systemd services
- **User configuration** — Non-root user setup, shell configuration
- **Secrets management** — ansible-vault for sensitive data
- **Remote hosts** — Expand beyond local-only execution
- **Idempotent removal** — Tasks to uninstall packages from specific profiles

