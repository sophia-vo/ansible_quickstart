# Ansible Quickstart — Line-by-Line Explanation


## Important notes before running
- The Docker setup publishes each container’s SSH port `22` to a different port on your laptop: `2221`, `2222`, and `2223`.

---

# 1. Install Ansible

```bash
pip install ansible
```

| Line | Explanation |
|---|---|
| `pip install ansible` | Uses Python’s package installer, `pip`, to install the `ansible` package into your current Python environment. After this, commands such as `ansible`, `ansible-playbook`, and `ansible-inventory` should be available. |

---

# 2. Create the project folder

```bash
mkdir ansible_quickstart && cd ansible_quickstart
```

| Part | Explanation |
|---|---|
| `mkdir ansible_quickstart` | Creates a new directory named `ansible_quickstart`. |
| `&&` | Runs the command on the right only if the command on the left succeeds. |
| `cd ansible_quickstart` | Changes your current terminal directory into the new project folder. |

---

# 3. Create local Docker lab folders

```bash
mkdir -p lab/ssh
cd lab
```

| Line | Explanation |
|---|---|
| `mkdir -p lab/ssh` | Creates the `lab` folder and the nested `ssh` folder. The `-p` flag means “create parent directories if needed” and do not error if they already exist. |
| `cd lab` | Moves your terminal into the `lab` directory. This matters because the Dockerfile and `docker-compose.yml` are created inside `lab`. |

---

# 4. Create or copy your SSH public key

```bash
test -f ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -C ansible-lab
cp ~/.ssh/id_ed25519.pub ssh/authorized_keys
```

| Line / Part | Explanation |
|---|---|
| `test -f ~/.ssh/id_ed25519.pub` | Checks whether the SSH public key file `~/.ssh/id_ed25519.pub` already exists. `~` means your home directory. `-f` means “is a regular file.” |
| `||` | Means “or.” The command on the right runs only if the command on the left fails. In this case, a new key is generated only if the public key does not already exist. |
| `ssh-keygen` | Creates a new SSH key pair. |
| `-t ed25519` | Chooses the Ed25519 key type, a modern SSH key algorithm. |
| `-f ~/.ssh/id_ed25519` | Saves the private key at `~/.ssh/id_ed25519` and the public key at `~/.ssh/id_ed25519.pub`. |
| `-N ""` | Sets an empty passphrase for the key. This makes automation easier, but for real production keys you may want a passphrase or a secrets manager. |
| `-C ansible-lab` | Adds the comment `ansible-lab` to the public key so you can identify what it is for. |
| `cp ~/.ssh/id_ed25519.pub ssh/authorized_keys` | Copies your public key into the lab’s `ssh/authorized_keys` file. The Dockerfile later copies this into the container so SSH logins are allowed for your key. |

---

# 5. Dockerfile

```Dockerfile
FROM ubuntu:24.04 # Starts the image from the official Ubuntu 24.04 base image. Everything below builds on top of this operating system image.

RUN apt-get update && \ # Runs `apt-get update` inside the image build. This downloads the latest package index. The `&&` means continue only if it succeeds. The trailing `\` continues the command on the next line.
    apt-get install -y openssh-server sudo python3 python3-apt && \ # Installs required packages: `openssh-server` for SSH access, `sudo` for privilege escalation, `python3` because Ansible modules need Python on managed nodes, and `python3-apt` so Ansible’s apt-related modules can work properly on Ubuntu/Debian. `-y` automatically answers yes.
    rm -rf /var/lib/apt/lists/* # Deletes downloaded apt package index files to keep the Docker image smaller.

RUN useradd -m -s /bin/bash ansible && \ # Creates a Linux user named `ansible`. `-m` creates a home directory, and `-s /bin/bash` sets Bash as the login shell.
    echo 'ansible ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ansible && \ # Creates a sudoers rule allowing the `ansible` user to run any command with sudo without typing a password. This is useful for Ansible automation.
    chmod 0440 /etc/sudoers.d/ansible # Sets safe permissions on the sudoers file. Sudo requires strict permissions for security.

RUN mkdir -p /var/run/sshd /home/ansible/.ssh # Creates the SSH daemon runtime directory and the `ansible` user’s `.ssh` directory. The `-p` flag creates parent directories as needed.

COPY ssh/authorized_keys /home/ansible/.ssh/authorized_keys # Copies the public key file from your local Docker build context into the container image. This authorizes SSH login as the `ansible` user.

RUN chown -R ansible:ansible /home/ansible/.ssh && \ # Changes ownership of the `.ssh` directory and its contents to the `ansible` user and group. SSH will reject keys if ownership is wrong.
    chmod 700 /home/ansible/.ssh && \ # Sets the `.ssh` directory permissions so only the owner can read, write, or enter it.
    chmod 600 /home/ansible/.ssh/authorized_keys # Sets the authorized keys file so only the owner can read/write it. SSH expects these strict permissions.

EXPOSE 22 # Documents that the container listens on port `22`, the standard SSH port. This does not publish the port by itself; Docker Compose does that later.

CMD ["/usr/sbin/sshd", "-D", "-e"] # Sets the default command when the container starts. It runs the SSH daemon. `-D` keeps it in the foreground so the container stays alive. `-e` sends logs to stderr, which Docker can capture.
```

