# Grok Build CLI on Windows ARM64

How to get xAI's `grok` CLI working on a Snapdragon X laptop when the native ARM64 build crashes
with a stack overflow the moment it opens a TLS connection. On the stable channel the fix is to run
the x64 build under Windows' x64 emulation. The alpha channel has a native ARM64 build that does not
crash. This page has the fix first, then the evidence, so it can be re-applied without re-deriving
it.

Last reviewed: 2026-09-15, Grok Build CLI 1.0.30 (stable) and 1.0.32 (alpha), Samsung Galaxy
Book4 Edge (Snapdragon X Elite), Windows 11 ARM64 Insider build 29667, PowerShell 7.6.5.

## Symptom

Running `grok`, `grok login`, `grok login --device-code` or `grok update --check` in PowerShell
prints one line and exits:

```text
thread 'main' (34592) has overflowed its stack
```

`grok --version` and `grok doctor` work. The Windows Application log gets one Application Error
event per attempt: exception `0xC00000FD` (`STATUS_STACK_OVERFLOW`), faulting module `grok.exe`,
the same fault offset every time. No `~\.grok\auth.json` is ever written because no login completes.

## Cause

The ARM64 build recurses without bound inside the rustls TLS client handshake, right after the TCP
connection to any xAI host succeeds. Commands that do not open an HTTPS connection are unaffected.
The x64 build of the same version does not have the problem. The evidence is under "Diagnosis"
below.

Which ARM64 builds are affected:

| ARM64 build | Channel | `update --check --json` |
|---|---|---|
| 1.0.30 | stable | overflows its stack |
| 1.0.31 | none (downloadable by URL only) | overflows its stack |
| 1.0.32 | alpha | returns JSON |

The 1.0.30 crash reproduces on Windows builds 29648 and 29667. xAI has published nothing about the
fix: the upstream repository has issues disabled and no releases, and none of its recent commits
mention ARM64, TLS or the stack.

## Fix

Option A keeps the stable channel and runs the x64 build. Option B moves to the native alpha build.

### Option A: run the x64 build under emulation

Download the x64 build of the installed version, validate it, keep the ARM64 originals in a backup
folder, and copy the x64 file over both `grok.exe` and `agent.exe`. xAI's installer copies the one
downloaded binary to both names, so both have to be swapped or the agent subprocess still crashes.

The block stops at the first failure and never touches the install until the download has passed
three checks. `curl.exe` does not throw on failure in PowerShell, so its exit code is tested by
hand. Backups are copies, so the originals survive anything that goes wrong later in the block.

```powershell
$ErrorActionPreference = 'Stop'
$bin = "$env:USERPROFILE\.grok\bin"
$ver = (& "$bin\grok.exe" --version) -replace '^grok (\S+).*', '$1'      # 1.0.30 at time of writing
$x64 = "$env:TEMP\grok-$ver-windows-x86_64.exe"

function Get-PeMachine([string]$Path) {
    $fs = [IO.File]::OpenRead($Path); $br = New-Object IO.BinaryReader($fs)
    $fs.Seek(0x3C, 'Begin') | Out-Null; $pe = $br.ReadInt32()
    $fs.Seek($pe + 4, 'Begin') | Out-Null; $m = $br.ReadUInt16(); $br.Close()
    switch ($m) { 0x8664 { 'x64' } 0xAA64 { 'ARM64' } 0x14C { 'x86' } default { '0x{0:X}' -f $m } }
}

# 1. Download. curl.exe reports failure through its exit code only.
curl.exe -fsSL -o $x64 "https://x.ai/cli/grok-$ver-windows-x86_64.exe"
if ($LASTEXITCODE -ne 0) { throw "download failed, curl exit code $LASTEXITCODE" }
# mirror, if x.ai refuses:
# curl.exe -fsSL -o $x64 "https://storage.googleapis.com/grok-build-public-artifacts/cli/grok-$ver-windows-x86_64.exe"

# 2. Validate before touching the install: plausible size, X.AI signature, x64 image.
if ((Get-Item $x64).Length -lt 50MB) { throw "download is too small to be the CLI" }
$sig = Get-AuthenticodeSignature $x64
if ($sig.Status -ne 'Valid' -or $sig.SignerCertificate.Subject -notmatch 'CN=X\.AI LLC') {
    throw "signature check failed: $($sig.Status) / $($sig.SignerCertificate.Subject)"
}
if ((Get-PeMachine $x64) -ne 'x64') { throw "downloaded file is not an x64 image" }

# 3. Back up by copying, so the originals are intact whatever happens next.
Get-Process grok, agent -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
$bak = "$bin\arm64-$ver-backup"
New-Item -ItemType Directory -Force $bak | Out-Null
Copy-Item "$bin\grok.exe"  "$bak\grok.exe"  -Force
Copy-Item "$bin\agent.exe" "$bak\agent.exe" -Force

# 4. Replace both binaries and confirm each now matches the validated download.
Copy-Item $x64 "$bin\grok.exe"  -Force
Copy-Item $x64 "$bin\agent.exe" -Force
$want = (Get-FileHash $x64).Hash
foreach ($f in 'grok.exe', 'agent.exe') {
    if ((Get-FileHash "$bin\$f").Hash -ne $want) { throw "$f does not match the download; restore from $bak" }
}
Remove-Item $x64
```

