# OpenCode on Windows ARM64

How to get OpenCode working on a Snapdragon X laptop when the native ARM64 build refuses to start
its TUI with a `bun:ffi` error. OpenCode now ships as two npm packages. The 1.x line, `opencode-ai`,
is still built with a Bun that has no FFI on Windows ARM64, and every ARM64 release of it through
1.19.0 fails. The 2.x line, `@opencode/cli`, is built with Bun 1.4.2 and runs natively. So the fix
is to install v2. If you have to stay on 1.x, the second option below swaps in the x64 build and
turns off self-update. This page has the fix first, then the evidence.

Last reviewed: 2026-10-15, `opencode-ai` 1.19.0 and `@opencode/cli` 2.0.29, Samsung Galaxy Book4
Edge (Snapdragon X Elite), Windows 11 ARM64 Insider build 29683, PowerShell 7.6.5, Node 26.7.0.
The 1.18.31 failure was also seen on a second ARM64 laptop.

## Symptom

Starting the TUI of a 1.x install, plain `opencode`, exits 1 at once:

```text
Error: Unexpected error

Failed to initialize OpenTUI render library: bun:ffi dlopen() is not available in this build (TinyCC is disabled)
```

Commands that do not draw a screen are not a reliable signal. `auth list`, `models` and `run` have
failed with the same message on 1.18.30 in one session and succeeded with the byte-identical binary
in another, while the TUI failed both times. The difference is unexplained. `opencode --version`
always works.

There are no crash events in the Windows Application log, because this is a clean error exit rather
than a native fault. OpenCode's own log under `~\.local\share\opencode\log` gets nothing from the
failing run, since the failure happens before logging starts.

## Cause

OpenCode is compiled with Bun. Its terminal renderer, OpenTUI, loads a native DLL through
`bun:ffi`. Bun 1.3 compiles `bun:ffi` out on `windows-aarch64` because TinyCC, which it used for
FFI, has no ARM64 backend. An OpenCode ARM64 binary built with such a Bun cannot start its renderer.
Bun 1.4.0 (2026-08-20) replaced TinyCC with an engine-native FFI.

Every `opencode-ai` ARM64 release from 1.18.26 through 1.19.0, plus the nightly of 2026-10-14,
embeds Bun 1.3.14 and fails. The 1.x release tags still pin `bun@1.3.14`. The repository's
`dev` branch moved its publish workflow to Bun 1.4.2 in PR #47446 on 2026-09-05, and that branch is
what `@opencode/cli` is built from; its first release, 2.0.0, was published on 2026-09-11. The ARM64
build of 2.0.29 embeds Bun 1.4.2 and starts its TUI.

