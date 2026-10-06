# Ansible Assignment-5: GitHub Actions Runner Role

## Overview

This assignment demonstrates an **Ansible Role for configuring a GitHub Actions self-hosted runner**.

The role is designed to be:

* Version-specific
* OS-independent
* Variable-driven
* Configurable using Jinja2 templates
* Supported by Ansible handlers
* Reusable across Ubuntu/Debian and RedHat-based systems

The role installs and configures the GitHub Actions runner on a target EC2 instance and automatically starts the runner when its generated startup script changes.

---

## Assignment Requirements

The role should provide the following features:

1. Version-specific installation
2. OS-independent implementation
3. Variableized configuration
4. Use of Jinja2 templates
5. Templates for configuration files containing dynamic values
6. Separate handlers
7. Ability to execute the role on Ubuntu, CentOS/RedHat, or both

---

## Project Structure

```text
Assignment-05/
├── github_runner_role/
│   ├── defaults/
│   │   └── main.yml
│   ├── files/
│   ├── handlers/
│   │   └── main.yml
│   ├── meta/
│   │   └── main.yml
│   ├── tasks/
│   │   └── main.yml
│   ├── templates/
│   │   └── start-runner.sh.j2
│   ├── tests/
│   │   ├── inventory
│   │   └── test.yml
│   └── README.md
├── gitrunans.yml
├── inventory
└── README.md
```

---

# 1. Version-Specific Installation

The GitHub Actions runner version is defined as a variable in `defaults/main.yml`.

```yaml
github_runner_version: "2.336.0"
```

The download URL is dynamically generated using the version variable:

```yaml
github_runner_download_url: "https://github.com/actions/runner/releases/download/v{{ github_runner_version }}/actions-runner-linux-{{ github_runner_architecture }}-{{ github_runner_version }}.tar.gz"
```

This allows the runner version to be changed without modifying the role tasks.

---

# 2. Variableized Configuration

The role uses variables instead of hard-coded values.

Example variables:

```yaml
github_runner_user: githubrunner
github_runner_dir: /home/githubrunner/actions-runner

github_runner_version: "2.336.0"
github_runner_architecture: "x64"

github_runner_url: "https://github.com/behaldivyam/ansible-github-runner"
github_runner_name: "github-runner"
```

The variables control the runner user, installation directory, version, architecture, GitHub repository, and runner name.

The GitHub registration token is supplied at runtime and is not stored in the repository.

---

# 3. OS Independence

The role uses Ansible facts to identify the target operating system.

For Debian-based systems:

```yaml
- name: Install GitHub Actions runner prerequisites on Debian
  ansible.builtin.apt:
    name: "{{ github_runner_packages }}"
    state: present
    update_cache: true
  when: ansible_os_family == "Debian"
```

For RedHat-based systems:

```yaml
- name: Install GitHub Actions runner prerequisites on RedHat
  ansible.builtin.dnf:
    name: "{{ github_runner_packages }}"
    state: present
  when: ansible_os_family == "RedHat"
```

The same role can therefore select the appropriate package manager based on the target operating system.

**Testing note:** The role was successfully tested on Ubuntu 24.04. The RedHat-based implementation is included but was not separately tested during this assignment.

---

# 4. Runner Installation

The role creates a dedicated user for the GitHub Actions runner:

```yaml
- name: Create GitHub Actions runner user
  ansible.builtin.user:
    name: "{{ github_runner_user }}"
    comment: GitHub Actions Runner
    create_home: true
    shell: /bin/bash
```

It then creates the runner directory and assigns ownership to the runner user.

The runner archive is downloaded using the version-specific URL.

An archive existence check is also performed before downloading:

```yaml
- name: Check if GitHub Actions runner archive exists
  ansible.builtin.stat:
    path: "{{ github_runner_archive }}"
  register: github_runner_archive_check
```

The download task runs only when the archive does not already exist:

```yaml
when: not github_runner_archive_check.stat.exists
```

This avoids unnecessarily downloading the large runner archive again on the same target.

---

# 5. Jinja2 Template

A Jinja2 template is used to create the GitHub Actions runner startup script.

Template:

```text
github_runner_role/templates/start-runner.sh.j2
```

The template contains a dynamic variable:

```bash
#!/bin/bash

# GitHub Actions runner startup script
# Managed by Ansible

cd "{{ github_runner_dir }}"

exec ./run.sh
```

