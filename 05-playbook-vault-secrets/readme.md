# Ansible: Separation of Concerns & Ansible Vault

Securely managing sensitive variables using external files and encrypting them with Ansible Vault from a master EC2 instance.

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

### Step 13: Create the Secrets File

Create a file named `secrets.yml` to store your sensitive variables separately:

```bash
vim secrets.yml
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Paste the following configuration:

```yaml
api_key: "sddsvdsv345345353"
```

3. Save and exit by pressing <kbd>Esc</kbd> followed by <kbd>:wq</kbd> (or <kbd>:x</kbd>) and pressing <kbd>Enter</kbd>.

---

### Step 14: Create the Playbook File

Create a playbook named `show-secrets.yml` that loads variables from the external secrets file:

```bash
vim show-secrets.yml
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Paste the following playbook definition:

```yaml
- name: user secrets from secrets.yml file
  hosts: servers
  become: yes

  vars_files:
    - secrets.yml

  tasks:
    - name: Show API key
      debug:
        msg: "My API key is {{ api_key }}"
```

3. Save and exit by pressing <kbd>Esc</kbd>, typing <kbd>:wq</kbd> (or <kbd>:x</kbd>), and pressing <kbd>Enter</kbd>.

---

### Step 15: Run the Playbook (Unencrypted)

Verify that the playbook runs and prints the variable successfully:

```bash
ansible-playbook show-secrets.yml
```

---

### Step 16: Create a Password Vault File

Create a file containing the encryption password to automatically decrypt secrets during playbook execution:

```bash
vim password-vault.txt
```

1. Enter your password (e.g., `nur1234`).
2. Save and exit.

Set read-only permissions on the file for the owner to ensure security:

```bash
chmod 600 password-vault.txt
ls -l password-vault.txt
```

---

### Step 17: Encrypt the Secrets File

Encrypt the `secrets.yml` file using Ansible Vault:

```bash
ansible-vault encrypt secrets.yml --vault-password-file password-vault.txt
```

---

### Step 18: Run the Playbook (Encrypted)

If you attempt to run the playbook normally, it will fail because the secrets file is encrypted:

```bash
ansible-playbook show-secrets.yml
```

To run the playbook successfully, provide the password vault file to decrypt the variables:

```bash
ansible-playbook show-secrets.yml --vault-password-file password-vault.txt
```

---

## Author

Built as a hands-on DevOps practice project using Ansible and AWS EC2.
