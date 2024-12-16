# Slurm Cluster Setup with GPU Monitoring

__GPUd:__ GPUd is a lightweight real-time monitor of your cluster's GPUs. It detects and reports hardware and system-level issues, enabling earlier signaling of downtime.

__Lepton AI:__ Lepton AI integrates with GPUd to offer a dashboard for monitoring GPU health and performance across your cluster.

## Foundations

### Clone the Slurm Crusoe Template project

```bash
git clone https://github.com/crusoecloud/slurm.git
```

### Populate `~/.crusoe/config`

```properties
[default]
access_key_id = "your_access_key"
default_project = "default"
secret_key = "your_secret_key"
```

### Modify the example under `slurm/examples/example.tfvars`


```terraform
location = "your_cluster_location"
project_id = "your_project_id"
ssh_public_key_path = "~/.ssh/id_rsa.pub"
vpc_subnet_id = "your_vpn_subnet_id"

slurm_head_node_count = 1
slurm_head_node_type = "c1a.8x"
slurm_login_node_count = 1
slurm_login_node_type = "a40.1x"
slurm_nfs_node_type = "s1a.20x"
slurm_nfs_home_size = "128GiB"
slurm_compute_node_type = "a40.1x"
slurm_compute_node_count = 1

slurm_users = [{
  name = "your.name"
  uid = 1001 
  ssh_pubkey = "ssh-rsa KEY00001 your.name@localhost"
}]
```


### Install Ansible on the Controller Machine


**Debian/Ubuntu:**

```bash
sudo apt update
sudo apt install ansible
```

**Red Hat/CentOS:**

```bash
sudo yum install ansible
```

**macOS:**

```bash
brew install ansible
```

### Initialize and Apply Terraform


```bash
terraform init
terraform apply
```

## GPUd Installation and Configuration


### Create an Ansible Playbook `./gpud_install.yml`


```yaml
---
- name: Install and Configure gpud
  hosts: all
  become: true
  user: ubuntu
  tasks:
    - name: Check if GPU is present
      shell: lspci | grep -i nvidia
      register: gpu_check
      ignore_errors: true
      changed_when: false

    - name: Skip if no GPU found
      when: gpu_check.rc != 0
      debug:
        msg: "No NVIDIA GPU detected. Skipping gpud installation on {{ inventory_hostname }}."

    - name: Install gpud
      when: gpu_check.rc == 0
      block:
        - name: Download and execute gpud install script
          shell: curl -fsSL https://pkg.gpud.dev/install.sh | sh
          register: gpud_install_output

        - name: Start gpud with optional Lepton AI token
          shell: "sudo gpud up {{ ' --token ' + lepton_ai_token if lepton_ai_token is defined else '' }}"
          register: gpud_up_output
          changed_when: "'successfully started gpud' not in gpud_up_output.stdout"

        - name: Display gpud start output
          debug:
            msg: "{{ gpud_up_output.stdout }}"
        - name: Display gpud install output
          debug:
            msg: "{{ gpud_install_output.stdout }}"

```

### Configure Ansible Inventory

```bash
crusoe compute vms list -f json | jq -r '.[] | "\(.name) ansible_host=\(.network_interfaces[0].ips[0].public_ipv4.address)"' > ./hosts.ini
```

### Run the Ansible Playbook

### Local Dashboards Only
```bash
ansible-playbook -i ./hosts.ini ./gpud_install.yml 
```

#### Local Dashboard Access

_Assuming `204.52.16.221` is the public IP address of a Slurm node_

```bash
ssh -f your.name@204.52.16.221 -L 15132:localhost:15132 -N
```

open a web browser and navigate to `http://localhost:15132`.


### Using Lepton.ai
```bash
ansible-playbook -i ./hosts.ini ./gpud_install.yml -e "lepton_ai_token=<YOUR_LEPTON_AI_TOKEN>"
```
#### Connect to Lepton.ai Dashboard

If you provided a Lepton AI token, your nodes should now be connected to your Lepton AI dashboard. You can view the GPU metrics in the Lepton AI dashboard under Utilities -> Machines.