Here, `{{ github_runner_dir }}` is replaced by the value defined in the Ansible variables.

---

# 6. Ansible Template Module

The Jinja2 template is rendered using Ansible's `template` module.

```yaml
- name: Deploy GitHub Actions runner startup script
  ansible.builtin.template:
    src: start-runner.sh.j2
    dest: "{{ github_runner_dir }}/start-runner.sh"
    owner: "{{ github_runner_user }}"
    group: "{{ github_runner_user }}"
    mode: "0755"
  notify: Start GitHub Actions runner
```

The `template` module takes the `.j2` file, processes the Jinja2 variables, and creates the final `start-runner.sh` file on the target EC2 instance.

---

# 7. Handler Implementation

The role uses a separate handler instead of starting the runner directly inside the task.

`handlers/main.yml`:

```yaml
---
- name: Start GitHub Actions runner
  ansible.builtin.shell:
    cmd: "sudo -u {{ github_runner_user }} ./start-runner.sh > runner.log 2>&1 &"
    chdir: "{{ github_runner_dir }}"
...
```

The template task notifies this handler:

```yaml
notify: Start GitHub Actions runner
```

The handler therefore runs when the template task reports a change.

### Handler Flow

```text
Jinja2 Template
      ↓
template module
      ↓
start-runner.sh created/updated
      ↓
Task reports "changed"
      ↓
notify handler
      ↓
Start GitHub Actions runner
      ↓
run.sh
      ↓
GitHub Runner connected
```

If the template does not change, the handler is not triggered.

---

# 8. Playbook

The role is called from the main playbook:

```yaml
---
- name: Setup GitHub Actions self-hosted runner
  hosts: github_runner
  become: true
  gather_facts: true

  roles:
    - github_runner_role
...
```

The target hosts are defined in the inventory under the `github_runner` group.

---

# 9. Execution

The role can be executed using:

```bash
ansible-playbook -i inventory gitrunans.yml
```

The GitHub runner registration token should be supplied securely at runtime rather than stored in the repository.

---

# 10. Validation and Testing

The role was tested on an Ubuntu 24.04 EC2 instance.

The playbook completed successfully:

```text
PLAY RECAP
github-runner : ok=10 changed=2 unreachable=0 failed=0 skipped=2
```

The generated startup script was verified on the target:

```text
-rwxr-xr-x 1 githubrunner githubrunner 128 Oct 1 13:42 start-runner.sh
```

The runner process was also verified:

```text
githubr+ 4795 4794 0 13:42 pts/4 00:00:00 /bin/bash /home/githubrunner/actions-runner/run.sh
```

The runner log confirmed successful communication with GitHub:

```text
√ Connected to GitHub

Current runner version: '2.336.0'
2026-10-01 13:42:18Z: Listening for Jobs
```

The runner was visible in GitHub with the status:

```text
Idle
```

This confirms that the role successfully installed, configured, and started the GitHub Actions self-hosted runner.

---

# 11. Idempotency Consideration

The role includes checks to avoid unnecessary work.

For example, the runner archive is checked before downloading:

```yaml
register: github_runner_archive_check
```

and:

```yaml
when: not github_runner_archive_check.stat.exists
```

The runner configuration also uses:

```yaml
creates: "{{ github_runner_dir }}/.runner"
```

This prevents the configuration command from being executed again when the runner is already configured.

The template task only triggers the handler when the generated startup script changes.

---

# 12. Key Ansible Concepts Demonstrated

This assignment demonstrates the following Ansible concepts:

* Ansible Roles
* Role defaults
* Variables
* OS-specific conditionals
* Ansible facts
* Package management
* File management
* `stat` module
* `get_url` module
* `unarchive` module
* `command` module
* `template` module
* Jinja2 templating
* Handlers
* Notifications
* Idempotent execution
* Runtime variables/secrets

---

# 13. Conclusion

This assignment demonstrates a reusable Ansible role for deploying a GitHub Actions self-hosted runner.

The implementation supports version-specific installation, variable-driven configuration, OS-specific package handling, Jinja2 templating, and handler-based runner startup.

The role was successfully executed on an Ubuntu 24.04 EC2 instance, and the resulting GitHub Actions runner successfully connected to GitHub and reached the **Idle** state.
