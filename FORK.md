# Fork notes

Personal fork of [AltTab](https://github.com/lwouis/alt-tab-macos) that unlocks the paid
**AltTab Pro** features for personal use. AltTab is GPL-3.0, so modifying my own copy is explicitly
permitted.

> **Audience:** this is a private reference for me and for coding agents working in this repo — how
> the fork is patched, built, authenticated, and kept in sync. It is **not** documentation for other
> users. It's a fork-only file; upstream never sees it, so it won't cause rebase conflicts.

---

## 1. Unlock requirements & patch

### Requirements

Any unlock approach (current or redesigned after an upstream rebase) must satisfy:

1. **Keep it as simple as possible** — smallest patch that works; prefer one choke point over scattered edits.
2. **Do not change current features unless necessary** — no unrelated refactors, UI cleanups, or behavior changes beyond unlocking Pro.
3. **All Pro features work** — gated shortcuts, switcher search/lock, appearance styles, Auto window sizing, etc.

Cosmetic Pro UI leftovers (badges, "Pro activated" button) are acceptable; stripping them is out of scope unless required.

### Current patch

Everything keys off a single source of truth: `LicenseManager.shared.state` (enum `LicenseState`:
`.trial` / `.pro` / `.proExpired` / `.trialExpired`). Force it to `.pro` and the whole paywall
falls away at once. The patch lives in two files, both gated on the same `unlockProForFree` flag:
**`src/pro/license/LicenseManager.swift`** (the unlock) and **`src/vendors/SparkleDelegate.swift`**
(auto-update suppression — see "Keeping the unlock durable" below).

In `LicenseManager.swift`:

1. A flag, defaulted off:
   ```swift
   var unlockProForFree = false
   ```
2. Turned on for the production singleton only (the `static let shared` factory):
   ```swift
   manager.unlockProForFree = true
   ```
3. Short-circuit at the top of `computeState()`:
   ```swift
   func computeState() -> LicenseState {
       if unlockProForFree { return .pro }
       // ...original logic untouched...
   }
   ```

### Why this single spot

`computeState()` is what `initialize()` / `refreshState()` funnel through, so forcing it cascades
everywhere with no per-call-site edits:

- **All gated features unlock** — extra keyboard shortcuts (slots 2–9), switcher type-to-search and
  lock-search, the App Icons / Titles appearance styles, and Auto window sizing.
  (`ProFeature.attemptUse()` returns early when `isProAvailable`; degradable prefs never get
  downgraded because `isProLocked` is false.)
- **The trial/nag popups go silent** — `ProTransitionScheduler.computeNextFireDate()` returns
  `nil` for `.pro`, so none of the Day 1/12/15/21/35 popups are ever scheduled; the event-triggered
  Day-4 tour is suppressed by `ProTransitionManager`'s own `.pro` check. The menu bar also drops
  the "Get Pro" item.
- **No network calls** — `scheduleAsyncRevalidationIfNeeded()` and `revalidateWithServer()` return
  early when `unlockProForFree` is on, so leftover Keychain license entries never trigger a
  validate call (and cannot flip state back to `.trialExpired`).

### Why the flag (instead of hard-coding `.pro`)

The flag defaults to `false`, and only `LicenseManager.shared` flips it on. The unit tests in
`src/pro/license/LicenseManagerTests.swift` build their own `LicenseManager` instances (flag stays
`false`), so they keep exercising the real trial/expiry logic and stay green.

### Keeping the unlock durable (auto-update)

Forcing `.pro` unlocks the features, but Sparkle would happily replace the whole app: its feed
points at upstream's official appcast, the default policy auto-checks (and can auto-install), so an
unlocked build would, within days, "update" itself to the **signed, locked** upstream binary and
re-lock Pro.

The fix lives in **`src/vendors/SparkleDelegate.swift`**: `feedURLString(for:)` returns `nil` when
`unlockProForFree` is on. There is **no `SUFeedURL` in `Info.plist`**, so a `nil` feed leaves Sparkle
with nothing to check — scheduled background checks and auto-install both become silent no-ops, and
no upstream build can ever be found or downloaded. Gating on the same flag keeps unlock and
no-auto-update consistent; a vanilla build (flag off) still updates normally.

Minor, accepted side effects in unlock builds: the manual "Check for updates now…" button errors out
(it would otherwise offer the locked upstream build), and the feedback window's "Report a bug"
preflight falls through its existing 5-second timeout once per session before showing the form.

### Known cosmetic leftovers (intentional — minimal scope)

Functionality is unlocked, but the now-inert Pro UI is **not** stripped: Settings still shows a
green "Pro activated" button and "PRO" badges, and the menu bar still has "My Account". Harmless.
(Removing them would be a separate, larger change across `AppearanceTab.swift`, `ControlsTab.swift`,
`Menubar.swift`, `SettingsWindow.swift`, `ProBadgeView.swift`.)

### One-time gotcha for prior-trial machines

If a machine already ran the 14-day trial to expiry, AltTab downgraded its appearance prefs and
saved the originals into `remembered*` keys. Forcing `.pro` does **not** auto-restore them — just
re-pick App Icons / Titles / Auto once in Settings → Appearance; it sticks afterwards.

---

## 2. Repo layout, git identity & auth

### Remotes (only two)

```
origin    = https://github.com/leo0570/alt-tab-macos.git   (my personal repo — push here)
upstream  = https://github.com/lwouis/alt-tab-macos.git    (the real upstream — pull releases)
```

### Branches

- **`master`** — clean mirror of upstream. No patch on it.
- **`pro-unlock`** — the working branch: unlock patch + this doc + the CI workflow. Build from here.

Keeping the patch off `master` is deliberate: every upstream sync stays a clean rebase, and
`git diff master pro-unlock` always shows exactly the delta.

### Identity & auth (this is the tricky part — read before pushing)

This machine uses **two GitHub accounts**: a global/default one, and `leo0570` scoped to
**this repo only**. The scoping is entirely in this repo's local config — global config is untouched.

- **Commit identity** (repo-local, private email):

  ```
  user.name  = leo0570
  user.email = 274127332+leo0570@users.noreply.github.com   # GitHub no-reply → email stays private
  ```

  The global commit identity (my default account) is unchanged and applies to every other repo.

- **Push auth** is via the **GitHub CLI (`gh`)**, scoped to this repo:

  ```
  credential.https://github.com.helper =                       # empty entry resets inherited helper
  credential.https://github.com.helper = !gh auth git-credential
  ```

  The empty first value drops the inherited system `osxkeychain` helper (the global/default account)
  for this repo, so `gh` provides the credential instead. Other repos still use `osxkeychain`.

- ⚠️ **`gh`'s active account decides who pushes.** It must be `leo0570`. If pushes 403, fix with:
  ```bash
  gh auth switch --user leo0570
  ```
  (Switching `gh` to the other account for other work and forgetting to switch back is the only failure mode.)

There are **no SSH keys / host aliases** involved — auth is pure HTTPS via `gh`.

---

## 3. Building locally (on a Mac with Xcode)

Per `AGENTS.md`, drive `xcodebuild`, not the Xcode GUI.

**Compile + run (Debug)** — the quick inner loop:

```bash
xcodebuild -project alt-tab-macos.xcodeproj -scheme Debug -configuration Debug -derivedDataPath DerivedData
open DerivedData/Build/Products/Debug/AltTab.app
```

> ⚠️ A **Debug** build is **not portable** — it loads Sparkle via an absolute path into _this_
> machine's `DerivedData`. Fine for local dev, useless if copied elsewhere. For a portable app build
> **Release** (see CI below), which embeds frameworks via relative `@rpath`.

**Run the tests:**

```bash
xcodebuild test -project alt-tab-macos.xcodeproj -scheme Test -configuration Release
# or, prettier:
scripts/run_tests.sh
```

---

## 4. Building in CI (GitHub Actions) — for use without a local Mac

The fork ships one custom workflow, **`.github/workflows/build-on-tag.yml`**: it builds a portable
**Release** app with **ad-hoc signing** (no Apple cert / no notarization) and publishes the zip on
the **Releases** page when a `*-unlock` tag is pushed. (`ci_cd.yml` is upstream's full release
pipeline — left untouched; it won't run here because it needs secrets I don't have.) Actions is
already enabled on the repo.

**Trigger a versioned build** — push a `*-unlock` tag:

```bash
git tag v11.3.0-unlock && git push origin v11.3.0-unlock
```

The **Run workflow** button on `pro-unlock` still compiles (version falls back to `11.3.0`) but
does not publish a download — only tag pushes create a Release.

**Download the build** — open the matching release (permanent, no login):
`https://github.com/<owner>/<repo>/releases/tag/v<version>-unlock` → download
`AltTab-<version>-unlock.zip`

**Install the downloaded build on another Mac:**

```bash
# unzip the zip, then:
mv AltTab.app /Applications/
xattr -dr com.apple.quarantine /Applications/AltTab.app   # clear the download quarantine
open /Applications/AltTab.app
```

Then grant **Accessibility** (required) and **Screen Recording** (only for Thumbnails / window
previews) in System Settings → Privacy & Security.

**Updating to a new version** — do a clean install; don't just replace the app bundle:

1. **Quit AltTab** and **delete `/Applications/AltTab.app` entirely** (don't overlay the old copy).
2. **Remove stale permissions** in System Settings → Privacy & Security → **Accessibility** and
   **Screen Recording** — delete any AltTab entries left from the previous build.
3. Install the new build as above, then re-grant Accessibility / Screen Recording.
4. In AltTab → **Settings → General**, set **Don't check for updates periodically** and set
   **Crash reports policy** to **Never send crash reports**. (The update setting is now
   belt-and-suspenders — unlock builds disable the Sparkle feed in code, see §1 "Keeping the unlock
   durable" — but it's harmless and reassuring to set anyway.)

Caveats for an ad-hoc-signed build:

- **Permissions reset on each rebuild** (the ad-hoc signature changes with the binary) — re-grant
  Accessibility/Screen Recording after updating. Once per version, not daily.
- **Managed (MDM) Macs.** If IT forces Gatekeeper to "identified developers only" or controls TCC
  via profiles, an unsigned app may be blocked — policy, not fixable in CI. The only workaround is
  signing + notarizing with a personal Apple Developer ID ($99/yr), wired in via repo secrets.

---

## 5. Updating to a new upstream release

The patch is one tiny, stable spot, so syncing is a near-frictionless rebase:

```bash
git fetch upstream --tags
git rebase v11.4.0 pro-unlock        # replay the unlock patch onto the new release tag
# resolve conflicts only if upstream rewrote computeState() — rare, one file, seconds
git tag v11.4.0-unlock               # name the patched build (distinct from upstream's tag!)
git push origin pro-unlock           # update the branch
git push origin v11.4.0-unlock       # push JUST this tag → triggers the CI build
```

After downloading the release zip, follow the **Updating to a new version** steps in §4 (full
uninstall, revoke Accessibility/Screen Recording in System Settings, disable auto-update and crash
reports in the app).

> **Agent note:** `build-on-tag.yml` mirrors upstream's `ci_cd.yml` / `scripts/` build steps inline.
> If those change on rebase, update the fork workflow too.
>
> **Agent note:** The unlock patch may stop working after a rebase (e.g. upstream rewrites the license
> flow). If it does, revisit the codebase and design a new approach that still meets the **Unlock
> requirements** in §1.

Keep `master` mirrored too, if desired:

```bash
git switch master && git merge --ff-only upstream/master && git switch pro-unlock
```

Tag rules:

- **Name tags `vX.Y.Z-unlock`** so they never collide with upstream's `vX.Y.Z`.
- **Push the tag individually** (`git push origin vX.Y.Z-unlock`) — never `git push origin --tags`,
  which would dump all of upstream's `v8.x…` tags onto the repo.

---

## 6. Project conventions & key files (from `AGENTS.md`)

- Pure **Swift 5.8**, no Interface Builder, no SwiftUI. Compact code, guard clauses for the happy
  path, small focused methods, low-latency focus.
- Tests are **co-located**: a feature is `Foo.swift` + `FooTests.swift` + `FooSpecs.md` in one folder.
- **Keychain/signing invariant:** the Developer ID, Team ID, and bundle ID must stay stable across
  official builds (changing them orphans stored license keys). Irrelevant to local/CI ad-hoc builds.

Where the paywall lives:

```
src/pro/license/         LicenseManager (state machine), RemoteLicenseClient, Keychain, Endpoints
src/pro/ProFeature.swift registry of every gated capability + the attemptUse() gate
src/pro/scheduling/      ProTransitionManager + the Day-X nag popups + the scheduler
src/pro/ui/              Pro badges, gradient buttons, prompt windows
config/*.xcconfig        build settings (base/debug/release); DOMAIN & API_DOMAIN defaults in base
```

---

## 7. Quick reference

| Task                           | Command                                                                                                                          |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| Local dev build + run          | `xcodebuild -scheme Debug -configuration Debug -derivedDataPath DerivedData && open DerivedData/Build/Products/Debug/AltTab.app` |
| Run tests                      | `scripts/run_tests.sh`                                                                                                           |
| See the delta                  | `git diff master pro-unlock`                                                                                                     |
| Fix a push 403                 | `gh auth switch --user leo0570`                                                                                                  |
| Sync to a new release          | `git fetch upstream --tags && git rebase vX.Y.Z pro-unlock`                                                                      |
| Trigger a CI build             | `git tag vX.Y.Z-unlock && git push origin vX.Y.Z-unlock`                                                                         |
| Clear quarantine on a download | `xattr -dr com.apple.quarantine /Applications/AltTab.app`                                                                        |
