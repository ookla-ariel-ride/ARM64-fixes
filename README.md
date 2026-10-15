# ARM64-fixes

Tested workarounds for developer CLIs that fail on Windows on ARM laptops (Snapdragon X). Each page
starts from the exact error message, says whether the current release still needs a workaround, and
gives a PowerShell block for the fix. The evidence is on the same page, so it can be rechecked after
the tool updates.

## Fixes

| Tool | What you see | Where it stands | Last tested |
|---|---|---|---|
| [Grok Build CLI](GROK-CLI-WINDOWS-ARM64.md) | `thread 'main' has overflowed its stack` on `grok login` or any command that goes online | Fixed upstream. Only the ARM64 builds 1.0.30 and 1.0.31 crash; stable 1.0.52 and alpha 1.0.60 run natively. A crashing build cannot update itself, so rerun xAI's installer. | 2026-10-15 |
| [OpenCode](OPENCODE-WINDOWS-ARM64.md) | `Failed to initialize OpenTUI render library: bun:ffi dlopen() is not available in this build (TinyCC is disabled)` | Fixed in OpenCode v2 (`@opencode/cli`), which runs natively. Every ARM64 release of the 1.x package `opencode-ai` through 1.19.0 still fails; to stay on 1.x, run the x64 build under emulation and turn off self-update. | 2026-10-15 |

## Using a fix

You need Windows 11 on ARM64 and PowerShell 7. The OpenCode page needs npm, because both OpenCode
packages install with it.

1. Find your error message in the table and open that page.
2. Follow the Fix section. Where a page swaps a binary, the block validates the download and backs
   up the original before it replaces anything, and it stops at the first failure.
3. Run the commands under Verify.
4. If you want the original back, run the Revert block.

Updating a tool can put a broken ARM64 build back, and OpenCode 1.x does it on its own unless
self-update is off. After any update, run the Verify commands again before relying on it.

Every page has a Symptom, Cause, Fix, Verify, Gotchas and Diagnosis section, and a Revert section
where the fix changes files. The Diagnosis section lists the commands that found the cause, which you
can reuse on the next tool that fails in a similar way.

## Test machine

All results come from a Samsung Galaxy Book4 Edge (Snapdragon X Elite) on Windows 11 ARM64 Insider
builds 29648 to 29683 with PowerShell 7.6.5. The OpenCode 1.18.31 failure was also seen on a second
ARM64 laptop. Other hardware and Windows builds may behave differently, and the pages record the
versions each result applies to.

## Reporting results

If a fix stops working, or a release changes what a page says, open an issue. Include the tool
version, your Windows build from `winver`, and the full output of the page's Verify commands.

## Secret scanning

Commits are scanned with [gitleaks](https://github.com/gitleaks/gitleaks), locally by a pre-commit
hook and on GitHub by the `gitleaks` workflow. To turn on the hook in your clone:

```powershell
winget install Gitleaks.Gitleaks
git config core.hooksPath .githooks
```

## License

MIT. See [LICENSE](LICENSE).
