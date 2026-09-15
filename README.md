# ARM64-fixes

Tested workarounds for developer CLIs that fail on Windows on ARM laptops (Snapdragon X). Each page
starts from the exact error message and gives a PowerShell block that applies the fix. The evidence
for each fix is on the same page, so it can be rechecked after the tool updates.

## Fixes

| Tool | What you see | Fix | Last tested |
|---|---|---|---|
| [Grok Build CLI](GROK-CLI-WINDOWS-ARM64.md) | `thread 'main' has overflowed its stack` on `grok login` or any command that goes online | On stable (1.0.30), run the x64 build under emulation. The native ARM64 build of alpha 1.0.32 works. | 2026-09-15 |
| [OpenCode](OPENCODE-WINDOWS-ARM64.md) | `Failed to initialize OpenTUI render library: bun:ffi dlopen() is not available in this build (TinyCC is disabled)` | Run the x64 build under emulation and turn off OpenCode's self-update. No native ARM64 release from 1.18.26 to 1.18.31 starts the TUI. | 2026-09-15 |

## Using a fix

You need Windows 11 on ARM64 and PowerShell 7. The Grok fix uses the `curl.exe` that ships with
Windows; the OpenCode fix needs npm, because OpenCode is installed with `npm i -g opencode-ai`.

1. Find your error message in the table and open that page.
2. Run the page's Fix block in PowerShell 7. It validates the download and backs up the original
   binary before it replaces anything, and it stops at the first failure.
3. Run the commands under Verify.
4. If you want the original back, run the Revert block.

Updating the tool usually puts the broken ARM64 build back. After any update, run the Verify
commands again before relying on it.

Every page has the same sections: Symptom, Cause, Fix, Verify, Gotchas, Revert and Diagnosis. The
Diagnosis section lists the commands that found the cause, which you can reuse on the next tool
that fails in a similar way.

## Test machine

All results come from a Samsung Galaxy Book4 Edge (Snapdragon X Elite) on Windows 11 ARM64 Insider
build 29667 with PowerShell 7.6.5. The OpenCode 1.18.31 failure was also seen on a second ARM64
laptop. Other hardware and Windows builds may behave differently, and the pages record the versions
each result applies to.

## Reporting results

If a fix stops working, or a new release makes the native ARM64 build work, open an issue. Include
the tool version, your Windows build from `winver`, and the full output of the page's Verify commands.

## Secret scanning

Commits are scanned with [gitleaks](https://github.com/gitleaks/gitleaks), locally by a pre-commit
hook and on GitHub by the `gitleaks` workflow. To turn on the hook in your clone:

```powershell
winget install Gitleaks.Gitleaks
git config core.hooksPath .githooks
```

## License

MIT. See [LICENSE](LICENSE).