---

# 6. Docker Compose file

```yml
services: # Top-level Docker Compose key that defines the containers/services to run.
  node1: # Defines a service named `node1`. This becomes one managed Ansible target in your lab.
    build: . # Tells Docker Compose to build the image using the Dockerfile in the current directory, `.`.
    container_name: ansible-node1 # Gives the container the fixed Docker name `ansible-node1`.
    ports: # Starts the list of port mappings for this container.
      - "2221:22" # Maps port `2221` on your laptop to port `22` inside the container. You SSH to `127.0.0.1:2221`, and Docker forwards that to this container’s SSH server.

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

---

# 7. Start Docker Desktop on macOS

```bash
open -a Docker
```

| Line | Explanation |
|---|---|
| `open -a Docker` | On macOS, opens the application named Docker, which starts Docker Desktop and the Docker daemon. |

---

# 8. Check Docker

```bash
docker info
```

| Line | Explanation |
|---|---|
| `docker info` | Prints information about the Docker client and server/daemon. If this works, Docker is running and your terminal can talk to it. |

---

# 9. Build and start the containers

```bash
docker compose up -d --build
```

| Part | Explanation |
|---|---|
| `docker compose` | Uses Docker Compose, the tool for running multi-container apps from a Compose file. |
| `up` | Creates and starts the services defined in `docker-compose.yml`. |
| `-d` | Detached mode: runs containers in the background instead of taking over your terminal. |
| `--build` | Rebuilds the image from the Dockerfile before starting the containers. |

---

# 10. Return to the Ansible project directory

```bash
cd ..
```

| Line | Explanation |
|---|---|
| `cd ..` | Moves one directory up. Since you were in `ansible_quickstart/lab`, this returns you to `ansible_quickstart`. |

---

# 11. Ansible inventory.ini

```ini
[myhosts] # Creates an Ansible inventory group named `myhosts`. Hosts listed below this line belong to that group.
node1 ansible_host=127.0.0.1 ansible_port=2221 # Defines an Ansible host alias called `node1`. Ansible connects to `127.0.0.1` on SSH port `2221`, which Docker forwards to the first container.
node2 ansible_host=127.0.0.1 ansible_port=2222
node3 ansible_host=127.0.0.1 ansible_port=2223