If the block throws at step 1 or 2, the install is untouched. If it throws at step 3 or 4, the
backup folder holds the originals and the "Revert" section puts them back.

The fix does not touch PATH, the registry or any system file. `~\.grok\bin` is already on the user
PATH from xAI's installer.

### Option B: the native ARM64 alpha build

Same shape as option A, but the download is the ARM64 build of alpha 1.0.32, and the downloaded
file has to complete a TLS handshake before it goes anywhere near the install. The backup folder is
named after the architecture of the files it holds, so it works whether the install is the original
ARM64 build or already swapped by option A.

```powershell
$ErrorActionPreference = 'Stop'
$bin = "$env:USERPROFILE\.grok\bin"
$ver = '1.0.32'                                                  # first ARM64 build seen to finish a TLS handshake
$cur = (& "$bin\grok.exe" --version) -replace '^grok (\S+).*', '$1'
$arm = "$env:TEMP\grok-$ver-windows-aarch64.exe"

function Get-PeMachine([string]$Path) {
    $fs = [IO.File]::OpenRead($Path); $br = New-Object IO.BinaryReader($fs)
    $fs.Seek(0x3C, 'Begin') | Out-Null; $pe = $br.ReadInt32()
    $fs.Seek($pe + 4, 'Begin') | Out-Null; $m = $br.ReadUInt16(); $br.Close()
    switch ($m) { 0x8664 { 'x64' } 0xAA64 { 'ARM64' } 0x14C { 'x86' } default { '0x{0:X}' -f $m } }
}

# 1. Download. curl.exe reports failure through its exit code only.
curl.exe -fsSL -o $arm "https://x.ai/cli/grok-$ver-windows-aarch64.exe"
if ($LASTEXITCODE -ne 0) { throw "download failed, curl exit code $LASTEXITCODE" }

# 2. Validate before touching the install: plausible size, X.AI signature, ARM64 image, and a
#    TLS handshake run from the downloaded file itself.
if ((Get-Item $arm).Length -lt 50MB) { throw "download is too small to be the CLI" }
$sig = Get-AuthenticodeSignature $arm
if ($sig.Status -ne 'Valid' -or $sig.SignerCertificate.Subject -notmatch 'CN=X\.AI LLC') {
    throw "signature check failed: $($sig.Status) / $($sig.SignerCertificate.Subject)"
}
if ((Get-PeMachine $arm) -ne 'ARM64') { throw "downloaded file is not an ARM64 image" }
$null = & $arm update --check --json
if ($LASTEXITCODE -ne 0) { throw "downloaded build failed the TLS smoke test, exit code $LASTEXITCODE" }

# 3. Back up by copying, so the current files are intact whatever happens next.
Get-Process grok, agent -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
$bak = "$bin\$((Get-PeMachine "$bin\grok.exe").ToLower())-$cur-backup"
New-Item -ItemType Directory -Force $bak | Out-Null
Copy-Item "$bin\grok.exe"  "$bak\grok.exe"  -Force
Copy-Item "$bin\agent.exe" "$bak\agent.exe" -Force

# 4. Replace both binaries and confirm each matches the validated download.
Copy-Item $arm "$bin\grok.exe"  -Force
Copy-Item $arm "$bin\agent.exe" -Force
$want = (Get-FileHash $arm).Hash
foreach ($f in 'grok.exe', 'agent.exe') {
    if ((Get-FileHash "$bin\$f").Hash -ne $want) { throw "$f does not match the download; restore from $bak" }
}
Remove-Item $arm
```

