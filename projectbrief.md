# Project brief

What this repository is for, who it serves, the standard every page is held to, and where things
stand. Read this before changing a page or adding one. The README is the front door for someone
with an error on their screen; this brief is for whoever maintains the pages.

Last reviewed: 2026-10-16.

## Purpose

Developer CLIs ship native Windows ARM64 builds that sometimes do not work, while the same
version's x64 build runs under Windows' emulation without complaint. When that happens, the person
at the keyboard gets one terse error, no guidance, and a tool they cannot use. This repository
holds a page per tool that turns the error into a fix in one sitting, and holds the evidence that
produced the fix so it can be rechecked when the tool updates.

The repository exists because this work was done twice, once per tool, and the second time went
faster for having the first page to copy from. The page format is the product; the two fixes are
the first two uses of it.

## Audience

Two readers, in this order:

- Someone on a Snapdragon X laptop with the error text in front of them. They want to know whether
  the current release still has the problem, and if so, one block to run and one command to
  confirm it worked. They do not want a history lesson.
- Whoever rechecks a page after a release. They want the exact commands that found the cause, the
  traps that produced wrong results last time, and which claims are measured versus inferred.

Both readers are served by the same page, with the fix first and the evidence after.

## Scope

In scope: a tool whose native Windows ARM64 build fails and whose x64 build, or a later native
build, works. The fix is a local change to the install: swapping a binary, switching a channel,
turning off self-update, rerunning an installer. Each page records which versions were tested, on
which Windows build, and what the result was.

Out of scope: patching the tools themselves, carrying binaries in the repository, fixes that need
administrator rights or registry edits, and anything that was not run on a real ARM64 machine. A
claim that has not been reproduced here does not go on a page as a fact.

## What a page is held to

Every page follows the same section order, listed in the README. Beyond the layout, each page has
to meet these conditions:

- It describes the current state, under a dated "Last reviewed" line. When a release changes the
  picture, the page is rewritten, not appended to. Git history is the changelog.
- The fix leads. A reader who stops after the Fix and Verify sections has everything they need.
- Every version named on the page was run on this machine, and the page says what was run. A table
  row is a measurement. A sentence that starts "the maintainer says" is reported, not confirmed.
- A fix block validates before it touches anything: file exists, right architecture, signature or
  hash where one exists, and a round trip such as `--version`. It copies the original to a backup
  folder before replacing it. It stops at the first failure, with `$ErrorActionPreference = 'Stop'`
  and an explicit exit-code check for tools like `npm` and `curl.exe` that do not throw.
- Verify means the thing that actually failed. For a TUI, that is the TUI starting in a real
  console window, not a non-interactive command that happens to pass.
- The Diagnosis section is complete enough to rerun. Commands are given in full, with the output
  that was seen, so the next person can tell "still broken" from "my setup is different".
- Every PowerShell block parses in PowerShell 7 without errors. Blocks that change an install have
  been run, as written, against a scratch copy before they are published, and against the real
  install when that is what the session was for.
- The prose is plain. No dashes as connectors, no bold labels, no "not X but Y", no hedges stacked
  on claims that were measured. If a claim cannot be stated plainly it has not been tested enough.

## How validation is done safely

These rules come from mistakes made during the first two reviews.

- Everything happens in a scratch folder first: npm `--prefix` installs, direct downloads by
  version, copies of the real `bin` folder. The real install changes only when the session's
  purpose is to change it, and its version and architecture are checked again at the end.
- `grok` is never launched with no arguments during testing. The TUI checks for updates on start
  and replaces the real install within a minute, from any folder, including a scratch copy, and
  there is no opt-out. The only safe probes are `--version`, `update --check --json`, `doctor`,
  `-p` and a bounded `login --device-code`.
- Every OpenCode launch runs with `OPENCODE_DISABLE_AUTOUPDATE=1`, installed or scratch. The
  installed copy also has `"autoupdate": false` in its global config.
