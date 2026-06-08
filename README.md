# Ansible KVM + Ceph RBD VM Provisioning

This project provisions Linux virtual machines on a KVM/libvirt host using **Ansible**, with VM root disks stored directly on **Ceph RBD**.

It automates the full VM lifecycle required for initial provisioning:

* Install required packages on the KVM host
* Define a libvirt bridge network backed by an existing Linux bridge
* Register a Ceph secret in libvirt
* Import a cloud image into Ceph RBD
* Resize the RBD root disk
* Generate per-VM cloud-init ISO files
* Render libvirt domain XML files
* Define and start VMs through `virsh`
* Clean up temporary local artifacts

---

## Architecture Overview

The architecture is simple and production-friendly:

```text
+----------------------+
|   Ansible Control    |
|   localhost / KVM    |
+----------+-----------+
           |
           | virsh / libvirt
           |
+----------v-----------+
|      KVM Host        |
|                      |
|  br0 Linux Bridge    |
|  libvirt br0-net     |
|  QEMU/KVM            |
+----------+-----------+
           |
           | RBD protocol
           |
+----------v-----------+
|      Ceph Cluster    |
|                      |
|  pool: vm-rbd        |
|  user: client.libvirt|
|  image: vm01-root    |
|  image: vm02-root    |
+----------------------+
```

Each VM uses:

* Ceph RBD as the root disk
* A cloud-init ISO as the seed disk
* Static IP configuration via cloud-init network config
* Password-based SSH access for the configured user
* A libvirt bridged network connected to the host bridge `br0`

---

## Repository Structure

```text
.
├── group_vars
│   └── all.yml
├── inventory
│   └── vms.yml
└── playbooks
    ├── provision.yml
    └── roles
        ├── ceph_rbd
        │   └── tasks
        │       └── main.yaml
        ├── cleanup
        │   └── tasks
        │       └── main.yaml
        ├── cloud_init
        │   ├── tasks
        │   │   └── main.yaml
        │   └── templates
        │       ├── meta-data.j2
        │       ├── network-config.j2
        │       └── user-data.j2
        ├── common
        │   └── tasks
        │       └── main.yaml
        ├── libvirt_vm
        │   ├── tasks
        │   │   └── main.yaml
        │   └── templates
        │       └── vm.xml.j2
        └── network
            └── tasks
                └── main.yml
```

---

## Requirements

This project is designed to run on the KVM host itself.

The Ansible playbook uses:

```yaml
hosts: localhost
connection: local
become: true
```

So you should run it directly on the machine that has:

* KVM
* libvirt
* access to the Ceph cluster
* access to the existing Linux bridge `br0`
* the local cloud image file

---

## Host Requirements

The KVM host should have:

* Ubuntu/Debian-based OS
* KVM enabled
* libvirt installed and running
* QEMU installed
* Ceph client access
* Ansible installed
* an existing Linux bridge named `br0`

Check KVM support:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

If the output is greater than `0`, CPU virtualization is available.

Check libvirt:

```bash
systemctl status libvirtd
virsh list --all
```

Check the bridge:

```bash
ip link show br0
bridge link
```

The playbook defines a libvirt network named `br0-net`, but it expects the actual Linux bridge `br0` to already exist on the host.

---

## Install Basic Dependencies

Install Ansible and libvirt tooling:

```bash
sudo apt update
sudo apt install -y \
  ansible \
  qemu-kvm \
  libvirt-daemon-system \
  libvirt-clients \
  virtinst \
  bridge-utils
```

Start and enable libvirt:

```bash
sudo systemctl enable --now libvirtd
```

---

## Ceph Requirements

The Ceph cluster must already exist.

You need:

* a Ceph pool for VM disks
* a Ceph client user for libvirt
* a valid Ceph key
* monitor IP addresses
* working Ceph client connectivity from the KVM host

Example values used by this project:

```yaml
ceph_pool: vm-rbd
ceph_user: libvirt

ceph_monitors:
  - 192.168.1.201
  - 192.168.1.202
  - 192.168.1.203
```

This means the project expects a Ceph user like:

```text
client.libvirt
```

and RBD images like:

```text
vm-rbd/vm01-root
vm-rbd/vm02-root
```

