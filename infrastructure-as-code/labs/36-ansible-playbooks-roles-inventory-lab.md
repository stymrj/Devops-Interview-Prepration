# Ansible Hands-On Lab

Build real Ansible muscle: 10 exercises covering inventory, playbooks, roles, vault, and debugging. Use a control machine (your laptop or a VM) and 1-2 target VMs/containers with SSH access (Ubuntu works best).

---

## Exercise 1: Verify Agentless Connectivity

**Goal:** Confirm Ansible can reach targets over SSH with no agent installed.

```bash
# Install ansible
pip install ansible
ansible --version

# Create a minimal inventory
cat > lab-inventory.ini <<'EOF'
[web]
web1 ansible_host=192.168.1.11 ansible_user=ubuntu
EOF

# Ping all hosts (uses the ping module, not ICMP)
ansible all -i lab-inventory.ini -m ping
```

**Expected output:** `"ping": "pong"` for each host.

**Why it matters:** This proves the core Ansible promise — SSH in, modules execute, nothing installed permanently. Every playbook you ever write starts here.

---

## Exercise 2: Run Ad-Hoc Commands Across Hosts

**Goal:** Do real work without writing a file.

```bash
ansible all -i lab-inventory.ini -m shell -a "uptime" 
ansible all -i lab-inventory.ini -m ansible.builtin.apt -a "name=htop state=present" --become
ansible all -i lab-inventory.ini -m ansible.builtin.setup -a "filter=ansible_distribution*"
```

**Expected output:** Uptime lines, a `changed` result for the package install, and facts filtered to distribution info.

**Why it matters:** Ad-hocs are your incident-response muscle — check state, install a tool, gather facts in seconds across the fleet.

---

## Exercise 3: Write Your First Playbook

**Goal:** Go from ad-hoc to a repeatable playbook.

```bash
cat > 01-web.yml <<'EOF'
- name: Set up web basics
  hosts: web
  become: true
  tasks:
    - name: Install nginx and curl
      ansible.builtin.apt:
        name: [nginx, curl]
        state: present
        update_cache: true
    - name: Ensure nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
EOF
ansible-playbook -i lab-inventory.ini 01-web.yml
```

**Expected output:** Play recap showing tasks as `changed` on first run.

**Why it matters:** This is the fundamental unit of Ansible work. Repeatable, reviewable, version-controlled server config.

---

## Exercise 4: Prove Idempotency

**Goal:** See that reruns change nothing.

```bash
ansible-playbook -i lab-inventory.ini 01-web.yml
# Read the recap carefully
```

**Expected output:** Every task shows `ok=4 changed=0` — zero changes on the second run.

**Why it matters:** Idempotency is why ops teams trust Ansible on schedules and in CI. If your playbook isn't idempotent, fix it before it touches prod.

---

## Exercise 5: Use Group Variables and Separate Environments

**Goal:** Organize inventory the production way.

```bash
mkdir -p inventories/dev inventories/prod group_vars
cat > inventories/dev/hosts.ini <<'EOF'
[web]
dev-web1 ansible_host=192.168.1.11 ansible_user=ubuntu
EOF
cat > group_vars/web.yml <<'EOF'
nginx_worker_connections: 512
app_env: development
EOF
cat > 02-vars.yml <<'EOF'
- hosts: web
  tasks:
    - name: Show effective config
      ansible.builtin.debug:
        msg: "env={{ app_env }} workers={{ nginx_worker_connections }}"
EOF
ansible-playbook -i inventories/dev/hosts.ini 02-vars.yml
```

**Expected output:** Debug message rendering the group var values.

**Why it matters:** This is how you avoid hardcoding per-environment values. Dev and prod stay separate; one playbook serves both.

---

## Exercise 6: Render Configs with Jinja2 Templates

**Goal:** Generate per-host config files from one template.

```bash
cat > nginx.conf.j2 <<'EOF'
worker_connections {{ nginx_worker_connections }};
server {
    listen 80;
    server_name {{ inventory_hostname }};
    location / { proxy_pass http://127.0.0.1:3000; }
}
EOF
cat > 03-template.yml <<'EOF'
- hosts: web
  become: true
  tasks:
    - name: Deploy rendered nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/conf.d/app.conf
      notify: Reload nginx
  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
EOF
ansible-playbook -i inventories/dev/hosts.ini 03-template.yml
ansible-playbook -i inventories/dev/hosts.ini 03-template.yml  # run twice
```

