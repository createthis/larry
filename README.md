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
# 1. Create a new LV named 'data' using 100% of the available space
sudo lvcreate -l 100%FREE -n data ubuntu-vg

# 2. Format it as ext4 (using journaling is optional; here we keep defaults)
sudo mkfs.ext4 /dev/ubuntu-vg/data

# 3. Create the mount point
sudo mkdir /data

# 4. Mount with noatime for performance
sudo mount -o noatime /dev/ubuntu-vg/data /data

# 5. Persist in fstab
echo '/dev/ubuntu-vg/data /data ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab

sudo chown -R jesse:jesse /data
cd /data
```


# move ollama data dir
```bash
mkdir -p /data/ollama/models
sudo chown ollama:ollama /data/ollama/models
sudo systemctl edit ollama.service
```

Add these lines:

```
[Service]
Environment="OLLAMA_MODELS=/data/ollama/models"
Environment="OLLAMA_KEEP_ALIVE=-1"
```

`CTRL+O`
`CTRL+X`

Restart ollama daemon:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

Check the logs and status to make sure it worked:
```bash
systemctl status ollama.service
sudo journalctl -u ollama.service
```

# Run Deepseek V3
This is a super long download, even on fiber, so use `screen`! That way,
if your SSH session times out, your download continues in the background.
```bash
screen
ollama run --verbose deepseek-v3:671b-q8_0
```

If you need to resume the ssh session later:
```bash
ssh jesse@larry
screen -x
```

# Run DeepSeek-V3-0324
```bash
screen
ollama run --verbose lordoliver/DeepSeek-V3-0324:671b-q8_0
```

This version talks about youtube scripts unprompted, so I think the `Modelfile` must include a custom prompt.
Let's remove it (only works if you also have `deepseek-v3:671b-q8_0` downloaded):

```bash
ollama stop lordoliver/DeepSeek-V3-0324:671b-q8_0
mkdir /data/DeepSeek-V3-0324
cd /data/DeepSeek-V3-0324
ollama show deepseek-v3:671b-q8_0 --modelfile > Modelfile.deepseek
ollama show lordoliver/DeepSeek-V3-0324:671b-q8_0 --modelfile > Modelfile.old
cp Modelfile.deepseek Modelfile
vim Modelfile.old
# copy the FROM line then open Modelfile in a new vim tab
tabe Modelfile
# paste the FROM line from Modelfile.old into Modelfile
ollama create DeepSeek-V3-0324:671b-q8_0 -f ./Modelfile
ollama run DeepSeek-V3-0324:671b-q8_0
```
