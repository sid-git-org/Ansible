
# 🛠 Ansible Essentials and Day-to-Day Summary

## 📌 Core Definitions

- **Playbook**: A YAML file that defines a series of plays/tasks to be executed on a group of hosts.
- **Play**: A mapping between hosts and tasks to be executed on them.
- **Task**: A single action to be executed, like installing a package or copying a file.
- **Role**: A pre-defined directory structure for organizing related tasks, handlers, files, and variables.
- **Handler**: A task triggered by `notify` from another task, usually used for restarting services.
- **Inventory**: A list of hosts or groups where tasks are executed.
- **Module**: A unit of work Ansible ships with (e.g., `copy`, `yum`, `command`).

---

## 📂 Playbook Structure

```yaml
- name: Sample Playbook
  hosts: webservers
  become: yes
  tasks:
    - name: Install package
      apt:
        name: nginx
        state: present
      notify: restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

---

## 🚨 Error Handling in Ansible

### 1. `ignore_errors: yes`
Continue even if the task fails.

### 2. `failed_when`
Custom failure condition.

### 3. `block`, `rescue`, `always`
Try-Catch-Finally structure.

```yaml
- block:
    - name: risky task
      command: /bin/false
  rescue:
    - debug: msg="Recovered from failure"
  always:
    - debug: msg="Always run this"
```

### 4. `register` + `when` + `failed`
Control flow based on task output.

### 5. `max_fail_percentage`
Allow partial failure in large inventories.

### 6. `any_errors_fatal: true`
Stop all if one fails.

---

## 🔁 Handlers and `flush_handlers`

- Handlers run **at the end** of play by default.
- Use `meta: flush_handlers` to run all notified handlers **immediately**.

```yaml
- name: Update config
  template:
    src: conf.j2
    dest: /etc/app.conf
  notify: restart app

- meta: flush_handlers
```

> `flush_handlers` runs **all handlers** notified up to that point.

---

## 🚀 Resuming Playbook After Failure

### Resume from a task:
```bash
ansible-playbook site.yml --start-at-task="Task name"
```

### Resume from a specific host:
```bash
ansible-playbook site.yml --limit host59
```

### Resume both:
```bash
ansible-playbook site.yml --start-at-task="Task name" --limit host59
```

### Use `.retry` file:
```bash
ansible-playbook site.yml
ansible-playbook site.yml --limit @site.retry
```

### Use `serial` in playbook:
```yaml
- hosts: all
  serial: 10
```

---

## ⚡ Concurrency & Performance Tuning in Ansible

### **1. `forks` — Parallel Execution**
Controls how many hosts Ansible will work on at the same time. Default: 5.

Set in `ansible.cfg`:
```ini
[defaults]
forks = 20
```
CLI override:
```bash
ansible-playbook site.yml -f 20
```

### **2. `serial` — Rolling Batches**
Executes tasks in batches instead of all at once.

```yaml
- hosts: webservers
  serial: 5
  tasks:
    - name: Restart nginx
      service:
        name: nginx
        state: restarted
```

### **3. `batch`**
Defines groups of hosts when `serial` is used.

### **4. Combining `forks` and `serial`**
Forks control **parallelism within a batch**.

### **5. `max_fail_percentage`**
Adds tolerance for failures in a batch.

### **6. Async & Poll**
Allows background execution of tasks:
```yaml
- name: Long-running task
  shell: /usr/bin/long_task.sh
  async: 300
  poll: 0
```

---

## 📦 Ansible Roles

Roles group tasks, handlers, files, variables, etc., into a **standard directory structure**:

```
my_role/
├── tasks/
├── handlers/
├── templates/
├── files/
├── vars/
├── defaults/
├── meta/
└── tests/
```

Create a role:
```bash
ansible-galaxy init my_role
```

Use roles in playbook:
```yaml
- hosts: webservers
  roles:
    - nginx
```

---

## 📚 Ansible Learning Roadmap

### **Pending Topics to Master:**
- Advanced conditionals & loops.
- Variable precedence.
- Jinja2 templates.
- Vault (secrets management).
- Dynamic inventory.
- Testing with Molecule.
- CI/CD integrations.
- Performance optimizations.
- Real-life use cases like zero-downtime deployments.

---

## 🔑 Variable Precedence

**Highest → Lowest:**
1. Extra-vars (`-e`).
2. Task vars.
3. Block vars.
4. Role vars.
5. Host/group vars.
6. Inventory vars.
7. Role defaults.

---

## 🗄 Fact Caching

Used to store gathered facts for reuse, improving speed.

**Example (jsonfile backend):**
```ini
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache
fact_caching_timeout = 86400
```

---

## 🏃 Strategies in Ansible

### **1. Linear (default)**
Tasks run sequentially on all hosts.

### **2. Free Strategy**
Tasks run as fast as possible on each host independently:
```yaml
- hosts: all
  strategy: free
```

---

## 🔄 Reboot Module
Reboots a host and waits for it to come online:
```yaml
- name: Reboot server
  reboot:
    reboot_timeout: 600
```

---

## 🔄 Ansible Pull Mode
Instead of pushing config, the host **pulls** from a Git repo:
```bash
ansible-pull -U https://github.com/your/repo.git playbook.yml
```

---

---

# 🔀 Advanced Conditionals & Loops in Ansible

## **1. Conditionals (`when`)**

### **Basic Example**
```yaml
- name: Install nginx on Debian-based systems
  apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"
```

### **Using Variables**
```yaml
- name: Restart service if required
  service:
    name: nginx
    state: restarted
  when: service_restart | bool
