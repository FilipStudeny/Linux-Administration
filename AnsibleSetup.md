# Ansible setup
```
                 ----------
                 | ROUTER |
                 ----------
                     |
     -------------------------------------
    |          |           |              |
-------  -----------  ------------  ------------
| web |  | ansible |  | client-1 |  | client-2 |
-------  -----------  ------------  ------------
```


- admin - Administration server, has Ansible installed
- client-1 / client-2 - Modyfied by admin server

## SSH setup

1. Connect to each station via SSH
```
ssh <IP>
```
2. Create an SSH key pair (with passphrase) for normal user account
```
ssh-keygen -t ed25519 -C "user key"
```
3. Copy key to each client
```
ssh-copy-id -i <ssh pub file> <user>@<IP>
```
4. Create SSH specific to Ansible
```
ssh-keygen -t ed25519 -C "user key"

ssh-copy-id -i <ssh pub file> <user>@<IP>
```


## Ansible
Install package
```
sudo apt ansible
```

Create inventory list with IP addresses of clients
```
vim inventory
__________________________________
192.168.1.10
192.168.1.11
```
Test connections
```
ansible all --key-file ~/.ssh/ansible -i inventory -u root -m ping
```

Should return for each IP in file on succesfull connection
```
192.168.1.11 | SUCCESS => {
    "ansible_facts":{
        "disocvered_interpreter:python":"/usr/bin/python"
    },
    "changed" : false,
    "ping" : "pong"
}
```
## Basics
Create ansible config - **ansible.cfg** file
```
vim ansible.cfg
__________________________________
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible
```

Shorten command for testing connections using **ansible.cfg**:
```
#FROM
ansible all --key-file ~/.ssh/ansible -i inventory -u root -m ping

#TO 
ansible all -u root -m ping
```

Show hosts:
```
ansible all --list-hosts
__________________________________
hosts (2):
    192.168.1.10
    192.168.1.11
```

Gather client data:
```
ansible all -u root -m gather_facts
__________________________________
- IP addresses
- OS version
- Keys, visible password etc...
```

## Making changes via Ansible - Basics
Updating packages
```
ansible all -u root -m shell -a 'yum update'
__________________________________
192.168.1.10 | CHANGED | rc=0 >>
CentOS Stream 9 - BaseOS                         12 kB/s |  11 kB     00:00
CentOS Stream 9 - AppStream                      20 kB/s |  11 kB     00:00
CentOS Stream 9 - Extras packages                19 kB/s |  12 kB     00:00
Závislosti vyřešeny.
Není co dělat.
Hotovo!

192.168.1.11 | CHANGED | rc=0 >>
CentOS Stream 9 - BaseOS                         15 kB/s |  11 kB     00:00
CentOS Stream 9 - AppStream                      13 kB/s |  11 kB     00:00
CentOS Stream 9 - Extras packages                13 kB/s |  12 kB     00:00
Závislosti vyřešeny.
Není co dělat.
Hotovo!
```

Run command with root password
```
ansible all -u root -m shell -a 'yum update' --become --ask-become-pass
__________________________________
BECOME password: <root password>

192.168.1.10 | CHANGED | rc=0 >>
CentOS Stream 9 - BaseOS                         12 kB/s |  11 kB     00:00
CentOS Stream 9 - AppStream                      20 kB/s |  11 kB     00:00
CentOS Stream 9 - Extras packages                19 kB/s |  12 kB     00:00
Závislosti vyřešeny.
Není co dělat.
Hotovo!

192.168.1.11 | CHANGED | rc=0 >>
CentOS Stream 9 - BaseOS                         15 kB/s |  11 kB     00:00
CentOS Stream 9 - AppStream                      13 kB/s |  11 kB     00:00
CentOS Stream 9 - Extras packages                13 kB/s |  12 kB     00:00
Závislosti vyřešeny.
Není co dělat.
Hotovo!
```

Install packages - Git
```
ansible all -u root -m shell -a 'yum -y install git' --become --ask-become-pass
```

## Playbooks
### List installed packages:

Create playbook: **list_packages.yml**
```
vim list_packages.yml
__________________________________
---

- hosts: all
  become: true

  tasks:
  - name: Get installed packages
    command: yum list installed
    register: yum_list

  - name: Save installed packages into file
    copy:
      content: "{{ yum_list.stdout }}"
      dest: /root/yum_list.txt
```


```
$ ansible-playbook -u root --ask-become-pass playbooks/list_packages.yml
__________________________________
BECOME password:

PLAY [all] ************************************************************************************************************************************

TASK [Gathering Facts] ************************************************************************************************************************
ok: [192.168.1.10]
ok: [192.168.1.11]

TASK [Get installed packages] *****************************************************************************************************************
changed: [192.168.1.11]
changed: [192.168.1.10]

TASK [Save installed packages into file] ******************************************************************************************************
changed: [192.168.1.11]
changed: [192.168.1.10]

PLAY RECAP ************************************************************************************************************************************
192.168.1.10               : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.1.11               : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Create users

Create group called **developers**, create directory called **/web** and give user that are part of the developers group access to this directory. Create two users **web_developer** and **web_administrator** and assign them to the developers group:

```
vim playbooks/new_user.yml
__________________________________
---

- name: Create developers group and web_developer user
  hosts: all
  become: true

  vars:
    fox: "fox"

  tasks:
    - name: Create group called Developers
      group:
        name: developers
        state: present

    - name: Create web directory with only developers access
      file:
        path: /web
        state: directory
        mode: "0770"
        owner: root
        group: developers

    - name: Create user called - web_developer
      user:
        name: web_developer
        state: present
        groups: developers
        createhome: yes
        home: /home/web_developer
        shell: /bin/bash
        password: "{{ fox | password_hash('sha512', 'A512') }}"

    - name: Create user called - web_administrator
      user:
        name: web_administrator
        state: present
        groups: developers
        createhome: yes
        home: /home/web_administrator
        shell: /bin/bash
        password: "{{ fox | password_hash('sha512', 'A512') }}"
```

### Install Apache2
Install apache and give the user web_administrator access to the directory

```
vim playbooks/install_apache.yml
__________________________________
---

- name: Install apache and grant access to web_administrator
  hosts: all
  become: true

  tasks:
    - name: Install Apache2
      package:
        name: httpd
        state: present

    - name: Ensure Apache2 is started
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Grant access to web_administrator for Apache directory
      file:
        path: /var/www/html
        state: directory
        mode: "0770"
        owner: apache
        group: apache

    - name: Add web_administrator to Apache group
      user:
        name: web_administrator
        groups: apache
        append: yes

```

### Add sudo access for web_administrator
Add sudo access for **web_administrator** so he can start/stop/restart apache service
```
vim playbooks/grant_sudo_access.yml
__________________________________
---

- name: Add sudo access for users
  hosts: all
  become: true

  tasks:
    - name: Add web_administrator to sudoers file
      lineinfile:
        path: /etc/sudoers
        line: 'web_administrator ALL=(ALL) NOPASSWD: /bin/systemctl start httpd, /bin/systemctl stop httpd, /bin/systemctl restart httpd'
        validate: 'visudo -cf %s'
      notify: Restart sudoers file check

  handlers:
    - name: Restart sudoers file check
      command: visudo -c
      ignore_errors: yes

```
