After my previous tutorial, by now you should have a barebones machine, flashed with Ubuntu Server LTS, configured declaratively via Autoinstall. This machine can automatically connect to you home network because it has all necessary settings, and you can SSH into it right after install.

In this section, I will show how can you apply configuration to this machine using Ansible. Instead of doing everything by hand through shell (installing packages, creating folders etc.) everything will be done through Ansible in a declarative, reusable manner. 

Even if at some time later you decide that you need to reflash your server from scratch these playbooks will allow you to set up everything to desired state in **short period of time**, with **minimal manual intervention**.

I will present playbooks I've created to bring my home server to desired state.


# 1 What is Ansible, plus necessary terminology and concepts

I don't want to make this post an introduction to Ansible. There are many high-quality sources one can learn about Ansible from. I will present just some concepts that are necessary to know in order to understand my approach.

**Ansible** is an open-source automation tool that provides a simple and efficient way to automate IT infrastructure tasks. It uses a declarative language called YAML to describe the desired state of systems and performs actions to **ensure that the systems match that state.** 

This is different from shell scripts in many ways: 

- Ansible uses a declarative approach, where you define the desired state of the system and let Ansible handle the execution details. Shell scripts, on the other hand, are imperative, where you **explicitly list out the steps to be executed.**
- Ansible ensures **idempotent execution**, which means the same playbook can be run multiple times without causing unintended side effects. Ansible checks the current state of the system and only makes necessary changes to achieve the desired state. Shell scripts often **require manual checks and conditions** to avoid unintended changes or duplicate actions.
- Ansible uses YAML, a **human-readable and easy-to-write format**, for defining playbooks and tasks. This makes Ansible playbooks more readable and maintainable compared to shell scripts, which often involve more complex syntax.


This makes Ansible more convenient to use instead of shell scripts, if that's only possible. Another thing is that Ansible is **push-based**. While we'd run shell scripts directly on remote host, with Ansible we "push" the desired configuration from our local machine.


I've organized my playbooks as Ansible Roles. This allows me to self-contain each of my tasks. This was very useful during experimenting phase when doing my home server, because if some role malfunctioned and I had problems with fixing it, or even when I changed my thought, all that was needed from me was to comment out particular role from list of main roles to execute. 

Roles, which still need polishing or I'm experimenting with, I can run from separate playbook, and have my battle-tested main one unaffected.


## File structure


My project structure follows standard Ansible file hierarchy:

```text
my_project/
├── inventory/
│   ├── hosts
│   └── group_vars/
│       └── all.yml
├── roles/
│   └── my_role/
│       ├── defaults/
│       │   └── main.yml
│       ├── files/
│       │   └── myfile.txt
│       ├── handlers/
│       │   └── main.yml
│       ├── meta/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       │   └── mytemplate.j2
│       ├── vars/
│       │   └── main.yml
│       └── README.md
└── main.yml

```

From `main.yml` I run all my tasks which I know are ready to be applied to my server. 

If I know something isn't ready and want to test more, then I usually create separate playbook with that specific role to separate it from my known working set.

All my roles reside under `roles/` folder. Each role usually has their `tasks` and `files` folder. For my needs I usually do not need to use any more folders.





Going through important points inside my main.yaml file:

```yml
- hosts: local-1
  become: yes
  vars_files:
    - vars/main.yaml
  ignore_errors: "{{ ansible_check_mode }}"
```

This code block defines an Ansible playbook that targets a host named "local-1". The playbook executes with elevated privileges (`become: yes`) to perform tasks that require administrative access. It also includes a list of variable files (`vars/main.yaml`) that contain predefined variables used during playbook execution.

Setting `ignore_errors` to `ansible_check_mode` allows the playbook to continue executing even if errors occur during check mode.

My playbook sources some global variables which are frequently reused in multiple tasks:

```yml
# Administrative user
admin_username: "admin" # change to yours
admin_home_path: "/home/admin" # change to yours

# Dedicated user for running containers
container_user: "dockeruser"
container_group: "dockeruser"
container_user_uid: 2000
container_user_gid: 2000
```

`admin_username` and `admin_home_path`  are for default user's username and home directory path. These are the same that we set in `identity` section in autoinstall.yaml:

```yml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: ubuntu-server
    username: admin # can be custom
    password: "PWD HASH"
```





```yml
  pre_tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
```

This task uses the apt module to update the package cache on the system. It ensures that the latest package information is available for package management.

