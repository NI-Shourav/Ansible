# Ansible: Loop Installing Multiple Tools on Worker Nodes

Running an Ansible Playbook utilizing loops to install multiple packages (unzip, docker.io, nginx, tree) on managed nodes using Ansible from a master EC2 instance.

[![Ansible](https://img.shields.io/badge/Ansible-%23EE0000.svg?style=for-the-badge&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-%23E95420.svg?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)

---

## Project Architecture

```text
                   ┌──────────────────────┐
                   │  Ansible Master PC   │
                   │    Ubuntu 24.04      │
                   └──────────┬───────────┘
                              │
               ┌──────────────┴──────────────┐
               │                             │
     ┌─────────▼─────────┐       ┌──────────▼─────────┐
     │   Managed Node 1  │       │   Managed Node 2  │
     │    Ubuntu 24.04   │       │    Ubuntu 24.04   │
     └───────────────────┘       └────────────────────┘
```

---

## Step-by-Step Implementation

### Step 1: Launch EC2 Instances

Launch **3 EC2 instances** on AWS.

#### Recommended OS
* **Ubuntu Server 24.04 LTS**

> [!WARNING]
> Avoid using experimental or non-LTS Ubuntu releases, as default directories and package behaviors may differ.

---

### Step 2: Rename the EC2 Instances

For clearer inventory management, assign distinct names to each instance:

| Their Roles | Instance Name |
| :--- | :--- |
| **Master Node** | `Ansible-master-pc` |
| **Managed Node 1** | `Managed-node-1` |
| **Managed Node 2** | `Managed-node-2` |

---

### Step 3: Connect to the Master Node

Establish an SSH connection to the master EC2 instance:

```bash
ssh -i "your-key.pem" ubuntu@<MASTER_PUBLIC_IP>
```

> [!NOTE]
> Replace `your-key.pem` with your actual private key file name, and `<MASTER_PUBLIC_IP>` with the public IP address of your Master EC2 instance.

---

### Step 4: Update the Master Server

Synchronize repository indexes and update installed packages on the master node:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### Step 5: Install Ansible

Register the official Ansible PPA repository and install the runtime:

```bash
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

---

### Step 6: Verify Ansible Installation

Confirm that Ansible is installed correctly:

```bash
ansible --version
```

---

### Step 7: Transfer the PEM Key to the Master Node

To allow the master node to manage the client instances, you need to copy your private SSH key from your local machine to the master EC2 instance.

#### Local PC Actions

Navigate to the directory containing your key (typically `Downloads`):

```bash
cd Downloads
```

Ensure the key file has appropriate read-only permissions:

```bash
chmod 400 your-key.pem
```

Copy the file using Secure Copy (`scp`):

```bash
scp -i "your-key.pem" your-key.pem ubuntu@<MASTER_PUBLIC_IP>:/home/ubuntu
```

> [!NOTE]
> Replace `your-key.pem` with your actual key file, and `<MASTER_PUBLIC_IP>` with your Master EC2 instance's public IP address.

---

### Step 8: Configure the Ansible Inventory File

Navigate to the Ansible configuration directory and update the hosts inventory:

```bash
cd /etc/ansible
sudo vim hosts
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Add the following host definitions:

```ini
[servers]
server1 ansible_host=<MANAGED_NODE_1_PUBLIC_IP>
server2 ansible_host=<MANAGED_NODE_2_PUBLIC_IP>

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/your-key.pem
```

> [!NOTE]
> In the configuration above:
> * Replace `<MANAGED_NODE_1_PUBLIC_IP>` and `<MANAGED_NODE_2_PUBLIC_IP>` with the public IP addresses of your managed instances.
> * Replace `your-key.pem` with your actual private key file name.

3. Save and exit by typing <kbd>:wq</kbd> and pressing <kbd>Enter</kbd>.

---

### Step 9: Configure Ansible Settings

Disable SSH host key checking to prevent interactive prompts when connecting to new managed nodes:

```bash
sudo vim ansible.cfg
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Append the following settings:

```ini
[defaults]
host_key_checking=False
```

3. Save and exit by typing <kbd>:wq</kbd> and pressing <kbd>Enter</kbd>.

---

### Step 10: Test Connectivity to Managed Nodes

Verify that the master node can successfully reach and authenticate with all managed nodes:

```bash
ansible -m ping servers
```

#### Expected Successful Output

```json
server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
server2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

### Step 11: Update Managed Nodes

Update package definitions and upgrade installed packages on all managed servers:

```bash
ansible -a "sudo apt update" servers
ansible -a "sudo apt upgrade -y" servers
```

---

### Step 12: Create a Playbooks Directory

Create a dedicated directory to organize your Ansible playbooks and navigate into it:

```bash
mkdir playbooks
cd playbooks
```

---

### Step 13: Create the Playbook File

Create a new playbook file named `package-installer.yml` using a text editor such as Vim:

```bash
vim package-installer.yml
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Paste the following playbook definition:

```yaml
- name: Install My Packages
  hosts: servers
  become: yes

  vars:
    packages_to_install:
      - unzip
      - docker.io
      - nginx
      - tree

  tasks:
    - name: Install Packages
      apt:
        name: "{{ item }}"
        state: present
      loop: "{{ packages_to_install }}"

    - name: Print installation status
      loop: "{{ packages_to_install }}"
      debug:
        msg: "installing {{ item }}"
```

3. Save and exit the editor by pressing <kbd>Esc</kbd>, typing <kbd>:x</kbd> (or <kbd>:wq</kbd>), and pressing <kbd>Enter</kbd>.

---

### Step 14: Run the Ansible Playbook

Execute the playbook to run the package installation tasks on the managed nodes:

```bash
ansible-playbook package-installer.yml
```

---

## Author

Built as a hands-on DevOps practice project using Ansible and AWS EC2.
