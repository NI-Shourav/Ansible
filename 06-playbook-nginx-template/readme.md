# Ansible: Deploy HTML Homepage on Nginx Managed Nodes

Deploying a custom HTML web page on managed nodes by configuring and running an Nginx deployment Ansible playbook from a master EC2 instance.

[![Ansible](https://img.shields.io/badge/Ansible-%23EE0000.svg?style=for-the-badge&logo=ansible&logoColor=white)](https://www.ansible.com/)
[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-%23E95420.svg?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Nginx](https://img.shields.io/badge/Nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)

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

### Step 12: Install Nginx on Managed Nodes

Deploy the Nginx web server across the managed fleet:

```bash
ansible -a "sudo apt install nginx -y" servers
```

---

### Step 13: Verify Nginx Installation

Open a web browser and navigate to the public IP address of either managed node:

```text
http://<MANAGED_NODE_PUBLIC_IP>
```

You should see the default **Welcome to nginx!** page, indicating Nginx is running.

---

### Step 14: Create a Playbooks Directory

Create a dedicated directory to organize your Ansible playbooks and navigate into it:

```bash
mkdir playbooks
cd playbooks
```

---

### Step 15: Create the Custom HTML Page

Create a file named `index.html` which will be deployed as the web server's welcome page:

```bash
vim index.html
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Paste the following HTML code:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to Ansible</title>

    <style>
        *{
            margin:0;
            padding:0;
            box-sizing:border-box;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        body{
            height:100vh;
            display:flex;
            justify-content:center;
            align-items:center;
            background: linear-gradient(135deg,#0f172a,#1e293b,#334155);
            overflow:hidden;
        }

        .container{
            text-align:center;
            background: rgba(255,255,255,0.08);
            backdrop-filter: blur(12px);
            padding:60px 50px;
            border-radius:25px;
            box-shadow:0 20px 40px rgba(0,0,0,0.4);
            max-width:800px;
            width:90%;
            border:1px solid rgba(255,255,255,0.15);
        }

        .logo{
            font-size:80px;
            margin-bottom:20px;
        }

        h1{
            font-size:3.5rem;
            color:#ffffff;
            margin-bottom:15px;
        }

        .highlight{
            color:#4ade80;
        }

        p{
            color:#cbd5e1;
            font-size:1.2rem;
            line-height:1.8;
            margin-bottom:30px;
        }

        .badge{
            display:inline-block;
            padding:12px 28px;
            background:#4ade80;
            color:#0f172a;
            font-weight:bold;
            border-radius:50px;
            font-size:1rem;
            box-shadow:0 0 20px rgba(74,222,128,.5);
        }

        .footer{
            margin-top:30px;
            color:#94a3b8;
            font-size:0.95rem;
        }

        .circle{
            position:absolute;
            border-radius:50%;
            background:rgba(74,222,128,0.08);
        }

        .c1{
            width:300px;
            height:300px;
            top:-100px;
            left:-100px;
        }

        .c2{
            width:250px;
            height:250px;
            bottom:-80px;
            right:-80px;
        }
    </style>
</head>
<body>

    <div class="circle c1"></div>
    <div class="circle c2"></div>

    <div class="container">
        <div class="logo">⚙️</div>

        <h1>Welcome to <span class="highlight">Ansible</span></h1>

        <p>
            Dear Students,<br>
            Welcome to your Ansible Automation Journey.
            Today you will learn how to automate servers,
            manage infrastructure efficiently, and simplify
            complex system administration tasks.
        </p>

        <div class="badge">
            🚀 Happy Learning & Automating
        </div>

        <div class="footer">
            Powered by Ansible | Infrastructure as Code | DevOps
        </div>
    </div>

</body>
</html>
```

3. Save and exit by pressing <kbd>Esc</kbd> followed by <kbd>:wq</kbd> (or <kbd>:x</kbd>) and pressing <kbd>Enter</kbd>.

---

### Step 16: Create the Playbook File

Create a playbook named `setup-nginx.yaml` to deploy the HTML file to the managed nodes:

```bash
vim setup-nginx.yaml
```

1. Press <kbd>i</kbd> to enter **Insert Mode**.
2. Paste the following playbook definition:

```yaml
---
- name: Configure and Deploy Nginx
  hosts: servers
  become: yes

  tasks:
    - name: Ensure Nginx is installed (Debian/Ubuntu)
      apt:
        name: nginx
        state: present
        update_cache: yes
      when: ansible_facts['os_family'] == "Debian"

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deploy the welcome HTML file
      template:
        src: index.html
        dest: /var/www/html/index.html

    - name: Restart Nginx to ensure changes are live
      service:
        name: nginx
        state: restarted
```

> [!IMPORTANT]
> The lines `src: index.html` and `dest: /var/www/html/index.html` are very important as they map the source template from the master node to the destination path on the worker nodes.

3. Save and exit by pressing <kbd>Esc</kbd> followed by <kbd>:wq</kbd> (or <kbd>:x</kbd>) and pressing <kbd>Enter</kbd>.

---

### Step 17: Run the Ansible Playbook

Execute the playbook to run the tasks and configure the Nginx web server on the managed nodes:

```bash
ansible-playbook setup-nginx.yaml
```

---

### Step 18: Verify the Custom Web Page

Open a web browser and navigate to the public IP address of either managed node:

```text
http://<MANAGED_NODE_PUBLIC_IP>
```

You should see the custom **Welcome to Ansible** page with the gradient background and rocket badge, indicating a successful deployment.

---

## Author

Built as a hands-on DevOps practice project using Ansible and AWS EC2.