```yml
roles: 
    - role: motd # setup MOTD banner
    - role: sshd_harden # harden SSHD config
    - role: sshd_banner # setup SSHD banner
    - role: sudo # setup sudo config
    # - role: autoupdate-Debian
    - role: hifis.unattended_upgrades
      # Do automatic removal of all unused dependencies after the upgrade
      unattended_remove_unused_dependencies: true
      # Remove unused automatically installed kernel-related packages (kernel images, kernel headers and kernel version locked tools)
      unattended_remove_unused_kernel_packages: true 
      unattended_update_days: {"Tue"}
      # Define verbosity level of APT for periodic runs. The output will be sent to root.
      unattended_verbose: 2
      when: ansible_facts['os_family'] == 'Debian' 
    - role: tmux # install and configure tmux
    - role: vim # ensure proper vim config
    - role: zsh # install zsh on machine, set it for admin suer
    - role: gantsign.oh-my-zsh # install oh-my-zsh
      users:
      - username: admin
    - role: zsh-plugins # configure oh-my-zsh plugins
    - role: docker-infra # setup docker infrastructure
    - role: ufw # configure UFW firewall
```

These roles are executed in the playbook. They are the heart of my automation. There are various roles which I've written for my needs and two which are external. 

These roles ensure my target system will have various configuration applied, including:

- MOTD and SSHD banner for SSH connections
- my hardened configuration for `sudo` and `sshd`
- unattended system upgrades enabled, thanks to `hifis.unattended_upgrades` role
- my configuration files deployed  for `tmux`, `vim` and `zsh`
- `oh-my-zsh` installed for my main user thanks to `gantsign.oh-my-zsh` role, along with necessary plugins used my be
- ensure docker is available on remote machine, along with basic configuration for my projects
- ensure proper UFW firewall configuration


Those two external roles need to be installed separately. To learn more how they function, what tunable options they have I suggest visiting their Ansible Galaxy page or Github page, if available (the same for any other role).

To be able to use my playbook, these roles will need to be installed on local machine:

```bash
ansible-galaxy install gantsign.oh-my-zsh
ansible-galaxy install hifis.unattended_upgrades
```

Each my role usually follows simple pattern:
- it ensures that particular packages are present on remote server
- it usually copies some kind of config file to remote server
- it ensures proper permissions are set
- if needed, it restarts some daemons, usually via handlers

```yml
- name: Install vim package
  apt:
    name: vim
    state: present
```

For configuration files, I just copy prepared config onto machine. There is of course potential to introduce modularity to this process, and either use Jinja2 templates to create more generic configuration files with various config sections controllable via variables and conditionals, or use modules such as `lineinfile` or `blockinfile` to modify files on remote host in place, but in my case **it was much easier for me to just modify my config files locally when there is need and push this new file to remote server.**

```yml
- name: copy .vimrc config files to remote hosts
  copy:
    src: files/.vimrc
    dest: "{{ admin_home_path }}/.vimrc"
    owner: "{{ admin_username }}"
    group: "{{ admin_username }}"
    mode: "0644"
```

Some roles are for daemons which after supplying new configuration, need to be reloaded or restarted. I have necessary handlers prepared for this scenario:

```yml
- name: "Copy sshd_config.d 99-banner.conf files"
  copy:
    src: "files/99-banner.conf"
    dest: "/etc/ssh/sshd_config.d/99-banner.conf"
    mode: 0600
    owner: root
    group: root
  notify: Reload SSHD
```

```yml
handlers:
    - name: "Reload UFW"
      ufw:
        # state: enabled
        state: reloaded

    - name: "Reload SSHD"
      service:
        name: sshd
        state: restarted
```

I need to mention changes I make to sshd, sudo and firewall as these are definitely changes you may want to tweak for your need.


Here's my config for sshd daemon. It's a snap-in file installed on remote host in `/etc/ssh/sshd_config.d/91-harden.conf` location, so it doesn't replace default config. This choice was deliberate as I did not want to mess with default config.

```ini
# Disable root login via SSH
PermitRootLogin no

# Disable password authentication, only allow key-based authentication
PasswordAuthentication no

# Disable X11 forwarding, which allows remote graphical applications
X11Forwarding no

# Change the SSH port to 1337 (example port, you can use your desired port)
Port 1337

# Do not allow empty passwords
PermitEmptyPasswords no

# Disable challenge-response authentication (e.g., keyboard-interactive)
ChallengeResponseAuthentication no

# Disable GSSAPI authentication (Kerberos authentication)
GSSAPIAuthentication no

# Specifies the users allowed to log in via SSH
AllowUsers admin ansible
```

The most important change I introduced is different SSH port. If you want to keep SSH on port 22, please make sure to modify that line. Other settings involve X11 forwarding, root login, disabled password authentication,