---

## Example Ceph User

On a Ceph admin node, you can create a dedicated libvirt client user.

Example:

```bash
ceph auth get-or-create client.libvirt \
  mon 'profile rbd' \
  osd 'profile rbd pool=vm-rbd' \
  mgr 'profile rbd pool=vm-rbd'
```

Then get the key:

```bash
ceph auth get-key client.libvirt
```

Use that value in:

```yaml
ceph_secret_value: "BASE64_OR_CEPH_CLIENT_KEY"
```

The current playbook stores this value in:

```yaml
ceph_secret_value: "AQDpfiFqg1p0MxAAiPG5yVEl3bqvvL+9vG6LqQ=="
```

> Important: Do not commit real production Ceph keys to a public repository. Use Ansible Vault or CI/CD secrets instead.

---

## Cloud Image Requirement

This project imports a local cloud image into Ceph RBD.

Configured path:

```yaml
local_cloud_image: /opt/ubuntu-22.04-converted.raw
```

The image should exist before running the playbook.

Example download:

```bash
cd /opt

sudo wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img \
  -O ubuntu-22.04.qcow2
```

Convert it to raw:

```bash
sudo qemu-img convert -O raw \
  /opt/ubuntu-22.04.qcow2 \
  /opt/ubuntu-22.04-converted.raw
```

Verify:

```bash
qemu-img info /opt/ubuntu-22.04-converted.raw
```

---

## Configuration Files

### Global Variables

File:

```text
group_vars/all.yml
```

Example:

```yaml
ceph_pool: vm-rbd
ceph_user: libvirt

disk_size: 50G

local_cloud_image: /opt/ubuntu-22.04-converted.raw

ceph_monitors:
  - 192.168.1.201
  - 192.168.1.202
  - 192.168.1.203

ceph_secret_uuid: "11111111-2222-3333-4444-555555555555"
ceph_secret_value: "AQDpfiFqg1p0MxAAiPG5yVEl3bqvvL+9vG6LqQ=="

ssh_user: ubuntu
ssh_password: "Xyz@123321"

vm_gateway: 192.168.1.1
```

Variable explanation:

| Variable            | Description                                             |
| ------------------- | ------------------------------------------------------- |
| `ceph_pool`         | Ceph RBD pool used for VM root disks                    |
| `ceph_user`         | Ceph client user used by libvirt/QEMU                   |
| `disk_size`         | Final size of each VM root disk                         |
| `local_cloud_image` | Local raw cloud image that will be imported into RBD    |
| `ceph_monitors`     | Ceph monitor IP addresses used by libvirt XML           |
| `ceph_secret_uuid`  | UUID of the libvirt secret used for Ceph authentication |
| `ceph_secret_value` | Ceph client key                                         |
| `ssh_user`          | User created inside each VM by cloud-init               |
| `ssh_password`      | Password configured for the VM user                     |
| `vm_gateway`        | Default gateway configured inside each VM               |

---

### VM Inventory

File:

```text
inventory/vms.yml
```

Example:

```yaml
vms:
  - name: vm01
    ip: 192.168.1.50
    memory: 4194304
    vcpu: 3

  - name: vm02
    ip: 192.168.1.51
    memory: 4194304
    vcpu: 2
```

Memory is configured in KiB because libvirt XML uses:

```xml
<memory unit='KiB'>{{ vm.memory }}</memory>
```

Example:

```text
4194304 KiB = 4 GiB
```

Each VM entry creates:

* one Ceph RBD root disk
* one cloud-init ISO
* one libvirt domain XML
* one running VM

---

## Provisioning Pipeline

The main playbook is:

```text
playbooks/provision.yml
```

```yaml
- name: VM Provisioning Pipeline
  hosts: localhost
  connection: local
  become: true

  vars_files:
    - ../inventory/vms.yml
    - ../group_vars/all.yml

  roles:
    - common
    - network
    - ceph_rbd
    - cloud_init
    - libvirt_vm
    - cleanup
```

Execution order:

```text
common
  ↓
network
  ↓
ceph_rbd
  ↓
cloud_init
  ↓
libvirt_vm
  ↓
cleanup
```

