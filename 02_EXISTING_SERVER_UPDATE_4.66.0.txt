# 2. Update the existing Ubuntu server — uSOPp.ai 4.66.0

This guide is for the existing **SpinConnect Ubuntu Docker installation**, previously shown as `usopp-container-app-1`, with data volume `usopp-container_data` and installer settings under `/home/lmw/.config/usopp-container`.

Run Windows transfer commands on the laptop. Run Bash commands after SSH login to Ubuntu. Finish users' active tests, exports and collection work before updating. Run the launcher as the **same Linux user that installed it** (`lmw` in your example), without prefixing it with `sudo`; otherwise it will look in another home directory.

The update stops the application, backs up the complete existing volume, then starts 4.66.0 on that same volume. It preserves accounts, passwords, saved results, knowledge, settings, model files and the current listening port/address. The image update does not reset existing passwords.

## 1. Collect the four release files on Windows

Copy these files into `%USERPROFILE%\Downloads\uSOPp-4.66.0`:

```text
uSOPp_AI_4.66.0_Container_Launchers.zip
uSOPp_AI_4.66.0_Container_Launchers.zip.sha256
uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz
uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256
```

Keep the image `.tar.gz` compressed. The launcher ZIP alone cannot install the image.

## 2. Copy the files to Ubuntu

In **Windows PowerShell**:

```powershell
$ErrorActionPreference = 'Stop'
Set-Location -LiteralPath (Join-Path $env:USERPROFILE 'Downloads\uSOPp-4.66.0')
$ServerAddress = Read-Host 'Enter the existing Ubuntu server IP or hostname'
$Remote = "lmw@$ServerAddress"
ssh $Remote 'mkdir -p ~/usopp-updates/4.66.0'
if ($LASTEXITCODE -ne 0) { throw 'SSH connection failed.' }
scp .\uSOPp_AI_4.66.0_Container_Launchers.zip .\uSOPp_AI_4.66.0_Container_Launchers.zip.sha256 .\uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz .\uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256 "${Remote}:~/usopp-updates/4.66.0/"
if ($LASTEXITCODE -ne 0) { throw 'File transfer failed.' }
ssh $Remote
```

Check the SSH host fingerprint when connecting. Your prompt should now show the Ubuntu server, such as `lmw@SpinConnect`.

If you already downloaded these four files directly onto Ubuntu, skip the transfer and place them in `$HOME/usopp-updates/4.66.0`.

## 3. Verify the current installation and saved volume

In the **Ubuntu SSH terminal**:

```bash
whoami
docker version
docker compose version
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
test -f "$HOME/.config/usopp-container/image"
test -f "$HOME/.config/usopp-container/port"
test -f "$HOME/.config/usopp-container/admin-password.txt"
cat "$HOME/.config/usopp-container/image"
cat "$HOME/.config/usopp-container/port"
docker inspect usopp-container-app-1 --format '{{range .Mounts}}{{if eq .Destination "/data"}}{{.Name}}{{end}}{{end}}'
docker volume inspect usopp-container_data --format '{{.Name}}'
```

Do not print the password file. Both volume commands should identify `usopp-container_data` for your installation. If the container name, volume or saved configuration differs, stop and use those actual installation details; do not initialize a replacement store.

If a `test -f` command fails, locate the original install user's configuration before updating. If Docker access is denied, restore that user's Docker access instead of running the launcher under a different home directory.

## 4. Verify the release and extract its launchers

```bash
cd "$HOME/usopp-updates/4.66.0"
sha256sum --check uSOPp_AI_4.66.0_Container_Launchers.zip.sha256
sha256sum --check uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256
```

Both must say `OK`. Stop if either fails.

If `unzip` is missing, install it once:

```bash
sudo apt-get update
sudo apt-get install -y unzip
```

Then extract into a new directory:

```bash
unzip -n uSOPp_AI_4.66.0_Container_Launchers.zip -d launchers
test -f launchers/usopp-container.sh
test -f launchers/compose.yaml
bash -n launchers/usopp-container.sh
```

Use the **4.66.0 launcher**, not an old copied script.

## 5. Run the update with its automatic full backup

Allow enough free disk space for the loaded image and a full backup, including any installed GGUF models:

```bash
df -h "$HOME"
docker system df
```

After everyone finishes active work, run:

```bash
USOPP_BACKUP_DIRECTORY="$HOME/usopp-backups" \
  bash ./launchers/usopp-container.sh update usopp-ai:4.66.0 \
  "$HOME/usopp-updates/4.66.0/uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz"
```

The command loads the downloaded image, stops the old container, creates a complete backup and checksum, then replaces the container. Keep the printed backup path. This step can take several minutes.

If the updater reports a backup failure, fix that error before retrying. It intentionally leaves the application stopped with its existing volume retained. Do not continue to a fresh install.

## 6. Verify the image, data mount and API

```bash
bash ./launchers/usopp-container.sh status
bash ./launchers/usopp-container.sh logs
docker inspect usopp-container-app-1 --format '{{.Config.Image}}'
docker inspect usopp-container-app-1 --format '{{range .Mounts}}{{println .Destination "<-" .Source}}{{end}}'
docker inspect usopp-container-app-1 --format '{{.State.Health.Status}}'
ls -lh "$HOME/usopp-backups"
```

Expected: image `usopp-ai:4.66.0`; `/data` still uses the original named volume. Health may initially say `starting`; rerun the health command after startup.

Read the saved bind address and port to check the API:

```bash
usopp_port=$(cat "$HOME/.config/usopp-container/port")
usopp_bind=$(cat "$HOME/.config/usopp-container/bind-address")
if [ "$usopp_bind" = '0.0.0.0' ]; then
  usopp_bind='127.0.0.1'
fi
curl --fail --show-error "http://${usopp_bind}:${usopp_port}/api/health"
```

Expected API JSON contains `"version":"4.66.0"` (whitespace may differ). Use the same LAN/VPN browser address as before, sign in with the existing account, then press `Ctrl+F5`. Confirm **UI 4.66.0 | API 4.66.0** and review saved dashboards/results.

## 7. Check Chopper after updating

In **Settings → Chopper setup → Local CPU model**:

1. Existing saved model choices and downloaded weights are retained.
2. To upgrade to 3B, click **Install 3B** if needed, then **Use 3B primary**. A network connection is needed for that one-time model download; model files are not inside the image archive.
3. Keep the installed **1.5B fallback** and **Start with the server** enabled if required.
4. Wait for **Ready**, then check **AI Assistant → Chat**. Project Intelligence remains collapsed until clicked.

## 8. Routine commands after the update

From `$HOME/usopp-updates/4.66.0`:

```bash
bash ./launchers/usopp-container.sh status
bash ./launchers/usopp-container.sh logs
```

To stop after active work finishes:

```bash
bash ./launchers/usopp-container.sh stop
```

To start:

```bash
bash ./launchers/usopp-container.sh start
```

Keep the original data volume, prior image and the pre-update backup. Updating the image does not require cloning Git, copying application source or installing Python/Node on Ubuntu.
