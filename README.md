# Getting started
Once Ubuntu `24.04.2` is installed:

# Get IP of larry
In the KVM:
```bash
ip addr
```

In my case it is `192.168.0.146`.

# Add larry to /etc/hosts
```bash
sudo vim /etc/hosts
```

The line to add will look like:
```
192.168.0.146 larry
```

# Setup SSH
Send your public key and private key to larry via SFTP:
```bash
sftp jesse@larry
cd .ssh
put id_rsa
put id_rsa.pub
exit
```

Populate `authorized_keys`:
```bash
ssh jesse@larry
cd .ssh
cat id_rsa.pub >> authorized_keys
```

Configure sshd:
```bash
# uncomment Pub Key Auth and set to yes
# set Password Auth to no
sudo vim /etc/ssh/sshd_config
# set Password Auth to no
sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
```

Restart sshd:
```bash
sudo systemctl restart ssh
```

Attempt login:
```bash
ssh jesse@larry
```

# install ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

# create /data partition
```bash
# 1. Create a new 2TB LV named 'data'
sudo lvcreate -L 2T -n data ubuntu-vg

# 2. Format it as ext4 (using journaling is optional; here we keep defaults)
sudo mkfs.ext4 /dev/ubuntu-vg/data

# 3. Create the mount point
sudo mkdir /data

# 4. Mount with noatime for performance
sudo mount -o noatime /dev/ubuntu-vg/data /data

# 5. Persist in fstab
echo '/dev/ubuntu-vg/data /data ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab

sudo chown -R jesse:jesse /data
```

# Install DeepSeek-V3-0324
```bash
sudo apt update
sudo apt install git-lfs
git lfs install
cd /data
git lfs clone https://huggingface.co/deepseek-ai/DeepSeek-V3-0324 DeepSeek-V3-0324
```

# Run DeepSeek-V3-0324
```bash
cd DeepSeek-V3-0324
ollama import model.yml
ollama run DeepSeek-V3-0324
```
