# K3s Deployment with Ansible

This repository contains instructions and configurations for deploying Kubernetes K3s cluster using Ansible.

## References

- [Techno Tim's k3s-ansible Repository](https://github.com/techno-tim/k3s-ansible)
- [YouTube Tutorial](https://www.youtube.com/watch?v=CbkEWcUZ7zM)

## Requirements

### Hardware
- Minimum 3 VMs:
  - 1 Master Node
  - 1 Worker Nodes
  - 1 Ansible Server (for deploying k3s)

### System Preparation
1. Update and upgrade all VM machines
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. Install Ansible on all VMs
   - Follow [official Ansible documentation](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html)

3. User Configuration:
   - Create the same user on all VMs
   - Add the user to sudo group and grant full permissions:
     ```bash
     sudo usermod -aG sudo username
     echo "username ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/username
     ```

4. Update NTP time on all VMs:
   ```bash
   sudo timedatectl set-ntp off
   sudo timedatectl set-time "YYYY-MM-DD HH:MM:SS"
   sudo timedatectl set-timezone "Region/City"
   sudo systemctl restart systemd-timesyncd
   sudo timedatectl set-ntp on
   ```

5. Set hostname for all VMs
   ```bash
   sudo hostnamectl set-hostname <hostname>
   ```

6. Install required library on Ansible server:
   ```bash
   sudo pip3 install netaddr
   ```
## 8. Communications

Ensure that the necessary rules and policies for routing are properly configured when the virtual machines (VMs) are not within the same network.



## Deployment Steps

### 1. Create SSH Key on Ansible Server

Create SSH key for password-less authentication:
```bash
ssh-keygen -b 2048 -t rsa
```

Verify the key creation:
```bash
cd ~/.ssh
ls
```
You should see `id_rsa` (private key) and `id_rsa.pub` (public key)

### 2. Copy SSH Public Key to All VMs

Create/update authorized_keys file on each target VM:
```bash
cd ~/.ssh/
# Create the authorized_keys file if it doesn't exist
touch authorized_keys
# Add the public key from the Ansible server to the authorized_keys file
# The content of id_rsa.pub from the Ansible server should be added here
```

Verify connection from Ansible server to VMs without password:
```bash
ssh -i ~/.ssh/id_rsa username@VM_IP_ADDRESS
```

### 3. Clone the Repository

On the Ansible server:
```bash
git clone https://github.com/shlomo-b/k3s.git
```

### 4. Configure the Deployment

Edit configuration files:

#### site.yml
- Set `apiserver_endpoint` (VIP) when using multiple master nodes
- Set K3s version (e.g., v1.30)

#### hosts.ini
- Define the IPs for master and worker nodes

#### ping.yml
- Use for testing communication between Ansible server and nodes

### 5. Test Connectivity

Test communication from Ansible server to worker nodes:
```bash
ansible-playbook -i hosts ping.yml
```

Verify connectivity to the master node:
```bash
ansible-playbook -i hosts.ini ping.yml
```

You should see "REACHABLE" for all nodes if connectivity is successful.

### 6. Deploy K3s

Run the deployment playbook:
```bash
ansible-playbook site.yml -i hosts.ini -vvv
```

If you encounter errors with apiserver, add the following to your hosts.ini:
```
[all:vars]
apiserver_endpoint=<IP_ADDRESS>
```

### 7. Verify Deployment

On the master node:
```bash
kubectl get nodes
```

To change the role of a node (for example, if node3 doesn't have a proper role assigned):
```bash
k3s kubectl label node node3 node-role.kubernetes.io/worker=worker
```

For detailed node information:
```bash
kubectl get nodes -o wide
```

### 8. Remove K3s

Run the reset playbook:
```bash
ansible-playbook reset.yml -i hosts.ini -vvv
```

## Notes

- The apiserver_endpoint (VIP) is necessary when using multiple master nodes for high availability
- With only one master node, a VIP is optional but useful for future scalability
- Ensure the same username is used across all VMs for consistency
- NTP synchronization is critical for proper cluster operation