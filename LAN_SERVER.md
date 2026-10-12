# Shared uSOPp server on LAN/VPN

Release examples below use **4.65.13**. Replace the version with the release you actually download. One server runs the app; five regular users plus Admin use browsers through the existing LAN/VPN. The development laptop remains separate.

## 1. Prepare the server

- Windows 10/11 PC acting as the shared server: install [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/), enable its WSL 2 backend and Linux containers, and start Docker Desktop. Allow at least 6 GB RAM for Docker initially; larger PLC projects need more. Docker Desktop is not supported on Windows Server editions; use the Ubuntu host route or an Ubuntu VM for those machines.
- Ubuntu: install [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) and the Compose plugin. Use an account with Docker access.
- Download the private Release's **Container Launchers ZIP**, **Linux amd64 image archive** and their SHA256 files. Extract the launchers into one installation folder. Keep the image archive there.
- Choose an unused port, such as 8201. Find the server's LAN address with `ipconfig` on Windows or `hostname -I` on Ubuntu. Use the address reached by your VPN; reserve it on DHCP if needed.

Only the server needs Docker. User PCs need the browser, VPN and their uSOPp accounts.

## 2. Install on Windows

Open PowerShell in the extracted launcher folder. Verify the image archive against its supplied checksum:

```powershell
Get-FileHash .\uSOPp_AI_4.65.13_Container_linux_amd64.tar.gz -Algorithm SHA256
```

Use your server's LAN IPv4 address in place of `192.168.19.10`:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 install -Image usopp-ai:4.65.13 -ImageArchive .\uSOPp_AI_4.65.13_Container_linux_amd64.tar.gz -Port 8201 -BindAddress 192.168.19.10
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 status
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 logs
```

Choose the initial Admin password when prompted. It seeds only a new account; updates do not reset passwords. Check Docker's reported health and log. Open **http://192.168.19.10:8201** from a VPN/LAN user PC.

If Windows Firewall blocks access, the server administrator can add a rule for the actual trusted LAN/VPN ranges. Replace these example ranges before running in an elevated PowerShell:

```powershell
New-NetFirewallRule -DisplayName 'uSOPp LAN VPN' -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8201 -RemoteAddress 192.168.19.0/24,10.8.0.0/24 -Profile Domain,Private
```

## 3. Install on Ubuntu

```bash
sha256sum -c uSOPp_AI_4.65.13_Container_linux_amd64.tar.gz.sha256
USOPP_PORT=8201 USOPP_BIND_ADDRESS=192.168.19.10 bash usopp-container.sh install usopp-ai:4.65.13 ./uSOPp_AI_4.65.13_Container_linux_amd64.tar.gz
bash usopp-container.sh status
bash usopp-container.sh logs
```

Open **http://192.168.19.10:8201**. The installer saves the chosen image, port, bind address and volume for later start/update commands. Restrict access through your server/network firewall to existing LAN/VPN users. Docker-published ports need rules appropriate to Docker's firewall integration; a UFW rule alone may not restrict them. See [Docker packet filtering](https://docs.docker.com/engine/network/packet-filtering-firewalls/).

## 4. Create users and install the CPU model

Visitors can click **Explore demo** on the landing page to fill `demo` / `demo`
and then click **Sign in**. Each demo login has its own workspace and a fixed
30-minute session. Demo access does not consume one of the five regular-user
slots. Shared rule/mapping changes and administration use personal accounts.

Sign in as Admin. Create up to five regular accounts. Each account keeps its own saved settings, imported metadata and test history. Admin can review everyone's tests.

In **Settings → Chopper · AI connection**, click **Set up small CPU model**. The one-time 1.12 GB download runs in the background, verifies SHA256, selects the included llama.cpp runtime and enables startup on restart. Wait for **Ready**. No Ollama installation is required. Existing browser-only saved settings migrate when their owner signs in to the updated app from that browser.

For PLC projects use **Browse / upload folder**. Windows project paths are not Linux container paths. Live DataInspect requires a network route from Docker to the PLC IP/port over the factory network/VPN.

## 5. Stop, backup and start

Finish active tests and wait for settings to save. On Windows:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 stop
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 backup -BackupDirectory D:\uSOPp-backups
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 start
```

On Ubuntu:

```bash
bash usopp-container.sh stop
USOPP_BACKUP_DIRECTORY="$HOME/usopp-backups" bash usopp-container.sh backup
bash usopp-container.sh start
```

Backup refuses a volume used by a running container. It includes the entire persistent store: accounts, full results, reports, source snapshots, rules, knowledge, configurations and installed model. It creates a `.tar.gz` and `.sha256`; keep both. Default backups go under `%LOCALAPPDATA%\uSOPp-container\backups` or `~/.config/usopp-container/backups`. Copy them to separate storage.

## 6. Update without losing data

