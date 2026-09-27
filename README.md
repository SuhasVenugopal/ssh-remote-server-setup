# SSH Remote Server Setup

> **Project:** [roadmap.sh/projects/ssh-remote-server-setup](https://roadmap.sh/projects/ssh-remote-server-setup)

A basic remote Linux server, configured to allow secure SSH access using two independent SSH key pairs — plus a stretch-goal setup of `fail2ban` to protect against brute-force login attempts.

---

## 📋 Project Goal

Set up a remote Linux server and configure it so it can be accessed via SSH using **two separate SSH key pairs**, either directly or through a shortcut alias.

---

## 🖥️ 1. Server Setup

- **Provider:** AWS EC2
- **OS:** Ubuntu 26.04 LTS
- **Access:** Public IP address, key-based SSH only (no password login)

> Replace `<server-ip>` with your actual server's public IP throughout this guide.

---

## 🔑 2. Generate Two SSH Key Pairs

Run these on your **local machine** (not the server):

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/devops-proj-id
ssh-keygen -t rsa -b 4096 -f ~/.ssh/server_key
```

This creates two independent key pairs:

| File | Purpose |
|---|---|
| `~/.ssh/devops-proj-id` / `.pub` | Key pair #1 |
| `~/.ssh/server_key` / `.pub` | Key pair #2 |

Lock down the private keys so SSH will accept them:

```bash
chmod 400 ~/.ssh/devops-proj-id
chmod 400 ~/.ssh/server_key
```

> ⚠️ **Note:** If you're on WSL, generate/store your keys inside the Linux filesystem (e.g. `~/.ssh/`), **not** under `/mnt/c/...`. Windows-mounted drives (NTFS) don't support Linux permission bits, so `chmod` won't actually apply there.

---

## 📤 3. Add Both Public Keys to the Server

Copy both public keys' contents into the server's `~/.ssh/authorized_keys` file in one go — `cat` can take multiple files at once, printing them one after another:

```bash
cat ~/.ssh/devops-proj-id.pub ~/.ssh/server_key.pub | ssh -i ~/.ssh/devops-proj-id user@<server-ip> "cat >> ~/.ssh/authorized_keys"
```

---

## ✅ 4. Verify Both Keys Work Independently

```bash
ssh -i ~/.ssh/devops-proj-id user@<server-ip>
ssh -i ~/.ssh/server_key    user@<server-ip>
```

Both commands should log you in without a password — confirming each key works on its own.

---

## ⚙️ 5. Set Up an SSH Config Alias

On your **local machine**, create/edit `~/.ssh/config`:

```
Host devops-server
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/devops-proj-id
```

Set correct permissions:

```bash
chmod 600 ~/.ssh/config
```

Now you can connect with a short alias instead of the full command:

```bash
ssh devops-server
```

---

## 🛡️ 6. Stretch Goal: Install `fail2ban`

Installed **on the remote server** (fail2ban protects the machine receiving connections, not the one initiating them):

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo systemctl status fail2ban
```

### Live Verification

Checked the SSH jail's status:

```bash
sudo fail2ban-client status sshd
```

**Result:** Within minutes of the server going live, fail2ban had already logged several failed login attempts from automated bots scanning the public IP — confirming it was actively monitoring.

To confirm the ban mechanism itself, 5 deliberate failed logins were made from a test machine:

```bash
ssh wronguser@<server-ip>   # repeated 5 times
```

Fail2ban correctly banned the offending IP:

```
Currently banned: 1
Banned IP list:   <banned-ip>
```

A follow-up connection attempt from the banned IP was refused at the connection level:

```
ssh: connect to host <server-ip> port 22: Connection refused
```

The IP was then manually unbanned to confirm recovery:

```bash
sudo fail2ban-client set sshd unbanip <banned-ip>
```

Access was successfully restored, completing the full ban → block → unban → reconnect cycle.

---

## 📌 Summary

| Requirement | Status |
|---|---|
| Remote Linux server provisioned | ✅ |
| Two SSH key pairs created | ✅ |
| Both keys verified working independently | ✅ |
| SSH config alias set up and tested | ✅ |
| Stretch goal: fail2ban installed & verified live | ✅ |

**Outcome:** The server can be securely accessed via SSH using either key, or via the `devops-server` alias — and is actively protected against brute-force login attempts.