---

## Role: common

Path:

```text
playbooks/roles/common/tasks/main.yaml
```

This role installs the required packages:

```yaml
- name: Install required packages
  apt:
    pkg:
      - genisoimage
      - python3-passlib
      - ceph-common
      - libvirt-clients
      - qemu-utils
    state: present
```

Installed tools:

| Package           | Purpose                                    |
| ----------------- | ------------------------------------------ |
| `genisoimage`     | Creates cloud-init ISO files               |
| `python3-passlib` | Allows Ansible to generate password hashes |
| `ceph-common`     | Provides `rbd` and Ceph client tools       |
| `libvirt-clients` | Provides `virsh`                           |
| `qemu-utils`      | Provides `qemu-img`                        |

---

## Role: network

Path:

```text
playbooks/roles/network/tasks/main.yml
```

This role defines a libvirt network named `br0-net`.

```xml
<network>
  <name>br0-net</name>
  <forward mode='bridge'/>
  <bridge name='br0'/>
</network>
```

It then runs:

```bash
virsh net-define /tmp/net.xml
virsh net-start br0-net
virsh net-autostart br0-net
```

This does not create the Linux bridge `br0`.

It only tells libvirt to use the already existing host bridge.

Verify:

```bash
virsh net-list --all
virsh net-info br0-net
```

Expected result:

```text
Name:           br0-net
Active:         yes
Autostart:      yes
```

---

## Role: ceph_rbd

Path:

```text
playbooks/roles/ceph_rbd/tasks/main.yaml
```

This role handles Ceph/libvirt integration and VM disk creation.

It performs these steps:

1. Defines a libvirt Ceph secret
2. Sets the secret value
3. Creates a temporary Ceph keyring
4. Checks whether each VM RBD image already exists
5. Builds a list of missing RBD images
6. Imports the cloud image into Ceph
7. Resizes the imported image to the configured size

The generated RBD image name format is:

```text
{{ ceph_pool }}/{{ vm.name }}-root
```

Example:

```text
vm-rbd/vm01-root
vm-rbd/vm02-root
```

Check RBD images manually:

```bash
rbd ls vm-rbd --id libvirt --keyring /tmp/ceph.keyring
```

Check one image:

```bash
rbd info vm-rbd/vm01-root --id libvirt --keyring /tmp/ceph.keyring
```

---

## Important Idempotency Behavior

The playbook checks whether an RBD image already exists.

If the image exists, it is skipped.

If the image does not exist, it is imported and resized.

This means:

```text
new VM RBD missing → create/import/resize/start
existing VM RBD found → skip disk creation
```

In the current implementation, the `libvirt_vm` role loops over:

```yaml
missing_rbd
```

So only VMs with newly created RBD disks are defined and started.

This is useful for first-time provisioning and adding new VMs, but it means existing VMs are not redefined on every run.

If you change the VM XML template and want to apply it to already existing VMs, you should either:

* manually redefine the domain
* destroy and undefine the VM
* change the role logic to loop over `vms` instead of `missing_rbd`

Manual redefine example:

```bash
virsh destroy vm01
virsh undefine vm01

ansible-playbook playbooks/provision.yml
```

---

## Role: cloud_init

Path:

```text
playbooks/roles/cloud_init/tasks/main.yaml
```

This role creates one cloud-init seed ISO per VM.

For each VM, it creates:

```text
/tmp/vm01/user-data
/tmp/vm01/meta-data
/tmp/vm01/network-config
```

Then it creates:

```text
/var/lib/libvirt/images/vm01-cloudinit.iso
```

The ISO is attached to the VM as a CD-ROM.

---

## Cloud-init user-data

Template:

```text
playbooks/roles/cloud_init/templates/user-data.j2
```

```yaml
#cloud-config
ssh_pwauth: true
users:
  - name: {{ ssh_user }}
    shell: /bin/bash
    sudo: 
      - ALL=(ALL) NOPASSWD:ALL
    lock_passwd: false
    password: {{ ssh_password | password_hash('sha512') }}

chpasswd:
  list: |
    {{ ssh_user }}:{{ ssh_password }}
  expire: false

growpart:
  mode: auto
  devices: ['/']
  ignore_growroot_disabled: false

resize_rootfs: true
```

