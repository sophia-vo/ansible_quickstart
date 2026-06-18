## Install Ansible.

```bash
pip install ansible
```

## Create a project folder on your filesystem.

```bash
mkdir ansible_quickstart && cd ansible_quickstart
```

## Building an inventory:

Inventories organize managed nodes in centralized files that provide Ansible with system information and network locations. Using an inventory file, Ansible can manage a large number of hosts with a single command.

Create a file named inventory.ini in the ansible_quickstart directory that you created in the preceding step.

Add a new [myhosts] group to the inventory.ini file and specify the IP address or fully qualified domain name (FQDN) of each host system.

## Local containers with Docker:

From your ansible_quickstart directory:

```bash
mkdir -p lab/ssh
cd lab
```

## Create or copy your SSH public key:

```bash
test -f ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C ansible-lab
cp ~/.ssh/id_ed25519.pub ssh/authorized_keys
```

## Create Dockerfile (in lab):

```Dockerfile
FROM ubuntu:24.04

RUN apt-get update && \
    apt-get install -y openssh-server sudo python3 python3-apt && \
    rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash ansible && \
    echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible && \
    chmod 0440 /etc/sudoers.d/ansible

RUN mkdir -p /var/run/sshd /home/ansible/.ssh

COPY ssh/authorized_keys /home/ansible/.ssh/authorized_keys

RUN chown -R ansible:ansible /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh && \
    chmod 600 /home/ansible/.ssh/authorized_keys

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D", "-e"]
```

## Create docker-compose.yml (in lab):

```yml
services:
  node1:
    build: .
    container_name: ansible-node1
    ports:
      - "2221:22"

  node2:
    build: .
    container_name: ansible-node2
    ports:
      - "2222:22"

  node3:
    build: .
    container_name: ansible-node3
    ports:
      - "2223:22"
```

## In separate terminal: (make sure Docker Desktop is installed)

```bash
open -a Docker
```

## Check it with:

```bash
docker info
```

## Back in intial terminal in lab:

Start the containers:

```bash
docker compose up -d --build
```

## Go back to your ansible_quickstart directory:

```bash
cd ..
```

## Create inventory.ini:

```ini
[myhosts]
node1 ansible_host=127.0.0.1 ansible_port=2221
node2 ansible_host=127.0.0.1 ansible_port=2222
node3 ansible_host=127.0.0.1 ansible_port=2223

[myhosts:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
```

## Test SSH manually:

```bash
ssh -p 2221 ansible@127.0.0.1
```

Then `exit` back to ansible_quickstart.

## Then test Ansible:

```bash
ansible-inventory -i inventory.ini --list
ansible -i inventory.ini myhosts -m ping
```

You should see something like:

```text
node1 | SUCCESS => {
    "ping": "pong"
}
node2 | SUCCESS => {
    "ping": "pong"
}
node3 | SUCCESS => {
    "ping": "pong"
}
```

## Creating a playbook:

**Playbook**
A list of plays that define the order in which Ansible performs operations, from top to bottom, to achieve an overall goal.

**Play**
An ordered list of tasks that maps to managed nodes in an inventory.

**Task**
A reference to a single module that defines the operations that Ansible performs.

**Module**
A unit of code or binary that Ansible runs on managed nodes. Ansible modules are grouped in collections with a Fully Qualified Collection Name (FQCN) for each module.

Create a file named playbook.yml in your ansible_quickstart directory, that you created earlier, with the following content:

```yml
- name: My first play
  hosts: myhosts
  tasks:
   - name: Ping my hosts
     ansible.builtin.ping:

   - name: Print message
     ansible.builtin.debug:
       msg: Hello world
```

## Run your playbook.

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Should return:

```text
PLAY [My first play] *************************************************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************
ok: [node3]
ok: [node2]
ok: [node1]

TASK [Ping my hosts] *************************************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Print message] *************************************************************************************************************************************
ok: [node1] => {
    "msg": "Hello world"
}
ok: [node2] => {
    "msg": "Hello world"
}
ok: [node3] => {
    "msg": "Hello world"
}

PLAY RECAP ***********************************************************************************************************************************************
node1                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node3                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Gathering Facts from all servers:

Current directory:
```
ansible_quickstart/
  inventory.ini
  playbook.yml
  lab/
```

Create **gather_facts.yml:**
```
- name: Gather basic server facts
  hosts: myhosts
  gather_facts: true

  tasks:
    - name: Show OS information
      ansible.builtin.debug:
        msg:
          - "Host: {{ inventory_hostname }}"
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
          - "Architecture: {{ ansible_architecture }}"
          - "Memory MB: {{ ansible_memtotal_mb }}"
          - "Python: {{ ansible_python_version }}"
```

Run it: `ansible-playbook -i inventory.ini gather_facts.yml`

Should return something similar to:
```
PLAY [Gather basic server facts] ***********************************************

TASK [Gathering Facts] *********************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Show OS information] *****************************************************
ok: [node1] => {
    "msg": [
        "Host: node1",
        "OS: Ubuntu 24.04",
        "Architecture: aarch64",
        "Memory MB: 7836",
        "Python: 3.12.3"
    ]
}
ok: [node2] => {
    "msg": [
        "Host: node2",
        "OS: Ubuntu 24.04",
        "Architecture: aarch64",
        "Memory MB: 7836",
        "Python: 3.12.3"
    ]
}
ok: [node3] => {
    "msg": [
        "Host: node3",
        "OS: Ubuntu 24.04",
        "Architecture: aarch64",
        "Memory MB: 7836",
        "Python: 3.12.3"
    ]
}

PLAY RECAP *********************************************************************
node1                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node3                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

## Creating and Validating Server Specs

Create a folder: `mkdir -p vars`.

Create `vars/server_specs.yml`:

```yml
expected_specs:
  os_family: Debian
  min_memory_mb: 128
  required_user: ansible
  required_packages:
    - openssh-server
    - sudo
    - python3
    - curl
```

We include curl since containers may not have it installed to test compliance failure later.

Create `validate_build.yml`:
```yml
- name: Validate server build against expected specs
  hosts: myhosts
  gather_facts: true
  become: true

  vars_files:
    - vars/server_specs.yml

  tasks:
    - name: Collect installed package facts
      ansible.builtin.package_facts:
        manager: auto

    - name: Check if required user exists
      ansible.builtin.getent:
        database: passwd
        key: "{{ expected_specs.required_user }}"
      register: user_check
      failed_when: false

    - name: Build list of missing packages
      ansible.builtin.set_fact:
        missing_packages: >-
          {{
            expected_specs.required_packages
            | reject('in', ansible_facts.packages.keys())
            | list
          }}

    - name: Build compliance findings
      ansible.builtin.set_fact:
        compliance_findings:
          host: "{{ inventory_hostname }}"
          os_family_actual: "{{ ansible_os_family }}"
          os_family_expected: "{{ expected_specs.os_family }}"
          memory_mb_actual: "{{ ansible_memtotal_mb }}"
          memory_mb_required: "{{ expected_specs.min_memory_mb }}"
          required_user: "{{ expected_specs.required_user }}"
          user_exists: "{{ user_check.ansible_facts.getent_passwd is defined }}"
          missing_packages: "{{ missing_packages }}"
          compliant: >-
            {{
              ansible_os_family == expected_specs.os_family
              and ansible_memtotal_mb >= expected_specs.min_memory_mb
              and user_check.ansible_facts.getent_passwd is defined
              and missing_packages | length == 0
            }}

    - name: Print compliance result
      ansible.builtin.debug:
        var: compliance_findings

    - name: Fail if server is not compliant
      ansible.builtin.assert:
        that:
          - compliance_findings.compliant | bool
        fail_msg: "Server {{ inventory_hostname }} is NOT compliant: {{ compliance_findings }}"
        success_msg: "Server {{ inventory_hostname }} is compliant."
```