```ini
# https://www.tecmint.com/sudoers-configurations-for-setting-sudo-in-linux/
# more: man sudoers

# If set, sudo will insult users when they enter an incorrect password.
Defaults	insults

# Defaults	logfile="/var/log/sudo.log"
# To log hostname and the four-digit year in the custom log file, use log_host and log_year parameters respectively as follows:
Defaults  log_host, log_year, logfile="/var/log/sudo.log"


# Number of minutes that can elapse before sudo will ask for a passwd again
Defaults        timestamp_timeout=20
```

For `sudo`, I enable insults, change logging settings and change number of minutes `sudo` will not ask me again for password.

Also I think it is very important to look at UFW configuration, because it is also very much tailored for my needs. My UFW playbook:

- copies created by me custom UFW profiles onto remote machine
- enables those custom profiles
- sets default policies for incoming and outgoing traffic

```yml
- name: Transfer custom UFW profile files
  copy:
    #  loop directive is used to iterate over each file matched by the fileglob lookup plugin
    src: "{{ item }}"
    dest: "/etc/ufw/applications.d/"
    owner: root
    group: root
    mode: 0644
  with_fileglob:
    - "files/*.profile"
  notify: "Reload UFW"
```

When using ufw module, there is a distinction between using `port:` and `name:` directive. 

When you provide the name of a service rather than a port number ufw looks the name up in `/etc/services` and reads the port number from it.

```bash
# root@domain:/home/qbus/docker-compose# ufw app list
# WARN: Skipping 'http': also in /etc/services
# WARN: Skipping 'https': also in /etc/services
# WARN: Skipping 'ntp': also in /etc/services
# WARN: Skipping 'ssh': also in /etc/services
# Available applications:
#   DHCPv6
#   OpenSSH
#   dhcp
#   dns
#   qbittorrent
#   ssh-hardened
```

This is why, even though I've created profiles named `ssh`, `http`, `https`, I had to use `port:` to reference them because port is for those services which are defined in `/etc/services`.

For custom profiles, which also do not share their name with ports defined in  `/etc/services`, use `name:`

```yml
- name: Enable default ports
  ufw:
    rule: allow
    port: '{{ item.port }}'
    comment: '{{ item.comment }}'
  loop:
    - port: ssh
      comment: "SSH (standard port, for endlessh)"
    - port: https
      comment: HTTP/S
    - port: http
      comment: HTTP
    - port: ntp
      comment: NTP 
  notify: "Reload UFW"

- name: Enable custom profiles
  ufw:
    rule: allow
    name: '{{ item.name }}'
    comment: '{{ item.comment }}'
  loop:
    - name: ssh-hardened
      comment: "SSH (custom hardened port)"
    - name: dns
      comment: "DNS"
    - name: qbittorrent
      comment: "Qbittorrent client"
  notify: "Reload UFW"
```

All my custom roles are defined in `ufw/files/*.profile`. 

Coming back to custom SSH port, if want to specify your own custom port number for SSH, remember to modify it both in sshd config AND inside `ssh-hardened.profile` - otherwise there is very high chance you will get locked out of remote machine!


```ini
[ssh-hardened]
title=SSH, hardened through custom config
description=Secure Shell protocol
ports=1337/tcp
```

If you don't want to expose particular port which I decided to do, remember to remove it or at least comment it out, as you don't want random ports exposed on your machine, especially if it's connected to the internet.


The last two things I want do describe are `ansible.cfg` and `inventory` files.

My ansible.cfg is very simple:

```ini
[defaults]
remote_user = ansible
private_key_file = /path/to/id_rsa # private key
host_key_checking = False
```

It only sets `ansible` as user to in through SSH as during playbook execution, along with path to SSH private key. It corresponds to whatever was put into Autoinstall.yaml for `ansible` user:

```yml
#cloud-config
autoinstall:
  version: 1
  user-data:
    users:
      - name: ansible
        gecos: Ansible User
        sudo: "ALL=(ALL) NOPASSWD:ALL"
        groups: sudo
        lock_passwd: true
        shell: /bin/bash
        ssh_authorized_keys:
          - "SSH PUBLIC KEY"
```

`inventory` just contains static IP address for my server (I have only one) in my LAN network. There are two positions, one references different SSH port which is changes after playbook is run first time. After that all I need is to change `hosts` in `main.yml` to reference proper host.

```ini
[local-1-pre]
192.168.100.69
[local-1]
192.168.100.69 ansible_port=1337

```

Now, all that is needed is to run Ansible playbook and wait couple minutes before al changes are applied.

```bash
ansible-playbook -i inventory main.yaml
```

![[Pasted image 20230529172118.png]]

After that you can SSH into server and enjoy your ready to use setup :) Here's my using `tmux`, after applying my roles

![[Pasted image 20230529203257.png]]