```

---

## **2. Custom Failure & Change Conditions**

### **`failed_when`**
Override failure detection:
```yaml
- name: Check command output
  command: /bin/false
  register: result
  failed_when: "'ERROR' in result.stderr"
```

### **`changed_when`**
Override changed detection:
```yaml
- name: Run script but mark as not changed
  command: ./script.sh
  changed_when: false
```

---

## **3. Loops**

### **Simple Loop**
```yaml
- name: Install multiple packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - curl
    - git
```

### **Loop with Dictionaries**
```yaml
- name: Create multiple users
  user:
    name: "{{ item.name }}"
    state: present
    groups: "{{ item.groups }}"
  loop:
    - { name: 'john', groups: 'admin' }
    - { name: 'jane', groups: 'developers' }
```

### **with_items (old style)**
```yaml
- name: Install multiple packages
  yum:
    name: "{{ item }}"
    state: present
  with_items:
    - httpd
    - vim
```

---

## **4. until & retries (Looping Until Success)**
Retry a task until it succeeds:
```yaml
- name: Wait for service
  shell: systemctl is-active nginx
  register: service_status
  until: service_status.stdout == "active"
  retries: 5
  delay: 10
```

---

## **5. Combining Loops and Conditionals**
```yaml
- name: Install packages only if variable is true
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - mysql
  when: install_packages | default(true)
```

---

---

# 🧠 Advanced Conditionals & Loops

## 1) `when` — Conditional execution
Run a task only if a condition is true.

```yaml
- name: Install httpd only on RedHat
  yum:
    name: httpd
    state: present
  when: ansible_facts['os_family'] == "RedHat"
```

### Multiple conditions
```yaml
- name: Restart only if changed and service enabled
  service:
    name: nginx
    state: restarted
  when:
    - result.changed
    - nginx_enabled | default(true)
```

### Using filters
```yaml
- name: Run only on even hosts (web02, web04 ...)
  debug:
    msg: "Even host"
  when: inventory_hostname | regex_search('[02468]$')
```

---

## 2) `failed_when` — Custom failure logic
Override Ansible’s default idea of success/failure.

```yaml
- name: Run command but fail if specific string appears
  command: my_script.sh
  register: out
  failed_when: "'FATAL' in out.stdout"
```

### `changed_when` — Control changed status
```yaml
- name: Check something but never report changed
  command: /usr/bin/true
  changed_when: false
```

### Both together
```yaml
- name: Curl endpoint and decide success manually
  command: curl -s -o /dev/null -w "%{http_code}" http://localhost/health
  register: http_code
  changed_when: false
  failed_when: http_code.stdout | int != 200
```

---

## 3) Loops

### `loop` (preferred) vs legacy `with_items`
```yaml
- name: Create users
  user:
    name: "{{ item.name }}"
    state: present
    uid: "{{ item.uid }}"
  loop:
    - { name: 'alice', uid: 2001 }
    - { name: 'bob',   uid: 2002 }
```

### `with_items` (older syntax)
```yaml
- name: Install packages
  yum:
    name: "{{ item }}"
    state: present
  with_items:
    - vim
    - git
```

### `with_dict`
```yaml
- name: Create users from dict
  user:
    name: "{{ item.key }}"
    uid: "{{ item.value.uid }}"
  with_dict:
    alice: { uid: 2001 }
    bob:   { uid: 2002 }
```

### `with_subelements`
Useful for iterating parent/child structures.

```yaml
vars:
  users:
    - name: alice
      keys:
        - ssh-rsa AAA...
        - ssh-ed25519 BBB...
    - name: bob
      keys:
        - ssh-rsa CCC...

tasks:
  - name: Install authorized keys
    authorized_key:
      user: "{{ item.0.name }}"
      key: "{{ item.1 }}"
    with_subelements:
      - "{{ users }}"
      - keys
```

### `with_nested`
```yaml
- name: Combine two lists
  debug:
    msg: "user={{ item[0] }} env={{ item[1] }}"
  with_nested:
    - [ 'alice', 'bob' ]
    - [ 'dev', 'prod' ]
```

### `until` (retries with condition)
```yaml
- name: Wait for service to be healthy
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 30
  delay: 5
```

---

## 4) Loop controls: `loop_control`
```yaml
- name: Show index and label
  debug:
    msg: "index={{ idx }} item={{ item }}"
  loop: [a, b, c]
  loop_control:
    index_var: idx
    label: "{{ item | upper }}"
```

---

## 5) Combining `when` with registered results
```yaml
- name: Try command
  command: /usr/bin/might_fail
  register: cmd
  ignore_errors: yes

- name: Do recovery if failed
  debug:
    msg: "Recovering..."
  when: cmd is failed
```

> Other helpful tests: `is succeeded`, `is skipped`, `is changed`

---

## 6) Example putting it all together
```yaml
- hosts: app
  tasks:
    - name: Deploy package list
      yum:
        name: "{{ item }}"
        state: present
      loop: "{{ ['nginx', 'jq', 'curl'] }}"
      when: ansible_facts['os_family'] == 'RedHat'

    - name: Hit health check until it becomes 200
      uri:
        url: http://localhost/health
        return_content: yes
      register: hc
      until: hc.status == 200 and ('UP' in hc.content)
      retries: 20
      delay: 3
      changed_when: false

    - name: Mark failed if health body has ERROR
      debug:
        msg: "{{ hc.content }}"
      failed_when: "'ERROR' in hc.content"
      changed_when: false
```

---
