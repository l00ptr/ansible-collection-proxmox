# Ansible Collection - maxhoesel.proxmox

A collection for managing Proxmox VE and Backup Server hosts.

Currently contains the following components:

#### Roles

- lxc_container: Creates and bootstraps LXC containers for Ansible access
- lxc_container_to_ostemplate: Generates a ostemplate image from an existing container
- pve: Perform basic system-level setup tasks on PVE nodes (such as setting the PVE repository and configuring the root user)
- pbs: Perform basic system-level setup tasks on Proxmox Backup Server hosts (such as setting the PBS repository and configuring the root user + S.M.A.R.T)

#### Modules

- ha: Manage a hosts HA settings

# Requirements & Installation

Several roles in this collection require the `proxmoxer` and `requests` Python module.

- Installation with `pip` (venv recommended): `pip3 install proxomxer requests`

To install this role: `ansible-galaxy collection install maxhoesel.proxmox`

# License

GNU GPL3 or later

# Author

Max Hösel <ansible@maxhoesel.de>
