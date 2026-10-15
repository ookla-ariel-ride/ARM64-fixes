# Grok Build CLI on Windows ARM64

xAI's `grok` CLI crashed on Snapdragon X laptops with a stack overflow the moment it opened a TLS
connection. That was true of the native ARM64 builds 1.0.30 and 1.0.31. Every ARM64 build tested
since, from alpha 1.0.32 up to the current stable 1.0.52 and alpha 1.0.60, completes the handshake,
so a current install needs no workaround. This page is for anyone still on one of the two crashing
builds, which cannot update themselves because the update check is itself a TLS request. The fix is
to rerun xAI's installer. The evidence and the diagnosis commands follow, so the next build that
behaves this way can be checked the same way.

Last reviewed: 2026-10-15, Grok Build CLI 1.0.52 (stable) and 1.0.60 (alpha), Samsung Galaxy Book4
Edge (Snapdragon X Elite), Windows 11 ARM64 Insider build 29683, PowerShell 7.6.5.

## Symptom

On 1.0.30 or 1.0.31, running `grok`, `grok login`, `grok login --device-code` or
`grok update --check` in PowerShell prints one line and exits:

```text
thread 'main' (34592) has overflowed its stack
```

`grok --version` and `grok doctor` work. The Windows Application log gets one Application Error
event per attempt: exception `0xC00000FD` (`STATUS_STACK_OVERFLOW`), faulting module `grok.exe`,
the same fault offset every time. No `~\.grok\auth.json` is ever written because no login completes.

## Cause

The ARM64 build recurses without bound inside the rustls TLS client handshake, right after the TCP
connection to any xAI host succeeds. Commands that do not open an HTTPS connection are unaffected.
The x64 build of the same version does not have the problem. The evidence is under "Diagnosis".

Which ARM64 builds are affected, by the `update --check --json` smoke test:

| ARM64 build | Channel when tested | Result | Tested |
|---|---|---|---|
| 1.0.30 | stable | overflows its stack | 2026-09-15 |
| 1.0.31 | none (downloadable by URL only) | overflows its stack | 2026-09-15 |
| 1.0.32 | alpha | returns JSON | 2026-09-15 |
| 1.0.34 | stable | returns JSON | 2026-09-17 |
| 1.0.35 | alpha | returns JSON | 2026-09-17 |
| 1.0.52 | stable | returns JSON | 2026-10-15 |
| 1.0.60 | alpha | returns JSON | 2026-10-15 |

The 1.0.30 crash reproduced on Windows builds 29648 and 29667. xAI has published nothing about it:
the upstream repository has issues disabled and no releases, and none of its commits mention ARM64,
TLS or the stack.

## Fix

Rerun xAI's installer. It downloads the current stable build with PowerShell's own HTTP client, so it
works from a machine whose installed `grok` cannot open a connection:

```powershell
irm https://x.ai/cli/install.ps1 | iex
```

The script reads the channel from `GROK_CHANNEL` (`stable`, `alpha` or `enterprise`), takes the
version from `https://x.ai/cli/<channel>`, downloads `grok-<version>-windows-aarch64.exe`, and copies
that one file to both `~\.grok\bin\grok.exe` and `~\.grok\bin\agent.exe`. It also puts the `grove*`
helper executables and a bundled Git beside them, writes `installer = "internal"` (and the channel,
if not stable) under `[cli]` in `~\.grok\config.toml`, and adds `~\.grok\bin` to the user PATH if it
is missing. The only `grok` command it runs is `completions powershell`, which does not go online.

`grok update` is the same download once the installed build can reach x.ai, and it is how 1.0.34
moved to 1.0.52 on the test machine. It leaves the previous binary beside the new one as
`grok.exe.old`.

If an earlier version of this page left an x64 build in `~\.grok\bin`, nothing special is needed. The
installer and `grok update` both replace `grok.exe` and `agent.exe` with the ARM64 build, and the
`arm64-1.0.30-backup` folder from that swap can be deleted.

## Verify

```powershell
grok --version               # grok 1.0.52 (0920c8169e27) [stable]
grok update --check --json   # returns JSON; on a crashing build this command overflows
grok login                   # browser flow, or: grok login --device-code
Test-Path "$env:USERPROFILE\.grok\auth.json"   # True after login
```