This config:

* enables SSH password authentication
* creates the configured user
* grants passwordless sudo
* sets the user password
* grows the root partition
* resizes the root filesystem

---

## Cloud-init network-config

Template:

```text
playbooks/roles/cloud_init/templates/network-config.j2
```

```yaml
network:
  version: 2
  ethernets:
    enp1s0:
      dhcp4: false
      addresses:
        - {{ item.ip }}/24
      routes:
        - to: default
          via: {{ vm_gateway }}
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

Each VM gets a static IP from `inventory/vms.yml`.

Example:

```yaml
- name: vm01
  ip: 192.168.1.50
```

VM `vm01` will get:

```text
192.168.1.50/24
gateway: 192.168.1.1
dns: 1.1.1.1, 8.8.8.8
```

Important:

The network interface name inside the VM is expected to be:

```text
enp1s0
```

This depends on the libvirt device order and guest OS naming. If your VM does not get the IP address, check the actual interface name from the VM console and update the template if needed.

---

## Cloud-init meta-data

Template:

```text
playbooks/roles/cloud_init/templates/meta-data.j2
```

```yaml
instance-id: {{ item.name }}
local-hostname: {{ item.name }}
```

This sets the VM hostname.

---

## Role: libvirt_vm

Path:

```text
playbooks/roles/libvirt_vm/tasks/main.yaml
```

This role:

1. Renders a VM XML file into `/tmp`
2. Defines the VM with `virsh define`
3. Starts the VM with `virsh start`

Generated XML path:

```text
/tmp/vm01.xml
/tmp/vm02.xml
```

---

## Libvirt VM XML

Template:

```text
playbooks/roles/libvirt_vm/templates/vm.xml.j2
```

Main VM properties:

```xml
<domain type='kvm'>
  <name>{{ vm.name }}</name>
  <memory unit='KiB'>{{ vm.memory }}</memory>
  <currentMemory unit='KiB'>{{ vm.memory }}</currentMemory>
  <vcpu placement='static'>{{ vm.vcpu }}</vcpu>
```

The VM uses:

```xml
<cpu mode='host-passthrough' check='none' migratable='on'/>
```

This exposes the host CPU features directly to the guest.

The root disk is a Ceph RBD network disk:

```xml
<disk type='network' device='disk'>
  <driver name='qemu' type='raw'/>
  <auth username='{{ ceph_user }}'>
    <secret type='ceph' uuid='{{ ceph_secret_uuid }}'/>
  </auth>
  <source protocol='rbd' name='{{ ceph_pool }}/{{ vm.name }}-root'>
    {% for mon in ceph_monitors %}
    <host name='{{ mon }}' port='6789'/>
    {% endfor %}
  </source>
  <target dev='vda' bus='virtio'/>
</disk>
```

The cloud-init ISO is attached as a CD-ROM:

```xml
<disk type='file' device='cdrom'>
  <driver name='qemu' type='raw'/>
  <source file='/var/lib/libvirt/images/{{ vm.name }}-cloudinit.iso'/>
  <target dev='sda' bus='sata'/>
  <readonly/>
</disk>
```

The VM network is attached to the libvirt bridge network:

```xml
<interface type='network'>
  <source network='br0-net'/>
  <model type='virtio'/>
</interface>
```

---

## Ceph Monitor Port Note

The XML currently uses:

```xml
<host name='{{ mon }}' port='6789'/>
```

Port `6789` is the classic Ceph monitor v1 port.

If your Ceph cluster only exposes msgr2 on port `3300`, update the XML accordingly:

```xml
<host name='{{ mon }}' port='3300'/>
```

Use the port that matches your Ceph monitor configuration.

---

## Role: cleanup

Path:

```text
playbooks/roles/cleanup/tasks/main.yaml
```

This role removes temporary sensitive files:

```yaml
- name: Cleanup temp artifacts
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /tmp/ceph.keyring
    - /tmp/sec.xml
