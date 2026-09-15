# OpenCode on Windows ARM64

How to get OpenCode working on a Snapdragon X laptop when the native ARM64 build refuses to start
its TUI with a `bun:ffi` error. The fix is to run the x64 build under Windows' x64 emulation and
turn off OpenCode's self-update, which otherwise puts the ARM64 build back. There is no working
native release to pin. This page has the fix first, then the evidence, so it can be re-applied
without re-deriving it.

Last reviewed: 2026-09-15, OpenCode 1.18.31 installed with `npm i -g opencode-ai`, Samsung Galaxy
Book4 Edge (Snapdragon X Elite), Windows 11 ARM64 Insider build 29667, PowerShell 7.6.5, Node 26.7.0.
First written: 2026-09-12 on OpenCode 1.18.30 and Windows build 29648. 1.18.31 was also seen to fail
the same way on a second ARM64 laptop.

## Symptom

Starting the TUI, plain `opencode`, exits 1 at once:

```text
Error: Unexpected error

Failed to initialize OpenTUI render library: bun:ffi dlopen() is not available in this build (TinyCC is disabled)
```

Commands that do not draw a screen are not a reliable signal. On 2026-09-12, `auth list`, `models`
and `run` all failed with the same message on 1.18.30. On 2026-09-15 the byte-identical binary ran
`auth list` and `models` successfully and still failed to start the TUI. That difference was not
explained. `opencode --version` worked on every run.

There are no crash events in the Windows Application log, because this is a clean error exit rather
than a native fault. OpenCode's own log under `~\.local\share\opencode\log` gets nothing from the
failing run, since the failure happens before logging starts.

## Cause

OpenCode is compiled with Bun. Its terminal renderer, OpenTUI, loads a native DLL through
`bun:ffi`. Stable Bun compiles `bun:ffi` out on `windows-aarch64` because TinyCC, which it used for
FFI, has no ARM64 backend. An OpenCode ARM64 binary built with such a Bun cannot start its renderer.

Every OpenCode ARM64 release tested, 1.18.26 through 1.18.31, embeds Bun 1.3.14 and fails to start
the TUI. This is not a regression in one release. Bun 1.4.0, released 2026-08-20, has an
engine-native FFI, but OpenCode has not moved to it. The upstream issues have been open since March;
the details are under "Diagnosis".

## Fix

Two steps: swap in the x64 binary, then stop OpenCode from updating itself back to ARM64.

### 1. Run the x64 build under emulation

Install the x64 platform package into a scratch prefix, validate it, and copy its binary over the
one the npm shim runs. npm refuses to install a package whose `cpu` field says `x64` on an ARM64
host, and does so silently under `--silent`, so the platform override flags are required.

The block stops at the first failure and never touches the install until the package has been
checked. `npm` does not throw on failure in PowerShell, so its exit code is tested by hand. The
backup is a copy, so the original survives anything that goes wrong later. npm packages carry no
Authenticode signature, so the checks here are existence, architecture and a version round trip.

```powershell
$ErrorActionPreference = 'Stop'
$pkg = "$env:APPDATA\npm\node_modules\opencode-ai"
$ver = & "$env:APPDATA\npm\opencode.cmd" --version       # 1.18.31 at time of writing
$t   = "$env:TEMP\opencode-x64"

function Get-PeMachine([string]$Path) {
    $fs = [IO.File]::OpenRead($Path); $br = New-Object IO.BinaryReader($fs)
    $fs.Seek(0x3C, 'Begin') | Out-Null; $pe = $br.ReadInt32()
    $fs.Seek($pe + 4, 'Begin') | Out-Null; $m = $br.ReadUInt16(); $br.Close()
    switch ($m) { 0x8664 { 'x64' } 0xAA64 { 'ARM64' } 0x14C { 'x86' } default { '0x{0:X}' -f $m } }
}

# 1. Install the x64 platform package. npm reports failure through its exit code only.
New-Item -ItemType Directory -Force $t | Out-Null
npm install --prefix $t --no-audit --no-fund --cpu x64 --os win32 --force "opencode-windows-x64@$ver"
if ($LASTEXITCODE -ne 0) { throw "npm install failed, exit code $LASTEXITCODE" }

# 2. Validate before touching the install: the file exists and is an x64 image.
$x64 = "$t\node_modules\opencode-windows-x64\bin\opencode.exe"
if (-not (Test-Path $x64)) { throw "npm produced no binary at $x64; the --cpu and --os overrides are required on ARM64" }
if ((Get-PeMachine $x64) -ne 'x64') { throw "installed file is not an x64 image" }

# 3. Back up by copying, so the original is intact whatever happens next.
Get-Process opencode -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue
$bak = "$pkg\bin\arm64-$ver-backup"
New-Item -ItemType Directory -Force $bak | Out-Null
Copy-Item "$pkg\bin\opencode.exe" "$bak\opencode.exe" -Force

# 4. Replace the binary and confirm it matches the package and answers with the same version.
Copy-Item $x64 "$pkg\bin\opencode.exe" -Force
if ((Get-FileHash "$pkg\bin\opencode.exe").Hash -ne (Get-FileHash $x64).Hash) {
    throw "opencode.exe does not match the x64 package; restore from $bak"
}
$got = & "$env:APPDATA\npm\opencode.cmd" --version
if ($got -ne $ver) { throw "opencode --version answered '$got', expected '$ver'; restore from $bak" }
Remove-Item $t -Recurse -Force
```

