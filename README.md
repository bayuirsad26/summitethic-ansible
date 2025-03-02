# Ansible VPS Deployment Project

This project automates the deployment and configuration of a **Debian/Ubuntu-based** VPS using Ansible. With a **modular** approach, each component (role) can be configured independently and adapted easily. The project covers initial system setup (bootstrap), user security and configuration, Traefik deployment for a reverse proxy, and Mailcow deployment for a Dockerized mail server.

## Table of Contents

1. [Key Features](#key-features)
2. [Project Structure](#project-structure)
3. [Prerequisites](#prerequisites)
4. [Installation of Required Ansible Collections](#installation-of-required-ansible-collections)
5. [Inventory & Global Variables](#inventory--global-variables)
6. [Roles and Their Purposes](#roles-and-their-purposes)
   - [Bootstrap (Root Setup)](#bootstrap-root-setup)
   - [User Setup](#user-setup)
   - [Traefik](#traefik)
   - [Mailcow](#mailcow)
7. [How to Run This Project](#how-to-run-this-project)
   - [Initial Bootstrap](#initial-bootstrap)
   - [Regular Operation](#regular-operation)
   - [Running Specific Roles (e.g., Traefik and Mailcow only)](#running-specific-roles)
8. [Troubleshooting](#troubleshooting)
9. [Contribution Guidelines](#contribution-guidelines)
10. [License](#license)
11. [References](#references)

---

## 1. Key Features

- **Bootstrap (Root Setup):**

  - Updates, upgrades, and cleans system packages.
  - Creates an admin user with passwordless sudo.
  - Configures SSH (creates `.ssh`, copies public key).

- **User Setup & Security:**

  - **SSH Hardening:** Disables root and password authentication.
  - **Firewall & Fail2ban:** Installs and configures UFW and fail2ban.
  - **Docker Installation:** Configures Docker repository, installs Docker, and adds the admin user to the Docker group.
  - **Swap File Management:** Automatically calculates and creates a swap file based on system RAM.

- **Traefik Deployment:**

  - Deploys Traefik using Docker Compose.
  - Manages TLS certificates (ACME) and sets up reverse proxy routing.

- **Mailcow Deployment:**
  - Clones the Mailcow-dockerized repository.
  - Templates the Mailcow environment configuration.
  - Integrates Mailcow with Traefik for routing and certificate management.
  - Uses an override file without the obsolete `version` attribute and with a networks section to properly reference the external network `traefik-public`.
  - Provides default fallback values for environment variables (e.g., `TZ`, `DBNAME`, etc.) in the template.

---

## 2. Project Structure

```
project/
├── ansible.cfg                  # Global Ansible configuration (inventory, callbacks, etc.)
├── README.md                    # Project documentation (this file)
├── requirements.yml             # Defines required Ansible collections
├── inventories/
│   └── production/
│       ├── group_vars/
│       │   ├── all.yml          # Global variables for all hosts
│       │   └── all.yml.example  # Example global variables file
│       ├── hosts.yml            # Main production inventory
│       └── hosts.yml.example    # Example inventory file
├── playbooks/
│   ├── bootstrap.yml       # Initial bootstrap playbook (runs as root on new servers)
│   └── site.yml            # Regular configuration playbook (runs as non-root)
└── roles/
    ├── mailcow/
    │   ├── defaults/
    │   │   └── main.yml         # Default variables for Mailcow
    │   ├── meta/
    │   │   └── main.yml         # Metadata for Mailcow role
    │   ├── tasks/
    │   │   └── main.yml         # Tasks for deploying Mailcow
    │   ├── templates/
    │   │   ├── mailcow.env.j2   # Environment configuration template for Mailcow
    │   │   └── (override file is generated via task)
    │   └── vars/
    │       └── main.yml         # (Optional) Additional variables if needed
    ├── root_setup/         # Bootstrap role (initial system setup)
    │   ├── defaults/
    │   │   └── main.yml         # Default variables for system updates & admin user creation
    │   ├── tasks/
    │   │   └── main.yml         # Tasks for updating system, creating admin user, etc.
    │   ├── vars/
    │   │   └── main.yml         # Contains admin user password (encrypted with Vault)
    │   └── handlers/
    │       └── main.yml         # Handlers (e.g., reboot after updates)
    ├── traefik/            # Traefik deployment configuration
    │   ├── defaults/
    │   │   └── main.yml         # Default variables for Traefik (image, ports, etc.)
    │   ├── meta/
    │   │   └── main.yml         # Metadata for Traefik role
    │   ├── tasks/
    │   │   └── main.yml         # Tasks for deploying Traefik
    │   ├── templates/
    │   │   └── traefik-docker-compose.yml.j2  # Docker Compose template for Traefik
    │   └── vars/
    │       └── main.yml         # Additional Traefik variables (e.g., ACME email)
    └── user_setup/         # User security and system configuration tasks
        ├── defaults/
        │   └── main.yml         # Default variables for SSH hardening, firewall, Docker, swap, etc.
        ├── files/
        │   ├── fail2ban.jail.local  # Fail2ban configuration file
        │   └── id_ed25519_summitethic.pub  # SSH public key
        ├── meta/
        │   └── main.yml         # Metadata for user_setup role
        ├── handlers/
        │   └── main.yml         # Handlers for restarting SSH & fail2ban
        └── tasks/
            └── main.yml         # Tasks for security, firewall, Docker installation, and swap configuration
```

---

## 3. Prerequisites

- **Ansible**: Version 2.9 or later.
- **Python**: Installed on the VPS (usually pre-installed on Debian/Ubuntu).
- **SSH Access**: Ensure you can access the VPS with a user that has sudo privileges.
- **Docker Compatibility**: Verify that the VPS supports Docker.
- **Required Ansible Collections**: As listed in `requirements.yml` (e.g., `community.docker`, `ansible.posix`, `community.general`).

---

## 4. Installation of Required Ansible Collections

To automatically install all required collections, run:

```bash
ansible-galaxy collection install -r requirements.yml
```

This ensures that collections like `community.docker`, `ansible.posix`, and `community.general` are properly installed.

---

## 5. Inventory & Global Variables

### Inventory

- **hosts.yml**: Define your VPS host(s). Example:

  ```yaml
  all:
    hosts:
      vps.summitethic.europe:
        ansible_host: 192.0.2.10
        # For bootstrap phase, connect as root. For regular operations, the ansible_user is used.
        # You can override remote_user via your playbooks.
        ansible_python_interpreter: /usr/bin/python3.12
  ```

### Global Variables

- **all.yml**: Set global variables. Example:

  ```yaml
  ansible_user: example
  mailcow_hostname: mail.example.com
  mailcow_dbname:
  mailcow_dbuser:
  mailcow_dbpass:
  mailcow_dbroot:
  mailcow_redispass:
  mailcow_additional_san:
  traefik_acme_email: admin@example.com
  ```

---

## 6. Roles and Their Purposes

### Bootstrap (Root Setup)

- **Purpose**:

  - Updates and cleans system packages.
  - Creates an admin user with passwordless sudo.
  - Configures SSH (creates `.ssh`, copies public key).

- **Key Files**:
  - `root_setup/defaults/main.yml`: Default variables (e.g., admin user, SSH directory).
  - `root_setup/tasks/main.yml`: Tasks for system update, user creation, and SSH configuration.
  - `root_setup/vars/main.yml`: Contains the hashed admin password (encrypted with Ansible Vault).
  - `root_setup/handlers/main.yml`: Reboots the system if necessary.

### User Setup

- **Purpose**:

  - **SSH Hardening**: Disables root and password authentication.
  - **Firewall & Fail2ban**: Installs and configures UFW and fail2ban.
  - **Docker Installation**: Sets up Docker, configures repositories, and adds the user to the Docker group.
  - **Swap File Management**: Creates a swap file based on system memory.

- **Key Files**:
  - `user_setup/defaults/main.yml`: Variables for SSH hardening, firewall, Docker, and swap.
  - `user_setup/tasks/main.yml`: Tasks for configuring SSH, UFW, fail2ban, Docker installation, and swap.
  - `user_setup/files/`: Contains configuration files (e.g., `fail2ban.jail.local` and the SSH public key).
  - `user_setup/handlers/main.yml`: Handlers for restarting SSH and fail2ban.

### Traefik

- **Purpose**:

  - Deploys Traefik using Docker Compose.
  - Manages TLS certificates (ACME) and sets up reverse proxy routing.

- **Key Files**:
  - `traefik/defaults/main.yml`: Variables for the Traefik image, ports, and volumes.
  - `traefik/tasks/main.yml`: Tasks to create necessary directories, template the Docker Compose file, and deploy Traefik.
  - `traefik/templates/traefik-docker-compose.yml.j2`: Docker Compose template for Traefik (without the obsolete `version` attribute and with a networks section).
  - `traefik/vars/main.yml`: Additional Traefik variables (e.g., ACME email).

### Mailcow

- **Purpose**:

  - Clones the Mailcow-dockerized repository.
  - Templates the Mailcow environment configuration.
  - Integrates Mailcow with Traefik for routing and certificate management.
  - Generates an override file for Docker Compose that removes the obsolete `version` attribute and defines the external network `traefik-public`.
  - Uses default values for environment variables to avoid warnings.

- **Key Files**:
  - `mailcow/defaults/main.yml`: Default variables (Git repo, branch, destination, ports).
  - `mailcow/tasks/main.yml`: Tasks for deploying Mailcow, including cloning the repository, templating the environment configuration, opening required ports, and deploying using Docker Compose.
  - `mailcow/templates/mailcow.env.j2`: Template for Mailcow environment configuration with fallback default values for variables.
  - `mailcow/vars/main.yml`: (Optional) Additional variables if needed.

---

## 7. How to Run This Project

### Initial Bootstrap

For a new server (when root login is still available), run the bootstrap playbook:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/bootstrap.yml --ask-vault-pass
```

This executes the `root_setup` role to update the system, create the admin user, configure SSH, and set up passwordless sudo.

### Regular Operation

After the initial bootstrap (when root login is disabled), run the main site playbook:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml --ask-vault-pass
```

This playbook uses the non-root user (defined in your global variables) to execute roles such as `user_setup`, `traefik`, and `mailcow`.

### Running Specific Roles

To run only specific roles (e.g., Traefik and Mailcow), use tags:

```bash
ansible-playbook -i inventories/production/hosts.yml playbooks/site.yml --tags "traefik,mailcow" --ask-vault-pass
```

---

## 8. Troubleshooting

- **SSH Connection Errors**:  
  Verify host IPs, SSH keys, and ensure the correct user is used in the inventory.

- **Docker or Role Failures**:  
  Run with increased verbosity using `ansible-playbook -vvv` to diagnose issues and ensure required collections are installed.

- **Ansible Vault Issues**:  
  Use `--ask-vault-pass` (or a vault password file) with the correct password.

- **Environment Variable Warnings in Mailcow**:  
  Ensure that all necessary environment variables (e.g., `TZ`, `DBNAME`, `DBUSER`, `DBPASS`, `REDISPASS`, `DBROOT`, `ADDITIONAL_SAN`) are defined in your global variables (in `inventories/production/group_vars/all.yml`) or let the fallback defaults in the template apply.

- **Docker Compose Override Errors**:  
  Verify that the Mailcow override file generated by the playbook includes a networks section defining `traefik-public` as external and that the obsolete `version` attribute is removed.

---

## 9. Contribution Guidelines

Contributions are welcome! Please follow these steps:

1. **Fork the Repository**: Create your personal copy.
2. **Create a Feature Branch**: For example, `feature/improved-logging`.
3. **Commit Changes**: Write clear commit messages.
4. **Submit a Pull Request**: Describe your modifications and reasoning.

Follow Ansible best practices to ensure tasks remain idempotent and secure.

---

## 10. License

This project is licensed under the **[MIT License](LICENSE)**. Feel free to use, modify, and distribute with appropriate attribution.

---

## 11. References

- [Ansible Documentation](https://docs.ansible.com/)
- [community.docker Collection](https://galaxy.ansible.com/community/docker)
- [ansible.posix Collection](https://galaxy.ansible.com/ansible/posix)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)

---

By following these instructions and using the provided project structure, you can deploy and maintain your VPS dynamically. The clear separation between bootstrap and regular operations ensures that initial root-level configuration is only run once, and subsequent configurations use the non-root admin user to avoid lockout issues.