`grok update --check --json` is the right smoke test. It is the smallest command that performs a
full TLS handshake, and it needs no account.

Run the binary from `~\.grok\bin`. Launched from some other folder with no arguments, `grok` opens
the interactive TUI. `update --check --json` and `--version` answer normally from anywhere.

## Gotchas

Never launch `grok` with no arguments to test a build, from any folder. The TUI checks for updates
on start and installs one into `~\.grok\bin` within about a minute, without asking. During the
2026-09-17 review a scratch copy of 1.0.33 launched that way replaced the real install with 1.0.34.
There is no documented opt-out, and `update --check --json` reports `"autoUpdate": null`.

Code signing changed between channels. Stable 1.0.52 is signed by `X.AI LLC` (Palo Alto); alpha
1.0.60 is signed by `X Corp.` (Bastrop, Texas). Any script that checks the Authenticode signer has
to accept both subjects.

The `grove.exe`, `grove-credential.exe` and `grove-fsmonitor.exe` files that the installer adds are
ARM64 and have not been tested here.

The `model-gateway` Claude Code plugin reads `~\.grok\auth.json` for its Grok route. A build that
cannot log in shows up there as "Grok CLI auth is missing. Run `grok` and log in again."

There is no public bug to follow. The upstream repository `xai-org/grok-build` has issues disabled;
the only report channel is `/feedback` inside the TUI, which is unreachable on a build that cannot
log in.

## Diagnosis

How the cause was pinned down, with the commands. Each step is cheap and reusable for the next tool
that behaves this way.

### Find the binary and check its architecture

`Get-Command grok` points at `~\.grok\bin\grok.exe`. Read the PE header's machine field rather
than trusting the file name; `0xAA64` is ARM64 and `0x8664` is x64.

```powershell
function Get-PeMachine([string]$Path) {
    $fs = [IO.File]::OpenRead($Path); $br = New-Object IO.BinaryReader($fs)
    $fs.Seek(0x3C, 'Begin') | Out-Null; $pe = $br.ReadInt32()
    $fs.Seek($pe + 4, 'Begin') | Out-Null; $m = $br.ReadUInt16(); $br.Close()
    switch ($m) { 0x8664 { 'x64' } 0xAA64 { 'ARM64' } 0x14C { 'x86' } default { '0x{0:X}' -f $m } }
}
Get-PeMachine "$env:USERPROFILE\.grok\bin\grok.exe"                      # ARM64
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture       # Arm64
```

The installed binary was native ARM64 and validly signed. Git Bash and other emulated shells report
`AMD64` from `uname`; only the .NET call is trustworthy for the OS.

### Read the crash records

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'Application'; ProviderName = 'Application Error'; StartTime = (Get-Date).AddHours(-3) } |
    Where-Object { $_.Message -match 'grok\.exe' } |
    Select-Object TimeCreated, @{ n = 'Msg'; e = { ($_.Message -split "`n")[0..6] -join ' | ' } } | Format-List