Run it: `ansible-playbook -i inventory.ini validate_build.yml`

Expected error output:
```
PLAY [Validate server build against expected specs] ******************************************************************************************************

TASK [Gathering Facts] ***********************************************************************************************************************************
ok: [node2]
ok: [node3]
ok: [node1]

TASK [Collect installed package facts] *******************************************************************************************************************
ok: [node2]
ok: [node3]
ok: [node1]

TASK [Check if required user exists] *********************************************************************************************************************
ok: [node2]
ok: [node3]
ok: [node1]

TASK [Build list of missing packages] ********************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Build compliance findings] *************************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Print compliance result] ***************************************************************************************************************************
ok: [node1] => {
    "compliance_findings": {
        "compliant": false,
        "host": "node1",
        "memory_mb_actual": 7836,
        "memory_mb_required": 128,
        "missing_packages": [
            "curl"
        ],
        "os_family_actual": "Debian",
        "os_family_expected": "Debian",
        "required_user": "ansible",
        "user_exists": true
    }
}
ok: [node2] => {
    "compliance_findings": {
        "compliant": false,
        "host": "node2",
        "memory_mb_actual": 7836,
        "memory_mb_required": 128,
        "missing_packages": [
            "curl"
        ],
        "os_family_actual": "Debian",
        "os_family_expected": "Debian",
        "required_user": "ansible",
        "user_exists": true
    }
}
ok: [node3] => {
    "compliance_findings": {
        "compliant": false,
        "host": "node3",
        "memory_mb_actual": 7836,
        "memory_mb_required": 128,
        "missing_packages": [
            "curl"
        ],
        "os_family_actual": "Debian",
        "os_family_expected": "Debian",
        "required_user": "ansible",
        "user_exists": true
    }
}

TASK [Fail if server is not compliant] *******************************************************************************************************************
[ERROR]: Task failed: Action failed: Server node1 is NOT compliant: {'host': 'node1', 'os_family_actual': 'Debian', 'os_family_expected': 'Debian', 'memory_mb_actual': 7836, 'memory_mb_required': 128, 'required_user': 'ansible', 'user_exists': True, 'missing_packages': ['curl'], 'compliant': False}
Origin: /Users/sophiavo/ansible_quickstart/validate_build.yml:54:9

52           var: compliance_findings
53
54       - name: Fail if server is not compliant
           ^ column 9

fatal: [node1]: FAILED! => {
    "assertion": "compliance_findings.compliant | bool",
    "changed": false,
    "evaluated_to": false,
    "msg": "Server node1 is NOT compliant: {'host': 'node1', 'os_family_actual': 'Debian', 'os_family_expected': 'Debian', 'memory_mb_actual': 7836, 'memory_mb_required': 128, 'required_user': 'ansible', 'user_exists': True, 'missing_packages': ['curl'], 'compliant': False}"
}
[ERROR]: Task failed: Action failed: Server node2 is NOT compliant: {'host': 'node2', 'os_family_actual': 'Debian', 'os_family_expected': 'Debian', 'memory_mb_actual': 7836, 'memory_mb_required': 128, 'required_user': 'ansible', 'user_exists': True, 'missing_packages': ['curl'], 'compliant': False}
Origin: /Users/sophiavo/ansible_quickstart/validate_build.yml:54:9

52           var: compliance_findings
53
54       - name: Fail if server is not compliant
           ^ column 9

fatal: [node2]: FAILED! => {
    "assertion": "compliance_findings.compliant | bool",
    "changed": false,
    "evaluated_to": false,
    "msg": "Server node2 is NOT compliant: {'host': 'node2', 'os_family_actual': 'Debian', 'os_family_expected': 'Debian', 'memory_mb_actual': 7836, 'memory_mb_required': 128, 'required_user': 'ansible', 'user_exists': True, 'missing_packages': ['curl'], 'compliant': False}"
}
[ERROR]: Task failed: Action failed: Server node3 is NOT compliant: {'host': 'node3', 'os_family_actual': 'Debian', 'os_family_expected': 'Debian', 'memory_mb_actual': 7836, 'memory_mb_required': 128, 'required_user': 'ansible', 'user_exists': True, 'missing_packages': ['curl'], 'compliant': False}
Origin: /Users/sophiavo/ansible_quickstart/validate_build.yml:54:9

52           var: compliance_findings
53
54       - name: Fail if server is not compliant
           ^ column 9

fatal: [node3]: FAILED! => {
    "assertion": "compliance_findings.compliant | bool",
    "changed": false,
    "evaluated_to": false,
    "msg": "Server node3 is NOT compliant: {'host': 'node3', 'os_family_actual': 'Debian', 'os_family_expected': 'Debian', 'memory_mb_actual': 7836, 'memory_mb_required': 128, 'required_user': 'ansible', 'user_exists': True, 'missing_packages': ['curl'], 'compliant': False}"
}

PLAY RECAP ***********************************************************************************************************************************************
node1                      : ok=6    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
node2                      : ok=6    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
node3                      : ok=6    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0  
```