- A TUI test runs the binary in its own console window through a batch file and keeps only stderr
  and the exit code. In the batch file the redirection goes before the `echo`, as
  `>"file" echo %errorlevel%`, because the other order turns an exit code of 0 or 1 into a handle
  redirection.
- In PowerShell 7.6 a function parameter named `$Args` is always empty. A probe helper with that
  parameter launches the bare TUI, which is the one thing the Grok rule forbids. Parameters are
  named `$Arguments`.
- Downloads and extracted packages are untrusted. Each goes into its own empty folder, and scripts
  that read them live elsewhere.

## Status, 2026-10-16

Grok Build CLI. The stack overflow in the rustls handshake affected the ARM64 builds 1.0.30 and
1.0.31 only. Alpha 1.0.32 was the first build to pass, stable caught up at 1.0.34, and the current
stable 1.0.52 and alpha 1.0.60 both pass. The page's fix is to rerun xAI's installer, which was
checked against a copy of a crashing 1.0.30 install. The x64 and alpha swap blocks that the page
used to carry were removed when stable started working; they are in the history at `e82549c`. xAI
has published nothing about the bug and has no public tracker.

OpenCode. The 1.x package, `opencode-ai`, is built with Bun 1.3.14, which has no `bun:ffi` on
Windows ARM64, and every ARM64 release from 1.18.26 through 1.19.0 fails to start its TUI. The 2.x
package, `@opencode/cli`, is built with Bun 1.4.2 and runs natively; 2.0.29 passed the TUI test,
answered prompts and ran a shell command through its tool. The page leads with installing v2 and
keeps the x64 swap plus self-update switches as the 1.x option. The upstream issues are closed; the
maintainers consider the problem fixed, which is true of v2 and not of 1.x.

The README table carries the same summary with the "Last tested" dates.

## Open items

- OpenCode's self-update switches have not been seen blocking a real install. The update check,
  read out of the 1.18.31 binary, auto-installs only a patch release, and the only pending release
  for any 1.x install is the minor bump to 1.19.0. The test becomes possible when a 1.19.x ships.
- Whether OpenCode v2 has the same two self-update switches has not been checked.
- The `grove*.exe` helpers that the Grok installer adds are ARM64 and untested.
- Only one machine produced these results, plus one confirmation of the OpenCode 1.18.31 failure
  on a second laptop. Reports from other hardware would strengthen every table.
- `docs/session-handoff.md` is an untracked note from the first review session and is out of date.
  It should be deleted or replaced by this brief.

## When to revisit a page

- A new release of the tool, on any channel the page names. Check the channel pointers or npm
  dist-tags, install the new build into scratch, and run the page's Verify commands on it.
- Movement on an upstream issue or pull request the page links to.
- A new Windows Insider build on the test machine. The Grok crash was checked on three builds
  because a dependency crash can be build-sensitive.
- A report from another machine that disagrees with a table row.

The check takes about an hour per tool when nothing has changed and an afternoon when a page needs
rewriting. Both reviews so far were done with a coding agent driving the commands; the pages are
written so that either a person or an agent can repeat them.

## How changes land

One branch per change, cut from `main`, merged through a pull request. The `gitleaks` workflow
runs on every push and the same scan runs as a pre-commit hook in clones that have enabled it.
Commit messages say what was observed, in the past tense, and claim nothing that was not seen in
the session. No tool or model is credited as an author.

## Adding a tool

1. Reproduce the failure on the ARM64 build and confirm the x64 build, or a later native build,
   does not have it. Capture the exact error, the event log entries if any, and the tool's own log.
2. Find the binary, read its architecture from the PE header, and work out how the tool is
   installed and updated, because the update path is usually what undoes a fix.
3. Test every candidate build in scratch and record a table: version, channel, build date, the
   measurement that distinguishes broken from working.
4. Write the fix block to the standard above and run it against a scratch copy of the install,
   then against the real one.
5. Write the page in the section order, with the fix first. Add a row to the README table.
6. Open a pull request. Say in the description what was run and what was not.