Run against a copy of an option-A install (x64 1.0.30) in a scratch folder, this block leaves
`grok 1.0.32 (e21ee47a3bbf) [alpha]` plus an `x64-1.0.30-backup` folder, and the swapped binary
answers `update --check --json` and `-p "reply pong"`. The prompt runs inside `grok.exe` with no
`agent.exe` child, so that test does not exercise the native `agent.exe`.

The trade-off is the alpha channel itself, which xAI describes as "faster updates, may have bugs".
See "Gotchas" for how updates behave after this swap.

## Verify

```powershell
grok --version               # grok 1.0.30 (04b7ffed98c6) [stable]  or  grok 1.0.32 (e21ee47a3bbf) [alpha]
grok update --check --json   # returns JSON; on a crashing build this command overflows
grok login                   # browser flow, or: grok login --device-code
Test-Path "$env:USERPROFILE\.grok\auth.json"   # True after login
```

`grok update --check --json` is the right smoke test. It is the smallest command that performs a
full TLS handshake, and it needs no account.

Run the binary from `~\.grok\bin`. Launched from some other folder, the x64 build opens the
interactive TUI even for `--version`. That is how it detects an installed layout, and it goes away
once the file is in place. The ARM64 1.0.32 build answers `--version` normally from a scratch folder.

## Gotchas

`grok update` re-downloads the `windows-aarch64` build. On stable that is 1.0.30 and the crash comes
back. After any update, run `grok update --check --json` before anything else; if it overflows,
repeat the fix.

After option B, the binary still reads the channel from `~\.grok\config.toml`, which says stable
unless the installer was run with `GROK_CHANNEL=alpha`. Its update check answers
`"currentVersion":"1.0.32","latestVersion":"1.0.30","updateAvailable":false,"channel":"stable"`, so
it does not offer the older build. `grok update --stable` or `grok update --version 1.0.30` would
install a crashing build. `grok update --help` lists `--alpha` to switch to the alpha channel, and
xAI's installer run with `GROK_CHANNEL=alpha` writes `channel = "alpha"` under `[cli]`. Neither has
been tested with this fix.

The installer also puts `grove.exe`, `grove-credential.exe` and `grove-fsmonitor.exe` in
`~\.grok\bin`. They are ARM64, neither option touches them, and they have not been tested.

The `model-gateway` Claude Code plugin reads `~\.grok\auth.json` for its Grok route. A silent revert
shows up there as "Grok CLI auth is missing. Run `grok` and log in again."

There is no public bug to follow. The upstream repository `xai-org/grok-build` has issues disabled;
the only report channel is `/feedback` inside the TUI, which is unreachable on a build that cannot
log in. Report it from a working build.

## Revert

Restores the most recent backup, which undoes the last swap: option A's backup puts the original
ARM64 build back, option B's puts back whatever it replaced. It copies the files back, confirms both
landed, and only then removes the backup. `Move-Item` errors are non-terminating by default, so a
locked destination would otherwise let the block fall through to the delete and lose the only copy.

```powershell
$ErrorActionPreference = 'Stop'
$bin = "$env:USERPROFILE\.grok\bin"
$bak = Get-ChildItem "$bin\*-backup" -Directory | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if (-not $bak) { throw "no backup folder under $bin" }
Get-Process grok, agent -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue

foreach ($f in 'grok.exe', 'agent.exe') {
    Copy-Item "$($bak.FullName)\$f" "$bin\$f" -Force
    if ((Get-FileHash "$bin\$f").Hash -ne (Get-FileHash "$($bak.FullName)\$f").Hash) {
        throw "$f was not restored; backup left in place"
    }
}
Remove-Item $bak.FullName -Recurse
```