[myhosts:vars] # Defines variables that apply to every host in the `myhosts` group.
ansible_user=ansible # Tells Ansible to log in as the Linux user named `ansible`.
ansible_ssh_private_key_file=~/.ssh/id_ed25519 # Tells Ansible which private SSH key to use when connecting. This private key matches the public key copied into the containers.
ansible_python_interpreter=/usr/bin/python3 # Tells Ansible to use `/usr/bin/python3` on the managed nodes. Many Ansible modules execute Python code remotely.
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new' # Passes an SSH option to automatically accept new host keys the first time you connect. This avoids an interactive prompt in the lab.
```

---

# 12. Test SSH manually

```bash
ssh -p 2221 ansible@127.0.0.1
```

| Part | Explanation |
|---|---|
| `ssh` | Starts an SSH connection. |
| `-p 2221` | Uses port `2221` instead of the default SSH port `22`. |
| `ansible@127.0.0.1` | Logs in as user `ansible` to host `127.0.0.1`, which is your local machine. Docker forwards this to `node1`. |

Then type:

```bash
exit
```

Closes the SSH session and returns you to your original local terminal.

---

# 13. Test Ansible inventory and connectivity

```bash
ansible-inventory -i inventory.ini --list
ansible -i inventory.ini myhosts -m ping
```

| Line | Explanation |
|---|---|
| `ansible-inventory -i inventory.ini --list` | Reads `inventory.ini` and prints Ansible’s parsed view of the inventory as structured data. This verifies Ansible can understand your inventory file. |
| `ansible -i inventory.ini myhosts -m ping` | Runs an ad-hoc Ansible command against the `myhosts` group using the `ping` module. This checks that Ansible can connect and run Python on each managed node. |

---

# 14. Basic playbook.yml

```yml
- name: My first play # Starts a YAML list item representing one Ansible play. The play is named `My first play`.
  hosts: myhosts # Tells Ansible to run this play on every host in the `myhosts` inventory group.
  tasks: # Starts the list of tasks that will run on those hosts.
   - name: Ping my hosts # Starts the first task and gives it a human-readable name shown in Ansible output.
     ansible.builtin.ping: # Runs Ansible’s built-in `ping` module. This is not ICMP ping; it checks Ansible connectivity and Python execution.

   - name: Print message
     ansible.builtin.debug: # Runs Ansible’s built-in `debug` module, which prints information to the playbook output.
       msg: Hello world # Passes the message argument to the debug module, telling it to print `Hello world`.
```

---

# 15. Run the basic playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

| Part | Explanation |
|---|---|
| `ansible-playbook` | Runs an Ansible playbook file. |
| `-i inventory.ini` | Tells Ansible which inventory file to use. |
| `playbook.yml` | The playbook file to execute. |

---

# 16. gather_facts.yml

```yml
- name: Gather basic server facts # Starts a play named `Gather basic server facts`.
  hosts: myhosts # Runs this play against all hosts in the `myhosts` group.
  gather_facts: true # Tells Ansible to collect system information before tasks run. These facts become variables such as OS, memory, architecture, and Python version.

  tasks:
    - name: Show OS information
      ansible.builtin.debug:
        msg: # Starts the message value. Here it is a YAML list of strings.
          - "Host: {{ inventory_hostname }}" # Prints the Ansible inventory name of the current host, such as `node1`. `{{ ... }}` is Jinja2 variable syntax.
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}" # Prints the OS distribution name and version collected from facts, such as `Ubuntu 24.04`.
          - "Architecture: {{ ansible_architecture }}" # Prints the CPU architecture, such as `x86_64` or `aarch64`.
          - "Memory MB: {{ ansible_memtotal_mb }}" # Prints the total memory in megabytes.
          - "Python: {{ ansible_python_version }}" # Prints the Python version found on the managed node.
```

Run command:

```bash
ansible-playbook -i inventory.ini gather_facts.yml
```

---

# 17. Create vars folder

```bash
mkdir -p vars
```

---

# 18. vars/server_specs.yml

```yml
expected_specs: # Creates a dictionary/object named `expected_specs`. The values under it define your compliance baseline.
  os_family: Debian # Says the expected operating system family is Debian. Ubuntu belongs to the Debian family in Ansible facts.
  min_memory_mb: 128 # Requires at least 128 MB of memory.
  required_user: ansible # Requires that a user named `ansible` exists on each server.
  required_packages: # Starts a list of packages that must be installed.
    - openssh-server # Requires the SSH server package.
    - sudo # Requires the sudo package.
    - python3 # Requires Python 3.
    - curl # Requires curl. In this lab, this is intentionally missing at first so validation fails and remediation can fix it.
