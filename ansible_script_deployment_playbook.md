# 📜Automate Script Deployment and Scheduling with Ansible

In the world of DevOps, automation is not just a convenience—it's a superpower. And when it comes to configuring remote systems at scale, **Ansible** continues to shine as one of the most elegant and efficient tools in the toolbox.

In this article, we’ll walk through an Ansible playbook designed to automate two crucial tasks:
1. **Deploying a shell script to remote servers**, and
2. **Scheduling it with crontab** so it runs at regular intervals.


![Linux-Ansible](./images/linux-ansible.png)

## 📜 Anatomy of the Playbook

Here's the heart of the automation:

```yaml
- name: Deploy Script and Make a Crontab Entry
  hosts: all
  become: true
  gather_facts: false
```

We begin by defining a playbook that runs on **all hosts**, with `become: true` to elevate privileges when needed. No facts gathering means faster execution, since we're dealing with static tasks.

### 🔧 Variables Section

```yaml
vars:
  - SCRIPT_NAME: <script-name>
  - SCRIPT_SRC: ./
  - SCRIPT_DEST: <destination-path-on-remote-server>
  - SCRIPT_USER_OWNER: "centos"
  - SCRIPT_GROUP_OWNER: "centos"

  - DEPLOY_FOR_USER: "centos"
  - MINUTE: "*/10"
  - HOUR: "*"
  - DAY: "*"
  - MONTH: "*"
  - WEEKDAY: "*"
```

These variables give you full control over where and how your script is deployed, and at what schedule it should run. Want it every hour? Change `MINUTE: 0`. Different user? Change `DEPLOY_FOR_USER`, `SCRIPT_USER_OWNER`, `SCRIPT_GROUP_OWNER`.

### 📂 Task 1: Ensure the Directory Exists

```yaml
- name: "Create the directory if it does not exist"
  ansible.builtin.file:
    path: "{{ SCRIPT_DEST }}"
    state: directory
    mode: '0755'
```

Simple and effective—if the target directory isn't there, it's created.

### 📄 Task 2: Copy the Script

```yaml
- name: "Place Script at the desired path"
  ansible.builtin.copy:
    src: "{{ SCRIPT_SRC }}/{{ SCRIPT_NAME }}"
    dest: "{{ SCRIPT_DEST }}/{{ SCRIPT_NAME }}"
    owner: "{{ SCRIPT_USER_OWNER }}"
    group: "{{ SCRIPT_GROUP_OWNER }}"
    mode: '0755'
```

This copies the script from your local directory (`SCRIPT_SRC`) to the destination on the remote server (`SCRIPT_DEST`) with proper permissions and ownership.

### ⏰ Task 3: Add a Crontab Entry

```yaml
- name: "make crontab entry for the script"
  ansible.builtin.cron:
    name: "Test Script deployed by Ansible"
    job: "{{ SCRIPT_DEST }}/{{ SCRIPT_NAME }}"
    user: "{{ DEPLOY_FOR_USER }}"
    minute: "{{ MINUTE }}"
    hour: "{{ HOUR }}"
    day: "{{ DAY }}"
    month: "{{ MONTH }}"
    weekday: "{{ WEEKDAY }}"
```

This is where the magic happens. A cron job is created (or updated if it already exists) to schedule the script’s execution. With the default values, it will run every 10 minutes.

## 📦 Complete Playbook
```yaml
# Deploy Bash Script with Ansible

- name: Deploy Script and Make a Crontab Entry
  hosts: all
  become: true
  gather_facts: false

  # update these variable as per the need
  vars:

    # script details
  - SCRIPT_NAME: <script-name>
  - SCRIPT_SRC: ./
  - SCRIPT_DEST: <destination-path-on-remote-server>
  - SCRIPT_USER_OWNER: "centos"
  - SCRIPT_GROUP_OWNER: "centos"

    # crontab details
  - DEPLOY_FOR_USER: "centos"
  - MINUTE: "*/10"
  - HOUR: "*"
  - DAY: "*"
  - MONTH: "*"
  - WEEKDAY: "*"

  tasks:

  # create operations script directory if it doesn't exist already
  - name: "Create the directory if it does not exist"
    ansible.builtin.file:
      path: "{{ SCRIPT_DEST }}"
      state: directory
      mode: '0755'

  # copy the script on the host
  - name: "Place Script at the desired path"
    ansible.builtin.copy:
      src: "{{ SCRIPT_SRC }}/{{ SCRIPT_NAME }}"
      dest: "{{ SCRIPT_DEST }}/{{ SCRIPT_NAME }}"
      owner: "{{ SCRIPT_USER_OWNER }}"
      group: "{{ SCRIPT_GROUP_OWNER }}"
      mode: '0755'

  # edit the crontab
  - name: "make crontab entry for the script"
    ansible.builtin.cron:
      name: "Test Script deployed by Ansible"
      job: "{{ SCRIPT_DEST }}/{{ SCRIPT_NAME }}"
      user: "{{ DEPLOY_FOR_USER }}"
      minute: "{{ MINUTE }}"
      hour: "{{ HOUR }}"
      day: "{{ DAY }}"
      month: "{{ MONTH }}"
      weekday: "{{ WEEKDAY }}"
```

## How to Use This Playbook

1. **Save the playbook** as `deploy_script.yml`
2. **Place your shell script** in the same directory
3. **Customize variables** if needed (script name, user, schedule)
4. Run the playbook:

```bash
ansible-playbook -i inventory.ini deploy_script.yml
```

Make sure your `inventory.ini` file lists the remote hosts you want to target.

---

## 🎯 Why Use This?

This approach is:
- **Repeatable**: Run it anytime to redeploy or update the cron schedule.
- **Scalable**: Works across any number of hosts defined in your inventory.
- **Idempotent**: Ansible ensures tasks are only applied when necessary.

Whether you’re deploying monitoring scripts, maintenance tasks, or log rotators, this playbook gives you a clean and automated pathway.

---

> *This article builds upon our previous post on [Setting up a local Ansible lab with Vagrant](./ansible_lab_vagrant.md), where we prepared the perfect sandbox to try out playbooks like this.*

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