Download the new launcher ZIP and image. Use the new launcher folder, finish active tests, then replace `<new-version>` and `<new-image-path>`:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 update -Image usopp-ai:<new-version> -ImageArchive <new-image-path> -BackupDirectory D:\uSOPp-backups
```

```bash
USOPP_BACKUP_DIRECTORY="$HOME/usopp-backups" bash usopp-container.sh update usopp-ai:<new-version> <new-image-path>
```

The command loads the image, stops the app, makes a complete backup and starts the selected release on the existing volume. Existing users, passwords, test IDs/results, rule configuration, edited parameter mappings and model weights remain. Port and bind address are preserved. A backup failure leaves the app stopped with its data retained. Read the log/status after an update; startup is not proof of completed migrations or a working live PLC route.

## 7. Restore or move to another server

Install the matching release on the other server first; do not add production data there yet. Copy the backup **and checksum** to it, stop the app, then restore to a new empty named volume:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 restore -BackupFile D:\uSOPp-backups\usopp-server-EXAMPLE.tar.gz -DataVolume usopp-restored-20261010
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 start
```

```bash
bash usopp-container.sh restore "$HOME/usopp-backups/usopp-server-EXAMPLE.tar.gz" usopp-restored-20261010
bash usopp-container.sh start
```

The command validates the checksum and paths, refuses a nonempty destination and switches installer settings only after extraction succeeds. The original volume remains separately available. Restored users keep their original passwords; the new host's initial-password prompt does not reset them. Restore a pre-update backup with the matching older release for rollback; do not attach a migrated newer database directly to an older release.

This restore flow is for container backups. Native Windows and source/demo installations use their own data roots. Do not delete volumes during upgrades. GitHub publishing does not back up the private server store.

## Implementation status

Install/update/backup/restore and the CPU runtime build are prepared in the source and release tooling. They have not been run on a Docker destination: Docker is unavailable on the development laptop. Complete image/build, restore and concurrent-user verification on the destination before using this as production storage.

## 8. Replace the old Ubuntu source demo with a fresh container

Use this procedure for the original `~/usopp-demo/usopp_ai_2.0` installation
managed by `npm start` / `scripts/ubuntu_demo.py`. The requested fresh install
deletes that application's source, virtual environments, accounts, results,
settings, mappings, knowledge and rules. It creates new container accounts/data.

### Download the new release manually