```

---

# 19. validate_build.yml

```yml
- name: Validate server build against expected specs # Starts a play named `Validate server build against expected specs`.
  hosts: myhosts # Runs the validation on all hosts in the `myhosts` group.
  gather_facts: true # Collects facts such as OS family and memory before tasks run.
  become: true # Uses privilege escalation, usually sudo, for tasks that may need elevated permissions.

  vars_files: # Starts a list of external variable files to load.
    - vars/server_specs.yml # Loads your expected baseline values from `vars/server_specs.yml`.

  tasks:
    - name: Collect installed package facts
      ansible.builtin.package_facts: # Uses Ansible’s package facts module to collect installed package information.
        manager: auto # Lets Ansible automatically detect the package manager, such as apt on Ubuntu.

    - name: Check if required user exists
      ansible.builtin.getent: # Uses the `getent` module to query system databases such as users and groups.
        database: passwd # Queries the passwd database, which stores user account information.
        key: "{{ expected_specs.required_user }}" # Looks up the username from your baseline file, which is `ansible`.
      register: user_check # Saves the task result into a variable named `user_check`. Later tasks can inspect it.
      failed_when: false # Prevents the task from failing the play if the user does not exist. This lets you record noncompliance instead of stopping immediately.

    - name: Build list of missing packages
      ansible.builtin.set_fact: # Creates or updates Ansible variables/facts during the play.
        missing_packages: >- # Creates a variable named `missing_packages`. `>-` is YAML folded block syntax, useful for multi-line expressions that become one logical value.
          {{ 
            expected_specs.required_packages
            | reject('in', ansible_facts.packages.keys())
            | list
          }}
        # (1) Starts a Jinja2 expression. 
        # (2) Reads the required package list from `vars/server_specs.yml`. 
        # (3) Filters out packages that are already installed. What remains are package names not found in the installed package facts.
        # (4) Converts the filtered result into a normal list.
        # (5) Ends the Jinja2 expression.

    - name: Build compliance findings
      ansible.builtin.set_fact: # Sets a new variable during the play.
        compliance_findings: # Creates a dictionary/object named `compliance_findings`.
          host: "{{ inventory_hostname }}" # Stores the current host’s inventory name.
          os_family_actual: "{{ ansible_os_family }}" # Stores the OS family found by Ansible facts.
          os_family_expected: "{{ expected_specs.os_family }}" # Stores the expected OS family from your baseline file.
          memory_mb_actual: "{{ ansible_memtotal_mb }}" # Stores the actual memory found by Ansible.
          memory_mb_required: "{{ expected_specs.min_memory_mb }}" # Stores the minimum required memory from the baseline.
          required_user: "{{ expected_specs.required_user }}" # Stores the username that should exist.
          user_exists: "{{ user_check.ansible_facts.getent_passwd is defined }}" # Stores true/false depending on whether the earlier `getent` check found the user.
          missing_packages: "{{ missing_packages }}" # Stores the list of packages that were missing.
          compliant: >- # Creates a final true/false compliance value using a multi-line Jinja2 expression.
            {{
              ansible_os_family == expected_specs.os_family
              and ansible_memtotal_mb >= expected_specs.min_memory_mb
              and user_check.ansible_facts.getent_passwd is defined
              and missing_packages | length == 0
            }}
        # (1) Starts the compliance expression.
        # (2) Checks whether the actual OS family matches the expected OS family.
        # (3) Requires actual memory to be greater than or equal to the minimum.
        # (4) Requires the user lookup to have succeeded.
        # (5) Requires the missing package list to be empty.
        # (6) Ends the compliance expression.

    - name: Print compliance result
      ansible.builtin.debug: # Uses the debug module to print information.
        var: compliance_findings # Prints the value of the variable `compliance_findings`.

    - name: Fail if server is not compliant
      ansible.builtin.assert: # Uses the assert module to enforce a condition.
        that: # Starts the list of conditions that must be true.
          - compliance_findings.compliant | bool # Requires the `compliant` value to be true. The `| bool` filter converts it to a Boolean.
        fail_msg: "Server {{ inventory_hostname }} is NOT compliant: {{ compliance_findings }}" # Message shown if the assertion fails. It includes the host name and findings.
        success_msg: "Server {{ inventory_hostname }} is compliant." # Message shown if the assertion passes.
