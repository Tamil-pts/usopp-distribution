# 1. Windows installation — uSOPp.ai 4.66.0

Run commands in **Windows PowerShell**, one numbered step at a time. Stop if a command reports an error.

Choose **A: portable application** for this laptop. Choose **B: Docker container** if you specifically want Docker. They use separate account/data stores. Run one installation on a given port.

## A. Portable Windows application

### A1. Prepare the downloaded files

Create this folder and copy the following two downloaded files into it using File Explorer:

```text
%USERPROFILE%\Downloads\uSOPp-4.66.0

uSOPp_AI_4.66.0_Windows_x64.zip
uSOPp_AI_4.66.0_Windows_x64.zip.sha256
```

The ZIP in your screenshot is the correct Windows application. It includes its runtime; Python, Node, Git and Docker are unnecessary for this method.

### A2. Verify the ZIP

Open PowerShell and paste:

```powershell
$ErrorActionPreference = 'Stop'
$DownloadFolder = Join-Path $env:USERPROFILE 'Downloads\uSOPp-4.66.0'
Set-Location -LiteralPath $DownloadFolder
$ZipPath = Join-Path $DownloadFolder 'uSOPp_AI_4.66.0_Windows_x64.zip'
$Expected = ((Get-Content -LiteralPath "$ZipPath.sha256" -Raw).Trim() -split '\s+')[0]
$Actual = (Get-FileHash -LiteralPath $ZipPath -Algorithm SHA256).Hash
if ($Expected -notmatch '^[0-9a-fA-F]{64}$' -or $Actual -ine $Expected) {
    throw 'Checksum mismatch. Download the ZIP and its checksum again.'
}
Write-Host 'Windows ZIP checksum: OK'
```

### A3. If upgrading an existing portable installation, stop it first

Finish validations and exports. Open the **old extracted application folder** and double-click `Stop_uSOPp.cmd`.

Keep `%LOCALAPPDATA%\uSOPp.ai`; this is the saved account, knowledge and results directory. A development checkout and a Docker installation have their own stores and are not automatically migrated to the portable app.

### A4. Extract into a new version folder

Continue in the same PowerShell window:

```powershell
$InstallFolder = Join-Path $env:LOCALAPPDATA 'uSOPp-releases\4.66.0'
if (Test-Path -LiteralPath $InstallFolder) {
    throw 'This version folder already exists. Use its extracted uSOPp_AI folder, or choose a different new InstallFolder.'
}
Expand-Archive -LiteralPath $ZipPath -DestinationPath $InstallFolder
$AppFolder = Join-Path $InstallFolder 'uSOPp_AI'
Set-Location -LiteralPath $AppFolder
Test-Path -LiteralPath .\uSOPp_AI.exe
```

The last command must show `True`. Keep the entire `uSOPp_AI` folder, including `_internal` and the launchers.

### A5. Start the application

**If this Windows account already has uSOPp portable data**, run:

```powershell
.\Start_uSOPp.cmd
```

Use your existing login. Restarting does not reset its password.

**For the first portable installation**, choose an initial Admin password without printing it. Run this block instead of the command above:

```powershell
$SecurePassword = Read-Host 'Choose an initial Admin password, 12 to 128 characters' -AsSecureString
$PasswordPointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($SecurePassword)
try {
    $env:USOPP_INITIAL_ADMIN_PASSWORD = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($PasswordPointer)
    if ($env:USOPP_INITIAL_ADMIN_PASSWORD.Length -lt 12 -or $env:USOPP_INITIAL_ADMIN_PASSWORD.Length -gt 128) {
        throw 'Use a password with 12 to 128 characters.'
    }
    powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-local.ps1 start
    if ($LASTEXITCODE -ne 0) { throw 'Application launch failed.' }
}
finally {
    Remove-Item Env:USOPP_INITIAL_ADMIN_PASSWORD -ErrorAction SilentlyContinue
    [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($PasswordPointer)
}
```

Sign in as `Admin` with the password you chose. This seeds a new account only; it does not replace an existing Admin password.

### A6. Verify the running version

```powershell
$Health = Invoke-RestMethod -Uri 'http://127.0.0.1:8200/api/health'
$Health.version
if ($Health.version -ne '4.66.0') { throw 'Another application version is using port 8200.' }
$ExpectedDataRoot = Join-Path $env:LOCALAPPDATA 'uSOPp.ai'
if ($Health.applicationRoot -ine $ExpectedDataRoot) {
    throw 'Port 8200 is serving another installation, such as the development checkout or Docker. Stop it or use port 8201 for this portable package.'
}
Start-Process 'http://127.0.0.1:8200'
```

