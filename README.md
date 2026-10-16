# ARM64-fixes

Tested workarounds for developer CLIs that fail on Windows on ARM laptops, with the evidence that
found each cause and the commands to check whether a fix is still needed.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![gitleaks](https://img.shields.io/github/actions/workflow/status/ookla-ariel-ride/ARM64-fixes/gitleaks.yml?label=gitleaks)](https://github.com/ookla-ariel-ride/ARM64-fixes/actions/workflows/gitleaks.yml)

## About

Snapdragon X laptops run Windows 11 on ARM64. Most developer tools now ship a native ARM64 build,
and some of those builds are broken in ways the x64 build is not: a runtime compiled without a
feature on that architecture, a native library shipped for x64 only, a crash in a dependency that
nobody tested on ARM. The symptom is usually one terse error, the tool's issue tracker may or may
not know about it, and the x64 build under Windows' emulation often works fine.

Each page here starts from the exact error text, says whether the current release still needs a
workaround, and gives a PowerShell block that applies the fix without leaving the install in a worse
state than it found it. The diagnosis that pinned down the cause stays on the page, with the
commands, so anyone can rerun it against the next release instead of taking the page's word for it.

The pages describe the current state of each tool as of their "Last reviewed" line. They are not
changelogs; the git history holds the older fixes.

## Where each fix stands

| Tool | What you see | Where it stands | Last tested |
|---|---|---|---|
| [Grok Build CLI](GROK-CLI-WINDOWS-ARM64.md) | `thread 'main' has overflowed its stack` on `grok login` or any command that goes online | Fixed upstream. Only the ARM64 builds 1.0.30 and 1.0.31 crash; stable 1.0.52 and alpha 1.0.60 run natively. A crashing build cannot update itself, so rerun xAI's installer. | 2026-10-15 |
| [OpenCode](OPENCODE-WINDOWS-ARM64.md) | `Failed to initialize OpenTUI render library: bun:ffi dlopen() is not available in this build (TinyCC is disabled)` | Fixed in OpenCode v2 (`@opencode/cli`), which runs natively. Every ARM64 release of the 1.x package `opencode-ai` from 1.18.26 through 1.19.0 fails; to stay on 1.x, run the x64 build under emulation and turn off self-update. | 2026-10-15 |

## Using a fix

You need Windows 11 on ARM64 and PowerShell 7. The OpenCode page needs npm, because both OpenCode
packages install with it. The Grok page needs nothing beyond what Windows ships.

1. Find your error message in the table above and open that page.
2. Read the "Where it stands" line on the page. If the current release works natively, the page
   tells you how to get onto it and you can stop there.
3. Otherwise follow the Fix section. Where a page replaces a binary, the block downloads to a
   scratch folder, validates what it got, copies the original to a backup folder, and only then
   replaces anything. It stops at the first failure, so an install is either untouched or has a
   backup beside it.
4. Run the commands under Verify. For a TUI, the only reliable check is starting the TUI, and the
   pages include a function that does that from a script.
5. If you want the original back, run the Revert block.

Updating a tool can put a broken ARM64 build back, and OpenCode 1.x does it on its own unless its
self-update is off. After any update, run the Verify commands again before relying on it.

## How a page is laid out

Every page has the same sections, in this order:

| Section | What it holds |
|---|---|
| Symptom | The exact error text and where it appears, plus what the event log and the tool's own log show |
| Cause | What is wrong and why, with a table of which builds are affected |
| Fix | The PowerShell block, with what each step checks and what happens if it throws |
| Verify | Commands that prove the fix took, and the ones that cannot (a passing `auth list` on a broken OpenCode, for instance) |
| Gotchas | Self-update, channel switches, signing changes, anything that undoes the fix or misleads a test |
| Revert | How to put the original back, where the fix changed files |
| Diagnosis | The commands that found the cause, in the order they were run |
| Related | The other page and the upstream issues, pull requests and installer scripts |

The Diagnosis sections are the reusable part. The same handful of techniques found both causes
and will probably find the next one:

- Read the PE header's machine field to learn what architecture a binary really is, instead of
  trusting a file name or an installer's claim. Both pages carry a six-line `Get-PeMachine`.
- Probe a command through `Start-Process` with redirected output and a bounded wait, so a crash,
  a hang and a prompt that waits for a person all give a readable result.
- Pull a Bun-compiled binary's embedded Bun version with `BUN_BE_BUN=1` before running it.
- Query the Application event log for `0xC00000FD` and other fault codes filtered on the process
  name, and read the tool's own trace log (`GROK_LOG_FILE` with `RUST_LOG`, or OpenCode's
  `--print-logs`) to see the last line before it died.
- Install each candidate build into its own scratch folder, by exact version, and test it there.
  npm's `--prefix` does this for OpenCode; Grok's builds are downloadable by version from x.ai.
- Rule out the cheap explanations first: an unreachable proxy, a too-small stack, a leftover
  environment variable.

## Validating a page again

The pages invite re-checking, and both tools have traps for anyone who does it carelessly.

- Work in scratch folders only. Never test a fix block against the real install first.
- Never launch `grok` with no arguments to test a build. The TUI self-updates the real install
  within about a minute, from any folder, with no opt-out.
- Set `OPENCODE_DISABLE_AUTOUPDATE=1` for every OpenCode launch, installed or scratch.
- A TUI only counts as working if it stays up in a real console window. Redirected output is not
  a console, and the non-TUI commands pass on builds whose TUI fails.
- Check the real install's version and architecture at the end of the session as well as at the
  start.

## Test machine

All results come from a Samsung Galaxy Book4 Edge (Snapdragon X Elite) on Windows 11 ARM64 Insider
builds 29648 to 29683 with PowerShell 7.6.5. The OpenCode 1.18.31 failure was also seen on a second
ARM64 laptop. Other hardware and Windows builds may behave differently, and the pages record the
versions each result applies to.

## Contributing

Reports from other machines are the most useful thing you can add. If a fix stops working, or a
release changes what a page says, open an issue with the tool version, your Windows build from
`winver`, and the full output of the page's Verify commands.

A new page should cover a tool whose native ARM64 build fails on Windows and whose x64 build, or a
later native build, works. Use the section layout above, date the "Last reviewed" line, and put
the evidence for every claim on the page. A fix block has to validate what it downloaded, back up
before it replaces, and stop at the first failure. Prose is written for a person who has the error
in front of them, in plain sentences, without hedging or sales language.

Changes go through a branch and a pull request. The `gitleaks` workflow runs on every push, and
the same scan runs locally as a pre-commit hook once you enable it in your clone:

```powershell
winget install Gitleaks.Gitleaks
git config core.hooksPath .githooks
```

The `.gitignore` keeps coding-agent state, local secrets, logs, downloaded executables and backup
folders out of the repository, so running the fix blocks from a clone leaves nothing to commit by
accident.

## Related

- [`projectbrief.md`](projectbrief.md): why this repository exists, what it covers, the standards
  each page is held to, and what is still open.
- OpenCode upstream: [anomalyco/opencode#45875](https://github.com/anomalyco/opencode/issues/45875)
  and [anomalyco/opencode#47446](https://github.com/anomalyco/opencode/pull/47446).
- Grok Build CLI has no public tracker; the installer at `https://x.ai/cli/install.ps1` is the
  reference for how the tool is laid out.

## License

MIT. See [LICENSE](LICENSE).