```

Run command:

```bash
ansible-playbook -i inventory.ini validate_build.yml
```

---

# 20. remediate_packages.yml

```bash
touch remediate_packages.yml
```

```yml
- name: Remediate missing baseline packages 
  hosts: myhosts # Runs against every host in the `myhosts` inventory group.
  gather_facts: true
  become: true # Uses sudo/elevated privileges, required for installing packages.

  vars_files:
    - vars/server_specs.yml

  tasks:
    - name: Install required baseline packages
      ansible.builtin.apt: # Uses Ansible’s apt module for Debian/Ubuntu package management.
        name: "{{ expected_specs.required_packages }}" # Tells apt to install every package listed in `expected_specs.required_packages`.
        state: present # Ensures the packages are installed. If already installed, no change is made.
        update_cache: true # Runs an apt package index update before installing.
```

Check mode command:

```bash
ansible-playbook -i inventory.ini remediate_packages.yml --check
```

| Part | Explanation |
|---|---|
| `ansible-playbook` | Runs the playbook tool. |
| `-i inventory.ini` | Uses your inventory. |
| `remediate_packages.yml` | Runs the remediation playbook. |
| `--check` | Dry-run mode. Ansible predicts changes without actually applying them when modules support check mode. |

Diff/check command:

```bash
ansible-playbook -i inventory.ini remediate_packages.yml --check --diff
```

| Part | Explanation |
|---|---|
| `--check` | Runs in dry-run mode. |
| `--diff` | Shows before/after differences for modules that support diff output. This is more useful for file changes than package installation, but it is a common safe-review option. |

Actual run command:

```bash
ansible-playbook -i inventory.ini remediate_packages.yml
```

| Line | Explanation |
|---|---|
| `ansible-playbook -i inventory.ini remediate_packages.yml` | Runs the remediation for real and installs missing packages. |

---

# 21. Rerun validation

```bash
ansible-playbook -i inventory.ini validate_build.yml
```

| Line | Explanation |
|---|---|
| `ansible-playbook -i inventory.ini validate_build.yml` | Runs the validation playbook again to confirm the remediation fixed the missing package issue. |

---

# 22. Create reports folder and validation report file

```bash
mkdir -p reports
touch validation_report.yml
```

---

# 23. validation_report.yml

```yml
- name: Generate server validation report
  hosts: myhosts
  gather_facts: true
  become: true # Uses elevated privileges where needed, such as checking packages.

  vars_files:
    - vars/server_specs.yml

  tasks:
    - name: Collect installed package facts
      ansible.builtin.package_facts: # Collects installed package facts.
        manager: auto # Auto-detects the package manager.

    - name: Check if required user exists
      ansible.builtin.getent: # Queries a system database.
        database: passwd # Uses the user account database.
        key: "{{ expected_specs.required_user }}" # Looks up the required username from the baseline.
      register: user_check # Saves the lookup result for later use.
      failed_when: false # Does not fail immediately if the user is missing.

    - name: Build list of missing packages
      ansible.builtin.set_fact: # Creates a variable during play execution.
        missing_packages: >- # Starts a folded Jinja2 expression that produces the missing package list.
          {{
            expected_specs.required_packages
            | reject('in', ansible_facts.packages.keys())
            | list
          }}
        # (1) Starts the Jinja2 expression.
        # (2) Reads the required packages from the baseline file.
        # (3) Removes packages that are already installed from the required list.
        # (4) Converts the result to a list.
        # (5) Ends the expression.

    - name: Build report row
      ansible.builtin.set_fact: # Creates a variable.
        report_row: # Starts a dictionary named `report_row`.
          host: "{{ inventory_hostname }}" # Saves the host name.
          os_family: "{{ ansible_os_family }}" # Saves the detected OS family.
          memory_mb: "{{ ansible_memtotal_mb }}" # Saves detected memory in MB.
          user_exists: "{{ user_check.ansible_facts.getent_passwd is defined }}" # Saves whether the required user exists.
          missing_packages: "{{ missing_packages }}" # Saves the missing package list.
          compliant: >- # Starts the final compliance expression.
            {{
              ansible_os_family == expected_specs.os_family
              and ansible_memtotal_mb >= expected_specs.min_memory_mb
              and user_check.ansible_facts.getent_passwd is defined
              and missing_packages | length == 0
            }}
        # (1) Starts Jinja2.
        # (2) Checks OS compliance.
        # (3) Checks memory compliance.
        # (4) Checks user compliance.
        # (5) Checks package compliance.
        # (6) Ends Jinja2.

    - name: Print report row # Names the task that prints the report row.
      ansible.builtin.debug: # Uses debug output.
        var: report_row # Prints the variable `report_row`.

    - name: Save host report locally # Names the task that writes the report to a file.
      ansible.builtin.copy: # Uses the copy module to write content to a destination file.
        content: "{{ report_row | to_nice_json }}" # Converts the report row to nicely formatted JSON and uses it as file content.
        dest: "reports/{{ inventory_hostname }}_validation.json" # Writes a unique JSON file per host, such as `reports/node1_validation.json`.
      delegate_to: localhost # Runs the copy task on your local control machine instead of on the remote managed node.
      become: false # Disables sudo for this local file-writing task.
