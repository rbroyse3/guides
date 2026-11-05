# Module 11: Configuration management

**Time:** about 5 hours.
**Prerequisites:** modules 03 and 05.

## By the end you can

- Explain why servers built by hand drift, and how configuration management prevents it.
- Write an Ansible playbook that is idempotent: running it twice changes nothing the second time.
- Use variables, templates, and handlers.
- Preview changes with `--check --diff` before applying them.
- Rebuild a server from a clean snapshot and get the same result.

## Why this matters

A server configured by hand is a "pet": nobody remembers exactly how it was built, and rebuilding it is guesswork. A server defined in code is "cattle": you can rebuild it, review changes before they happen, and prove two servers match. This is how you "build things right."

## Concepts

- **Configuration drift.** Servers slowly diverge as people make one-off changes. Drift causes "works on server A but not server B" problems.
- **Declarative, idempotent.** You describe the **desired state** ("httpd is installed and running"), not the steps. Run it repeatedly and it only changes what's different.
- **Ansible.** Agentless: it connects over SSH (or locally), runs **modules**, and reports what **changed**. Core pieces: an **inventory** (what hosts), a **playbook** (YAML that lists tasks), **modules** (`dnf`, `file`, `copy`, `template`, `systemd`, `firewalld`), **handlers** (actions triggered only when something changed, like a restart), **variables**, and **Jinja2 templates**.
- **Check mode.** `--check --diff` shows what *would* change. Review before applying.
- **Roles** package tasks for reuse (a pointer only here). **Ansible Vault** encrypts secrets (also a pointer).
- **Other tools.** Puppet, Chef, Salt, and Terraform (for provisioning) solve related problems. The ideas transfer: desired state, idempotency, code in git, review before apply.
- **Everything in git.** The playbook is the documentation, and the history shows who changed what and why.
- **Don't fix servers by hand.** If you change a managed server manually, the next run will undo it, or worse, hide it. Change the code, then apply.

## Lab

Snapshot first (name it `before-ansible`). You'll restore it later.

### 1. Install and look around

```bash
sudo dnf install -y ansible-core
ansible --version
ansible-galaxy collection install ansible.posix     # needed for the firewalld module
mkdir -p ~/ansible-lab && cd ~/ansible-lab && git init
```

### 2. Ad-hoc commands

```bash
ansible localhost -c local -m ansible.builtin.ping
ansible localhost -c local -m ansible.builtin.setup -a 'filter=ansible_distribution*'
ansible localhost -c local -b -m ansible.builtin.dnf -a 'name=tree state=present'
ansible localhost -c local -b -m ansible.builtin.dnf -a 'name=tree state=present'   # run it twice
```

Look at the output: the first run says `changed`, the second says `ok`. That's idempotency.

### 3. A playbook that builds the web server from module 03

Create `site.yml`:

```yaml
- name: Lab web server
  hosts: localhost
  connection: local
  become: true
  vars:
    web_root: /srv/www
  tasks:
    - name: Install httpd
      ansible.builtin.dnf:
        name: httpd
        state: present

    - name: Web root exists with least privilege
      ansible.builtin.file:
        path: "{{ web_root }}"
        state: directory
        owner: root
        group: apache
        mode: "0750"

    - name: Index page
      ansible.builtin.copy:
        content: "hello from {{ ansible_hostname }}\n"
        dest: "{{ web_root }}/index.html"
        owner: root
        group: apache
        mode: "0640"

    - name: Site config
      ansible.builtin.template:
        src: lab.conf.j2
        dest: /etc/httpd/conf.d/lab.conf
        owner: root
        group: root
        mode: "0644"
      notify: Restart httpd

    - name: Open http in firewalld
      ansible.posix.firewalld:
        service: http
        permanent: true
        immediate: true
        state: enabled

    - name: httpd enabled and running
      ansible.builtin.systemd:
        name: httpd
        enabled: true
        state: started

  handlers:
    - name: Restart httpd
      ansible.builtin.systemd:
        name: httpd
        state: restarted
```

Create `templates/lab.conf.j2`:

```apache
<VirtualHost *:80>
    DocumentRoot "{{ web_root }}"
    <Directory "{{ web_root }}">
        Require all granted
    </Directory>
</VirtualHost>
```

### 4. Check, apply, and prove idempotency

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff       # preview: nothing changes
ansible-playbook site.yml                      # apply
ansible-playbook site.yml                      # again: changed=0
curl localhost
```

Why does the handler run on the first apply and not the second?

### 5. Drift

Break the server by hand: edit `/etc/httpd/conf.d/lab.conf`, change the mode of `/srv/www`, and stop httpd. Run `ansible-playbook site.yml --check --diff` and read what it reports. Then apply it and watch it repair everything. Explain what would happen if you'd *meant* that manual change.

### 6. Rebuild from nothing

Commit your playbook and template to git. Then **restore the clean snapshot**, install Ansible, pull your repo, and run the playbook once. Confirm you get the same working web server, with no manual steps. Time it.

### 7. Stretch: a second host and variables

With a second VM, add an inventory file, run the playbook against it over SSH (use your key from module 06), and use a host variable to give each server a different page. Look at roles (`ansible-galaxy init`) as the next step.

## Break it

1. **Not idempotent.** Replace the `file` task with `ansible.builtin.command: mkdir /srv/www`. Run it twice and read the error. What's the lesson about using `command` and `shell`?
2. **YAML.** Break the indentation and read the error from `--syntax-check`.
3. **A secret in the playbook.** Add a fake password to a vars file, commit it, and discuss what you'd do instead (Ansible Vault, a secrets manager) and what you'd do if it were real.
4. **A failing task.** Give the `dnf` task a package name that doesn't exist and see where the play stops and what state the server is left in.

## Check your understanding

- What is configuration drift, and what problems does it cause?
- What does idempotent mean, and how do you prove a playbook has it?
- What does `--check --diff` do, and when would you use it?
- Why use a handler instead of a task that always restarts the service?
- Why avoid `command` and `shell` when a module exists?
- Why shouldn't you edit a managed server by hand?
- How is a playbook in git also documentation?

## Deliverable

A git repository containing your playbook, template, and a README with: how to run it, what it builds, how you tested idempotency, and the time it took to rebuild from a clean snapshot. The repository must not contain secrets.

## Use AI well here

Ask Claude to review your playbook for idempotency problems, secrets, and missing handlers, then ask it to explain what each module does. Test every change with `--check --diff` first. Be wary of invented module names and options: check them with `ansible-doc <module>`.

## Go deeper

- Ansible documentation, and `ansible-doc <module>`
- Roles, collections, inventory groups, Ansible Vault, and `ansible-lint`
- Puppet or Chef docs if your team uses them; Terraform for provisioning infrastructure
- "Cattle, not pets," and immutable infrastructure