**Expected output:** First run `changed` + handler fired; second run `changed=0` and handler skipped.

**Why it matters:** Templates plus handlers are the bread and butter of config management — deploy config only when it changes, restart only when needed.

---

## Exercise 7: Build a Role

**Goal:** Package reusable automation in the standard layout.

```bash
ansible-galaxy init roles/nginx
ls roles/nginx  # tasks/ handlers/ defaults/ vars/ templates/ files/ meta/
# Move your nginx tasks into roles/nginx/tasks/main.yml
# Move the template into roles/nginx/templates/nginx.conf.j2
cat > 04-role.yml <<'EOF'
- hosts: web
  become: true
  roles:
    - nginx
EOF
ansible-playbook -i inventories/dev/hosts.ini 04-role.yml
```

**Expected output:** Role tasks execute exactly like inline tasks did.

**Why it matters:** Roles turn one-off playbooks into shareable, testable units. This is how teams scale Ansible without copy-paste chaos.

---

## Exercise 8: Encrypt Secrets with Ansible Vault

**Goal:** Keep secrets out of plaintext in git.

```bash
cat > vault-pass.txt <<'EOF'
lab-secret-password
EOF
ansible-vault create --vault-password-file vault-pass.txt group_vars/vault.yml
# Inside the editor, add: db_password: "supersecret123"
cat > 05-secret.yml <<'EOF'
- hosts: web
  tasks:
    - name: Use the secret safely
      ansible.builtin.debug:
        msg: "db password length is {{ db_password | length }}"
      no_log: true
EOF
ansible-playbook -i inventories/dev/hosts.ini 05-secret.yml --vault-password-file vault-pass.txt
cat group_vars/vault.yml  # confirm it's encrypted gibberish
rm vault-pass.txt  # never commit the password file
```

**Expected output:** Encrypted `$ANSIBLE_VAULT` header in the file; playbook decrypts in memory.

**Why it matters:** Vault is the minimum bar for secrets in Ansible. Plaintext passwords in repos are an instant interview red flag.

---

## Exercise 9: Do a Rolling Update with Serial

**Goal:** Update hosts in batches without full downtime.

```bash
cat > 06-rolling.yml <<'EOF'
- hosts: web
  become: true
  serial: 1
  tasks:
    - name: Update all packages
      ansible.builtin.apt:
        upgrade: dist
        update_cache: true
    - name: Reboot if needed
      ansible.builtin.reboot:
        reboot_timeout: 300
      when: false  # flip to true when you mean it
EOF
ansible-playbook -i inventories/dev/hosts.ini 06-rolling.yml
```

**Expected output:** Play recap shows hosts processed one batch at a time.

**Why it matters:** `serial` is how you patch fleets without taking everything down at once. Interviewers love asking about rolling strategies.

---

## Exercise 10: Debug a Failure Like a Pro

**Goal:** Practice the debugging workflow on a broken playbook.

```bash
cat > 07-broken.yml <<'EOF'
- hosts: web
  tasks:
    - name: Install a package that does not exist
      ansible.builtin.apt:
        name: definitely-not-a-real-package
        state: present
      become: true
EOF
ansible-playbook -i inventories/dev/hosts.ini 07-broken.yml --limit dev-web1
ansible-playbook -i inventories/dev/hosts.ini 07-broken.yml --limit dev-web1 -vvv
ansible-playbook -i inventories/dev/hosts.ini 07-broken.yml --check --diff
```

**Expected output:** Failure on the bad package; `-vvv` shows the exact module error; `--check` previews without changing anything.

**Why it matters:** `--limit`, `-vvv`, and `--check` are the three debugging tools you'll reach for every time. Practice them before you need them at 2 AM.

---

## Cleanup

```bash
rm -rf inventories group_vars roles lab-inventory.ini 0*.yml nginx.conf.j2
```

Done — 10 exercises, zero agents installed, full Ansible workflow practiced.