Open [Tamil-pts/usopp-distribution Releases](https://github.com/Tamil-pts/usopp-distribution/releases),
select the published version and download these four assets:

```text
uSOPp_AI_<version>_Container_Launchers.zip
uSOPp_AI_<version>_Container_Launchers.zip.sha256
uSOPp_AI_<version>_Container_linux_amd64.tar.gz
uSOPp_AI_<version>_Container_linux_amd64.tar.gz.sha256
```

The image archive and launcher ZIP must belong to the same release. The
automatic **Source code** ZIP is not the container installer. Obtain all four
files before removing the old app. If the image asset is absent, the maintainer
must build the latest source using the package workflow and publish the release
first; the launcher ZIP alone cannot install the application.

For a direct manual installation before publishing a release, either build
from the local source bundle on Ubuntu (section 9), or use the source
repository's **Actions -> Build private download packages** workflow. The
workflow can build the image even when Docker is absent on the development
laptop.

To transfer the files from Windows Downloads to the original Ubuntu server,
open PowerShell and replace the version below with the downloaded version:

```powershell
$usoppRelease = '<published-version>'
$usoppDownloadRoot = Join-Path $env:USERPROFILE 'Downloads'
ssh lmw@192.168.19.10 'mkdir -p "$HOME/usopp-container"'
$usoppReleaseFiles = @(
    "uSOPp_AI_${usoppRelease}_Container_Launchers.zip",
    "uSOPp_AI_${usoppRelease}_Container_Launchers.zip.sha256",
    "uSOPp_AI_${usoppRelease}_Container_linux_amd64.tar.gz",
    "uSOPp_AI_${usoppRelease}_Container_linux_amd64.tar.gz.sha256"
)
foreach ($usoppReleaseFile in $usoppReleaseFiles) {
    scp (Join-Path $usoppDownloadRoot $usoppReleaseFile) 'lmw@192.168.19.10:~/usopp-container/'
    if ($LASTEXITCODE -ne 0) { throw "Transfer failed: $usoppReleaseFile" }
}
```

Replace `lmw` and `192.168.19.10` if installing on a different server. If you
download directly in the server's browser, save the same files under
`~/usopp-container` instead.

### Check the downloaded files and Docker

On Ubuntu, run these commands, substituting the published version:

```bash
cd "$HOME/usopp-container"
usopp_release='<published-version>'
sha256sum -c "uSOPp_AI_${usopp_release}_Container_Launchers.zip.sha256"
sha256sum -c "uSOPp_AI_${usopp_release}_Container_linux_amd64.tar.gz.sha256"
docker info --format '{{.OSType}}'
docker compose version
```

Both checksum checks must report **OK**, and the Docker OS must report
**linux**. Install Docker Engine/Compose using section 1 if they are absent;
use an account with Docker access. Run the following stages only after these
checks succeed.

### Stop and delete the old source demo

Run the stop/status commands first:

```bash
cd "$HOME/usopp-demo/usopp_ai_2.0"
./.venv-ubuntu/bin/python scripts/ubuntu_demo.py stop
./.venv-ubuntu/bin/python scripts/ubuntu_demo.py status
```

Once status reports **stopped**, delete the complete old application directory:

```bash
cd "$HOME"
rm -rf -- "$HOME/usopp-demo/usopp_ai_2.0"
```

This removal targets the old source demo. Python/Node installed on the host are
not required by the container. Leave unrelated host services installed; the
original server has another Docker application using port 8200.

### Install the new container

These steps assume this server has no previous uSOPp container installation or
uSOPp data volume. An existing container volume is reused by the installer and
would need its own explicit removal to perform a complete reset.

```bash
cd "$HOME/usopp-container"
unzip "uSOPp_AI_${usopp_release}_Container_Launchers.zip"
USOPP_PORT=8201 USOPP_BIND_ADDRESS=192.168.19.10 \
  bash usopp-container.sh install "usopp-ai:$usopp_release" \
  "./uSOPp_AI_${usopp_release}_Container_linux_amd64.tar.gz"
bash usopp-container.sh status
bash usopp-container.sh logs
```

Use the new server's own LAN/VPN IPv4 address for `USOPP_BIND_ADDRESS`.
Choose a new initial Admin password when prompted. Open
`http://192.168.19.10:8201`, sign in as Admin and create the user accounts.
Configure rules, mappings, knowledge and the CPU model in the new installation.
The container carries its runtime; no `npm install` or application Python
environment is needed on the host.

For subsequent upgrades, use section 6's **update** command to keep the data
created in this fresh installation.

## 9. Build the container from local source files on Ubuntu

Use this when the release image has not been uploaded yet. It transfers a
private, reduced build bundle from the development PC, builds the Linux image
on Ubuntu and installs it locally. It excludes uploaded PLC projects, account
databases, local model weights, training data, B&R help source files and
dependency folders. Ubuntu needs Internet access during the Docker build to
download its pinned base images, packages and llama.cpp source.

On the development Windows PC, from the `usopp_ai_2.0` source folder, create
the bundle and checksum:

```powershell
$usoppVersion = (Get-Content .\package.json -Raw | ConvertFrom-Json).version
$usoppBundle = ".\release\private-build\uSOPp_AI_${usoppVersion}_Container_Build_Source.tar.gz"
python .\scripts\package_container_build.py --output $usoppBundle
python .\scripts\package_downloads.py --output ".\release\downloads\$usoppVersion"
```

Transfer those two files and the matching launcher ZIP/checksum to Ubuntu. For
version `4.65.13`, the filenames are:

```text
uSOPp_AI_4.65.13_Container_Build_Source.tar.gz
uSOPp_AI_4.65.13_Container_Build_Source.tar.gz.sha256
uSOPp_AI_4.65.13_Container_Launchers.zip
uSOPp_AI_4.65.13_Container_Launchers.zip.sha256
```

Run `scp` from Windows PowerShell, replacing `lmw` and the server IP if needed:

```powershell
$usoppVersion = '4.65.13'
$usoppSource = ".\release\private-build\uSOPp_AI_${usoppVersion}_Container_Build_Source.tar.gz"
$usoppLauncher = ".\release\downloads\$usoppVersion\uSOPp_AI_${usoppVersion}_Container_Launchers.zip"
scp $usoppSource "lmw@192.168.19.10:~/usopp-container/"
scp "$usoppSource.sha256" "lmw@192.168.19.10:~/usopp-container/"
scp $usoppLauncher "lmw@192.168.19.10:~/usopp-container/"
scp "$usoppLauncher.sha256" "lmw@192.168.19.10:~/usopp-container/"
```

On Ubuntu, verify both checksums, extract the source to its own build folder,
and build the image. The build takes several minutes and uses the Ubuntu
server's CPU and network:

```bash
cd "$HOME/usopp-container"
usopp_version='4.65.13'
sha256sum -c "uSOPp_AI_${usopp_version}_Container_Build_Source.tar.gz.sha256"
sha256sum -c "uSOPp_AI_${usopp_version}_Container_Launchers.zip.sha256"
mkdir -p "build-${usopp_version}"
tar -xzf "uSOPp_AI_${usopp_version}_Container_Build_Source.tar.gz" -C "build-${usopp_version}"
docker build --platform linux/amd64 --build-arg APP_VERSION="$usopp_version" \
  -t "usopp-ai:$usopp_version" "build-${usopp_version}"
```

Then install the image already built in Docker. Keep the existing app on port
8200 untouched; this example uses port 8201:

```bash
unzip -o "uSOPp_AI_${usopp_version}_Container_Launchers.zip"
USOPP_PORT=8201 USOPP_BIND_ADDRESS=192.168.19.10 \
  bash usopp-container.sh install "usopp-ai:$usopp_version"
bash usopp-container.sh status
bash usopp-container.sh logs
```

The Admin password prompt appears during the first container install. Wait for
`Health: healthy` before opening `http://192.168.19.10:8201`. Once the
container is running, the temporary build folder and transferred bundle may
be removed from `~/usopp-container`; keep the launcher ZIP for future
management commands. Do not remove Docker images or volumes belonging to other
applications.
