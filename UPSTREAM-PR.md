# Upstream PR: Vraxo/WuffPad `master` → CrazyPokemonDev/WuffPad `master`

## Submit it here

Open this link and press **Create pull request** (it is prefilled by GitHub, nothing else to change):

**https://github.com/CrazyPokemonDev/WuffPad/compare/master...Vraxo:WuffPad:master?expand=1**

Why this link and not an API call: the token this session runs with is a fine-grained
GitHub App installation on the **Vraxo** and **ProfoL** accounts only. It can read
`CrazyPokemonDev/WuffPad` (`pull: true, push: false`) but cannot create a pull request
there — `gh pr create --repo CrazyPokemonDev/WuffPad` returns
`Resource not accessible by integration (createPullRequest)`. Opening the PR needs a
click from an account GitHub recognises as able to comment on that repo (i.e. you).

## Sanity checks already done

* `CrazyPokemonDev/WuffPad` is **not archived**, issues enabled, default branch `master`.
* The fork is a genuine fork: `parent = CrazyPokemonDev/WuffPad`, own default branch `master`.
* Compare `CrazyPokemonDev:master...Vraxo:master` → merge base `94cf3e9`,
  **ahead 25, behind 0**, 24 files. Upstream `master` is exactly `94cf3e9`, so the PR
  contains only this work — no fork-only commits, no upstream drift.
* No `vraxo` references anywhere in the tracked tree (checked with `git grep`), so the
  patch is repo-agnostic. Nothing in it assumes the fork.
* No stale PRs from the fork are open upstream (only the two dormant Dependabot ones).

## Title

```
Make WuffPad start again: ship its helper files instead of downloading them
```

## Body

```markdown
## The problem

WuffPad no longer starts. It shows *"Cannot start application — the application is
missing required files"*, and the cause is in `MainWindow()`:

```csharp
try { FetchNewestFiles(); } catch { /* swallowed */ }
```

`FetchNewestFiles()` downloaded `versions.txt`, `language.xml`, `descriptions.dict`,
`standardKeys.db` and `emojis.txt` from `http://crazypokemondev.de/WuffPad/Resources/`.
That host — and `185.249.197.95` for uploads, `#closedlist` and feedback — was taken
offline, none of those files were ever in this repository, and the empty `catch` let the
window come up half-initialised. Upstream effectively reads: discontinued because the
server-side configuration files were lost.

This PR makes the editor work without the server and keeps a build available.

## What changed

**Helper files are embedded in the executable** (`Core/BuiltinResources.cs`)
They are extracted into `Resources/` on first start (`Core/AppData.cs`), or into
`%LOCALAPPDATA%\WuffPad\Resources` when the install folder is read-only. Missing, locked
or corrupt files fall back to the embedded copy, so a start-up can no longer fail on it.
The files are regenerated from the upstream langfiles of GreyWolfDev/Werewolf by the new
`tools/make-builtin-resources.py`: 655 description entries built from the in-file
`{0} - player name` comments, 655 keys, 319 emoji buttons.

**Updating is optional and never blocks** (`Core/ResourceUpdater.cs`)
Only runs when `UpdateUrl` (`App.config`) or `WUFFPAD_UPDATE_URL` is set. It refreshes in
the background with a 15 s timeout and only overwrites files the server actually answered
for — so the original "pull the newest files" behaviour is preserved for anyone who can
host them, without being a start-up requirement.

**Dead endpoints**
* `Info → #closedlist` / `#underdev` read the public `https://t.me/s/<channel>` pages
  instead of the dead PHP scripts (`Core/TelegramChannelFeed.cs`).
* Feedback opens a prefilled GitHub issue instead of the dead PHP mail script.
* Upload and Login now say the server is gone instead of silently failing. Note:
  uploading cannot be restored — there is no backend left to accept langfiles.
* The unused `Telegram.Bot` dependency is dropped.

**Bugs found on the way**
* Opening a `.xml` from Explorer constructed a throwaway `new MainWindow()` and left a
  second, empty window behind.
* The emoji grid was initialised twice; it is now rebuilt idempotently.
* The key list gained an empty entry from the file's trailing newline, so the save-time
  "missing string" check asked about a blank key.

**Toolchain**
Retargeted from .NET Framework 4.5 to 4.8. 4.5 is long EOL and its targeting pack is no
longer obtainable, so the solution could not be built at all. 4.8 ships with Windows 10
1903+ and Windows 11.

## Verification

`WuffPad.exe --selftest` (`Core/SelfTest.cs`) runs the start-up data path headless,
including a load → edit → save round trip, and writes a report to
`%TEMP%\wuffpad-selftest.txt`.

`.github/workflows/build.yml` builds with MSBuild, then — on a copy with **no**
`Resources/` folder — runs the self-test twice and opens the real window:

```
WuffPad opened its window: 'WuffPad'
Data folder after start-up: descriptions.dict, emojis.txt, language.xml, standardKeys.db
Result: PASS
second run: Result: PASS
```

`tools/smoke-test.ps1` reproduces that locally. `release.yml` attaches a `WuffPad.zip`
to a `v*` tag; a current build is at
https://github.com/Vraxo/WuffPad/releases/tag/v1.1.0 (432 KB, exe + four DLLs; the
`Resources` folder is rebuilt on first start).

## Still working

Opening, editing, adding and deleting strings and values, the emoji keyboard, find,
language properties, the experimental script window, and the save-time validation that
made WuffPad more than a text editor: placeholder, gif length, missing key and duplicate
key checks.

## Licensing note for the maintainer

This repository has no license file (the assembly metadata says "Open source"). Three of
the added data files are derived from GreyWolfDev/Werewolf, which is **GPL-3.0**:
`Resources/language.xml` (verbatim copy of its `English.xml`), `Resources/descriptions.dict`
and `Resources/standardKeys.db` (both generated from `English.xml`; the script documents
this). The emoji list is generated from its `Emoji.xml`. If WuffPad should be distributed
under a different license, those files are the ones to revisit — `language.xml` can simply
be regenerated from any langfile the maintainer prefers.
```

## If you would rather do it from the command line

From a clone with your own credentials (the fork is on your account, so `gh pr create`
works with your login):

```
gh pr create \
  --repo CrazyPokemonDev/WuffPad \
  --base master \
  --head Vraxo:master \
  --title "Make WuffPad start again: ship its helper files instead of downloading them" \
  --body-file upstream-pr-body.md
```

Expect the upstream to be dormant (last push 2022, one stale Dependabot PR). If it is
never merged, `v1.1.0` on the fork remains the working download.
