# Ansible Role for TAK Server Installation on Ubuntu 22.04

This repository provides an automated way to install the official TAK Server + MediaMTX on a fresh Ubuntu 22.04 virtual machine using Ansible. Tested on AWS, Raspberry pi and locally with Multipass.

---

## Requirements

- Multipass (local dev)
- Ansible installed on your local computer
- ssh key pair (`~/.ssh/id_ed25519.pub` must exist on your local computer)
- You should be able to ssh into your future tak server via ssh, with ubuntu/pi user. Modify the `ansible_user` on `inventory.ini` if necessary

---

## Bootstrap a Virtual Machine with Multipass (local dev)

### 1. Launch the VM with a random name and inject your SSH public key

```bash
export TAK_VM_NAME=tak-test-$RANDOM

multipass launch 22.04 \
  --name "$TAK_VM_NAME" \
  --memory 4G \
  --disk 10G \
  --cpus 2 \
  --cloud-init - <<EOF
#cloud-config
users:
  - default
  - name: ubuntu
    ssh_authorized_keys:
      - $(cat ~/.ssh/id_ed25519.pub)
EOF
```

### 2. Get the VM IP (local dev)

```bash
VM_IP=$(multipass info "$TAK_VM_NAME" --format json | jq -r ".info[\"$TAK_VM_NAME\"].ipv4[0]")
echo "VM IP: $VM_IP"
```

---

## Prepare Ansible Inventory

### 3. Edit `inventory.ini`:

```ini
[tak]
<REPLACE_WITH_VM_IP> ansible_user=ubuntu
```

Replace `<REPLACE_WITH_VM_IP>` with the value of `$VM_IP`.
Make sure the `ansible_user` is correct. On rasperry pi, the regular user is `pi`

### 4. Test Ansible SSH connection

```bash
ansible -i inventory.ini all -m ping
```

---

## Download TAK Server Files

Download the following files from the official TAK Server website:

- `takserver_5.4-RELEASEXX_all.deb`
- `takserver-public-gpg.key`
- `deb_policy.pol`

Rename the `.deb` file:

```bash
mv takserver_5.4-RELEASEXX_all.deb takserver_5.4-RELEASE_all.deb
```

Then copy all files to the role:

```bash
cp takserver_5.4-RELEASE_all.deb takserver-public-gpg.key deb_policy.pol roles/takserver/files/
```

---

## Extract the GPG Key ID

To extract the `id="..."` from the `.pol` file:

```bash
sed -n 's/.*id="\([^"]*\)".*/\1/p' roles/takserver/files/deb_policy.pol | head -n1
```

Take the output and set it in:

```yaml
# roles/takserver/defaults/main.yml
takserver_key_id: F06237936851F5B5  # Replace with correct value
```

---

## Run the Playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## Cleanup the VM (dev/optional)

```bash
multipass stop "$TAK_VM_NAME"
multipass delete "$TAK_VM_NAME"
multipass purge
```

---

### TODO

- check if open UFW are correct
- Install certificates
- Package mediamtx for ubuntu (so we dont have to guess which arch we are running)