Get-ChildItem "$env:LOCALAPPDATA\CrashDumps" | Where-Object Name -match grok
```

There were seven events in a few minutes, all `0xC00000FD` in `grok.exe` at fault offset
`0x60bcdc8`. The same offset every time means the same code site every time, so the crash is
deterministic. Windows Error Reporting had also written minidumps to `CrashDumps`, but no debugger
was installed to read them, and the trace log below made that unnecessary.

Filter on `grok.exe`. The same log can hold fail-fasts from unrelated processes, such as
`0xC0000409` from `WorkloadsSessionHost.exe`, which have nothing to do with `grok`.

### Probe commands with a timeout

A TUI takes the console and a login blocks forever, so every probe runs through `Start-Process` with
redirected output and a bounded wait. A probe still running at the timeout counts as a pass for a
command that was going to wait on a person anyway. The parameter is named `$Arguments` on purpose:
in PowerShell 7.6 a parameter named `$Args` is always empty, so the probe would launch the bare TUI
and trigger the self-update described under "Gotchas".

```powershell
function Invoke-Probe([string]$Exe, [string[]]$Arguments, [hashtable]$Env = @{}, [int]$WaitMs = 30000) {
    $out = New-TemporaryFile; $err = New-TemporaryFile; $saved = @{}
    foreach ($k in $Env.Keys) { $saved[$k] = [Environment]::GetEnvironmentVariable($k, 'Process'); [Environment]::SetEnvironmentVariable($k, $Env[$k], 'Process') }
    $p = Start-Process -FilePath $Exe -ArgumentList $Arguments -RedirectStandardOutput $out -RedirectStandardError $err -NoNewWindow -PassThru
    if (-not $p.WaitForExit($WaitMs)) { Stop-Process -Id $p.Id -Force; $code = 'TIMEOUT (still running)' } else { $code = '0x{0:X}' -f $p.ExitCode }
    foreach ($k in $Env.Keys) { [Environment]::SetEnvironmentVariable($k, $saved[$k], 'Process') }
    "=== $($Arguments -join ' ') -> $code"; Get-Content $out | Select-Object -First 8; Get-Content $err | Select-Object -First 8
    Start-Sleep 1; Remove-Item $out, $err
}
$g = "$env:USERPROFILE\.grok\bin\grok.exe"
Invoke-Probe $g @('login', '--device-code')          # 0xC00000FD within a second on 1.0.30
Invoke-Probe $g @('update', '--check', '--json')     # 0xC00000FD within a second on 1.0.30
Invoke-Probe $g @('doctor')                          # exits 0
Invoke-Probe $g @('-p', '"reply pong"')              # exits 1, "Not signed in", no crash
```

On 1.0.30 every command that opened an HTTPS connection died, and every command that did not ran
fine.

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

The x64 build of 1.0.30, downloaded to a scratch folder and run under emulation, ran all three
crashing commands for the full wait without a fault. With the trace log on, it continued past the
point the ARM64 build died:

```text
DEBUG rustls::client::hs: Using ciphersuite TLS13_AES_256_GCM_SHA384
DEBUG rustls::client::tls13: TLS1.3 encrypted extensions: ServerExtensions { ... }
DEBUG rustls::client::hs: ALPN protocol is Some(b"h2")
WARN  ... Failed to fetch models error=RequestFailed { status: 400, ... "Incorrect API key provided" ...
```

It completed TLS 1.3 sessions with `api.x.ai`, `auth.x.ai` and `cli-chat-proxy.grok.com` and got
HTTP responses back. Running that build under emulation was this page's fix until the native builds
caught up.

### Check newer builds

The channel pointers name the current version, and every build is downloadable by version, including
ones no channel points at:

```powershell
foreach ($c in 'stable', 'alpha') { "$c : " + (curl.exe -fsSL "https://x.ai/cli/$c") }   # stable : 1.0.52, alpha : 1.0.60
$v = '1.0.52'; $d = New-Item -ItemType Directory -Force "$env:TEMP\grok-$v"
curl.exe -fsSL -o "$d\grok.exe" "https://x.ai/cli/grok-$v-windows-aarch64.exe"
Get-PeMachine "$d\grok.exe"                                        # ARM64
(Get-AuthenticodeSignature "$d\grok.exe").SignerCertificate.Subject
Invoke-Probe "$d\grok.exe" @('update', '--check', '--json')
```

Each ARM64 build goes into its own scratch folder and is probed only with the commands above, never
launched bare. On 2026-10-15 both current builds passed:

```text
=== update --check --json -> 0x0
{"currentVersion":"1.0.52","latestVersion":"1.0.52","updateAvailable":false,"installer":"internal","channel":"stable","autoUpdate":null,"error":null}
=== update --check --json -> 0x0
{"currentVersion":"1.0.60","latestVersion":"1.0.52","updateAvailable":false,"installer":"internal","channel":"stable","autoUpdate":null,"error":null}
```

On 2026-09-15, 1.0.32 was the first ARM64 build to pass. Its device-code login reached
`accounts.x.ai` and a signed-in `-p "reply pong"` got `pong` back.

## Related

- [`OPENCODE-WINDOWS-ARM64.md`](OPENCODE-WINDOWS-ARM64.md): a different cause with the same shape,
  a native ARM64 build that fails and an x64 build that works under emulation.
- Installer reference: `https://x.ai/cli/install.ps1`, described under "Fix". `GROK_BIN_DIR`
  overrides the install folder and `GROK_VERSION` pins a version.
