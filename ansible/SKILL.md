---
name: ansible
description: Ansible coding conventions. Use when writing, generating or reviewing any Ansible code, including choosing the Ansible version, designing playbooks and roles, wiring dependencies, and adding validation.
---

Follow the `markdown` skill for README and documentation conventions.

## Stack

- Target platforms: RHEL and Fedora (use `dnf`, not `apt`).
- Preferred cloud: AWS, `us-east-2` region.
- All YAML files end in `.yaml`, never `.yml`.
- Python environment: `.venv` at the project root. Install with `pip install -r requirements.txt`.
- Always use **FQCN** (Fully Qualified Collection Names) for all module actions, including builtins: `ansible.builtin.copy`, not `copy`.
- Absolutely no sensitive information (passwords, keys, tokens) in the git repository. Use Ansible Vault for any secrets that must live alongside the code, and prefer external secret stores (e.g., AWS Secrets Manager with ExternalSecretsOperator) in production.

## Creating a New Ansible Project

Ansible projects live in repositories prefixed with `iac-` under the organization folder `~/projects/github/${organization_name}/iac-${project_name}`.

```shell
ORG=exarep
PROJECT=platform
REPO_DIR=~/projects/github/${ORG}/iac-${PROJECT}
mkdir -p ${REPO_DIR} && cd ${REPO_DIR}

python3 -m venv .venv
source .venv/bin/activate
echo "ansible" > requirements.txt
echo "ansible-lint" >> requirements.txt
pip install -r requirements.txt

mkdir -p playbooks roles inventories group_vars host_vars files templates
touch ansible.cfg
git init && git add . && git commit -m 'Ansible project init'
```

Always add a `README.md` covering clone, create the venv, install deps, and how to run the playbooks.

## Project Structure

```
iac-${project_name}/
├── ansible.cfg
├── inventories/
│   ├── dev/
│   │   ├── hosts.yaml
│   │   └── group_vars/
│   │       └── all.yaml
│   ├── test/
│   │   ├── hosts.yaml
│   │   └── group_vars/
│   │       └── all.yaml
│   └── prod/
│       ├── hosts.yaml
│       └── group_vars/
│           └── all.yaml
├── playbooks/
│   ├── pb-site.yaml
│   ├── pb-provision.yaml
│   └── pb-configure.yaml
├── roles/
│   └── example_role/
│       ├── defaults/
│       │   └── main.yaml
│       ├── handlers/
│       │   └── main.yaml
│       ├── meta/
│       │   └── main.yaml
│       ├── tasks/
│       │   └── main.yaml
│       ├── templates/
│       └── vars/
│           └── main.yaml
├── group_vars/
│   └── all.yaml
├── host_vars/
├── files/
├── templates/
├── collections/
│   └── requirements.yaml
├── requirements.txt
├── .gitignore
└── README.md
```

- **Playbooks** go in the `playbooks/` folder and are prefixed with `pb-`: `pb-site.yaml`, `pb-provision.yaml`, `pb-deploy.yaml`.
- **Roles** go in the `roles/` folder. Use `ansible-galaxy role init` to scaffold new roles, then rename any `.yml` files to `.yaml`.
- **Inventories** are organized by environment: `inventories/dev/`, `inventories/test/`, `inventories/prod/`.
- `group_vars/` and `host_vars/` at the project root hold cross-environment defaults. Environment-specific overrides go in `inventories/${env}/group_vars/`.

## `ansible.cfg`

```ini
[defaults]
inventory = inventories/dev/hosts.yaml
roles_path = roles
collections_path = collections
remote_user = ansible
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

## Collections

Declare collection dependencies in `collections/requirements.yaml` and install them before running playbooks.

```yaml
collections:
  - name: ansible.builtin
  - name: ansible.posix
  - name: amazon.aws
  - name: community.general
  - name: redhat.rhel_system_roles
```

```shell
ansible-galaxy collection install -r collections/requirements.yaml
```

## Playbook Conventions

- Every playbook starts with a `name` and clearly declares `hosts`, `become`, and any `vars_files` or `roles`.
- Use descriptive `name` fields on every play and every task. Never leave a task unnamed.
- Prefer roles and `ansible.builtin.include_role` / `ansible.builtin.import_role` for reusable logic. Keep playbooks thin orchestrators.
- Use tags for selective execution. Apply tags to roles and task includes, not to individual tasks inside a role.
- Group related tasks with `block` / `rescue` / `always` for error handling.

```yaml
---
- name: Configure web servers
  hosts: webservers
  become: true

  roles:
    - role: base
      tags: [base]
    - role: webserver
      tags: [webserver]

  tasks:
    - name: Verify service is running
      ansible.builtin.systemd:
        name: httpd
        state: started
        enabled: true
      tags: [verify]