## Remediate a failed check

`touch remediate_packages.yml`:
```yml
- name: Remediate missing baseline packages
  hosts: myhosts
  gather_facts: true
  become: true

  vars_files:
    - vars/server_specs.yml

  tasks:
    - name: Install required baseline packages
      ansible.builtin.apt:
        name: "{{ expected_specs.required_packages }}"
        state: present
        update_cache: true
```
Before running remediation in real environments, use check mode:
`ansible-playbook -i inventory.ini remediate_packages.yml --check`

Or diff mode:
`ansible-playbook -i inventory.ini remediate_packages.yml --check --diff`


Run it: `ansible-playbook -i inventory.ini remediate_packages.yml`

Expected output:
```
PLAY [Remediate missing baseline packages] **********************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Install required baseline packages] ***********************************************************************************************************
changed: [node3]
changed: [node1]
changed: [node2]

PLAY RECAP ******************************************************************************************************************************************
node1                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node3                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Then rerun validation: `ansible-playbook -i inventory.ini validate_build.yaml`

Expected output:
```
PLAY [Validate server build against expected specs] **************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************
ok: [node3]
ok: [node1]
ok: [node2]

TASK [Collect installed package facts] ***************************************************************************************************************
ok: [node3]
ok: [node2]
ok: [node1]

TASK [Check if required user exists] *****************************************************************************************************************
ok: [node3]
ok: [node2]
ok: [node1]

TASK [Build list of missing packages] ****************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Build compliance findings] *********************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Print compliance result] ***********************************************************************************************************************
ok: [node1] => {
    "compliance_findings": {
        "compliant": true,
        "host": "node1",
        "memory_mb_actual": 7836,
        "memory_mb_required": 128,
        "missing_packages": [],
        "os_family_actual": "Debian",
        "os_family_expected": "Debian",
        "required_user": "ansible",
        "user_exists": true
    }
}
ok: [node2] => {
    "compliance_findings": {
        "compliant": true,
        "host": "node2",
        "memory_mb_actual": 7836,
        "memory_mb_required": 128,
        "missing_packages": [],
        "os_family_actual": "Debian",
        "os_family_expected": "Debian",
        "required_user": "ansible",
        "user_exists": true
    }
}
ok: [node3] => {
    "compliance_findings": {
        "compliant": true,
        "host": "node3",
        "memory_mb_actual": 7836,
        "memory_mb_required": 128,
        "missing_packages": [],
        "os_family_actual": "Debian",
        "os_family_expected": "Debian",
        "required_user": "ansible",
        "user_exists": true
    }
}

