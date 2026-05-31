# Windows remote deployment — record on Windows, ship raw to our central

Goal: install the **thin-client** screenpipe agent (recorder + sync-shim) on a
Windows PC, driven remotely with commands from the Linux desktop, so it captures
frames+audio and ships **raw** to our central for GPU OCR/ASR/embedding.

The agent on Windows is two binaries:
- `screenpipe.exe`  — the fork-built recorder (`crates/screenpipe-engine` bin).
- `sync-shim.exe`   — the raw pusher (`product/sync-shim`).

Both auto-start via **At-Logon Scheduled Tasks** (screen capture needs the
interactive desktop session, NOT a session-0 service).

---

## 0. Decide the central address the Windows PC ships to

- **WireGuard-connected Windows PC** (e.g. the fleet Windows VM `10.10.0.7`):
  `SHIM_CENTRAL_URL = http://10.10.0.2:8090`  ← VERIFIED reachable (health 200).
- **Employee PC NOT on WireGuard** (the real fleet case): the central must be
  behind **public HTTPS** (`https://screenpipe.kaxtus.com` → reverse-proxy over
  WG to the desktop `:8090`). That proxy is the `environment`-lane cutover and is
  NOT wired yet. Until it is, use a WG-connected Windows PC.

The agent enrols with a per-agent token issued on the central:
```bash
# on the desktop (central host):
cd ~/Projects/screenpipe/product/central-server
set -a; . ~/.config/screenpipe-agent/central.env; set +a
.venv/bin/python -m app.cli issue-token --agent-key win-<name>
# prints spk_... ONCE — use it as -Token below.
```

---

## 1. Build the two .exe (ONE TIME — the heavy prerequisite)

A stock/upstream `screenpipe.exe` will NOT do: our thin-client `SCREENPIPE_DISABLE_OCR`
gate is a fork patch. Build the fork's bin on a Windows machine (windows-msvc).

On the Windows build box (elevated PowerShell):
```powershell
# toolchain
winget install --id Rustlang.Rustup -e --accept-source-agreements --accept-package-agreements
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --override `
  "--quiet --wait --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
winget install --id Git.Git -e
rustup default stable-x86_64-pc-windows-msvc

# source (push the branch first from the desktop, or copy the tree over)
git clone -b linux-build-fixes <fork-remote-url> screenpipe ; cd screenpipe

# build (engine bin has native deps — see screenpipe CONTRIBUTING for Windows;
# ffmpeg/onnxruntime may need vcpkg or the project's prebuilt deps step)
cargo build -p screenpipe-engine --bin screenpipe --release
cargo build --release --manifest-path product\sync-shim\Cargo.toml

# outputs:
#   target\release\screenpipe.exe
#   product\sync-shim\target\release\sync-shim.exe
```
> The **sync-shim** builds trivially (rusqlite/reqwest/image). The **engine** is
> the hard part on Windows (native capture + ffmpeg + ONNX). It is the supported
> path (the project ships a Windows GUI), but budget time for the dep setup.
>
> Build ONCE; the resulting two .exe are reused on every employee PC of the same
> arch — no per-PC build.

---

## 2. Enable a remote command channel on the target PC (one time)

Easiest is OpenSSH Server (built into Windows 10/11). On the target, once (console
or RDP), elevated:
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Set-Service sshd -StartupType Automatic ; Start-Service sshd
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server' -Enabled True `
  -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
# default shell -> PowerShell (so remote commands run pwsh)
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell `
  -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```
(Alternative: WinRM via `Enable-PSRemoting -Force` + `Invoke-Command`.)

NOTE the fleet Windows VM `10.10.0.7` was OFFLINE/unreachable at write time
(no ping, ports 22/3389/5985/5986 closed) — bring it up + do this step first.

---

## 3. Deploy + enroll remotely (from the desktop, every PC)

Use the driver: `product/packaging/windows/remote-deploy.sh` — it scp's the two
.exe + the 5 .ps1, then runs install + enroll over SSH:
```bash
cd ~/Projects/screenpipe/product/packaging/windows
./remote-deploy.sh \
  --host 10.10.0.7 --user <winuser> \
  --recorder /path/to/screenpipe.exe --shim /path/to/sync-shim.exe \
  --central-url http://10.10.0.2:8090 \
  --agent-id win-<name> --token spk_xxx \
  [--monitors "1,2"]
```
Equivalent manual commands (what the driver runs):
```bash
H=10.10.0.7 ; U=<winuser> ; DST='C:/Users/<winuser>/screenpipe-deploy'
ssh $U@$H "powershell -c New-Item -ItemType Directory -Force -Path $DST\bin | Out-Null"
scp screenpipe.exe sync-shim.exe $U@$H:"$DST/bin/"
scp install.ps1 enroll.ps1 run-recorder.ps1 run-shim.ps1 uninstall.ps1 $U@$H:"$DST/"
ssh $U@$H "powershell -ExecutionPolicy Bypass -File $DST\install.ps1 -RecorderBin $DST\bin\screenpipe.exe -ShimBin $DST\bin\sync-shim.exe"
ssh $U@$H "powershell -ExecutionPolicy Bypass -File $DST\enroll.ps1 -CentralUrl http://10.10.0.2:8090 -AgentId win-<name> -Token spk_xxx"
```
`enroll.ps1` writes recorder.json + shim.json and starts the two Scheduled Tasks.
The launchers already bake the thin-client config: recorder runs with
`SCREENPIPE_DISABLE_OCR/MDNS/KEYCHAIN=1` + `--audio-transcription-engine disabled`;
shim runs with `SHIM_RAW=true`.

---

## 4. Verify (remote)

```bash
# tasks running + API up on the Windows PC
ssh $U@$H "powershell -c Get-ScheduledTask -TaskName ScreenpipeRecorder,ScreenpipeSyncShim | ft TaskName,State"
ssh $U@$H "powershell -c (Invoke-WebRequest http://127.0.0.1:3030/health -UseBasicParsing).StatusCode"
```
```bash
# central is receiving from this agent (on the desktop):
PGPASSWORD=screenpipe docker exec central-server-db-1 psql -U screenpipe -d central -tAc \
 "select count(*) from frames f join agents a on a.id=f.agent_id where a.agent_key='win-<name>';"
```
Add the agent to the watcher by querying its row count, or just watch the central
totals climb.

---

## Caveats / open decisions
- **Engine Windows build is the gate.** Until `screenpipe.exe` is built on
  Windows, nothing else matters. Build it once on the Windows VM (or via CI).
- **WG vs public.** Real employee PCs aren't on WG → need the kaxtus→WG→:8090
  public proxy (environment lane) before they can reach the central.
- **Capture needs a logged-in desktop.** At-Logon tasks only run when the user is
  signed in (RDP/console). A locked screen still captures; a signed-out PC does not.
- **Monitor IDs differ on Windows** (1,2,… not the Linux 596/445). Pass `-Monitors`
  after checking `screenpipe vision list` (or omit to record all).
