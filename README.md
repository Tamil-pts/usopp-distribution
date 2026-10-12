# uSOPp.ai downloads

This repository contains installation instructions and release packages. Users do not need the private application source repository.

## Download the published release

Open this repository's **Releases**, select the required version and download
the package assets listed below. Container installation needs both the launcher
ZIP and Linux image archive, plus their checksum files:

```text
uSOPp_AI_<version>_Container_Launchers.zip
uSOPp_AI_<version>_Container_Launchers.zip.sha256
uSOPp_AI_<version>_Container_linux_amd64.tar.gz
uSOPp_AI_<version>_Container_linux_amd64.tar.gz.sha256
```

The automatically generated **Source code** archives contain this downloads
repository's documentation. Choose the named installation packages to install
uSOPp. A missing Linux image asset means the container package is not available
yet; the launcher ZIP alone cannot install the app.

For deleting the original Ubuntu source demo and creating new container
accounts/data, follow [Fresh Ubuntu installation](LAN_SERVER.md#8-replace-the-old-ubuntu-source-demo-with-a-fresh-container).

## Windows: portable application

1. Open **Releases** and download `uSOPp_AI_<version>_Windows_x64.zip`.
2. Right-click the ZIP, select **Extract all**, and keep the complete extracted folder.
3. Double-click `Start_uSOPp.cmd`; the browser opens at `http://127.0.0.1:8200`. Direct launch through `uSOPp_AI.exe` also works.
4. On a fresh installation sign in as **Admin**, password **usopp.2026**, then change it in **Settings → My account**. Existing installations retain their accounts.
5. Register up to five regular users plus Admin. See the app Help for each workspace.

No Python, Node.js, Git or Docker is required. Windows may flag an unsigned EXE; packages are currently unsigned. Accounts, uploads, results and history stay in `%LOCALAPPDATA%\uSOPp.ai`. Finish jobs and use `Stop_uSOPp.cmd` before upgrading. Closing the browser alone does not stop the backend. Keep the data folder; extract each new version into a new application folder.

For a different port or process status, open PowerShell in the extracted application folder:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-local.ps1 start -Port 8201
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-local.ps1 status
```

## Windows: container

The portable application is the simplest choice. For containers, first install [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/), enable the [WSL 2 backend](https://docs.docker.com/desktop/features/wsl/), and start Docker in **Linux containers** mode. Follow Docker's system and licensing requirements for your organization.

Download and extract `uSOPp_AI_<version>_Container_Launchers.zip`. Download `uSOPp_AI_<version>_Container_linux_amd64.tar.gz` into the same folder. Open PowerShell in that folder:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 install -Image usopp-ai:4.65.12 -ImageArchive .\uSOPp_AI_4.65.12_Container_linux_amd64.tar.gz
```

Use the version printed on your downloaded release in both places. Choose your initial Admin password when prompted; open `http://127.0.0.1:8200`. If that port is occupied, add `-Port 8201` to installation.

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 status
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 logs
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 stop
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 start
```

For an update, finish active jobs, download the new image and launchers, and use `update -Image usopp-ai:<new-version> -ImageArchive <new-image-path>`. A full backup is made before startup; the port, bind address and data volume are preserved. The password file in `%LOCALAPPDATA%\uSOPp-container` seeds only a new Admin; use account settings to change an existing password.

## Ubuntu: container

Install [Docker Engine](https://docs.docker.com/engine/install/ubuntu/) and the [Compose plugin](https://docs.docker.com/compose/install/linux/). Your account must have Docker access. Download and extract the same launchers and Linux amd64 image, then:

```bash
bash usopp-container.sh install usopp-ai:4.65.12 ./uSOPp_AI_4.65.12_Container_linux_amd64.tar.gz
bash usopp-container.sh status
bash usopp-container.sh logs
bash usopp-container.sh stop
bash usopp-container.sh start
```

Use `USOPP_PORT=8201 bash usopp-container.sh install ...` to choose another port. Update with `bash usopp-container.sh update usopp-ai:<new-version> <new-image-path>`. Installer settings and initial-password file remain under `~/.config/usopp-container`.

## Data, reports and PLC access

For the agreed shared LAN/VPN server, follow [LAN_SERVER.md](LAN_SERVER.md). It includes server binding, CPU model setup, full backups, updates and restore.

- A named Docker volume, `usopp-container_data`, holds accounts, private user workspaces, unique test IDs, reports, history, rules and knowledge. Stopping or replacing the container retains it. **Do not delete this volume when upgrading.** Finish active jobs before stopping or backing up the data.
- Native Windows and container installations use separate data stores. They do not automatically share accounts or test history.
- Use **Browse / upload folder** for local PLC projects: a Windows path such as `C:\Projects` is not a path inside a Linux container. Only `/data` is mounted by default.
- Live DataInspect needs network access from Docker to the PLC IP and port. Files can be inspected without a PLC; live reads need a real reachable controller.
- The browser port defaults to localhost. Choose the server LAN address with `-BindAddress` on Windows or `USOPP_BIND_ADDRESS` on Ubuntu as shown in the LAN guide. Keep one backend worker because job locks are process-local.
- The Linux image builds an included llama.cpp CPU runtime. In Admin settings choose **Set up small CPU model** to download and verify Qwen2.5-Coder 1.5B once into the persistent volume. No Ollama is needed. Model weights and settings survive updates.

The packages omit original application Python/TypeScript source and frontend source maps. Compiled bytecode, JavaScript and image contents can still be inspected or reverse engineered. They are not source encryption.

Each release supplies SHA256 checksum files. To check a Windows ZIP or image download:

```powershell
Get-FileHash .\uSOPp_AI_4.65.12_Windows_x64.zip -Algorithm SHA256
```

Compare the hash with that file's published `.sha256` companion. On Ubuntu use `sha256sum -c <filename>.sha256` from the download folder.
