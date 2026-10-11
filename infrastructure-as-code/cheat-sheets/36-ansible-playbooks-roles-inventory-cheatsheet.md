# Ansible Cheat Sheet

Dense one-page reference. Keep it open while writing playbooks.

## Core commands

```bash
ansible all -i inventory.ini -m ping                 # connectivity check
ansible web -i inventory.ini -m shell -a "uptime"    # ad-hoc command
ansible-playbook -i inventory.ini site.yml           # run a playbook
ansible-playbook site.yml --check --diff             # dry run with diff
ansible-playbook site.yml --limit web1               # target subset
ansible-playbook site.yml --tags deploy              # run tagged tasks
ansible-playbook site.yml --skip-tags slow           # skip tagged tasks
ansible-playbook site.yml -e "env=prod"              # extra vars (win)
ansible-playbook site.yml -vvv                       # max verbosity
ansible-playbook site.yml --start-at-task "Install"  # resume mid-playbook
ansible-playbook site.yml --step                     # confirm each task
ansible-inventory -i inventory.ini --graph            # visualize inventory
ansible-inventory -i inventory.ini --list             # dump parsed inventory
```

## Inventory (INI)

```ini
[web]
web1 ansible_host=10.0.0.5 ansible_user=ubuntu ansible_port=2222
web2

[db]
db1

[prod:children]
web
db

[web:vars]
http_port=8080
```

Dynamic (AWS EC2 plugin), `inventory.aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2
regions: [ap-south-1]
keyed_groups:
  - key: tags.Role
    separator: ""
compose:
  ansible_host: public_ip_address
```

## Playbook skeleton

```yaml
- name: Label the play
  hosts: web
  become: true            # sudo
  gather_facts: true
  vars:
    app_port: 3000
  tasks:
    - name: Install pkg        # always name tasks
      ansible.builtin.apt:
        name: nginx
        state: present         # present | absent | latest
      tags: [install]
      when: ansible_os_family == "Debian"
      notify: Restart nginx
  handlers:
    - name: Restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

## Most-used modules

| Module | Does what |
|---|---|
| `ansible.builtin.apt` / `yum` / `dnf` | package install/remove |
| `ansible.builtin.copy` | push static file |
| `ansible.builtin.template` | render Jinja2 template |
| `ansible.builtin.file` | dirs, links, permissions |
| `ansible.builtin.lineinfile` | edit one line in a file |
| `ansible.builtin.service` | start/stop/enable service |
| `ansible.builtin.user` / `group` | manage accounts |
| `ansible.builtin.command` / `shell` | run commands (not idempotent) |
| `ansible.builtin.git` | clone repos idempotently |
| `ansible.builtin.uri` | HTTP calls, health checks |
| `ansible.builtin.debug` | print vars (`msg:`) |
| `ansible.builtin.set_fact` | create host vars mid-run |
| `ansible.builtin.include_role` | call a role dynamically |
| `community.docker.*` | container management |

## Loops & conditionals

```yaml
- name: Install many packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop: [nginx, curl, htop]

- name: Copy per-app configs
  ansible.builtin.template:
    src: "{{ item }}.j2"
    dest: "/etc/app/{{ item }}.conf"
  loop: "{{ apps }}"

- name: Only on Debian family
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"

- name: Run if file changed
  ansible.builtin.command: /opt/deploy.sh
  register: deploy_out
  changed_when: "'DEPLOYED' in deploy_out.stdout"
  failed_when: deploy_out.rc != 0
```

## Jinja2 quick hits

```
{{ var }}                 {{ var | default('x') }}
{{ list | join(',') }}    {{ num | int + 1 }}
{% for h in groups['web'] %}{{ h }} {% endfor %}
{% if env == 'prod' %}...{% endif %}
{{ hostvars['web1'].ansible_host }}
{{ lookup('env', 'HOME') }}            # local env var
{{ lookup('file', 'key.pub') }}        # local file content
{{ query('aws_ssm', '/prod/db/pass') }}# pull from SSM/Vault
```

## Variable precedence (lowest → highest)

`role defaults` → `inventory` → `group_vars/all` → `group_vars/*` → `host_vars/*` → `play vars` → `task vars` → `extra vars (-e)`

## Roles layout

```
roles/myrole/
├── tasks/main.yml      # the work
├── handlers/main.yml   # notified restarts
├── defaults/main.yml   # overridable (lowest priority)
├── vars/main.yml       # fixed (high priority)
├── templates/          # *.j2 files
├── files/              # static files
└── meta/main.yml       # dependencies
```

```bash
ansible-galaxy init roles/myrole
ansible-galaxy install geerlingguy.nginx
```

## Vault

```bash
ansible-vault encrypt secrets.yml
ansible-vault edit secrets.yml
ansible-vault view secrets.yml
ansible-vault encrypt_string 's3cr3t' --name db_pass   # inline
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault
```

## Parallelism & strategy

```yaml
- hosts: web
  serial: 1            # rolling, one host at a time
  # serial: "25%"       # rolling, quarter of fleet
  # serial: [1, 5, 10]  # growing batches
  max_fail_percentage: 25
```

```bash
ansible-playbook site.yml -f 20      # 20 forks (default 5)
```

## Debugging one-liners

```bash
ansible-playbook site.yml --check --diff   # preview changes
ansible-playbook site.yml --limit badhost -vvv
ansible all -i inv -m ansible.builtin.setup -a "filter=ansible_*"
ansible localhost -m ansible.builtin.debug -a "var=hostvars"
```

## Facts you'll actually use

```
ansible_hostname  ansible_default_ipv4.address
ansible_os_family ansible_distribution ansible_distribution_version
ansible_processor_vcpus  ansible_memtotal_mb
ansible_interfaces  hostvars[inventory_hostname]
```