The upstream issues (#19130, #20767, #38520 and #45875) were all closed in October 2026. The
maintainer's closing note says the TUI "starts natively on Windows ARM64 without a workaround" since
#47446, which is true of v2 and not of 1.x; a user reported 1.19.0 still failing on 2026-10-11, and
the table under "Diagnosis" agrees.

## Fix

### Option 1: install OpenCode v2

Both packages install a command called `opencode`, and npm refuses to overwrite the existing shim
(`EEXIST: file already exists`), so remove 1.x first:

```powershell
npm uninstall -g opencode-ai
npm install -g @opencode/cli
opencode --version        # opencode v2.0.29
```

That sequence was run in a scratch global prefix (`npm_config_prefix`) on 2026-10-15: the install on
top of 1.x failed with `EEXIST`, the uninstall removed the `opencode-ai` folder, and the v2 install
then produced `opencode` and `opencode2` shims that both answer `opencode v2.0.29`. npm warns that
the package's `postinstall` script is not allowed, and the binary is in place anyway.

v2 also installs the command as `opencode2`. To keep 1.x and try v2 beside it, install v2 into its
own prefix and put that folder at the end of the user PATH, so `opencode` still runs 1.x and
`opencode2` runs v2:

```powershell
$p = "$env:LOCALAPPDATA\opencode2"
npm install --prefix $p @opencode/cli
$dir = "$p\node_modules\.bin"
$user = [Environment]::GetEnvironmentVariable('Path', 'User')
if (($user -split ';') -notcontains $dir) { [Environment]::SetEnvironmentVariable('Path', $user.TrimEnd(';') + ';' + $dir, 'User') }
# new terminal:  opencode2 --version
```

v2 reads the same `~\.config\opencode` folder as 1.x, including `opencode.jsonc`, and registers its
plugins in a new `cli.json` there. Plugins written for 1.x do not load; see "Gotchas".

### Option 2: stay on 1.x with the x64 build

Two steps: swap in the x64 binary, then stop OpenCode from updating itself back to ARM64.

#### 1. Run the x64 build under emulation

Install the x64 platform package into a scratch prefix, validate it, and copy its binary over the
one the npm shim runs. npm refuses to install a package whose `cpu` field says `x64` on an ARM64
host, and does so silently under `--silent`, so the platform override flags are required.

The block stops at the first failure and never touches the install until the package has been
checked. `npm` does not throw on failure in PowerShell, so its exit code is tested by hand. The
backup is a copy, so the original survives anything that goes wrong later. npm packages carry no
Authenticode signature, so the checks here are existence, architecture and a version round trip.

```powershell
$ErrorActionPreference = 'Stop'
$env:OPENCODE_DISABLE_AUTOUPDATE = '1'
$pkg = "$env:APPDATA\npm\node_modules\opencode-ai"
$ver = & "$env:APPDATA\npm\opencode.cmd" --version       # 1.19.0 at time of writing
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
backup folder holds the original and "Revert" puts it back. The block has run unchanged on 1.18.30,
1.18.31 and 1.19.0.

`opencode-windows-x64-baseline` is the build for CPUs without AVX2. Windows 11's emulator provides
AVX2 on this build, and the regular x64 package runs without complaint, so the baseline package is
not needed.

#### 2. Turn off self-update

When a newer release exists, starting the TUI upgrades OpenCode through npm, which installs the
ARM64 package again and undoes step 1. Set both of these before the next launch.

In `~\.config\opencode\opencode.jsonc`, at the top level:

```jsonc
"autoupdate": false
```

And for the user environment:

```powershell
[Environment]::SetEnvironmentVariable('OPENCODE_DISABLE_AUTOUPDATE', '1', 'User')
```

Both switches come from the 1.x binary's own update check, read out of the 1.18.31 executable. When
the TUI starts, the check returns at once if the global config has `autoupdate === false` or
`OPENCODE_DISABLE_AUTOUPDATE` is set. Otherwise it reads the latest tag from GitHub's releases API,
compares it with the running version, and installs it through npm only when the difference is a
patch release. A minor or major bump, such as 1.18.31 to 1.19.0, only shows an "update available"
notice. So the swap on 1.19.0 is undone by the next 1.19.x release, and the switches are what stop
that. They have not been watched blocking a real install: three launches of a scratch x64 1.18.31
on 2026-10-15, with no switch, the environment switch and the config switch, all left the package at
1.18.31, because the pending release was the minor bump to 1.19.0 and the control run could not
upgrade either. Updates then become manual: install the new version, rerun step 1, and run the TUI
check under "Verify".

## Verify

```powershell
opencode --version    # 1.19.0 for 1.x, or: opencode v2.0.29
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
Test-OpenCodeTui "$env:APPDATA\npm\node_modules\opencode-ai\bin\opencode.exe"      # 1.x
Test-OpenCodeTui "$env:APPDATA\npm\node_modules\@opencode\cli\bin\opencode.exe"    # v2
# PASS: TUI still running after 15 s
```

The `>"file" echo %errorlevel%` order in the batch file matters. Written as
`echo %errorlevel%> file`, an exit code of 0 or 1 turns into a handle redirection and the file
stays empty.

A v2 TUI killed this way can leave an `opencode.exe` process behind. Check with
`Get-Process opencode` afterwards and stop it.

To read a binary's embedded Bun without running it, set `BUN_BE_BUN=1`; the binary then prints its
Bun version instead of starting. 1.3.14 fails on ARM64 and 1.4.2 works:

```powershell
$env:BUN_BE_BUN = '1'; & "$env:APPDATA\npm\node_modules\@opencode\cli\bin\opencode.exe" --version; Remove-Item Env:BUN_BE_BUN
# 1.4.2
```

## Gotchas

Plugins written for 1.x do not load in v2. A 1.x plugin is a named export of a function that
returns hooks; v2 wants a default export with an `id` and a `setup` function (`Plugin.define` from
`@opencode/plugin`), and logs `Plugin must export a default definition with an id and an effect or
setup function` for anything else. A package can export both shapes, and tools that install plugins
into `~\.config\opencode` have been updating theirs; check for a newer version of the integration
before rewriting anything.

On 1.x, self-update undoes the swap. Launching the TUI of a swapped install while a newer release
exists logs `message=upgraded method=npm target=<version>`. Afterwards `bin\opencode.exe` is the
ARM64 build again and the `arm64-*-backup` folder is gone along with the old package directory.
This happened twice on the test machine before both switches in step 2 were set. `npm update -g`,
`npm i -g opencode-ai`, or any other reinstall of the package also undoes the swap. After any
update, run the TUI check before trusting it.

Do not pin an older 1.x release to stay native. 1.18.26 through 1.18.35 all fail to start the TUI
exactly like 1.19.0.

The global install lives under `%APPDATA%\npm`. `Get-Command opencode` shows the shims there
(`opencode`, `opencode.cmd`, `opencode.ps1`); the real binary is
`%APPDATA%\npm\node_modules\<package>\bin\opencode.exe`, which is what `opencode.cmd` runs. Each
package's `postinstall.mjs` copies the platform package's binary into that `bin\` folder.

## Revert

Undoes option 2. Copies the original back, confirms it landed, and only then removes the backup.
`Move-Item` errors are non-terminating by default, so a locked destination would otherwise let the
block fall through to the delete and lose the only copy.

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
on npm as `opencode-windows-arm64@<version>`; a backup and a fresh npm copy have the same hash.
Remove the self-update switches from step 2 as well if you want updates back. Undoing option 1 is
`npm uninstall -g @opencode/cli` followed by `npm install -g opencode-ai`.

## Diagnosis

How the cause was pinned down, with the commands, so the same steps work for the next release.

### Find the binary and check its architecture

```powershell
Get-Command opencode -All | Select-Object Name, CommandType, Source
Get-Content "$env:APPDATA\npm\opencode.cmd"      # names node_modules\opencode-ai\bin\opencode.exe
```

Both packages list one optional dependency per platform. On an ARM64 machine npm resolves the
`windows-arm64` one, and the package's `postinstall.mjs` copies its binary into `bin\`. Reading the
PE header with the `Get-PeMachine` helper from the fix block confirms a native ARM64 file:

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

When these fail, all three print the OpenTUI error and exit 1, `--print-logs` produces nothing, and
no file appears under `~\.local\share\opencode\log`. That places the failure before OpenCode's
logging starts. As the Symptom section says, these commands do not fail every time, so use
`Test-OpenCodeTui` from "Verify" for a dependable reproduction.

### Check the event log

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'Application'; StartTime = (Get-Date).AddDays(-7) } |
    Where-Object { $_.Message -match 'opencode|bun\.exe' } | Select-Object TimeCreated, ProviderName
```

No events turn up. The error is a controlled exit, and the message itself names the cause.

### Find the upstream issue

The error text is distinctive enough to search the tracker directly:

```powershell
gh issue list -R anomalyco/opencode --state all --search "TinyCC" --limit 10
```

Issues describing this error on Windows ARM64: #19130 (March 2026), #20767 (April), #38520 (July)
and #45875 (28 August 2026), all closed in October 2026. #45875 is the clearest write-up and names
two gaps: `bun:ffi` compiled out of Bun 1.3 on `windows-aarch64`, and `bun-pty` 0.4.8 shipping only
an x64 `rust_pty.dll`. The second was closed with a note that OpenCode now uses `bun-pty` 0.4.11; the
shell-tool run under "Test every ARM64 release" is the only check of that made here.

The Bun bump is PR #47446, merged 2026-09-05, which changes the publish workflow's `bun-version`
to 1.4.2. To see which Bun a tag was built with, read `.github/workflows/publish.yml` at that tag,
not the root `package.json`, whose `packageManager` field lags behind.

### Test every ARM64 release

`auth list` cannot find this bug, because it passes on every 1.x release below while the TUI fails
on all of them. Install each ARM64 platform package to a scratch prefix and run its binary three
ways:

```powershell
foreach ($v in '1.18.31', '1.19.0') {
    $t = "$env:TEMP\oc-$v"; New-Item -ItemType Directory -Force $t | Out-Null
    npm install --prefix $t --no-audit --no-fund "opencode-windows-arm64@$v" | Out-Null
    $exe = "$t\node_modules\opencode-windows-arm64\bin\opencode.exe"
    $env:BUN_BE_BUN = '1'; $bun = & $exe --version; Remove-Item Env:BUN_BE_BUN
    & $exe auth list *> $null; $auth = $LASTEXITCODE
    "$v bun $bun | auth list exit $auth | TUI " + (Test-OpenCodeTui $exe)
}
```

| Package | Published (UTC) | Embedded Bun | `auth list` | TUI |
|---|---|---|---|---|
| `opencode-windows-arm64` 1.18.26 | 2026-09-01 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.27 | 2026-09-02 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.28 | 2026-09-04 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.29 | 2026-09-04 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.30 | 2026-09-09 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.31 | 2026-09-14 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.32 | 2026-09-21 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.33 | 2026-09-28 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.34 | 2026-09-30 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.18.35 | 2026-10-06 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 1.19.0 | 2026-10-11 | 1.3.14 | exit 0 | FFI error, exit 1 |
| `opencode-windows-arm64` 0.0.0-dev-202610141901 | 2026-10-14 | 1.3.14 | | FFI error, exit 1 |
| `opencode-windows-x64` 1.19.0, emulated | 2026-10-11 | 1.3.14 | | still running at 15 s |
| `@opencode/cli-windows-arm64` 2.0.29 | 2026-10-14 | 1.4.2 | exit 0 | still running at 20 s |

The v2 binary also answered `models`, and with no credentials configured it ran two `run` prompts
against the free `opencode/deepseek-v4.1-flash-free` model: a plain reply, and one that made it
execute `echo pty-ok-7731` through its shell tool and return the output.

### Confirm with the x64 build

A plain `npm install` of `opencode-windows-x64` on ARM64 produces no binary and no error, because npm
skips a package whose declared `cpu` does not match the host and `--silent` hides the warning. With
`--cpu x64 --os win32 --force` it installs. Under emulation the binary passes `--version`,
`auth list` and `models`, and its TUI stays up under `Test-OpenCodeTui`.

## Related

- [`GROK-CLI-WINDOWS-ARM64.md`](GROK-CLI-WINDOWS-ARM64.md): a different cause with the same shape in
  xAI's Grok CLI, since fixed upstream.
- Upstream: `https://github.com/anomalyco/opencode/issues/45875` and
  `https://github.com/anomalyco/opencode/pull/47446`.