TASK [Fail if server is not compliant] ***************************************************************************************************************
ok: [node1] => {
    "changed": false,
    "msg": "Server node1 is compliant."
}
ok: [node2] => {
    "changed": false,
    "msg": "Server node2 is compliant."
}
ok: [node3] => {
    "changed": false,
    "msg": "Server node3 is compliant."
}

PLAY RECAP *******************************************************************************************************************************************
node1                      : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node3                      : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0  
```

## Generate a Simple Report

Create:
```bash
mkdir -p reports
touch validation_report.yml
```

validation_report.yml:
```yml
- name: Generate server validation report
  hosts: myhosts
  gather_facts: true
  become: true

  vars_files:
    - vars/server_specs.yml

  tasks:
    - name: Collect installed package facts
      ansible.builtin.package_facts:
        manager: auto

    - name: Check if required user exists
      ansible.builtin.getent:
        database: passwd
        key: "{{ expected_specs.required_user }}"
      register: user_check
      failed_when: false

    - name: Build list of missing packages
      ansible.builtin.set_fact:
        missing_packages: >-
          {{
            expected_specs.required_packages
            | reject('in', ansible_facts.packages.keys())
            | list
          }}

    - name: Build report row
      ansible.builtin.set_fact:
        report_row:
          host: "{{ inventory_hostname }}"
          os_family: "{{ ansible_os_family }}"
          memory_mb: "{{ ansible_memtotal_mb }}"
          user_exists: "{{ user_check.ansible_facts.getent_passwd is defined }}"
          missing_packages: "{{ missing_packages }}"
          compliant: >-
            {{
              ansible_os_family == expected_specs.os_family
              and ansible_memtotal_mb >= expected_specs.min_memory_mb
              and user_check.ansible_facts.getent_passwd is defined
              and missing_packages | length == 0
            }}

    - name: Print report row
      ansible.builtin.debug:
        var: report_row

    - name: Save host report locally
      ansible.builtin.copy:
        content: "{{ report_row | to_nice_json }}"
        dest: "reports/{{ inventory_hostname }}_validation.json"
      delegate_to: localhost
      become: false
```

Run it: `ansible-playbook -i inventory.ini validation_report.yml`

Expected output:
```
PLAY [Generate server validation report] *************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************
ok: [node3]
ok: [node1]
ok: [node2]

TASK [Collect installed package facts] ***************************************************************************************************************
ok: [node2]
ok: [node1]
ok: [node3]

TASK [Check if required user exists] *****************************************************************************************************************
ok: [node1]
ok: [node3]
ok: [node2]

TASK [Build list of missing packages] ****************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Build report row] ******************************************************************************************************************************
ok: [node1]
ok: [node2]
ok: [node3]

TASK [Print report row] ******************************************************************************************************************************
ok: [node1] => {
    "report_row": {
        "compliant": true,
        "host": "node1",
        "memory_mb": 7836,
        "missing_packages": [],
        "os_family": "Debian",
        "user_exists": true
    }
}
ok: [node2] => {
    "report_row": {
        "compliant": true,
        "host": "node2",
        "memory_mb": 7836,
        "missing_packages": [],
        "os_family": "Debian",
        "user_exists": true
    }
}
ok: [node3] => {
    "report_row": {
        "compliant": true,
        "host": "node3",
        "memory_mb": 7836,
        "missing_packages": [],
        "os_family": "Debian",
        "user_exists": true
    }
}

TASK [Save host report locally] **********************************************************************************************************************
changed: [node2 -> localhost]
changed: [node3 -> localhost]
changed: [node1 -> localhost]

PLAY RECAP *******************************************************************************************************************************************
node1                      : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node2                      : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
node3                      : ok=7    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

Check the reports:

`ls reports`:
```
node1_validation.json   node2_validation.json   node3_validation.json
```

`cat reports/node1_validation.json`:
```
{
    "compliant": true,
    "host": "node1",
    "memory_mb": 7836,
    "missing_packages": [],
    "os_family": "Debian",
    "user_exists": true
}%  
```