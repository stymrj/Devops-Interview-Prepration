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
