# Azure Ansible Inventory Hierarchy

This project demonstrates how to manage Azure VMs with Ansible and organise them into a group hierarchy using the Azure dynamic inventory plugin together with a static group definition.

## Overview

The dynamic inventory plugin (`azure.azcollection.azure_rm`) discovers Azure VMs and places each host into a group based on its `ansible_group` tag. A separate static YAML file defines the parent-child relationships between those groups. When both files live in the same `inventory/` directory, Ansible merges them into a single inventory tree.

```
all
  linux
    web
      web01
      web02
    db
      db01
```

## Repository structure

```
.
├── ansible.cfg                              # Points Ansible at the inventory/ directory
├── requirements.yml                         # Collection dependencies
├── vars/
│   └── azure.yml                            # Configurable variables (edit before running)
├── inventory/
│   ├── azure_rm.yml                         # Azure dynamic inventory plugin config
│   ├── group_hierarchy.yml                  # Static group parent-child definitions
│   └── group_vars/
│       └── linux.yml                        # Variables applied to the linux group
└── playbooks/
    ├── deploy_vms.yml                       # Provisions RHEL 10 VMs on Azure
    └── inventory_configuration.yml          # Configures the project, inventory & sources on Automation Controller
```

## Prerequisites

- Python 3.9+
- Ansible Core 2.15+
- An Azure subscription with an existing virtual network and subnet
- Azure CLI logged in (`az login`) or equivalent credentials configured
- An SSH key pair

Install the required collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Configuration

Edit `vars/azure.yml` before running any playbook. The file contains placeholders for all environment-specific values:

| Variable | Description |
|---|---|
| `azure_location` | Azure region (e.g. `eastus`) |
| `azure_resource_group` | Resource group for the VMs |
| `azure_network_resource_group` | Resource group containing the existing VNet |
| `azure_vnet_name` | Name of the existing virtual network |
| `azure_subnet_name` | Name of the existing subnet |
| `azure_ssh_public_key_file` | Absolute path to an existing SSH public key |

Default values that normally don't need changing:

| Variable | Default | Description |
|---|---|---|
| `azure_admin_username` | `cloud-user` | VM admin user |
| `azure_vm_size` | `Standard_B2s` | VM size (Gen2-capable) |
| `azure_image_sku` | `10-lvm-gen2` | RHEL 10 LVM Gen2 image |

## Playbooks

### Deploy VMs

Provisions two web servers and one database server on Azure, each running RHEL 10 with a static public IP and SSH-key-only authentication:

```bash
ansible-playbook playbooks/deploy_vms.yml -e @vars/azure.yml
```

The playbook creates three VMs in a loop. Each VM is tagged with `ansible_group` set to either `web` or `db`, which the dynamic inventory plugin uses to assign group membership.

### Configure Automation Controller

Sets up a project, inventory, and three inventory sources on Ansible Automation Platform Controller:

```bash
ansible-playbook playbooks/inventory_configuration.yml
```

Controller connection details (`controller_host`, `controller_username`, `controller_password` or `controller_oauthtoken`) should be provided via environment variables or a `~/.controller_cli.cfg` file.

The playbook creates:

1. **Project** -- points at this Git repository so Controller can sync the inventory files.
2. **Inventory** -- a Controller inventory to hold the hosts and groups.
3. **Three inventory sources** (all SCM-based, synced from the project):

| Source name | `source_path` | Purpose |
|---|---|---|
| Azure RM | `inventory/azure_rm.yml` | Discovers VMs and assigns groups from Azure tags |
| Group Hierarchy | `inventory/group_hierarchy.yml` | Defines parent-child group relationships |
| Combined | `inventory/` | Processes the entire directory, merging both sources |

All sources are set to `overwrite: true`, `update_on_launch: true`, and `verbosity: 2` (debug) for detailed sync output.

## How the inventory works

The key idea is separating **host discovery** from **group hierarchy**:

- `inventory/azure_rm.yml` -- the Azure RM plugin discovers hosts and creates flat groups based on the `ansible_group` tag on each VM (e.g. `web`, `db`).
- `inventory/group_hierarchy.yml` -- a static YAML file that nests those groups under parent groups (e.g. `web` and `db` are children of `linux`).
- `inventory/group_vars/linux.yml` -- variables inherited by all hosts in the `linux` group and its children (sets `ansible_user: cloud-user`).

When Ansible processes the `inventory/` directory, it merges all sources together. The result is a hierarchy where hosts discovered from Azure inherit both their tag-based group and any parent groups defined statically.

## License

See [LICENSE](LICENSE) for details.
