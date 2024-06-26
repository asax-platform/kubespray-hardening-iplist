# Equinix Metal

Kubespray provides support for bare metal deployments using the [Equinix Metal](http://metal.equinix.com).
Deploying upon bare metal allows Kubernetes to run at locations where an existing public or private cloud might not exist such
as cell tower, edge collocated installations. The deployment mechanism used by Kubespray for Equinix Metal is similar to that used for
AWS and OpenStack clouds (notably using Terraform to deploy the infrastructure). Terraform uses the Equinix Metal provider plugin
to provision and configure hosts which are then used by the Kubespray Ansible playbooks. The Ansible inventory is generated
dynamically from the Terraform state file.

## Local Host Configuration

To perform this installation, you will need a localhost to run Terraform/Ansible (laptop, VM, etc) and an account with Equinix Metal.
In this example, we are provisioning a m1.large CentOS7 OpenStack VM as the localhost for the Kubernetes installation.
You'll need Ansible, Git, and PIP.

```bash
sudo yum install epel-release
sudo yum install ansible
sudo yum install git
sudo yum install python-pip
```

## Playbook SSH Key

An SSH key is needed by Kubespray/Ansible to run the playbooks.
This key is installed into the bare metal hosts during the Terraform deployment.
You can generate a key new key or use an existing one.

```bash
ssh-keygen -f ~/.ssh/id_rsa
```

## Install Terraform

Terraform is required to deploy the bare metal infrastructure. The steps below are for installing on CentOS 7.
[More terraform installation options are available.](https://learn.hashicorp.com/terraform/getting-started/install.html)

Grab the latest version of Terraform and install it.

```bash
echo "https://releases.hashicorp.com/terraform/$(curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | jq -r -M '.current_version')/terraform_$(curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | jq -r -M '.current_version')_linux_amd64.zip"
sudo yum install unzip
sudo unzip terraform_0.14.10_linux_amd64.zip -d /usr/local/bin/
```

## Download Kubespray

Pull over Kubespray and setup any required libraries.

```bash
git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray
```

## Install Ansible

Install Ansible according to [Ansible installation guide](/docs/ansible/ansible.md#installing-ansible)

## Cluster Definition

In this example, a new cluster called "alpha" will be created.

```bash
cp -LRp contrib/terraform/packet/sample-inventory inventory/alpha
cd inventory/alpha/
ln -s ../../contrib/terraform/packet/hosts
```

Details about the cluster, such as the name, as well as the authentication tokens and project ID
for Equinix Metal need to be defined. To find these values see [Equinix Metal API Accounts](https://metal.equinix.com/developers/docs/accounts/).

```bash
vi cluster.tfvars
```

* cluster_name = alpha
* packet_project_id = ABCDEFGHIJKLMNOPQRSTUVWXYZ123456
* public_key_path = 12345678-90AB-CDEF-GHIJ-KLMNOPQRSTUV

## Deploy Bare Metal Hosts

Initializing Terraform will pull down any necessary plugins/providers.

```bash
terraform init ../../contrib/terraform/packet/
```

Run Terraform to deploy the hardware.

```bash
terraform apply -var-file=cluster.tfvars ../../contrib/terraform/packet
```

## Run Kubespray Playbooks

With the bare metal infrastructure deployed, Kubespray can now install Kubernetes and setup the cluster.

```bash
ansible-playbook --become -i inventory/alpha/hosts cluster.yml
```
