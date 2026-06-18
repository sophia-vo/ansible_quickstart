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
    apt-get install -y openssh-server sudo python3 && \
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

```yaml
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

## hen test Ansible:

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

Create a file named playbook.yaml in your ansible_quickstart directory, that you created earlier, with the following content:

```yaml
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
ansible-playbook -i inventory.ini playbook.yaml
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
