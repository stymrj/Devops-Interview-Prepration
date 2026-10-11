# Ansible Interview Preparation Guide

*How to Answer Ansible Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Ansible Basics](#ansible-basics)
2. [Inventory Management](#inventory-management)
3. [Playbooks and Tasks](#playbooks-and-tasks)
4. [Variables and Templating](#variables-and-templating)
5. [Roles and Reuse](#roles-and-reuse)
6. [Idempotency and Handlers](#idempotency-and-handlers)
7. [Ansible in Production](#ansible-in-production)

---

## Ansible Basics

### Q1: What is Ansible, and why would you pick it over Terraform or Chef?

**How to Answer:**

"Ansible is an agentless configuration management tool that automates provisioning, config, and deployments over SSH. I pick it when I need to configure servers that already exist — patching, installing packages, managing services — because it needs no agent installed on the target. Terraform builds the infrastructure, Ansible configures what's inside it. That's the split I use in real setups."

**Key Point:** "Terraform creates, Ansible configures — agentless over SSH."

---

### Q2: How does Ansible's architecture actually work?

**How to Answer:**

"There's a control node where Ansible runs, and managed nodes it talks to over SSH — nothing else. Modules are small programs pushed to the target, executed there, and removed when done. Facts are system info Ansible gathers automatically on each run, like OS version and IPs. Since there's no agent or master server, the setup stays dead simple."

**Key Point:** "Control node pushes modules over SSH; targets run them agentlessly."

---

### Q3: What are ad-hoc commands, and when do you use them?

**How to Answer:**

"I use ad-hoc commands for one-off tasks that aren't worth writing a playbook for. Things like checking uptime across servers, restarting a service, or verifying a package version. They're fast and disposable. If I find myself running the same ad-hoc twice, it becomes a playbook."

```bash
ansible webservers -i inventory.ini -m ping
ansible webservers -i inventory.ini -m apt -a "name=nginx state=present" --become
```

**Key Point:** "One-off tasks get ad-hocs; repeated tasks get playbooks."

---

## Inventory Management

### Q4: What is an Ansible inventory, and how do you organize one?

**How to Answer:**

"The inventory is just a list of hosts Ansible manages, grouped logically — webservers, dbservers, that kind of thing. I keep dev, staging, and prod as separate inventory files so a playbook never accidentally hits prod. Within groups I use variables for environment-specific values like ports or passwords. Groups can also be nested so common settings roll down."

```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com

[prod:children]
webservers
dbservers
```

**Key Point:** "Group hosts logically, separate environments by inventory file."

---

### Q5: What is a dynamic inventory, and when do you need one?

**How to Answer:**

"A dynamic inventory is a script or plugin that builds the host list at runtime instead of reading a static file. I need one the moment infra is autoscaling or Terraform-managed — static files go stale in hours. The AWS EC2 plugin queries the API and groups instances by tags, so new servers just show up. It keeps Ansible and the cloud in sync automatically."

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
keyed_groups:
  - key: tags.Role
    separator: ""
```

**Key Point:** "Dynamic inventory pulls live hosts from cloud APIs using tags."

---

## Playbooks and Tasks

### Q6: Walk me through the structure of a playbook.

**How to Answer:**

"A playbook is a YAML list of plays; each play maps a set of hosts to a list of tasks and runs top to bottom. Every task calls a module with arguments — install nginx, copy this file, restart that service. Plays run against inventory groups, so one playbook can touch webservers then dbservers in order. That's basically the whole mental model: plays, hosts, tasks."

```yaml
- name: Configure web tier
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
    - name: Deploy config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
```

**Key Point:** "Plays map hosts to ordered tasks; modules do the actual work."

---

### Q7: How does task execution order and parallelism work in Ansible?

**How to Answer:**

"Tasks run in the order they're written, and every task completes on all hosts before the next one starts. By default Ansible runs against hosts in parallel, up to five at a time — controlled by forks. I raise forks to 20 or 50 for large fleets so deploys don't crawl. There's also serial for rolling through hosts in batches, which is how I do zero-downtime updates."

**Key Point:** "Tasks run in order; forks control host parallelism, serial controls batches."

---

## Variables and Templating

### Q8: How do variables work in Ansible, and which ones win when they conflict?

**How to Answer:**

"Variables come from lots of places — inventory, group_vars, host_vars, playbooks, extra vars, facts. There's a fixed precedence: extra vars passed with -e always win, then host vars, then group vars, then play vars. I've been bitten by this in interviews where they ask about debugging a wrong value. I just run with -e last to override anything in a pinch, and I keep most config in group_vars for sanity."

**Key Point:** "Extra vars beat everything; precedence goes host, then group, then play."

---

### Q9: How do you use Jinja2 templates in playbooks?

**How to Answer:**

"Jinja2 templates let me generate config files dynamically — the nginx.conf with the right worker count per server, the app config with the right DB host per environment. I write the file once with placeholders and loop variables, and Ansible renders it per host at deploy time. Loops and conditionals inside templates handle the variation. One template, every environment covered."

```yaml
# template: nginx.conf.j2
upstream app { server {{ app_host }}:{{ app_port }}; }
# task renders it with host-specific vars
```

**Key Point:** "Templates render per-host configs from one Jinja2 source."

---

## Roles and Reuse

### Q10: What is an Ansible role, and why use one over a plain playbook?

**How to Answer:**

"A role is a packaged, reusable bundle of tasks, handlers, templates, files, and defaults in a standard directory layout. I use roles when the same logic — say installing and hardening nginx — needs to run across environments or projects. Playbooks then just list roles like a shopping list instead of duplicating tasks. Galaxy is the public registry where community roles live, but I mostly write my own."

**Key Point:** "Roles package reusable automation; playbooks consume them."

---

### Q11: What does the standard role directory structure look like?

**How to Answer:**

"Every role has tasks/main.yml for the actual work, handlers/main.yml for triggered restarts, defaults/main.yml for overridable variables, and vars/main.yml for things that shouldn't change. Templates go in templates/, static files in files/, and meta/main.yml declares dependencies on other roles. defaults are the lowest-priority variables, so callers can always override them. This layout is a convention Ansible knows by heart."

**Key Point:** "tasks, handlers, defaults, vars, templates, files — Ansible auto-loads them."

---

## Idempotency and Handlers

### Q12: What does idempotent mean in Ansible, and why does it matter?

**How to Answer:**

"Idempotent means running a playbook ten times gives the same result as running it once — Ansible only changes what drifted. This matters because in production you rerun playbooks constantly: after failures, on schedules, after config changes. If modules weren't idempotent, every rerun would pile up duplicate changes. I test this by running playbooks twice and expecting zero changes the second time."

**Key Point:** "Rerun safely: Ansible changes only what drifted from desired state."

---

### Q13: What are handlers, and how do they work?

**How to Answer:**

"Handlers are tasks that only run when notified by another task that changed something — the classic example is restarting nginx only when its config file actually changed. They run once at the end of the play, no matter how many tasks notified them. This avoids useless service restarts on every run. I name them clearly and keep them tiny."

```yaml
tasks:
  - name: Deploy nginx config
    ansible.builtin.template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx

handlers:
  - name: Restart nginx
    ansible.builtin.service:
      name: nginx
      state: restarted
```

**Key Point:** "Handlers run once at play end, only if notified of a change."

---

## Ansible in Production

### Q14: How do you handle secrets like passwords and API keys in Ansible?

**How to Answer:**

"I use Ansible Vault to encrypt secrets at rest — the vault file is checked into git encrypted, and the password lives in CI or a password manager, never in the repo. Inside playbooks I reference vault variables like normal ones; Ansible decrypts them in memory during the run. For serious setups I pull secrets from HashiCorp Vault or AWS Secrets Manager with a lookup plugin instead of storing them anywhere. Rule one: no plaintext secrets in git, ever."

```bash
ansible-vault encrypt group_vars/prod/vault.yml
ansible-playbook site.yml -i inventory/prod --ask-vault-pass
```

**Key Point:** "Vault-encrypt secrets; pull from a real secret store when you can."

---

### Q15: How would you debug a playbook that fails on some hosts but not others?

**How to Answer:**

"I start with -vvv to see exactly which task failed and what the module returned. Then I check gather_facts output for the failing hosts — different OS versions or missing packages are the usual culprit. I run the failing task in --check mode first, and use --limit to reproduce on just one bad host. Nine times out of ten it's a fact difference, like a missing Python version or a wrong interface name."

**Key Point:** "Debug with -vvv, compare facts, reproduce with --limit on one host."

---