After a browser refresh, the header should show **UI 4.66.0 | API 4.66.0**. If it shows an older UI, press `Ctrl+F5`.

If port 8200 is occupied, stop the old application or start this package on 8201 instead:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-local.ps1 start -Port 8201
```

For that choice use `http://127.0.0.1:8201` in the browser and health command. On a first installation, add `-Port 8201` to the launcher line inside the password block.

### A7. Configure offline Chopper

1. Open **Settings → Chopper setup → Local CPU model**.
2. Click **Set up local Chopper**, or **Install 3B** in the model list. The model is a separate one-time download; its weights are not inside the application ZIP.
3. Select **Use 3B primary** once installed and wait for **Ready**.
4. Keep **Start with the server** enabled. Install or retain the **1.5B fallback** if needed.
5. Open **AI Assistant → Chat**. Choose **Guide me step by step**, or attach/select a PLC project. Click **Project Intelligence** to expand its details.

If Chopper reports its model port is already in use after an old application was stopped, restart Windows or choose an unused **Model port** under Advanced settings, save and activate the model. The model port and the browser/API port are separate settings.

### A8. Daily commands

Run from the extracted `uSOPp_AI` folder:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-local.ps1 status
```

After finishing active work, stop with:

```powershell
.\Stop_uSOPp.cmd
```

Start again with `Start_uSOPp.cmd`. Closing the browser alone leaves the application running.

## B. Alternative: Windows Docker container

### B1. Prepare Docker and downloads

Install and start [Docker Desktop with the WSL 2 backend and Linux containers](https://docs.docker.com/desktop/setup/install/windows-install/). Then open a new PowerShell window.

Copy these four files into `%USERPROFILE%\Downloads\uSOPp-4.66.0`:

```text
uSOPp_AI_4.66.0_Container_Launchers.zip
uSOPp_AI_4.66.0_Container_Launchers.zip.sha256
uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz
uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz.sha256
```

The `.tar.gz` is a compiled Docker image archive. Keep it compressed; the installer loads it directly.

### B2. Verify both archives

```powershell
$ErrorActionPreference = 'Stop'
$DownloadFolder = Join-Path $env:USERPROFILE 'Downloads\uSOPp-4.66.0'
Set-Location -LiteralPath $DownloadFolder
docker info --format '{{.OSType}}'
if ($LASTEXITCODE -ne 0) { throw 'Start Docker Desktop first.' }
docker compose version
if ($LASTEXITCODE -ne 0) { throw 'Docker Compose is unavailable.' }
foreach ($Name in @('uSOPp_AI_4.66.0_Container_Launchers.zip', 'uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz')) {
    $File = Join-Path $DownloadFolder $Name
    $Expected = ((Get-Content -LiteralPath "$File.sha256" -Raw).Trim() -split '\s+')[0]
    $Actual = (Get-FileHash -LiteralPath $File -Algorithm SHA256).Hash
    if ($Expected -notmatch '^[0-9a-fA-F]{64}$' -or $Actual -ine $Expected) { throw "Checksum mismatch: $Name" }
    Write-Host "$Name : OK"
}
```

Docker's OS output must be `linux`.

### B3. Extract launchers and install

```powershell
$LauncherFolder = Join-Path $env:LOCALAPPDATA 'uSOPp-container-launchers\4.66.0'
if (Test-Path -LiteralPath $LauncherFolder) { throw 'Use the already extracted launcher folder, or choose a new LauncherFolder.' }
Expand-Archive -LiteralPath '.\uSOPp_AI_4.66.0_Container_Launchers.zip' -DestinationPath $LauncherFolder
Set-Location -LiteralPath $LauncherFolder
$ImageArchive = Join-Path $DownloadFolder 'uSOPp_AI_4.66.0_Container_linux_amd64.tar.gz'
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 install -Image usopp-ai:4.66.0 -ImageArchive $ImageArchive -Port 8200
if ($LASTEXITCODE -ne 0) { throw 'Container installation failed. Read the error before continuing.' }
```

Choose the Admin password when prompted. Open `http://127.0.0.1:8200`. If a Windows container installation already exists, use **update** instead of **install**; it preserves its configured port/data volume and makes a backup:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 update -Image usopp-ai:4.66.0 -ImageArchive $ImageArchive
```

### B4. Verify and manage

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 status
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 logs
docker inspect usopp-container-app-1 --format '{{.Config.Image}}'
```

Expected image: `usopp-ai:4.66.0`. Follow A7 for Chopper setup. Docker data is in `usopp-container_data`, separate from the native Windows data folder.

After active work finishes:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 stop
```

To start again:

```powershell
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\usopp-container.ps1 start
```