```

## Role Conventions

- Role names use snake_case: `install_packages`, `configure_firewall`.
- Put default variable values in `defaults/main.yaml`. Put internal constants in `vars/main.yaml`.
- Prefix all role variables with the role name to avoid collisions: `webserver_port`, `webserver_document_root`.
- Handlers restart or reload services. Always use `ansible.builtin.service` or `ansible.builtin.systemd` in handlers. Use `notify` on the tasks that change configuration.
- Templates use the `.j2` extension and live in the role's `templates/` folder.

```yaml
# roles/webserver/defaults/main.yaml
---
webserver_port: 8080
webserver_document_root: /var/www/html
```

```yaml
# roles/webserver/handlers/main.yaml
---
- name: Restart httpd
  ansible.builtin.systemd:
    name: httpd
    state: restarted
    enabled: true
```

## Task Conventions

- Always use FQCN: `ansible.builtin.copy`, `ansible.builtin.template`, `ansible.builtin.dnf`, `ansible.posix.firewalld`.
- Prefer `ansible.builtin.dnf` over `ansible.builtin.yum` (RHEL 8+ and Fedora).
- Tasks must be idempotent. Avoid `ansible.builtin.command` and `ansible.builtin.shell` when a purpose-built module exists. When shell is unavoidable, add `creates`, `removes`, or `changed_when` / `failed_when` to maintain idempotency.
- Use `ansible.builtin.template` over `ansible.builtin.copy` when the file needs variable substitution.
- Use `ansible.builtin.assert` for pre-flight validation of required variables.
- Use `loop` (not `with_items`) for iteration.

```yaml
- name: Install required packages
  ansible.builtin.dnf:
    name: "{{ item }}"
    state: present
  loop:
    - httpd
    - firewalld
    - mod_ssl

- name: Deploy configuration
  ansible.builtin.template:
    src: httpd.conf.j2
    dest: /etc/httpd/conf/httpd.conf
    owner: root
    group: root
    mode: "0644"
  notify: Restart httpd
```

## Variables and Secrets

- Use `group_vars/` and `host_vars/` for inventory-scoped variables. Keep variable definitions close to the inventory they apply to.
- Encrypt sensitive values with Ansible Vault. Store vault-encrypted files alongside the variable files they belong to with a `vault_` prefix: `group_vars/all/vault_secrets.yaml`.
- Reference vault variables with a `vault_` prefix and assign them to normal variable names in the clear-text file so the mapping is obvious:

```yaml
# group_vars/all/all.yaml
db_password: "{{ vault_db_password }}"

# group_vars/all/vault_secrets.yaml (vault-encrypted)
vault_db_password: supersecret
```

- Never commit unencrypted secrets. Add vault password files to `.gitignore`.

## Inventory

Use YAML format for inventory files. Group hosts by role and environment.

```yaml
# inventories/dev/hosts.yaml
---
all:
  children:
    webservers:
      hosts:
        web01.dev.example.com:
        web02.dev.example.com:
    databases:
      hosts:
        db01.dev.example.com:
```

## Containers

When managing containers with Ansible, assume Podman.

- Use `containers.podman.podman_container` and `containers.podman.podman_image`, not Docker modules.
- Name container definition files `Containerfile`, not `Dockerfile`.
- Source images from Red Hat registries: `registry.access.redhat.com/hi/...`, `registry.redhat.io`, `quay.io`.
- Add `containers.podman` to `collections/requirements.yaml`.

## Linting

Run `ansible-lint` before committing. The project `requirements.txt` should include `ansible-lint`.

```shell
source .venv/bin/activate
ansible-lint playbooks/
```

Address all warnings. In particular: every task must have a `name`, FQCN must be used, and `command`/`shell` tasks must have `changed_when` defined.

## `.gitignore`

```gitignore
.venv/
*.retry
collections/ansible_collections/
.vault_password
__pycache__/
*.pyc
```

## Running Playbooks

```shell
source .venv/bin/activate
ansible-galaxy collection install -r collections/requirements.yaml

# Run against a specific environment
ansible-playbook -i inventories/dev/hosts.yaml playbooks/pb-site.yaml

# Run with tags
ansible-playbook -i inventories/dev/hosts.yaml playbooks/pb-site.yaml --tags "base,webserver"

# Run with vault
ansible-playbook -i inventories/prod/hosts.yaml playbooks/pb-site.yaml --ask-vault-pass
```
