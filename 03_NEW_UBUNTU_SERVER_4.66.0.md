# 3. Install on a new Ubuntu server — uSOPp.ai 4.66.0

Use a fresh **64-bit x86 Ubuntu server**, for example Ubuntu 24.04 LTS or 22.04 LTS, with a normal login user and `sudo` access. The supplied image is Linux **amd64**, not ARM. Server users connect through a browser; only the server needs Docker.

Commands marked PowerShell run on the Windows laptop. Commands marked Bash run in the Ubuntu SSH terminal. Run one numbered step at a time and stop on errors.

## 1. Connect to Ubuntu and check its architecture

In **Windows PowerShell**:

```powershell
$ServerAddress = Read-Host 'Enter the new Ubuntu server IP or hostname'
$ServerUser = Read-Host 'Enter the Ubuntu login username'
$Remote = "${ServerUser}@${ServerAddress}"
ssh $Remote
```

In the **Ubuntu terminal**:

```bash
whoami
uname -m
cat /etc/os-release
```

Expected architecture: `x86_64` (Ubuntu package architecture `amd64`). Finish installing the OS and networking before continuing if this is not an Ubuntu host.

## 2. Install Docker Engine and Compose

If `docker version` and `docker compose version` already work, skip to step 3. For a fresh Ubuntu installation, the following uses [Docker's official Ubuntu apt repository](https://docs.docker.com/engine/install/ubuntu/):

If you skip Docker installation, make sure `curl` and `unzip` are installed for steps 4 and 6.

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl unzip
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

Allow your login account to use Docker, then reconnect. Docker-group membership grants administrative Docker access; this follows [Docker's Linux post-installation instructions](https://docs.docker.com/engine/install/linux-postinstall/).

```bash
sudo usermod -aG docker "$USER"
exit
```

Back in the **same Windows PowerShell window**, reconnect:

```powershell
ssh $Remote
```

In **Ubuntu**, verify:

```bash
docker version
docker compose version
docker info --format '{{.OSType}}/{{.Architecture}}'
```

Expected OS: `linux`. If access is denied, reconnect after confirming the group membership; do not run the uSOPp launcher with a different user's home directory.

## 3. Copy the four downloaded assets to the server

On Windows, put these files in `%USERPROFILE%\Downloads\uSOPp-4.66.0`:

```text
uSOPp_AI_4.66.0_Container_Launchers.zip
uSOPp_AI_4.66.0_Container_Launchers.zip.sha256
uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz
uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256
```

Open another **Windows PowerShell window** for the transfer:

```powershell
$ErrorActionPreference = 'Stop'
$ServerAddress = Read-Host 'Enter the new Ubuntu server IP or hostname'
$ServerUser = Read-Host 'Enter the Ubuntu login username'
$Remote = "${ServerUser}@${ServerAddress}"
Set-Location -LiteralPath (Join-Path $env:USERPROFILE 'Downloads\uSOPp-4.66.0')
ssh $Remote 'mkdir -p ~/usopp-releases/4.66.0'
if ($LASTEXITCODE -ne 0) { throw 'SSH connection failed.' }
scp .\uSOPp_AI_4.66.0_Container_Launchers.zip .\uSOPp_AI_4.66.0_Container_Launchers.zip.sha256 .\uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz .\uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256 "${Remote}:~/usopp-releases/4.66.0/"
if ($LASTEXITCODE -ne 0) { throw 'File transfer failed.' }
```

The `.tar.gz` is already a built image. Leave it compressed; no Git clone or Docker build is required on this server. If downloaded directly on Ubuntu, put the same four files in `$HOME/usopp-releases/4.66.0` and skip the transfer.

## 4. Verify files and extract the launchers

In the **Ubuntu terminal**:

```bash
cd "$HOME/usopp-releases/4.66.0"
sha256sum --check uSOPp_AI_4.66.0_Container_Launchers.zip.sha256
sha256sum --check uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256
```

Both must report `OK`. Then:

```bash
unzip -n uSOPp_AI_4.66.0_Container_Launchers.zip -d launchers
test -f launchers/usopp-container.sh
test -f launchers/compose.yaml
bash -n launchers/usopp-container.sh
hostname -I
```

Choose the server's actual LAN/VPN IPv4 address from its network configuration. The browser clients must be able to reach that address.

## 5. Install the image and choose the Admin password

Continue in **Ubuntu**. Enter the server's LAN IPv4 address when asked, for example your office network address:

```bash
read -r -p 'Server LAN IPv4 address: ' usopp_lan_ip
USOPP_PORT=8200 USOPP_BIND_ADDRESS="$usopp_lan_ip" \
  bash ./launchers/usopp-container.sh install usopp-ai:4.66.0 \
  "$HOME/usopp-releases/4.66.0/uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz"
```

Choose the **initial Admin password** when prompted: 12 to 128 characters. The password is not displayed. Run the launcher as your normal installation user; it saves its settings under that user's `~/.config/usopp-container`.

This creates a persistent volume named `usopp-container_data`. Keep it for future updates. The initial installation loads the image and starts the app automatically. For an occupied port, use `USOPP_PORT=8201` in this command and use 8201 in the following browser/health commands.

## 6. Verify startup

```bash
bash ./launchers/usopp-container.sh status
bash ./launchers/usopp-container.sh logs
docker inspect usopp-container-app-1 --format '{{.Config.Image}}'
docker inspect usopp-container-app-1 --format '{{.State.Health.Status}}'
curl --fail --show-error "http://${usopp_lan_ip}:8200/api/health"
```

Expected image: `usopp-ai:4.66.0`; API version: `4.66.0`. Health may initially say `starting`; repeat the check after startup. If the health request cannot connect, inspect the logs and confirm the chosen IP/port is available on the host.

On a LAN/VPN user PC, open:

```text
http://YOUR_SERVER_LAN_IP:8200
```

Replace `YOUR_SERVER_LAN_IP` with the address entered above. Sign in as `Admin` with the chosen password. Confirm **UI 4.66.0 | API 4.66.0**.

Your network administrator should allow this application port only for the intended LAN/VPN clients. Docker-published ports interact with Docker's firewall rules; follow the [firewall notes in Docker's Ubuntu installation guide](https://docs.docker.com/engine/install/ubuntu/#firewall-limitations) when setting access rules.

## 7. Set up users, Chopper and project access

1. Admin approves the required user accounts in **WorkHub → User management**.
2. Open **Settings → Chopper setup → Local CPU model**. Click **Set up local Chopper** or **Install 3B**, then select **Use 3B primary** and wait for **Ready**.
3. Model weights require a separate one-time download. They are stored in the persistent data volume. Install/retain **1.5B fallback** if needed and keep **Start with the server** enabled. Ollama is unnecessary for the bundled runtime.
4. Upload projects through the browser's folder/ZIP picker. A Windows path such as `C:\Projects` is not a path inside this Linux container.
5. Open **AI Assistant → Chat**, select a project and expand **Project Intelligence** when needed. For guided app use, choose **Guide me step by step**.
6. Enable live collection explicitly for the required DataInspect targets. The server/container needs a network route to the actual PLC IP and TCP port.

## 8. Stop, back up and restart

After active work finishes, run from `$HOME/usopp-releases/4.66.0`:

```bash
bash ./launchers/usopp-container.sh stop
USOPP_BACKUP_DIRECTORY="$HOME/usopp-backups" bash ./launchers/usopp-container.sh backup
bash ./launchers/usopp-container.sh start
```

Keep both the printed backup archive and its `.sha256` file, and copy them to separate backup storage. The backup includes account data, workspaces, reports, knowledge, settings and installed models. Future image updates should follow **02_EXISTING_SERVER_UPDATE_4.66.0.md**, using the new release's filenames/version.