If the block throws at step 1 or 2, the install is untouched. If it throws at step 3 or 4, the
backup folder holds the original and "Revert" puts it back. The block ran unchanged against 1.18.30
on 2026-09-12 and against 1.18.31 on 2026-09-15.

`opencode-windows-x64-baseline` is the build for CPUs without AVX2. Windows 11's emulator provides
AVX2 on this build, and the regular x64 package ran without complaint, so the baseline package was
not needed.

### 2. Turn off self-update

When a newer release exists, starting the TUI upgrades OpenCode through npm, which installs the
ARM64 package again and undoes step 1. Either of these stops it; set one before the next launch.

In `~\.config\opencode\opencode.jsonc`, at the top level:

```jsonc
"autoupdate": false
```

Or for the user environment:

```powershell
[Environment]::SetEnvironmentVariable('OPENCODE_DISABLE_AUTOUPDATE', '1', 'User')
```

Both switches come from the 1.18.31 binary's own update check, which returns early when the global
config has `autoupdate === false` or `OPENCODE_DISABLE_AUTOUPDATE` is set. Neither was tested
against a pending upgrade. Updates then become manual: install the new version, rerun step 1, and
run the TUI check under "Verify".

## Verify

```powershell
opencode --version    # 1.18.31
opencode              # the TUI opens instead of printing the FFI error
```

Launching the TUI is the only reliable check, because only the TUI is sure to start the renderer.
To check it from a script, give it its own console window, since a TUI needs one. This function
keeps only stderr and the exit code, and turns off self-update for the test run so a check cannot
replace the install:

```powershell
function Test-OpenCodeTui([string]$Exe, [int]$WaitSec = 15) {
    # The TUI needs a real console, so it runs in its own window and only its stderr and exit
    # code are kept. Auto-update is off so the test cannot replace the install.
    $tmp = (New-Item -ItemType Directory -Force "$env:TEMP\oc-tui-test").FullName
    $cmd = "$tmp\run.cmd"; $err = "$tmp\stderr.txt"; $rc = "$tmp\exitcode.txt"
    Remove-Item $err, $rc -ErrorAction SilentlyContinue
    Set-Content $cmd "@set OPENCODE_DISABLE_AUTOUPDATE=1`r`n@`"$Exe`" 2>`"$err`"`r`n@>`"$rc`" echo %errorlevel%" -Encoding ascii
    $p = Start-Process cmd.exe -ArgumentList '/c', "`"$cmd`"" -WorkingDirectory $tmp -WindowStyle Minimized -PassThru
    if ($p.WaitForExit($WaitSec * 1000)) {
        "FAIL exit $((Get-Content $rc -Raw).Trim()): $((Get-Content $err | Where-Object { $_ }) -join ' ')"
    } else {
        taskkill.exe /T /F /PID $p.Id | Out-Null
        "PASS: TUI still running after $WaitSec s"
    }
}
Test-OpenCodeTui "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe"
# PASS: TUI still running after 15 s
```

The `>"file" echo %errorlevel%` order in the batch file matters. Written as
`echo %errorlevel%> file`, an exit code of 0 or 1 turns into a handle redirection and the file
stays empty.

## Gotchas

OpenCode's self-update is the likeliest way to lose the fix. On 2026-09-15, launching the swapped
x64 1.18.30 TUI logged `message=upgraded method=npm target=1.18.31`, and afterwards
`bin\opencode.exe` was the ARM64 build and the `arm64-1.18.30-backup` folder was gone with the old
package directory. npm had reported the package's `postinstall` script as not allowed, and the ARM64
binary was in place anyway. Step 2 of the fix exists because of this.

`npm update -g`, `npm i -g opencode-ai`, or any other reinstall of the package also undoes step 1.
After any update, run the TUI check before trusting it.

Do not pin an older release to stay native. 1.18.26 through 1.18.29 fail to start the TUI exactly
like 1.18.30 and 1.18.31.

Once upstream ships an ARM64 build compiled with Bun 1.4 or later, the ARM64 package should work
again and the swap can be retired. `BUN_BE_BUN=1` makes a Bun-compiled binary print its embedded Bun
version instead of running, which tells you without installing it:

```powershell
$env:BUN_BE_BUN = '1'; & "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe" --version; Remove-Item Env:BUN_BE_BUN
# 1.3.14
```

Then confirm with `Test-OpenCodeTui` on the new ARM64 binary before adopting it.

The global install lives under `%APPDATA%\npm`. `Get-Command opencode` shows the shims there
(`opencode`, `opencode.cmd`, `opencode.ps1`); the real binary is
`%APPDATA%\npm\node_modules\opencode-ai\bin\opencode.exe`, which is what `opencode.cmd` runs.

## Revert

Copies the original back, confirms it landed, and only then removes the backup. `Move-Item` errors
are non-terminating by default, so a locked destination would otherwise let the block fall through
to the delete and lose the only copy.

```powershell
$ErrorActionPreference = 'Stop'
$pkg = "$env:APPDATA\npm\node_modules\opencode-ai"
$bak = Get-ChildItem "$pkg\bin\arm64-*-backup" -Directory | Select-Object -First 1
if (-not $bak) { throw "no backup folder under $pkg\bin" }
Get-Process opencode -ErrorAction SilentlyContinue | Stop-Process -Force -ErrorAction SilentlyContinue

Copy-Item "$($bak.FullName)\opencode.exe" "$pkg\bin\opencode.exe" -Force
if ((Get-FileHash "$pkg\bin\opencode.exe").Hash -ne (Get-FileHash "$($bak.FullName)\opencode.exe").Hash) {
    throw "opencode.exe was not restored; backup left in place"
}
Remove-Item $bak.FullName -Recurse
```

Reverting puts back a build whose TUI does not start. If the backup folder is gone, the same file is
on npm as `opencode-windows-arm64@<version>`; the 1.18.30 backup and a fresh npm copy had the same
hash. Remove the self-update switch from step 2 as well if you want updates back.

## Diagnosis

How the cause was pinned down, with the commands, so the same steps work for the next release.

### Find the binary and check its architecture

```powershell
Get-Command opencode -All | Select-Object Name, CommandType, Source
Get-Content "$env:APPDATA\npm\opencode.cmd"      # names node_modules\opencode-ai\bin\opencode.exe
```

The `opencode-ai` package lists one optional dependency per platform. On this machine npm resolved
`opencode-windows-arm64`, and the package's `postinstall.mjs` copied its binary into `bin\`. Reading
the PE header with the `Get-PeMachine` helper from the fix block confirmed a native ARM64 file:

```powershell
Get-PeMachine "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe"   # ARM64
```

### Reproduce with captured output

```powershell
$oc = "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe"
& $oc auth list
& $oc models
& $oc --log-level DEBUG --print-logs run "reply with the word pong"
```

On 2026-09-12 all three printed the OpenTUI error and exited 1. `--print-logs` produced nothing,
which placed the failure before OpenCode's logging starts. No new file appeared under
`~\.local\share\opencode\log`; the only log there was from 2026-09-01. It showed a normal boot, but
not which build or which command, so it does not show that a native TUI ever worked here.

On 2026-09-15 the first two commands succeeded with the same binary, so this reproduction is not
dependable. Use `Test-OpenCodeTui` from "Verify" instead.

### Check the event log

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'Application'; StartTime = (Get-Date).AddDays(-7) } |
    Where-Object { $_.Message -match 'opencode|bun\.exe' } | Select-Object TimeCreated, ProviderName
```