```

Run command:

```bash
ansible-playbook -i inventory.ini validation_report.yml
```

| Line | Explanation |
|---|---|
| `ansible-playbook -i inventory.ini validation_report.yml` | Runs the report playbook and writes report JSON files into the local `reports` folder. |

Check report files:

```bash
ls reports
cat reports/node1_validation.json
```

| Line | Explanation |
|---|---|
| `ls reports` | Lists files inside the `reports` directory. |
| `cat reports/node1_validation.json` | Prints the contents of the report file for `node1`. |

---

# 24. patch_baseline.yml

```bash
touch patch_baseline.yml
```

| Line | Explanation |
|---|---|
| `touch patch_baseline.yml` | Creates an empty playbook file named `patch_baseline.yml` or updates its timestamp. |

```yml
- name: Patch servers in controlled batches # Starts a play for patching servers.
  hosts: myhosts # Runs on all hosts in the group.
  become: true # Uses elevated privileges because package updates require sudo.
  gather_facts: true # Collects system facts before patching.
  serial: 1 # Runs the play on one host at a time instead of all hosts at once. This reduces risk during maintenance.

  tasks:
    - name: Pre-check connectivity
      ansible.builtin.ping: # Verifies Ansible can connect before patching.

    - name: Update apt package cache
      ansible.builtin.apt: # Uses Ansible’s apt module.
        update_cache: true # Runs the equivalent of `apt-get update`.

    - name: Apply available package updates
      ansible.builtin.apt: # Uses the apt module again.
        upgrade: dist # Performs a distribution-style upgrade, allowing package dependency changes. This is similar to `apt-get dist-upgrade`.

    - name: Check whether reboot is required
      ansible.builtin.stat: # Uses the stat module to check file metadata/existence.
        path: /var/run/reboot-required # Checks the Debian/Ubuntu file that indicates a reboot is required after updates.
      register: reboot_required # Saves the stat result into `reboot_required`.

    - name: Show reboot requirement
      ansible.builtin.debug: # Uses debug output.
        msg: "Reboot required: {{ reboot_required.stat.exists }}" # Prints true or false depending on whether `/var/run/reboot-required` exists.
```

Dry-run command:

```bash
ansible-playbook -i inventory.ini patch_baseline.yml --check
```

Runs the patch playbook in dry-run mode to preview likely changes.

Actual run command:

```bash
ansible-playbook -i inventory.ini patch_baseline.yml
```
Runs the patch playbook for real.

Alternative serial setting:

```yml
serial:
  - 1
  - 3
  - 5
