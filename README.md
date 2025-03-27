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

Add these lines (note this will expose the ollama API to your local network, so make sure you're secure first):

```
[Service]
Environment="OLLAMA_MODELS=/data/ollama/models"
Environment="OLLAMA_KEEP_ALIVE=-1"
Environment="OLLAMA_HOST=0.0.0.0"
Environment="OLLAMA_NUM_PARALLEL=1"
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

# Run Open WebUI
On a different machine (laptop, workstation, whatever), install Docker Desktop, then run this:

```bash
docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=http://larry:11434/ \
-v open-webui:/app/backend/data --name open-webui --restart always \
ghcr.io/open-webui/open-webui:main
```

On the same machine, in a browser, visit `http://localhost:3000/`.

In the chat, set the context window size to `16384`. Larger than that will exceed the 768G of RAM.
The first time I did this I forgot to add `Environment="OLLAMA_NUM_PARALLEL=1"` to my service config (see above)
and I got this error in the ollama service log:

```
 level=INFO source=server.go:105 msg="system memory" total="755.0 GiB" free="739.0 GiB" free_swap="7.9 GiB"
 level=WARN source=ggml.go:149 msg="key not found" key=deepseek2.vision.block_count default=0
 level=WARN source=server.go:133 msg="model request too large for system" requested="996.3 GiB" available=802049822720 total="755.0 GiB" free="739.0 GiB" swap="7.9 GiB"
 level=INFO source=sched.go:429 msg="NewLlamaServer failed" model=/data/ollama/models/blobs/sha256-e0edc8061214ae9dc247ba72e7995806287d0d6ac66a7807b65249f07db8a081 error="model requires more system memory (996.3 GiB) than is available (747.0 GiB)"
```

So apparently 1TB of RAM would buy you either 4x 16k context windows, **OR** a single 65k context window.
