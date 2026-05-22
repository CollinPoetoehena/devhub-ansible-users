# users

> Part of [dev-hub/Ansible](https://github.com/CollinPoetoehena/dev-hub/blob/main/Ansible.md) — see that file for conventions, structure guidelines, and the full role index.

Generic OS user, group, sudo, and SSH key management for Linux hosts. Designed to be reused across any host type (mgmtvm, jumphost, workload VMs, etc.).

## Background

On Linux, every process runs under a user identity. Users belong to one primary group and optionally supplementary groups. Access control — including `sudo` — is based on these identities.

Key concepts:
- **`/etc/passwd`** — stores user accounts (username, UID, home directory, shell, GECOS comment)
- **`/etc/group`** — stores group definitions and membership
- **`/etc/sudoers.d/`** — drop-in directory for sudo policy files; `visudo` validates syntax before any file is installed
- **`~/.ssh/authorized_keys`** — public keys whose holders may log in as this user
- **`~/.ssh/<keyfile>`** — private keys this user presents when initiating outbound SSH connections

For more detail see [GeeksforGeeks: User Management in Linux](https://www.geeksforgeeks.org/linux-unix/user-management-in-linux/) and [Red Hat Documentation: Managing Users and Groups](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/managing-users-and-groups_configuring-basic-system-settings.html), as well as the Linux man pages: [`useradd(8)`](https://man7.org/linux/man-pages/man8/useradd.8.html), [`sudoers(5)`](https://man7.org/linux/man-pages/man5/sudoers.5.html), [`sshd(8)`](https://man7.org/linux/man-pages/man8/sshd.8.html).

## Requirements

- **OS**: Any Linux distribution (RHEL/EL, Debian/Ubuntu, etc.), since user and group management is handled using standard Linux tools.
- **Ansible**: 2.14+
- **Collections**: `ansible.posix` — required for the SSH public-key authorisation task.  
  Install with:
  ```sh
  ansible-galaxy collection install ansible.posix
  ```

## Features

| Feature | Description |
|---------|-------------|
| Group management | Create OS groups; optionally grant passwordless sudo to all members |
| User management | Create OS users with home directory and primary group |
| Sudo management | Per-group and per-user `/etc/sudoers.d/` files; cleans up revoked grants |
| Private key deployment | Deploy one or more private key files to a user's `~/.ssh/` |
| Public key authorisation | Add one or more public keys to a user's `~/.ssh/authorized_keys` |

## Variables

### Groups

| Variable | Default | Description |
|----------|---------|-------------|
| `users_groups` | `[]` | List of groups to create. See format below. |

Each entry in `users_groups`:

| Field | Required | Description |
|-------|----------|-------------|
| `groupname` | yes | OS group name |
| `use_sudo` | no (default `false`) | Grant passwordless sudo to all group members |

### Users

| Variable | Default | Description |
|----------|---------|-------------|
| `users_list` | `[]` | List of OS users to create. See format below. |

Each entry in `users_list`:

| Field | Required | Description |
|-------|----------|-------------|
| `username` | yes | OS username |
| `group` | yes | Primary group name; automatically created if not in `users_groups` |
| `desc` | no | GECOS comment stored in `/etc/passwd` |
| `use_sudo` | no (default `false`) | Grant passwordless sudo |
| `ssh_private_keys` | no | List of private keys to deploy to `~/.ssh/`. Each entry: `{ src, dest }` |
| `ssh_public_keys` | no | List of public keys to authorise for login. Each entry: `{ key, comment }` |

#### `ssh_private_keys` entry fields

| Field | Required | Description |
|-------|----------|-------------|
| `src` | yes | Path on the Ansible controller to the private key file. NOTE: Private keys are sensitive so only the src path is used (instead of the full content) for security. |
| `dest` | yes | Filename (placed in `~/.ssh/`) or absolute path on the target host |

#### `ssh_public_keys` entry fields

| Field | Required | Description |
|-------|----------|-------------|
| `key` | yes | Full public key string (e.g. `ssh-rsa AAAA...`) |
| `comment` | no | Human-readable label added alongside the key in `authorized_keys` |

## Sudo behaviour

Two strategies are applied, chosen automatically per user:

1. **Group-level sudo** — a single `/etc/sudoers.d/group_<name>` file covers every member of the group. Used when `use_sudo: true` on a `users_groups` entry.
2. **User-level sudo** — a per-user `/etc/sudoers.d/user_<name>` file is created only when `use_sudo: true` on a user *and* the user's primary group does **not** already have sudo. This avoids redundant files.

Stale files are **removed** automatically to avoid orphaned sudo grants:
- Group sudoers file is removed when `use_sudo` is set to `false`.
- User sudoers file is removed when `use_sudo` is `false`, or when the user's group was granted sudo (making the per-user file redundant).

## Format

```yaml
users_groups:
  - groupname: as-admin
    use_sudo: true          # all members get passwordless sudo via group file

users_list:
  - username: ansibleremote
    desc: "Ansible service account"
    group: ansibleremote    # own group; use_sudo: true → per-user sudoers file
    use_sudo: true
    ssh_private_keys:
      - src: "{{ playbook_dir }}/keys/workload_id_rsa"
        dest: "workload_id_rsa"   # placed at /home/ansibleremote/.ssh/workload_id_rsa
      - src: "{{ playbook_dir }}/keys/backup_id_rsa"
        dest: "/opt/keys/backup_id_rsa"   # absolute path
    ssh_public_keys:
      - key: "ssh-rsa AAAA..."
        comment: "laptop key"

  - username: poetoec
    desc: "Collin Poetoëhena"
    group: as-admin         # sudo inherited from group; no per-user sudoers file added
    use_sudo: true

  - username: addmuser
    desc: "ADDM Scan User"
    group: addmuser
    use_sudo: false
```

> **Note on public vs private keys:**  
> - `ssh_public_keys` authorises who can **log in as** this user (written to `authorized_keys`).  
> - `ssh_private_keys` deploys keys this user will use to **connect elsewhere** (e.g. from a mgmtvm to workload VMs).

## Usage

### Requirements file

```yaml
---
roles:
  - name: users
    src: https://github.com/CollinPoetoehena/ansible-role-users.git
    scm: git
    version: 1.0.0
```

Then install with:

```sh
# NOTE: Example of roles path for -p is "roles/" (you can also specify this in ansible.cfg)
ansible-galaxy install -r requirements.yml -p <path/to/roles>
```

### Example playbook

```yaml
---
- hosts: mgmtvm
  become: true
  vars:
    users_groups:
      - groupname: as-admin
        use_sudo: true

    users_list:
      # Service account for Ansible to log in as; also gets a private key so it can SSH to workload VMs, etc. 
      # Can be used for CI/CD pipeline access; no interactive login needed, so own group and sudo via per-user file to avoid granting sudo to any other users that might share a group.
      - username: ansibleremote
        desc: "Ansible service account"
        group: ansibleremote
        use_sudo: true
        # Deploy a private key so this user can SSH to workload VMs
        ssh_private_keys:
          - src: "{{ playbook_dir }}/keys/workload_id_rsa"
            dest: "workload_id_rsa"
        # Authorise a public key so CI/CD can log in as ansibleremote
        ssh_public_keys:
          - key: "ssh-rsa AAAA..."
            comment: "ci-cd pipeline key"
      # Specific for a personal account, such as for an administrator, etc.
      - username: poetoec
        desc: "Collin Poetoëhena"
        group: as-admin
        use_sudo: true
      # Account for external monitoring tool; no sudo or SSH access needed since it only needs to run a local agent that doesn't connect anywhere or require elevated permissions.
      - username: addmuser
        desc: "ADDM Scan User"
        group: addmuser
        use_sudo: false

  roles:
    - role: users
```

### Selective execution with tags

```sh
# Only manage groups
ansible-playbook site.yml --tags groups

# Only manage sudo
ansible-playbook site.yml --tags sudo

# Only configure SSH keys
ansible-playbook site.yml --tags ssh

# Everything
ansible-playbook site.yml --tags users
```