## Diagnosis

How the cause was pinned down, with the commands. Each step is cheap and reusable for the next tool
that behaves this way.

### Find the binary and check its architecture

`Get-Command grok` points at `~\.grok\bin\grok.exe`. Read the PE header's machine field rather
than trusting the file name; `0xAA64` is ARM64 and `0x8664` is x64. `Get-PeMachine` is the helper
defined in the fix blocks above.

```powershell
Get-PeMachine "$env:USERPROFILE\.grok\bin\grok.exe"                      # ARM64
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture       # Arm64
```

The installed binary was native ARM64 and validly signed by X.AI LLC. Git Bash and other emulated
shells report `AMD64` from `uname`; only the .NET call is trustworthy for the OS.

### Read the crash records

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'Application'; ProviderName = 'Application Error'; StartTime = (Get-Date).AddHours(-3) } |
    Where-Object { $_.Message -match 'grok\.exe' } |
    Select-Object TimeCreated, @{ n = 'Msg'; e = { ($_.Message -split "`n")[0..6] -join ' | ' } } | Format-List
Get-ChildItem "$env:LOCALAPPDATA\CrashDumps" | Where-Object Name -match grok
```

There were seven events in a few minutes, all `0xC00000FD` in `grok.exe` at fault offset
`0x60bcdc8`. The same offset every time means the same code site every time, so this is
deterministic and not a flake. Windows Error Reporting had also written minidumps to `CrashDumps`,
but no debugger was installed to read them, and the trace log below made that unnecessary.

Filter on `grok.exe`. The same log can hold fail-fasts from unrelated processes, such as
`0xC0000409` from `WorkloadsSessionHost.exe`, which have nothing to do with `grok`.

### Probe commands with a timeout

A TUI takes the console and a login blocks forever, so every probe runs through `Start-Process` with
redirected output and a bounded wait. A probe still running at the timeout counts as a pass for a
command that was going to wait on a person anyway.

```powershell
function Invoke-Probe([string]$Exe, [string[]]$Args, [hashtable]$Env = @{}, [int]$WaitMs = 30000) {
    $out = New-TemporaryFile; $err = New-TemporaryFile; $saved = @{}
    foreach ($k in $Env.Keys) { $saved[$k] = [Environment]::GetEnvironmentVariable($k, 'Process'); [Environment]::SetEnvironmentVariable($k, $Env[$k], 'Process') }
    $p = Start-Process -FilePath $Exe -ArgumentList $Args -RedirectStandardOutput $out -RedirectStandardError $err -NoNewWindow -PassThru
    if (-not $p.WaitForExit($WaitMs)) { Stop-Process -Id $p.Id -Force; $code = 'TIMEOUT (still running)' } else { $code = '0x{0:X}' -f $p.ExitCode }
    foreach ($k in $Env.Keys) { [Environment]::SetEnvironmentVariable($k, $saved[$k], 'Process') }
    "=== $($Args -join ' ') -> $code"; Get-Content $out | Select-Object -First 8; Get-Content $err | Select-Object -First 8
    Remove-Item $out, $err
}
$g = "$env:USERPROFILE\.grok\bin\grok.exe"
Invoke-Probe $g @('login', '--device-code')          # 0xC00000FD within a second
Invoke-Probe $g @('update', '--check', '--json')     # 0xC00000FD within a second
Invoke-Probe $g @('doctor')                          # exits 0
Invoke-Probe $g @('-p', '"reply pong"')              # exits 1, "Not signed in", no crash
```

Every command that opens an HTTPS connection died, and every command that does not ran fine.

### Read the tool's own trace log

`grok` writes a log when `GROK_LOG_FILE` is set and honours `RUST_LOG`.

```powershell
Invoke-Probe $g @('update', '--check', '--json') @{ GROK_LOG_FILE = "$env:TEMP\grok-trace.log"; RUST_LOG = 'trace' }
Get-Content "$env:TEMP\grok-trace.log" -Tail 6
```

The last lines before the overflow were the same on every run:

```text
DEBUG reqwest::connect: starting new connection: https://x.ai/
DEBUG hyper_util::client::legacy::connect::http: connected to [2606:4700::6812:1350]:443
DEBUG rustls::client::hs: No cached session for DnsName("x.ai")
DEBUG rustls::client::hs: Not resuming any session
```

The TCP connection had succeeded and rustls was about to build the client hello. Nothing after that
line was ever logged. The overflowing thread was `main` when the blocking client ran the request and
`tokio-rt-worker` when the async client did, which rules out anything specific to one thread's
stack.

### Rule out the cheap explanations

An unreachable proxy stops the handshake from starting. With `HTTPS_PROXY=http://127.0.0.1:9` the
TUI came up and showed "error sending request" instead of crashing, so the crash needs a real
connection.

