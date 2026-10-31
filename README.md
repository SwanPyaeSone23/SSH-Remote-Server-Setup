# SSH Remote Server Setup

Goal of this project is to learn and practice the basics of Linux by setting up a
remote Linux server and configuring it to allow SSH connections using two
separate SSH key pairs.

## Requirements covered

- Provisioned a remote Linux server with a provider (e.g. DigitalOcean droplet, AWS EC2)
- Created **two** new SSH key pairs and added both public keys to the server
- Can connect to the server using **both** SSH keys
- Configured `~/.ssh/config` so the server can be reached with a short alias
- Stretch goal: installed and configured `fail2ban` to prevent brute force attacks

> **Important:** no private keys are (or ever will be) committed to this
> repository — the `.gitignore` in this project blocks them as a safety net.

## Steps

Replace `<SERVER_IP>` and `myuser` with your own server IP and login user.

### 1. Provision the server

Created a simple droplet on DigitalOcean (any provider works — AWS or others
are fine too) running Ubuntu LTS, then logged in for the first time with the
root password:

```bash
ssh root@<SERVER_IP>
```

### 2. Create a standard user (instead of using root)

```bash
adduser myuser
usermod -aG sudo myuser
```

### 3. Create two SSH key pairs (on the local machine)

```bash
ssh-keygen -t ed25519 -C "myuser@laptop-primary" -f ~/.ssh/id_ed25519_primary
ssh-keygen -t ed25519 -C "myuser@laptop-backup"  -f ~/.ssh/id_ed25519_backup
```

Both key pairs are generated locally. The **private keys never leave this
machine** and are excluded from git.

### 4. Copy both public keys to the server

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_primary.pub myuser@<SERVER_IP>
ssh-copy-id -i ~/.ssh/id_ed25519_backup.pub  myuser@<SERVER_IP>
```

Or append them manually on the server:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/authorized_keys <<'EOF'
<contents of id_ed25519_primary.pub>
<contents of id_ed25519_backup.pub>
EOF
chmod 600 ~/.ssh/authorized_keys
```

### 5. Configure password/key access settings on the server

Edit `/etc/ssh/sshd_config` (or files under `/etc/ssh/sshd_config.d/`) and set:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Then restart the service:

```bash
sudo systemctl restart ssh
```

### 6. Connect using both keys

Explicitly specifying the key:

```bash
ssh -i ~/.ssh/id_ed25519_primary myuser@<SERVER_IP>
ssh -i ~/.ssh/id_ed25519_backup  myuser@<SERVER_IP>
```

### 7. Set up `~/.ssh/config` for a short alias

```sshconfig
Host myserver
    HostName <SERVER_IP>
    User myuser
    IdentityFile ~/.ssh/id_ed25519_primary

Host myserver-backup
    HostName <SERVER_IP>
    User myuser
    IdentityFile ~/.ssh/id_ed25519_backup
```

Secure the file and test the alias:

```bash
chmod 600 ~/.ssh/config
ssh myserver
ssh myserver-backup
```

Both aliases now connect to the same server using their respective keys.

### 8. Stretch goal — install and configure fail2ban

```bash
sudo apt update && sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

Default `jail.conf` settings block repeated failed SSH logins. Verify it is
active with `sudo fail2ban-client status sshd`.

## Outcome

I can SSH into my server using both SSH keys, either by passing the key with
`-i` or through the `~/.ssh/config` aliases.