```
Starts a staged batch-size list. First batch runs on 1 host. Next batch runs on 3 hosts at a time. Later batches run on 5 hosts at a time. This lets you start cautiously and expand after initial success.

---

# 25. preflight.yml

```bash
touch preflight.yml
```

```yml
- name: Preflight checks before maintenance
  hosts: myhosts
  gather_facts: true # Collects system facts, including memory.
  become: true

  vars: # Defines variables directly inside this playbook.
    min_memory_mb: 128 # Sets the minimum acceptable memory to 128 MB.
    required_free_mb: 100 # Sets the minimum required free disk space on `/` to 100 MB.

  tasks:
    - name: Check connectivity
      ansible.builtin.ping: # Confirms Ansible can connect and run modules.

    - name: Check memory requirement
      ansible.builtin.assert: # Uses Ansible’s assert module to require a condition.
        that: # Starts the list of required conditions.
          - ansible_memtotal_mb >= min_memory_mb # Requires actual memory to be at least the minimum.
        fail_msg: "Not enough memory on {{ inventory_hostname }}. Found {{ ansible_memtotal_mb }} MB." # Message shown if the memory check fails.
        success_msg: "Memory check passed on {{ inventory_hostname }}." # Message shown if the memory check passes.

    - name: Check root filesystem free space in MB # Names the disk-space command task.
      ansible.builtin.command: df -Pm / # Runs `df -Pm /` on the remote host. `df` reports disk usage; `-P` uses portable output format; `-m` shows MB; `/` checks the root filesystem.
      register: root_df # Saves the command output into a variable named `root_df`.
      changed_when: false # Tells Ansible this command does not change the system, so report it as `ok` instead of `changed`.

    - name: Extract available root filesystem space
      ansible.builtin.set_fact: # Creates a new variable.
        root_available_mb: "{{ root_df.stdout_lines[1].split()[3] | int }}" # Reads line 2 of the `df` output, splits it into columns, takes column 4, and converts it to an integer. That column is available free space in MB. In actual SAP servers use ansible_mounts but since we are using Docker containers we use df -Pm /

    - name: Check root filesystem free space requirement # Names the disk-space assertion task.
      ansible.builtin.assert: # Uses the assert module.
        that: # Starts the condition list.
          - root_available_mb | int >= required_free_mb # Requires available root filesystem space to be at least the required amount.
        fail_msg: "Not enough free space on / for {{ inventory_hostname }}. Found {{ root_available_mb }} MB." # Message shown if disk space is insufficient.
        success_msg: "Disk space check passed on {{ inventory_hostname }}. Found {{ root_available_mb }} MB free." # Message shown if disk space is sufficient.
```

---

# 26. safe_maintenance.yml

```bash
touch safe_maintenance.yml
```

```yml
--- # YAML document start marker. Optional, but commonly used.
  - name: Safe maintenance structure
    hosts: myhosts
    become: true
    gather_facts: true
    serial: 1 # Runs one host at a time. This is safer for maintenance.

    tasks:
      - name: Maintenance workflow with error handling # Creates one task that contains a block/rescue/always structure.
        block: # Starts a group of tasks. If one task in this block fails, Ansible can run the `rescue` section.
          - name: Pre-check server # Names the first task inside the block.
            ansible.builtin.ping: # Checks connectivity.

          - name: Simulate stopping application # Names a simulated app-stop task.
            ansible.builtin.debug: # Uses debug instead of actually stopping an application.
              msg: "Stopping application on {{ inventory_hostname }}" # Prints which host would have its application stopped.

          - name: Simulate patching # Names a simulated patching task.
            ansible.builtin.debug: # Prints a message instead of actually patching.
              msg: "Patching {{ inventory_hostname }}" # Prints which host would be patched.

          - name: Simulate starting application # Names a simulated application-start task.
            ansible.builtin.debug: # Prints a message instead of starting a real app.
              msg: "Starting application on {{ inventory_hostname }}" # Prints which host would have the app started.

          - name: Simulate health check # Names a simulated health check task.
            ansible.builtin.debug: # Prints a message instead of running a real health check.
              msg: "Health check passed on {{ inventory_hostname }}" # Prints a simulated health-check success message.

        rescue: # Starts tasks that run only if something inside the `block` fails. Similar to exception handling.
          - name: Report failure
            ansible.builtin.debug: # Prints failure information.
              msg: "Something failed on {{ inventory_hostname }}. Running recovery steps." # Prints a message saying recovery is starting for that host.

          - name: Simulate rollback or restart # Names a simulated recovery task.
            ansible.builtin.debug: # Prints a message instead of actually rolling back or restarting.
              msg: "Attempting recovery on {{ inventory_hostname }}" # Prints which host would receive recovery action.

        always: # Starts tasks that run whether the block succeeds or fails.
          - name: Always write final status # Names the final status task.
            ansible.builtin.debug: # Prints a message.
              msg: "Maintenance attempt finished for {{ inventory_hostname }}" # Always prints that the maintenance attempt finished for the current host.