A bigger stack does not help. A copy of `grok.exe` with the PE optional header's
`SizeOfStackReserve` raised from 1 MB to 64 MB overflowed identically. That makes it unbounded
recursion, which no stack size fixes.

The machine's network settings were clean. `netsh winhttp show proxy` reported direct access, the
Internet Settings registry keys had no proxy or PAC, and no proxy variables were in the environment.

Whether a different TLS server also triggers it could not be isolated. The CLI opens its x.ai
connection first on every code path and dies there before any overridden host is reached.

### Confirm with the x64 build

The same version's x64 build, downloaded to a scratch folder and run under emulation, ran all three
crashing commands for the full wait without a fault. With the trace log on, it continued past the
point the ARM64 build died:

```text
DEBUG rustls::client::hs: Using ciphersuite TLS13_AES_256_GCM_SHA384
DEBUG rustls::client::tls13: TLS1.3 encrypted extensions: ServerExtensions { ... }
DEBUG rustls::client::hs: ALPN protocol is Some(b"h2")
WARN  ... Failed to fetch models error=RequestFailed { status: 400, ... "Incorrect API key provided" ...
```

It completed TLS 1.3 sessions with `api.x.ai`, `auth.x.ai` and `cli-chat-proxy.grok.com` and got
HTTP responses back, which is the basis for option A.

### Check newer builds

The channel pointers name the current version, and every build is downloadable by version, including
ones no channel points at:

```powershell
foreach ($c in 'stable', 'alpha') { "$c : " + (curl.exe -fsSL "https://x.ai/cli/$c") }   # stable : 1.0.30, alpha : 1.0.32
```

Download each ARM64 build to its own scratch folder and probe it with `Invoke-Probe`. 1.0.30 and
1.0.31 overflow. 1.0.32 runs cleanly:

```text
=== update --check --json -> 0x0
{"currentVersion":"1.0.32","latestVersion":"1.0.30","updateAvailable":false,"installer":"internal","channel":"stable","autoUpdate":null,"error":null}
DEBUG run_update_command: rustls::client::hs: Using ciphersuite TLS13_AES_256_GCM_SHA384
DEBUG run_update_command: rustls::client::hs: ALPN protocol is Some(b"h2")
=== login --device-code -> TIMEOUT (still running)
  https://accounts.x.ai/oauth2/device?user_code=...
=== -p "reply pong" -> 0x0
pong
```

The trace passes the point where 1.0.30 dies, the device-code flow reaches `accounts.x.ai`, and a
signed-in prompt gets a model reply. Option B rests on these results.

## Related

- [`OPENCODE-WINDOWS-ARM64.md`](OPENCODE-WINDOWS-ARM64.md): the same shape of problem in OpenCode,
  fixed the same way.
- Installer reference: `https://x.ai/cli/install.ps1` reads the channel from `GROK_CHANNEL`
  (`stable`, `alpha` or `enterprise`), the version from `https://x.ai/cli/<channel>`, downloads
  `grok-<version>-windows-<x86_64|aarch64>.exe`, and installs that one file as both `grok.exe` and
  `agent.exe`.