```

It removes:

```text
/tmp/ceph.keyring
/tmp/sec.xml
```

It does not delete:

* RBD images
* VMs
* cloud-init ISO files
* libvirt networks

---

## How to Run

Run from the project root:

```bash
sudo ansible-playbook playbooks/provision.yml
```

Or:

```bash
ansible-playbook playbooks/provision.yml --ask-become-pass
```

If the current user already has passwordless sudo, the first command is enough.

---

## Verify Provisioning

### Check libvirt network

```bash
virsh net-list --all
virsh net-info br0-net
```

### Check VMs

```bash
virsh list --all
```

Expected:

```text
 Id   Name   State
----------------------
 1    vm01   running
 2    vm02   running
```

### Check VM console

```bash
virsh console vm01
```

To exit the console:

```text
Ctrl + ]
```

### Check RBD images

Create a temporary keyring if needed:

```bash
cat >/tmp/ceph.keyring <<EOF
[client.libvirt]
  key = YOUR_CEPH_KEY
EOF
```

Then:

```bash
rbd ls vm-rbd --id libvirt --keyring /tmp/ceph.keyring
```

Expected:

```text
vm01-root
vm02-root
```

Check disk size:

```bash
rbd info vm-rbd/vm01-root --id libvirt --keyring /tmp/ceph.keyring
```

### Check SSH

```bash
ssh ubuntu@192.168.1.50
```

Default password from the sample config:

```text
Xyz@123321
```

---

## Adding a New VM

Edit:

```text
inventory/vms.yml
```

Add another VM:

```yaml
vms:
  - name: vm01
    ip: 192.168.1.50
    memory: 4194304
    vcpu: 3

  - name: vm02
    ip: 192.168.1.51
    memory: 4194304
    vcpu: 2

  - name: vm03
    ip: 192.168.1.52
    memory: 4194304
    vcpu: 2
```

Run:

```bash
sudo ansible-playbook playbooks/provision.yml
```

The playbook checks existing RBD images and only creates missing ones.

---

## Removing a VM

This project does not currently include a full destroy role.

To remove a VM manually:

```bash
virsh destroy vm01
virsh undefine vm01
```

Remove its cloud-init ISO:

```bash
rm -f /var/lib/libvirt/images/vm01-cloudinit.iso
```

Remove its RBD image:

```bash
cat >/tmp/ceph.keyring <<EOF
[client.libvirt]
  key = YOUR_CEPH_KEY
EOF

rbd rm vm-rbd/vm01-root --id libvirt --keyring /tmp/ceph.keyring
```

---

## Troubleshooting

### VM does not start

Check VM definition:

```bash
virsh dumpxml vm01
```

Check libvirt logs:

```bash
journalctl -u libvirtd -xe
```

Check QEMU logs:

```bash
ls -lah /var/log/libvirt/qemu/
cat /var/log/libvirt/qemu/vm01.log
```

---

### Ceph authentication error

Typical symptoms:

```text
error connecting to the cluster
Operation not permitted
failed to get secret
```

Check the libvirt secret:

```bash
virsh secret-list
virsh secret-dumpxml 11111111-2222-3333-4444-555555555555
```

Set the secret again:

```bash
virsh secret-set-value \
  --secret 11111111-2222-3333-4444-555555555555 \
  --base64 'YOUR_CEPH_KEY'
```

Check Ceph manually:

```bash
rbd ls vm-rbd --id libvirt --keyring /tmp/ceph.keyring
```

---

### RBD image already exists

If you see an error like:

```text
rbd: image already exists
```

It means the RBD image is already present in the Ceph pool.

Check it:

```bash
rbd info vm-rbd/vm01-root --id libvirt --keyring /tmp/ceph.keyring
```

Remove it only if you are sure:

```bash
rbd rm vm-rbd/vm01-root --id libvirt --keyring /tmp/ceph.keyring
```

---

### VM has no IP address

First check whether the VM booted:

```bash
virsh list --all
virsh console vm01
```

Inside the VM:

```bash
ip a
ip route
cloud-init status --long
```

Check cloud-init logs:

```bash
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log
```

Possible causes:

* wrong interface name in `network-config.j2`
* host bridge `br0` is not working
* libvirt network `br0-net` is inactive
* cloud-init ISO was not attached
* cloud-init did not detect the seed ISO

Check ISO attachment:

```bash
virsh dumpxml vm01 | grep cloudinit
```

Check libvirt network:

```bash
virsh net-info br0-net
```

Check bridge:

```bash
ip link show br0
bridge link
```

---

### Cloud-init did not run

Check if the ISO exists:

```bash
ls -lah /var/lib/libvirt/images/vm01-cloudinit.iso
```

Inspect ISO contents:

```bash
mkdir -p /mnt/cloudinit-test