```

Run command:

```bash
ansible-playbook -i inventory.ini safe_maintenance.yml
```
Runs the safe maintenance playbook. Because `serial: 1` is set, it processes `node1`, then `node2`, then `node3`. |

---

# 27. Directory structure example

```text
sap-automation/ # Root folder for a larger SAP automation project.
  inventories/ # Folder for inventory files by environment.
    dev.ini # Inventory file for development hosts.
    qa.ini # Inventory file for QA/test hosts.
    prod.ini # Inventory file for production hosts.

  group_vars/ # Folder for variables that apply to inventory groups.
    sap_app_servers.yml # Variables for SAP application server groups.
    hana_servers.yml # Variables for SAP HANA server groups.
    all.yml # Variables that apply to all inventory hosts/groups.

  vars/ # Folder for standalone variable files.
    sap_build_specs.yml # Baseline build requirements for SAP-related systems.
    patch_window.yml # Variables describing patch timing or maintenance windows.

  playbooks/ # Folder for Ansible playbooks.
    01_preflight.yml # First playbook: run checks before changing anything.
    02_validate_build.yml # Validate server build against requirements.
    03_pre_patch_report.yml # Generate report before patching.
    04_stop_sap.yml # Stop SAP services cleanly.
    05_patch_os.yml # Patch the operating system.
    06_reboot.yml # Reboot if required.
    07_start_sap.yml # Start SAP services again.
    08_health_check.yml # Validate SAP/system health after maintenance.
    09_post_patch_report.yml # Generate after-maintenance report.

  roles/ # Folder for reusable Ansible roles.
    validation/ # Role containing validation tasks.
    reporting/ # Role containing report-generation tasks.
    patching/ # Role containing OS patch tasks.
    sap_control/ # Role containing SAP start/stop/control tasks.
    health_check/ # Role containing health-check tasks.

  reports/ # Folder for generated output reports.
    pre/ # Pre-maintenance reports.
    post/ # Post-maintenance reports.
```

---

# 28. Local lab check list

```text
packages # Checks whether required packages are installed.
users # Checks whether required OS users exist.
memory # Checks whether the host has enough RAM.
disk # Checks whether there is enough free filesystem space.
OS # Checks operating system family/version requirements.
connectivity # Checks whether Ansible can connect and run modules.
```

---

# 29. Full playthrough commands

```bash
# Collects and displays basic host facts.
ansible-playbook -i inventory.ini gather_facts.yml

# Validates hosts against your expected specs. Initially this fails because `curl` is missing.
ansible-playbook -i inventory.ini validate_build.yml

# Previews remediation without applying changes.
ansible-playbook -i inventory.ini remediate_packages.yml --check

# Installs missing required packages.
ansible-playbook -i inventory.ini remediate_packages.yml

# Generates per-host JSON validation reports.
ansible-playbook -i inventory.ini validation_report.yml

# Runs safety checks before maintenance.
ansible-playbook -i inventory.ini preflight.yml

# Demonstrates a safe maintenance flow using block/rescue/always.
ansible-playbook -i inventory.ini safe_maintenance.yml
```

---

# 30. Master playbook concept

The final goal is one higher-level playbook that runs these stages in order:

| Step | Description |
|---|---|
| `1. preflight` | Check safety conditions before changing anything. |
| `2. validation` | Confirm whether servers meet the expected baseline. |
| `3. report` | Save the findings so you have evidence of current state. |
| `4. remediation` | Fix missing or noncompliant items. |
| `5. validation again` | Re-check after remediation to prove the system is compliant. |