Nothing. The error is a controlled exit, and the message itself names the cause.

### Find the upstream issue

The error text is distinctive enough to search the tracker directly:

```powershell
gh issue list -R anomalyco/opencode --state all --search "TinyCC" --limit 10
```

Issues describing this error on Windows ARM64: #19130 (March 2026), #20767 (April), #38520 (July)
and #45875 (28 August 2026). All four were still open on 2026-09-15. #45875 is the clearest
write-up. It names two gaps:

- Stable Bun ships `bun:ffi` compiled out on `windows-aarch64`, so OpenTUI cannot load its DLL.
  Fixed by building with Bun 1.4.0 or later, which has an engine-native FFI. PR #44946 moves the
  embedded Bun to 1.4.2; on 2026-09-15 it was open with passing checks and unmerged, and the
  repository's `packageManager` was still `bun@1.3.14`.
- `bun-pty` 0.4.8 ships only an x64 `rust_pty.dll`, which an ARM64 process cannot load, so shell
  sessions through `#pty` fail. A Windows ARM64 build is proposed in `sursaone/bun-pty#46`, still
  open on 2026-09-15; the newest `bun-pty` release was 0.4.10 from June.

Related and also open on 2026-09-15: PR #44665 ("fix(install): support windows-arm64"), issue
#44664 (the installer rejects `windows-arm64`), and issues #48518 and #49059, which are about the
Windows ARM64 desktop installer rather than the CLI. PR #45844 ("use x64 build on Windows ARM64
(native build lacks bun:ffi)") was closed unmerged.

### Test every ARM64 release

The first version of this page bisected releases with `auth list` and concluded that only 1.18.30
was broken. That was wrong: `auth list` passes on every release below while the TUI fails on all of
them. The re-check on 2026-09-15 installed each ARM64 platform package to a scratch prefix and ran
its binary three ways:

```powershell
foreach ($v in '1.18.26', '1.18.27', '1.18.28', '1.18.29', '1.18.30', '1.18.31') {
    $t = "$env:TEMP\oc-$v"; New-Item -ItemType Directory -Force $t | Out-Null
    npm install --prefix $t --no-audit --no-fund "opencode-windows-arm64@$v" | Out-Null
    $exe = "$t\node_modules\opencode-windows-arm64\bin\opencode.exe"
    $env:BUN_BE_BUN = '1'; $bun = & $exe --version; Remove-Item Env:BUN_BE_BUN
    & $exe auth list *> $null; $auth = $LASTEXITCODE
    "$v bun $bun | auth list exit $auth | TUI " + (Test-OpenCodeTui $exe)
}
```

| `opencode-windows-arm64` | Published (UTC) | Embedded Bun | `auth list` | TUI |
|---|---|---|---|---|
| 1.18.26 | 2026-09-01 21:50 | 1.3.14 | exit 0 | FFI error, exit 1 |
| 1.18.27 | 2026-09-02 21:38 | 1.3.14 | exit 0 | FFI error, exit 1 |
| 1.18.28 | 2026-09-04 15:36 | 1.3.14 | exit 0 | FFI error, exit 1 |
| 1.18.29 | 2026-09-04 23:45 | 1.3.14 | exit 0 | FFI error, exit 1 |
| 1.18.30 | 2026-09-09 03:32 | 1.3.14 | exit 0 | FFI error, exit 1 |
| 1.18.31 | 2026-09-14 17:48 | 1.3.14 | exit 0 | FFI error, exit 1 |
| x64 1.18.31, emulated | | | | still running at 15 s |

### Confirm with the x64 build

The first attempt to install `opencode-windows-x64` produced no binary and no error, because npm
skips a package whose declared `cpu` does not match the host and `--silent` hid the warning. With
`--cpu x64 --os win32 --force` it installed, and the binary passed `--version`, `auth list` and
`models` under emulation, and its TUI stays up under `Test-OpenCodeTui`. That was enough to apply
the swap.

## Related

- `GROK-CLI-WINDOWS-ARM64.md`: the same shape of problem in xAI's Grok CLI, fixed the same way.
- Upstream: `https://github.com/anomalyco/opencode/issues/45875` and
  `https://github.com/anomalyco/opencode/pull/44946`.
