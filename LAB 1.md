## Setup the `Docker Server` and `Jenkins Server` for CICD Lab.

### Task 1: Manually launch the Jump Server EC2 Instance

* Region: North Virginia (us-east-1).
* Use tag Name: `sirin-Jump-Server`
* AMI Type and OS Version: `Ubuntu 22.04 LTS`
* Instance type: `t2.micro`
* Create key pair with name: `sirin-DevOps-Keypair`
* Create security group with name: `sirin-DevOps-SG`
   (Include Ports: `22 [SSH],` `80 [HTTP],` `8080 [Jenkins],` `9999 [Tomcat],` and `4243 [Docker]`)
* Configure Storage: 10 GiB
* Click on `Launch Instance.`


### Task 2: Installing Terraform onto `Jump Server` to automate the creation of 2 more EC2 instances.
Once the Jump Server is up and running, SSH into the machine using `MobaXterm` or `Putty` with the username `ubuntu` and do the following:

```
sudo hostnamectl set-hostname JumpServer
bash
```
```
sudo apt update
```
```
sudo apt install wget unzip -y
```
```
wget https://releases.hashicorp.com/terraform/1.12.2/terraform_1.12.2_linux_386.zip
```
[Click here](https://developer.hashicorp.com/terraform/downloads) for Terraform's Latest Versions
```
unzip terraform_1.12.2_linux_386.zip
ls
```
```
sudo mv terraform /usr/local/bin
```
```
rm terraform_1.9.2_linux_amd64.zip
```
```
terraform -v
```

![image](https://github.com/user-attachments/assets/4dc530fa-b206-4bf2-a0eb-9499ff9e923a)

### Task 3: Install Python 3, pip, AWS CLI, and Ansible on to Jump Server
Install Python 3 and the required packages:
```
sudo apt install python3-pip -y
```
Installing `AWS CLI`, `boto`, `boto3`, and `ansible`
```
sudo pip3 install  boto boto3 awscli ansible --break-system-packages
```
For Authentication with AWS we need to provide `IAM User's CLI Credentials`
```
aws configure
```
Once configured, do a smoke test to check if your credentials are valid and got the access to AWS account.
```
aws s3 ls
```
(Or)
```
aws iam list-users
```

### Task 4: Create a host inventory file with the necessary permissions (Ansible need this Inventory file for identifying the targets)
```
sudo mkdir /etc/ansible && sudo touch /etc/ansible/hosts
```
Change the permissions. The below command gives `Read, Write and Execute` permissions to `Owner,` `Read and Write` permissions to the `Group,` and `Read and Write` permissions to `Others.`
```
sudo chmod 766 /etc/ansible/hosts
```
Cross verify using the below command
```
ls -la /etc/ansible/
```
![image](https://github.com/user-attachments/assets/25314a0c-8d3c-483b-96a8-0b7be2017424)


### Task 5: Utilizing Terraform, initiate the deployment of two new servers: `Docker-server` and `Jenkins-server` 

As a first step, create a keyPair using `ssh-keygen` Command.

**Note:**
1. This will create `id_rsa` and `id_rsa.pub` in Jump Machine in `/home/ubuntu/.ssh/` path.
2. While creating choose defaults like:
   * path as **/home/ubuntu/.ssh/id_rsa**
   * don't set up any passphrase, and just hit the '**Enter**' key for 3 questions it asks.
3. `-t rsa:` Specifies the type of key to create, in this case, RSA.
4. `-b 2048:` Specifies the number of bits in the key, 2048 bits in this case. The larger the number of bits, the stronger the key.
```
ssh-keygen -t rsa -b 2048 
```
![image](https://github.com/user-attachments/assets/2e81febd-047c-46e6-9c6b-1e9056613866)

Now Create the terraform directory and set up the config files in it
```
mkdir devops-labs && cd devops-labs
```
```
vi main.tf
```
Copy and paste the below code into `main.tf`
```
provider "aws" {
  region = var.region
}

resource "aws_key_pair" "mykeypair" {
  key_name   = var.key_name
  public_key = file(var.public_key)
}

# to create 2 EC2 instances
resource "aws_instance" "my-machine" {
  # Launch 2 servers
  for_each = toset(var.my-servers)

  ami                    = var.ami_id
  key_name               = var.key_name
  vpc_security_group_ids = [var.sg_id]
  instance_type          = var.ins_type
  depends_on             = [aws_key_pair.mykeypair]

  # Read from the list my-servers to name each server
  tags = {
    Name = each.key
  }

  # Ensure the .ssh directory exists before copying the file
  provisioner "remote-exec" {
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("/home/ubuntu/.ssh/id_rsa")  # Replace with the path to your private key
      host        = self.public_ip
    }

    inline = [
      "mkdir -p /home/ubuntu/.ssh"   # Ensure .ssh directory exists before copying the file
    ]
  }

  provisioner "file" {
    source      = "/home/ubuntu/.ssh/id_rsa"   # Source path on Host machine
    destination = "/home/ubuntu/.ssh/id_rsa"  # Destination path on EC2 instance
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("/home/ubuntu/.ssh/id_rsa")  # Replace with the path to your private key
      host        = self.public_ip
    }
  }

  # After the file is copied, change its permissions
  provisioner "remote-exec" {
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("/home/ubuntu/.ssh/id_rsa")  # Replace with the path to your private key
      host        = self.public_ip
    }

    inline = [
      "chmod 400 /home/ubuntu/.ssh/id_rsa"  # Change the permissions of the copied file after it's been transferred
    ]
  }

  provisioner "local-exec" {
    command = <<-EOT
      echo [${each.key}] >> /etc/ansible/hosts
      echo ${self.public_ip} >> /etc/ansible/hosts
    EOT
  }
}
```
Now, create the variables file with all variables to be used in the `main.tf` config file.
```
vi variables.tf
```
```
### **Note:** Change the following Inputs in `variables.tf.`

### Edit the **Allocated Region** (**Ex:** ap-south-1) & **AMI ID** of same region,
### Replace the same **Security Group ID** Created for the Jump Server
### Add your Name for **KeyPair** ("**YourName**-CICDlab-KeyPair")

variable "region" {
    default = "us-east-1"
}

# Change the SG ID. You can use the same SG ID used for your CICD Jump server
# Basically the SG should open ports 22, 80, 8080, 9999, and 4243
variable  "sg_id" {
    default = "sg-02b8781c33d6f584d" # us-east-1
}

# Choose a free tier Ubuntu AMI. You can use below. 
variable "ami_id" {
    default = "ami-0e86e20dae9224db8" # us-east-1; Ubuntu
}

# We are only using t2.micro for this lab
variable "ins_type" {
    default = "t2.micro"
}

# Replace 'yourname' with your first name
variable key_name {
    default = "sirin-Jenkins-Docker-KeyPair"
}

variable public_key {
    default = "/home/ubuntu/.ssh/id_rsa.pub"   #Ubuntu OS
}

variable "my-servers" {
  type    = list(string)
  default = ["sirin-Jenkins-Server", "sirin-Docker-Server"]
}
```
Now, execute the terraform commands to launch the new servers
```
terraform init
```
```
terraform plan
```
```
terraform apply -auto-approve
```
Once the Changes are Applies, Go to `EC2 Dashboard` and check that `2 New Instances` are launched. Also check the `inventory file` and ensure the below output.
```
cat /etc/ansible/hosts
```
![image](https://github.com/user-attachments/assets/b882382f-ba8b-4ca1-b9e7-a96bdbf664a9)


### Task 6: Check the access from `Jump Server to Jenkins` and `Jump Server to Docker`

From `Jump Server` SSH into `Jenkins-Server`, check they are accessible.

```
ssh ubuntu@<Jenkins ip address>
```
Set the hostname
```
sudo hostnamectl set-hostname Jenkins
bash
```
```
sudo apt update
```
Exit from the Jenkins server
```
exit
```
```
exit
```

Now from `Jump Server` SSH into `Docker-Server` and check they are accessible.
```
ssh ubuntu@<Docker ip address>  
```
Set the hostname as
```
sudo hostnamectl set-hostname Docker
bash
```
```
sudo apt update
```
Exit only from the Docker Server, not the Jump Server.
```
exit
```
```
exit
```


### Task 7: Use `Ansible` to deploy respective packages onto each of the servers 
In Jump Server Create a directory and change to it
```
cd ~ && mkdir ansible && cd ansible
```
Now, Create a playbook, which will deploy packages onto the `Docker-server` and `Jenkins-Server.` For this Create a new File with the name `main.yaml.`
```
vi main.yaml
```
Copy and paste the below code and save it.
```
---
- name: Install Jenkins (single file - final)
  hosts: sirin-Jenkins-Server
  become: yes
  gather_facts: yes

  vars:
    apt_lock_timeout: 900

    # Jenkins weekly repo + 2026 signing key (post key-rotation) [1](https://bing.com/search?q=Jenkins+minimum+required+Java+version+21+supported+Java+versions+21+25+May+2026)
    jenkins_repo_url: "https://pkg.jenkins.io/debian"
    jenkins_key_url: "https://pkg.jenkins.io/debian/jenkins.io-2026.key"
    jenkins_keyring: "/usr/share/keyrings/jenkins-keyring.asc"
    jenkins_repo: "deb [signed-by={{ jenkins_keyring }}] {{ jenkins_repo_url }} binary/"

    # Jenkins on your host requires Java 21+ (your journal showed min Java 21) [2](https://www.jenkins.io/blog/2023/03/27/repository-signing-keys-changing/)
    jenkins_java_pkg: "openjdk-21-jdk"
    jenkins_java_home: "/usr/lib/jvm/java-21-openjdk-amd64"

  pre_tasks:
    - name: Wait for apt/dpkg locks to be released
      ansible.builtin.shell: |
        set -e
        timeout {{ apt_lock_timeout }} bash -c '
          while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 \
             || fuser /var/lib/dpkg/lock >/dev/null 2>&1 \
             || fuser /var/lib/apt/lists/lock >/dev/null 2>&1 \
             || fuser /var/cache/apt/archives/lock >/dev/null 2>&1; do
            echo "APT/DPKG locked by another process. Waiting..."
            sleep 5
          done
        '
      args:
        executable: /bin/bash
      changed_when: false

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Install required packages (Java 21 + repo utilities)
      ansible.builtin.apt:
        name:
          - "{{ jenkins_java_pkg }}"
          - ca-certificates
          - curl
          - gnupg
          - fontconfig
        state: present
        update_cache: yes
        lock_timeout: "{{ apt_lock_timeout }}"

    # Cleanup old Jenkins repo/key entries to avoid Signed-By conflicts
    - name: Remove old Jenkins repo list files (if any)
      ansible.builtin.shell: rm -f /etc/apt/sources.list.d/jenkins*.list
      args:
        executable: /bin/bash
      changed_when: false

    - name: Remove old Jenkins keyring from /etc/apt/keyrings (if any)
      ansible.builtin.file:
        path: /etc/apt/keyrings/jenkins-keyring.asc
        state: absent

    - name: Ensure /usr/share/keyrings exists
      ansible.builtin.file:
        path: /usr/share/keyrings
        state: directory
        mode: "0755"

    - name: Download Jenkins signing key (2026)
      ansible.builtin.get_url:
        url: "{{ jenkins_key_url }}"
        dest: "{{ jenkins_keyring }}"
        mode: "0644"

    - name: Add Jenkins apt repository (signed-by)
      ansible.builtin.apt_repository:
        repo: "{{ jenkins_repo }}"
        filename: jenkins
        state: present

    - name: Update apt cache after adding Jenkins repo
      ansible.builtin.apt:
        update_cache: yes
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Install Jenkins
      ansible.builtin.apt:
        name: jenkins
        state: present
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Set JAVA_HOME for Jenkins (Java 21)
      ansible.builtin.lineinfile:
        path: /etc/default/jenkins
        regexp: '^JAVA_HOME='
        line: "JAVA_HOME={{ jenkins_java_home }}"

    - name: Reload systemd
      ansible.builtin.command: systemctl daemon-reload
      changed_when: false

    - name: Enable and restart Jenkins
      ansible.builtin.service:
        name: jenkins
        enabled: yes
        state: restarted


- name: Install Docker (single file - final, stable)
  hosts:  sirin-Docker-Server
  become: yes
  gather_facts: yes

  vars:
    apt_lock_timeout: 900
    docker_keyring: /etc/apt/keyrings/docker.gpg
    docker_repo: "deb [signed-by={{ docker_keyring }}] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"

  pre_tasks:
    - name: Wait for apt/dpkg locks to be released
      ansible.builtin.shell: |
        set -e
        timeout {{ apt_lock_timeout }} bash -c '
          while fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1 \
             || fuser /var/lib/dpkg/lock >/dev/null 2>&1 \
             || fuser /var/lib/apt/lists/lock >/dev/null 2>&1 \
             || fuser /var/cache/apt/archives/lock >/dev/null 2>&1; do
            echo "APT/DPKG locked by another process. Waiting..."
            sleep 5
          done
        '
      args:
        executable: /bin/bash
      changed_when: false

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Install prerequisites
      ansible.builtin.apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present
        update_cache: yes
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Ensure /etc/apt/keyrings exists
      ansible.builtin.file:
        path: /etc/apt/keyrings
        state: directory
        mode: "0755"

    - name: Download Docker repository key
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/ubuntu/gpg
        dest: /tmp/docker.gpg.key
        mode: "0644"

    - name: Convert Docker key to keyring
      ansible.builtin.shell: |
        set -o pipefail
        cat /tmp/docker.gpg.key | gpg --dearmor -o "{{ docker_keyring }}"
      args:
        executable: /bin/bash

    - name: Set permissions on Docker keyring
      ansible.builtin.file:
        path: "{{ docker_keyring }}"
        mode: "0644"

    - name: Add Docker apt repository
      ansible.builtin.apt_repository:
        repo: "{{ docker_repo }}"
        filename: docker
        state: present

    - name: Update apt cache after adding Docker repo
      ansible.builtin.apt:
        update_cache: yes
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Install Docker Engine
      ansible.builtin.apt:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
        state: present
        lock_timeout: "{{ apt_lock_timeout }}"

    - name: Enable and start Docker
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes
```
Run the above playbook to deploy the packages onto target servers.
```
ansible-playbook main.yaml
```
![image](https://github.com/user-attachments/assets/08b775c7-3f56-4aac-87bd-c0f97f01badb)


### Task 8 : Verify the Jenkins and Docker landing page.
Verify the Jenkins Landing Page: Open a web browser and navigate to your Jenkins landing page using your Jenkins server's public IP address. Replace `<Your_Jenkins_IP>` with the actual public IP.
```
http://<Your_Jenkins_IP>:8080/
```
Verify the Docker Landing Page: Open a web browser and access the Docker landing page using your Docker server's public IP address. Replace `<Your_Docker_IP>` with the actual public IP.
```
http://<Your_Docker_IP>:4243/version
```
![image](https://github.com/user-attachments/assets/0fef3915-1827-4d21-8ecf-4041d0ad5d5a)

--------
![image](https://github.com/user-attachments/assets/79549c26-2e4c-453c-aef9-7d1500207ddc)




