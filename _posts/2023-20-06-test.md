---
title: Automating deployments using Terraform with Proxmox and ansible
date: 2023-05-06 10:03:29 +0100
image:
  name:
categories: [Automation]
tags: [terraform, proxmox, ansible, automation]
math: false
mermaid: false
---

Over the years my home lab has grown and become more and more difficult to maintain, especially because some servers I build and forget as they function so well.

I have found recently though that moving to newer versions of operating systems can be difficult for the servers that I cant easily containerise at the moment.

For this reason I have moved over to using Terraform with Proxmox and ansible.

Telemate developed a Terraform provider that maps Terraform functionality to the Proxmox API, so start by defining the use of that provider in `provider.tf`

```terraform
terraform {
  required_version = ">=0.13.0"

  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "2.9.14"
    }
  }
}

provider "proxmox" {
  # Configuration options
  pm_api_url = var.proxmox_api_url
  pm_api_token_id = var.proxmox_api_token_id
  pm_api_token_secret = var.proxmox_api_token_secret

  # Optional: skip TLS Verification
  pm_tls_insecure = true
  pm_parallel     = 2
  pm_timeout      = 1200
}
```

{: file='provider.tf'}

Here we see two sections, the first of which contains the configuration for Terraform, it specifies the version of Terraform that is required along with all of the required providers and their required versions.

The second section contains the provider itself, and the configuration, a full list of arguments can be found [here](https://github.com/Telmate/terraform-provider-proxmox/blob/master/docs/index.md#argument-reference)

A provider is a plugin that allows Terraform to manage external APIs (such as proxmox)

Now this is no good without some servers or "resources", I create a file per resource for my lab, this keeps it simple for me, however you may want to do this differently depending on your requirements. Create a file with the resource name `bastion.tf` There is a lot more going on in this file, so I have added comments to it.

```terraform
# Specify the resource type, and then the name
resource "proxmox_vm_qemu" "bastion" {

  name = "td-bast01" # Name to call the VM
  desc = "Bastion" # Description for the VM
  target_node = var.proxmox_host # Proxmox target node

  clone = var.template_name  # The name of the template that this resource will be created from

  agent = 1 # is the qemu agent installed?

  os_type = "cloud-init" # The OS type of the image clone
  cores = 2 # number of CPU cores
  sockets = 1 # number of CPU sockets
  cpu = "host" # The CPU type
  memory = 4096 # Amount of memory to allocate
  onboot = true # start the VM on host startup
  scsihw = "virtio-scsi-pci" # Scsi hardware type
  bootdisk = "scsi0" # The boot disk scsi

  # This section contains the disk configuration, it can be duplicated for additional disks
  disk {
    size = "30G"
    type = "scsi"
    storage = "local-thin"
    iothread = 0
  }

  # if you want two NICs, just copy this whole network section and duplicate it
  network {
    model = "virtio"
    bridge = "vmbr0"
    tag = 20
  }

  ipconfig0 = "ip=172.16.20.43/24,gw=172.16.20.1"
  nameserver = "172.16.20.1"

  serial {
    id = 0
    type = "socket"
  }
  # sshkeys set using variables. the variable contains the text of the key.
  sshkeys = <<EOF
  ${var.ssh_key}
  EOF

  # Terraform has provisioners that allow the execution of commands / scripts on a local or remote machine.
  # Here I execute a command to update the VM.
  provisioner "remote-exec" {
    inline = ["sudo apt update", "sudo apt upgrade -y", "echo Done!"]

    connection {
      host        = "172.16.20.43"
      type        = "ssh"
      user        = "ubuntu"
      private_key = file(var.private_key_path)
    }
  }
  # Here I execute an ansible playbook to configure the VM.
  # I specify ANSIBLE_HOST_KEY_CHECKING as if run a second time and the VM is rebuil ansible wont connect unless this is set to false.
  provisioner "local-exec" {
    command = "ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -u ${var.ansible_user}  -l bastion -i ../ansible-deploy/inventory --private-key ${var.private_key_path} -e 'pub_key=${var.public_key_path}' --ssh-extra-args '-o UserKnownHostsFile=/dev/null' ../ansible-deploy/main.yml"
  }
}
```

{: file='bastion.tf'}

Through all of the files creates you will have noticed variables have been used against various configuration parameters, before they will work they need to be defined in a file, we will call this `vars.tf`

```terraform
# provider vars
variable "proxmox_api_url"{
  type = string
}

variable "proxmox_api_token_id" {
  type = string
}

variable "proxmox_api_token_secret" {
  type = string
}

# resource vars
variable "proxmox_host" {
  type = string
  default = "td-vh01"
}
variable "template_name" {
  type = string
  default = "jammy-template"
}
variable "ansible_user"{
  default = "ubuntu"
  type = string
}

variable "private_key_path"{
  type = string
}

variable "public_key_path"{
  type = string
}

variable "ssh_key" {
  type = string
  default = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINm8L9RZ4lDsRDm6Zx7jqOrQx9mO7FphqcrV5teyGVJN"
}

```

{: file='vars.tf'}

Now that we have defined the variables create a credential file to store all our variable information:

```terraform
proxmox_api_url = "https://127.0.0.1:8006/api2/json"
proxmox_api_token_id = ""
proxmox_api_token_secret = ""
private_key_path = "~/.ssh/id_ed25519"
public_key_path = "~/.ssh/id_ed25519.pub"
ansible_user = ""
```

{: file='credentials.auto.tfvars'}

> Don't commit this file to Git as it contains sensitive information
{: .prompt-warning }

Any variables in `vars.tf` that have a default value don't need to be defined in the credential file if the default value is sufficient.

## The Cloud-Init template

The configuration that is used utilises a cloud-init template, check out my previous post ([Proxmox template with cloud image and cloud init]({% post_url 2022-10-04-proxmox-template-with-cloud-image-and-cloud-init %})) where I cover how to set this up for use in Proxmox with Terraform.

## Usage

Now all of the files we require are created, lets get it running:

1. Install Terraform and Ansible

    ```shell
    apt install -y terraform ansible
    ```

1. Enter the directory where your Terraform files reside
1. Run `terraform init`, this will initialize your Terraform configuration and pull all the required providers.
1. Ensure that you have the `credential.auto.tfvars` file created and with your variables populated
1. Run `terraform plan -out plan` and if everything seems good `terraform apply`.

> Use `terraform apply --auto-approve` to automatically apply without a prompt
{: .prompt-tip }

> To destroy the infrastructure, run `terraform destroy`
{: .prompt-tip }

## Final Thoughts

There is so much more potential using Terraform and Ansible. I have just scratched the surface, but you could automate everything up to firewall configuration as well, this is something I still need to look into, but it would be great to deploy and configure the firewall based on each individual device.

If you have any cool ideas for using Terraform and Ansible please let me know in the comments below!

Until next time...
