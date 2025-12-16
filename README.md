<img src="img/header.jpg?raw=true" alt="Header Logo" width="100%" />
<br /><br />

# Ansible Role: **disk_partition**
[![Style: ansible-lint](https://img.shields.io/badge/style-ansible--lint-green)](#lint)
[![Tests: molecule](https://img.shields.io/badge/tests-molecule-blue)](#testing)
[![License: MIT](https://img.shields.io/badge/license-MIT-informational)](LICENSE)

Ansible role to **safely and idempotently prepare and mount Linux block devices**:

- Detects the current disk partition table label (`gpt` or `msdos`) and creates it if needed.
- Creates partitions with `community.general.parted` using ranges (`0%`, `25%`, `GiB`, etc.).
- Formats partitions with the specified filesystem (default `ext4`).
- Mounts and **persists** mounts in `/etc/fstab` with `ansible.posix.mount`.
- Includes **optional data migration** from an existing path (e.g., `/var/log`) with `rsync`.
- Optional hooks to **stop/start services** during migrations (journald, rsyslog, …).
- Supports `sdX` and `nvmeXnY` devices (automatic handling of the `p` suffix for NVMe/mmcblk).

> **Important**
> This role is designed for infrastructure/provisioning pipelines. Use with care: partitioning/formatting is inherently destructive if pointed at the wrong device.

---

## Table of Contents
- [Repository layout](#repository-layout)
- [Compatibility & requirements](#compatibility--requirements)
- [Installation](#installation)
  - [Recommended: separate requirements files](#recommended-separate-requirements-files)
  - [Alternative: install directly from Git](#alternative-install-directly-from-git)
  - [Install from Ansible Galaxy](#install-from-ansible-galaxy)
- [Inventory & useful commands](#inventory--useful-commands)
- [Variables](#variables)
  - [Structure of `disk_devices`](#structure-of-disk_devices)
  - [Service hints by path](#service-hints-by-path)
  - [Control variables & defaults](#control-variables--defaults)
- [Usage examples](#usage-examples)
  - [Recommended pattern: vars in host_vars/group_vars](#recommended-pattern-vars-in-host_varsgroup_vars)
  - [Inline vars (acceptable for one-off runs)](#inline-vars-acceptable-for-one-off-runs)
- [Tags](#tags)
- [Idempotence & safety](#idempotence--safety)
- [Troubleshooting](#troubleshooting)
- [Testing](#testing)
- [Test matrix / verified environments](#test-matrix--verified-environments)
- [License & author](#license--author)

---

## Repository layout

Below is the **project layout** used in the DevOps monorepo. It shows where inventories, playbooks and this role live, so the command examples make sense.

```
config-mgmt/ansible
.
├── ansible.cfg
├── .ansible
│   ├── collections/
│   │   └── ansible_collections/
│   └── roles/
│       └── jhonnygo.disk_partition/
│           ├── CHANGELOG
│           ├── LICENSE
│           ├── README.md
│           ├── defaults/
│           ├── files/
│           ├── filter_plugins/
│           ├── handlers/
│           ├── library/
│           ├── meta/
│           ├── module_utils/
│           ├── tasks/
│           │   ├── _per_device.yml
│           │   ├── _per_partition.yml
│           │   └── main.yml
│           ├── templates/
│           └── tests/
│
├── inventories/
│   ├── aws/
│   │   ├── dev/
│   │   │   ├── aws_ec2.yml
│   │   │   ├── group_vars/
│   │   │   │   ├── all.yml
│   │   │   │   ├── role_app.yml
│   │   │   │   └── role_db.yml
│   │   │   └── host_vars/
│   │   │       └── vm-dev-01.yml
│   │   └── stage/ ...
│   └── gcp/
│       ├── dev/
│       │   ├── gcp_compute.yml
│       │   ├── group_vars/
│       │   │   ├── all.yml
│       │   │   ├── role_app.yml
│       │   │   └── role_db.yml
│       │   └── host_vars/
│       │       └── vm-dev-01.yml
│       └── stage/ ...
│
├── playbooks/
│   ├── disk-partition.yml
│   └── ping.yml
└── requirements/
    ├── collections.yml
    └── roles.yml
```

### WSL note about `ansible.cfg` being ignored

If you run from `/mnt/c/...` you may see:

> `Ansible is being run in a world writable directory ..., ignoring it as an ansible.cfg source`

**Professional recommendation** (pick one):

1) Run the repo from the Linux filesystem (e.g., `~/workspace/...`) instead of `/mnt/c/...`, **or**  
2) Make the directory non-world-writable, **or**  
3) Set `ANSIBLE_CONFIG=/path/to/ansible.cfg` explicitly for runs.

---

## Compatibility & requirements

**OS**: Debian/Ubuntu (and derivatives).  
**Collections**:
- `ansible.posix`
- `community.general`

**Target packages** (installed automatically when supported):
- `parted`, `e2fsprogs`, `rsync` (rsync only used when migrations are configured)

---

## Installation

### Recommended: separate requirements files

Keep roles and collections in **separate** requirement files. This avoids ambiguity and matches what most production repos do.

**`requirements/collections.yml`**
```yaml
---
collections:
  - name: ansible.posix
  - name: community.general
```

**`requirements/roles.yml`**
```yaml
---
roles:
  - name: jhonnygo.disk_partition
    version: "0.1.0"   # pin to a released version
```

Install:

```bash
ansible-galaxy collection install -r requirements/collections.yml
ansible-galaxy role install -r requirements/roles.yml -p .ansible/roles
```

> Tip: `-p .ansible/roles` ensures the role is installed where your repo expects it.

### Alternative: install directly from Git

```bash
ansible-galaxy role install \
  git+https://github.com/jhonnygo-org/ansible-role-disk-partition.git \
  -p .ansible/roles
```

### Install from Ansible Galaxy

```bash
ansible-galaxy role install jhonnygo.disk_partition -p .ansible/roles
```

---

## Inventory & useful commands

### Inspect the inventory tree (graph)

AWS example:
```bash
ansible-inventory -i inventories/aws/dev/aws_ec2.yml --graph
```

GCP example:
```bash
ansible-inventory -i inventories/gcp/dev/gcp_compute.yml --graph
```

### Run the playbook (dry-run / check mode)

```bash
ansible-playbook -i inventories/gcp/dev/gcp_compute.yml playbooks/disk.yml \
  -l vm-dev-01 --tags disk --check --diff
```

### Real execution

```bash
ansible-playbook -i inventories/gcp/dev/gcp_compute.yml playbooks/disk.yml \
  -l vm-dev-01 --tags disk
```

---

## Variables

### Structure of `disk_devices`

Define each disk and its partitions. The role iterates over `disk_devices` and applies the layout idempotently.

```yaml
disk_devices:
  - device: /dev/sdb          # GCP typical extra disk
    label: gpt                # optional; gpt | msdos. defaults to disk_label_default
    parts:
      - number: 1
        start: "0%"
        end:   "50%"
        fs: ext4              # optional; defaults to disk_fs_default
        mkfs_opts: ""         # optional; e.g. "-F"
        mount:
          path: /var/log/app
          opts: defaults
          create_mode: "0755"
          create_owner: root
          create_group: root
          migrate_from: ""    # optional; when set, rsync data from this path
```

**Notes**
- For NVMe/mmcblk devices, partitions are `/dev/nvme1n1p1` (needs `p`).
- For `sdX` devices, partitions are `/dev/sdb1` (no `p`).
- The role handles this automatically.

### Service hints by path

When `migrate_from` is set, you may want to stop services that write to that path.

```yaml
disk_service_hints:
  "/var/log":
    stop:  [rsyslog, systemd-journald]
    start: [systemd-journald, rsyslog]
```

### Control variables & defaults

```yaml
disk_devices: []              # Required input (empty means "do nothing")
disk_label_default: gpt       # Default if a device has no label set
disk_fs_default: ext4         # Default FS if a partition has no fs set
disk_remove_lostfound: true   # Cosmetic cleanup for ext* filesystems
disk_debug: false             # Extra debug output
```

---

## Usage examples

### Recommended pattern: vars in host_vars/group_vars

This is the **most professional** approach:
- Playbooks remain thin and reusable.
- Environment-specific layouts live in inventory vars.
- You can apply different disk layouts per group or per host.

#### 1) Put disk layout under inventory vars

**If it is a single host** (e.g., `vm-dev-01`):

`inventories/gcp/dev/host_vars/vm-dev-01.yml`
```yaml
---
disk_fs_default: ext4
disk_remove_lostfound: true

disk_devices:
  - device: /dev/sdb
    label: gpt
    parts:
      - number: 1
        start: "0%"
        end: "50%"
        fs: ext4
        mount:
          path: /var/log/app
          opts: defaults
          create_mode: "0755"
      - number: 2
        start: "50%"
        end: "100%"
        fs: ext4
        mount:
          path: /var/log
          opts: defaults
          create_mode: "0755"
          migrate_from: /var/log

  - device: /dev/sdc
    label: gpt
    parts:
      - number: 1
        start: "0%"
        end: "100%"
        fs: ext4
        mount:
          path: /var/serv/app
          opts: defaults
          create_mode: "0755"

  - device: /dev/sdd
    label: gpt
    parts:
      - number: 1
        start: "0%"
        end: "100%"
        fs: ext4
        mount:
          path: /var/test/app
          opts: defaults
          create_mode: "0755"
```

**If it is a group** (e.g., all app servers):

`inventories/gcp/dev/group_vars/role_app.yml`
```yaml
---
disk_devices:
  - device: /dev/sdb
    label: gpt
    parts:
      - number: 1
        start: "0%"
        end: "100%"
        fs: ext4
        mount:
          path: /var/lib/app
          opts: defaults
```

> Rule of thumb:
> - `host_vars/<host>.yml` when layout is unique per instance.
> - `group_vars/<group>.yml` when layout is shared across a fleet.

#### 2) Keep the playbook thin

`playbooks/disk.yml`
```yaml
---
- name: Prepare disks (partition/format/mount)
  hosts: all
  become: true
  roles:
    - role: jhonnygo.disk_partition
      tags: [disk]
```

#### 3) Apply

Dry-run:
```bash
ansible-playbook -i inventories/gcp/dev/gcp_compute.yml playbooks/disk.yml \
  -l vm-dev-01 --check --diff
```

Real apply:
```bash
ansible-playbook -i inventories/gcp/dev/gcp_compute.yml playbooks/disk.yml \
  -l vm-dev-01
```

### Inline vars (acceptable for one-off runs)

Use only when prototyping or debugging.

```yaml
- hosts: vm-dev-01
  become: true
  roles:
    - role: jhonnygo.disk_partition
      vars:
        disk_devices:
          - device: /dev/sdb
            label: gpt
            parts:
              - { number: 1, start: "0%",  end: "100%", fs: ext4, mount: { path: /data, opts: defaults } }
```

---

## Tags

- `disk` — general role tag
- `disk:probe` — detections + kernel notifications (`lsblk`, `partprobe`, `udevadm`)
- `disk:mklabel` — disk label creation
- `disk:parted` — partition creation
- `disk:wait` — waits for partition nodes
- `disk:partition` — formatting, mounting, fstab, migrations
- `debug` — extra output (effective when `disk_debug: true`)

Example:
```bash
ansible-playbook -i inventories/gcp/dev/gcp_compute.yml playbooks/disk.yml \
  -l vm-dev-01 -t "disk:parted,disk:partition"
```

---

## Idempotence & safety

- `mkfs` is not forced if the filesystem already exists (idempotent).
- Mounting is done with `state: mounted` and persisted in `/etc/fstab`.
- If `migrate_from` is set, migration uses `rsync` and can stop/start services defined in `disk_service_hints`.
- After partitioning changes, the role runs `partprobe` and `udevadm settle` to synchronize kernel/udev.
- In check mode (`--check`), destructive tasks should be guarded to avoid modifying disks.

---

## Troubleshooting

### I can only reach hosts by IP, not by instance name
That is normal when:
- Dynamic inventory does not expose `name` as the `inventory_hostname`, or
- You do not configure `hostnames` options in the GCP inventory plugin.

Professional approach:
- Keep `inventory_hostname` stable (VM name is ideal),
- Use `ansible_host` to store the IP/FQDN used for SSH.

If your dynamic inventory currently shows only IPs, you can still:
- Limit by IP: `-l 35.x.x.x`
- Or adjust the inventory plugin to set `hostnames` / `compose` so `name` becomes the host key.

### WSL: `ansible.cfg` ignored
See [WSL note about ansible.cfg being ignored](#wsl-note-about-ansiblecfg-being-ignored).

### Partitions take time to appear
The role already runs `partprobe` and `udevadm settle` and waits for device nodes. If needed, increase retries/delays in the wait tasks.

---

## Testing

Molecule skeleton is included; adapt scenarios as needed.

```bash
pip install molecule molecule-plugins[docker] ansible-lint
molecule test
```

---

## Test matrix / verified environments

| Environment / Platform             | OS / Image           | Ansible | Result | Notes |
|-----------------------------------|----------------------|---------|--------|------|
| AWS EC2                           | Ubuntu 20.04/22.04   | 2.15+   | ✅ OK  | NVMe (`/dev/nvme...`) |
| GCP Compute Engine                | Ubuntu 22.04         | 2.15+   | ✅ OK  | SCSI (`/dev/sdb...`) |
| Docker (privileged)               | ubuntu:22.04         | 2.15+   | 🟡 Optional | Requires `--privileged` |

---

## License & author

**License:** MIT  
**Author:** Jhonny Alexander (JhonnyGO) — <support@jhoncytech.com>
