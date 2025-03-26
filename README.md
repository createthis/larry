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