sudo mount -o loop /var/lib/libvirt/images/vm01-cloudinit.iso /mnt/cloudinit-test

ls -lah /mnt/cloudinit-test
cat /mnt/cloudinit-test/user-data
cat /mnt/cloudinit-test/meta-data
cat /mnt/cloudinit-test/network-config

sudo umount /mnt/cloudinit-test
```

Expected files:

```text
user-data
meta-data
network-config
```

The ISO volume ID must be:

```text
cidata
```

The playbook already creates it using:

```bash
genisoimage -volid cidata
```

---

### Root disk did not resize inside VM

The playbook resizes the RBD image, and cloud-init is configured to grow the partition and filesystem.

Inside the VM:

```bash
lsblk
df -h
cloud-init status --long
```

If the disk is bigger but the partition/filesystem is not resized, run:

```bash
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
```

For XFS:

```bash
sudo xfs_growfs /
```

---

## Security Notes

This project currently uses plain YAML variables for sensitive values:

```yaml
ceph_secret_value: "..."
ssh_password: "..."
```

For real environments, use Ansible Vault:

```bash
ansible-vault encrypt group_vars/all.yml
```

Run with:

```bash
ansible-playbook playbooks/provision.yml --ask-vault-pass
```

Recommended improvements:

* use SSH keys instead of password authentication
* disable `ssh_pwauth` after initial provisioning
* store Ceph secrets in Ansible Vault or CI/CD secrets
* avoid committing real credentials
* restrict Ceph user capabilities to the required pool only
* use separate Ceph users per environment if needed

---

## Current Behavior Summary

This project currently provides:

| Feature                          | Status                         |
| -------------------------------- | ------------------------------ |
| Install required packages        | Yes                            |
| Define libvirt bridge network    | Yes                            |
| Use existing Linux bridge `br0`  | Yes                            |
| Define Ceph secret in libvirt    | Yes                            |
| Import cloud image into Ceph RBD | Yes                            |
| Resize RBD disk                  | Yes                            |
| Generate cloud-init ISO          | Yes                            |
| Static IP configuration          | Yes                            |
| Define libvirt VM                | Yes                            |
| Start libvirt VM                 | Yes                            |
| Cleanup temp keyring/secret XML  | Yes                            |
| Destroy VM                       | Manual                         |
| Remove RBD image                 | Manual                         |
| Reconfigure existing VM XML      | Manual / needs role adjustment |

---

## Suggested Future Improvements

Useful improvements for the next version:

* add a `destroy.yml` playbook
* add a role for deleting VMs and RBD images
* move secrets to Ansible Vault
* support SSH public keys
* support multiple bridges
* support per-VM gateway and DNS
* support per-VM disk size
* support cloud image download/conversion automatically
* support VM tags/metadata
* support `virtio-scsi`
* support UEFI/OVMF boot
* support idempotent VM XML updates
* support check mode where possible
* add CI linting with `ansible-lint` and `yamllint`

---

## Example Full Run

```bash
git clone <repo-url>
cd <repo>

sudo ansible-playbook playbooks/provision.yml
```

Verify:

```bash
virsh list --all
virsh net-list --all
rbd ls vm-rbd --id libvirt --keyring /tmp/ceph.keyring
ssh ubuntu@192.168.1.50
```

---

## Notes

This project is intended for environments where KVM hosts use Ceph RBD as the backend storage layer.

It is especially useful for:

* lab environments
* private cloud experiments
* Kubernetes node provisioning
* VM-based infrastructure automation
* Ceph-backed virtualization
* repeatable KVM provisioning without OpenStack or Proxmox

The main idea is:

```text
Ansible controls the flow
libvirt controls the VM
Ceph RBD stores the disk
cloud-init configures the guest
```
