# Fork notes

Personal fork of [AltTab](https://github.com/lwouis/alt-tab-macos) that unlocks the paid
**AltTab Pro** features for personal use. AltTab is GPL-3.0, so modifying your own copy is
explicitly permitted. (Redistributing a modified _binary_ also obliges you to offer the modified
_source_ under GPL-3.0 — which this repo is.)

This file documents the one change we make, how the repo is laid out, how to build it (locally or
in CI), and how to keep it in sync with upstream. It's a fork-only file; upstream never sees it,
so it won't cause rebase conflicts.

---

## 1. What was changed

Everything keys off a single source of truth: `LicenseManager.shared.state` (enum `LicenseState`:
`.trial` / `.pro` / `.proExpired` / `.trialExpired`). Force it to `.pro` and the whole paywall
falls away at once. The entire patch lives in **`src/pro/license/LicenseManager.swift`**:

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
  lock-search, the App Icons / Titles appearance styles, and Auto window sizing. (`ProFeature.attemptUse()`
  returns early when `isProAvailable`; degradable prefs never get downgraded because `isProLocked` is false.)
- **The trial/nag scheduler goes silent** — `ProTransitionScheduler.computeNextFireDate()` returns
  `nil` for `.pro`, so none of the Day 1/4/12/15/21/35 popups are ever scheduled. The Day-1 welcome
  is also suppressed, and the menu bar drops the "Get Pro" item.
- **No network calls** — the license server is only contacted when a license key exists in the
  Keychain; a free user has none, so nothing is ever sent.

### Why the flag (instead of hard-coding `.pro`)

The flag defaults to `false`, and only `LicenseManager.shared` flips it on. The unit tests in
`src/pro/license/LicenseManagerTests.swift` build their own `LicenseManager` instances (flag stays
`false`), so they keep exercising the real trial/expiry logic and stay green.

### Known cosmetic leftovers (intentional — minimal scope)

We unlock functionality only; we did **not** strip the now-inert Pro UI. So Settings still shows a
green "Pro activated" button and "PRO" badges, and the menu bar still has "My Account". They do
nothing harmful. (If you ever want them gone, that's a separate, larger change across
`AppearanceTab.swift`, `ControlsTab.swift`, `Menubar.swift`, `SettingsWindow.swift`, and `ProBadgeView.swift`.)

### One-time gotcha for prior-trial machines

If a machine already ran the 14-day trial to expiry, AltTab downgraded its appearance prefs and
saved the originals into `remembered*` keys. Forcing `.pro` does **not** auto-restore them. Just
re-pick App Icons / Titles / Auto once in Settings → Appearance; it'll stick from then on.

---

## 2. Repo / git layout

```
origin    = https://github.com/leo0570/alt-tab-macos.git    (fork repo — push here)
upstream  = https://github.com/lwouis/alt-tab-macos.git       (the real upstream — pull releases)
```

Branches:

- **`master`** — kept as a clean mirror of upstream (`origin/master`). No patch on it.
- **`pro-unlock`** — the working branch: the unlock patch + this doc + the CI workflow. Build from here.

Keeping the patch off `master` is deliberate: it makes every upstream sync a clean rebase, and
`git diff master pro-unlock` always shows exactly your delta.

If `upstream` isn't set yet:

```bash
git remote add upstream https://github.com/lwouis/alt-tab-macos.git
git fetch upstream --tags
```

---

## 3. Building locally (on a Mac with Xcode)

Per `AGENTS.md`, drive `xcodebuild`, not the Xcode GUI.

**Compile + run (Debug)** — the quick "does it work" loop:

```bash
xcodebuild -project alt-tab-macos.xcodeproj -scheme Debug -configuration Debug -derivedDataPath DerivedData
open DerivedData/Build/Products/Debug/AltTab.app
```

> ⚠️ A **Debug** build is **not portable** — it loads Sparkle via an absolute path into _this_
> machine's `DerivedData`. Great for local dev, useless if copied to another Mac. For a portable
> app, build **Release** (see CI below), which embeds frameworks via relative `@rpath`.

**Run the tests:**

```bash
xcodebuild test -project alt-tab-macos.xcodeproj -scheme Test -configuration Release
# or, with prettier output:
scripts/run_tests.sh
```

---

## 4. Building without a local Mac (GitHub Actions)

The fork ships one custom workflow, **`.github/workflows/build-on-tag.yml`**, that builds a
portable **Release** app with **ad-hoc signing** (no Apple Developer cert / no notarization) and
uploads it as a downloadable artifact. (`ci_cd.yml` is upstream's full release pipeline — left
untouched; it won't run for you because it needs secrets you don't have.)

**One-time:** enable Actions on the fork — https://github.com/leo0570/alt-tab-macos/actions →
"I understand my workflows, go ahead and enable them."

**Trigger a build** either by pushing a `*-unlock` tag, or via the **Run workflow** button on the
`pro-unlock` branch. Then open the finished run → **Artifacts** → download `AltTab-<version>`.

> The artifact download requires being signed in to GitHub and expires after 90 days. To instead
> get a permanent, public, no-login download on the **Releases** page, uncomment the
> "Publish GitHub Release" step at the bottom of the workflow (it makes the binary publicly
> downloadable).

**Install the downloaded build on another Mac:**

```bash
# unzip the artifact, then:
mv AltTab.app /Applications/
xattr -dr com.apple.quarantine /Applications/AltTab.app   # clear the download quarantine
open /Applications/AltTab.app
```

Then grant **Accessibility** (required) and **Screen Recording** (only for Thumbnails / window
previews) in System Settings → Privacy & Security.

Caveats for an ad-hoc-signed build:

- **Permissions reset on each rebuild.** The ad-hoc signature changes with the binary, so macOS
  treats each new build as a new app — re-grant Accessibility/Screen Recording after updating.
  (Once per version, not daily.)
- **Managed (MDM) Macs.** If IT forces Gatekeeper to "identified developers only" or controls TCC
  via profiles, an unsigned app may be blocked — that's policy, not something CI can fix. The only
  workaround there is signing + notarizing with your _own_ Apple Developer ID ($99/yr), wired into
  the workflow via repo secrets.

---

## 5. Updating to a new upstream release

The patch is one tiny, stable spot, so syncing is a near-frictionless rebase:

```bash
git fetch upstream --tags
git rebase v11.4.0 pro-unlock        # replay the unlock patch onto the new release tag
# resolve conflicts only if upstream rewrote computeState() — rare, one file, seconds to fix
git tag v11.4.0-unlock               # name your patched build (distinct from upstream's tag!)
git push origin pro-unlock           # update the branch
git push origin v11.4.0-unlock       # push JUST this tag → triggers the CI build
```

Keep `master` mirrored too, if you like:

```bash
git switch master && git merge --ff-only upstream/master && git switch pro-unlock
```

Tag rules:

- **Name your tags `vX.Y.Z-unlock`** so they never collide with upstream's `vX.Y.Z`.
- **Push the tag individually** (`git push origin v11.4.0-unlock`) — never `git push origin --tags`,
  which would dump all of upstream's `v8.x…` tags onto your fork.

---

## 6. Project conventions & key files (from `AGENTS.md`)

- Pure **Swift 5.8**, no Interface Builder, no SwiftUI. Compact code, guard clauses for the happy
  path, small focused methods, low-latency/responsiveness focus.
- Tests are **co-located**: a feature is `Foo.swift` + `FooTests.swift` + `FooSpecs.md` in the same folder.
- **Keychain/signing invariant:** the Developer ID, Team ID, and bundle ID must stay stable across
  official builds — changing them orphans users' stored license keys. (Irrelevant to our local/CI
  ad-hoc builds, but don't touch these if you ever set up real signing.)

Where the paywall lives, for reference:

```
src/pro/license/        LicenseManager (state machine), RemoteLicenseClient, Keychain, Endpoints
src/pro/ProFeature.swift   registry of every gated capability + the attemptUse() gate
src/pro/scheduling/     ProTransitionManager + the Day-X nag popups + the scheduler
src/pro/ui/             Pro badges, gradient buttons, prompt windows
config/*.xcconfig       build settings (base/debug/release); DOMAIN & API_DOMAIN defaults live in base
```

---

## 7. Quick reference

| Task                                   | Command                                                                                                                          |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Local dev build + run                  | `xcodebuild -scheme Debug -configuration Debug -derivedDataPath DerivedData && open DerivedData/Build/Products/Debug/AltTab.app` |
| Run tests                              | `scripts/run_tests.sh`                                                                                                           |
| See your delta                         | `git diff master pro-unlock`                                                                                                     |
| Sync to a new release                  | `git fetch upstream --tags && git rebase vX.Y.Z pro-unlock`                                                                      |
| Trigger a CI build                     | `git tag vX.Y.Z-unlock && git push origin vX.Y.Z-unlock`                                                                         |
| Clear quarantine on a downloaded build | `xattr -dr com.apple.quarantine /Applications/AltTab.app`                                                                        |
