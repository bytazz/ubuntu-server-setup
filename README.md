# ⚙️ Homelab Installation Guide — Ubuntu 22.04.5 LTS

> Practical guide to set up a homelab on **Ubuntu Server 22.04.5 LTS**.

---

## Index

1. 🚫 Disable cloud-init
2. 🌐 Configure network (static IP)
3. 🐳 Install Docker
4. 🖥️ Install Portainer (UI for Docker)
5. ⏰ Set time zone
6. 🔐 Harden and secure SSH
7. 🛡️ Prepare DNS — AdGuard Home (and systemd-resolved)
8. ☁️ Nextcloud — quick domain tweaks
9. 📚 Resources and references

---

## 1. 🚫 Disable cloud-init

> ℹ️ **cloud-init** may re-run configurations at boot.  
> If you don't use it, it's a good idea to disable it:

```bash
sudo touch /etc/cloud/cloud-init.disabled
```

**Optional:** To remove it completely:

```bash
sudo apt purge cloud-init -y
sudo rm -rf /etc/cloud/cloud.cfg.d /etc/cloud/clean.d /var/lib/cloud
sudo apt autoremove -y
```

---

## 2. 🌐 Configure network (static IP)

1. Edit your Netplan file (adjust according to your setup):

   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```

2. Example:

   ```yaml
   network:
     ethernets:
       enol: # Change "enol" to your interface name
         dhcp4: no
         addresses: [192.168.10.87/24]
         gateway4: 192.168.10.1
         nameservers:
           addresses: [1.1.1.1, 8.8.8.8]
     version: 2
   ```

3. Apply the changes:

   ```bash
   sudo netplan apply
   ```

> ⚠️ **Note:** Replace values (`addresses`, `gateway4`, `nameservers`) with those for your network.

---

## 3. 🐳 Install Docker

**1. Prepare APT repository and keys:**

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"${UBUNTU_CODENAME:-$VERSION_CODENAME}\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

**2. Install Docker packages:**

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**3. Check Docker status:**

```bash
sudo systemctl status docker
```

**4. Test with the hello-world image:**

```bash
sudo docker run hello-world
```

---

## 4. 🖥️ Install Portainer (UI for Docker)

1. Add your user to the docker group (you will need to log out or restart your session):

   ```bash
   sudo usermod -aG docker $USER
   ```

2. Create a persistent volume for Portainer data:

   ```bash
   docker volume create portainer_data
   ```

3. Run Portainer:

   ```bash
   docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:lts
   ```

4. Access Portainer from your browser:  
   `https://<SERVER_IP>:9443`

   > ⚠️ You will likely get a warning about the self-signed certificate. You can safely continue.
   > 
---

## 5. ⏰ Set time zone

1. Check the current time:

   ```bash
   timedatectl
   ```

2. Change time zone (example: Madrid):

   ```bash
   sudo timedatectl set-timezone Europe/Madrid
   timedatectl
   ```

---

## 6. 🔐 Harden and secure SSH

1. Edit main SSH config:

   ```bash
   sudo nano /etc/ssh/sshd_config
   ```

2. And cloud-init SSH config:

   ```bash
   sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
   ```

> 🛡️ **Recommendation:**  
> Disable password authentication and use public keys:

```text
PasswordAuthentication no
```

3. Restart the SSH service:

   ```bash
   sudo systemctl restart sshd
   ```

4. Generate a local key (if you don't have one yet):

   ```bash
   ssh-keygen -t ed25519
   ```

5. If there are old entries in known_hosts:

   ```bash
   ssh-keygen -R <SERVER_IP>
   ```

---

## 7. 🛡️ Prepare DNS — AdGuard Home (and systemd-resolved)

> If you plan to use AdGuard Home as a local resolver, it's common to disable `systemd-resolved` and manage `/etc/resolv.conf` manually:

```bash
sudo systemctl disable --now systemd-resolved
sudo systemctl mask systemd-resolved
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf
```

---

## 8. ☁️ Nextcloud — quick domain tweaks

1. If your Nextcloud installation is in Docker, edit `config.php` to add `trusted_domains`:

   ```bash
   sudo nano /docker/nextcloud/data/html/config/config.php
   ```

2. Add your domains/IPs to `trusted_domains` to avoid access errors.

---

## 📚 Resources and references

- [Docker docs](https://docs.docker.com/engine/install/ubuntu/)
- [Portainer](https://docs.portainer.io/start/install-ce/server/docker/linux)
