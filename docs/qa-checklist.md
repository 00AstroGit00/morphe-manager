# QA Checklist (targeted)

Manual validation checklist for recent high-impact behavior changes. Run on CI-capable environments where possible; root/install flows require physical devices.

## Environment prerequisites

- Android SDK/NDK + JDK toolchains for build/lint: `./gradlew :app:testDebugUnitTest :app:lint assembleDebug`
- Root-capable test devices for mount flows (KernelSU, Magisk, APatch as available).
- At least one device/emulator for non-root install and APK download helper flows.

## #1388 — Auto-mount respects "After patching"

**Fix:** root mounts auto-start only when `Settings > Patching > After patching` is **Install automatically** (`AfterPatching.INSTALL`). `Ask which installer` and `Install manually` must not auto-mount.

| Case | Setup | Expected |
|---|---|---|
| Manual, mount build | Rooted, patch mode produces mount build, After patching = Manual | Patching completes; **no** automatic mount. User can mount/install manually from success screen. |
| Ask installer, mount build | After patching = Ask which installer | **No** automatic mount (no installer to ask for mount). Manual action required. |
| Install automatically, mount build | After patching = Install automatically | Mount starts automatically after patching (subject to normal root/mount readiness). |
| Stock, not previously mounted | Stock app installed, not previously mounted; setting not Install automatically | Must **not** auto-mount after patching (regression case from issue report). |
| Previously mounted | App currently/previously mounted; setting not Install automatically | Must **not** auto-mount after patching. |
| Non-mount install | Standard (non-mount) build | Auto-install behavior unchanged except existing semantics: Install automatically only when allowed; Ask installer/manual behave as before. |

Verify on: KernelSU, Magisk, APatch (where available), YouTube/YouTube Music sample.

## #1392 — APK helper trust navigation / back gesture

**Fix:** tapping "trust" from download instructions dismisses the dialog before opening Settings > APK download helpers.

| Case | Expected |
|---|---|
| Download instructions → trust helper action → APK download helpers settings subpage | Back gesture/button exits settings subpage normally (not swallowed by hidden dialog). |
| Return to Home after trusting | APK flow state is sensible (download instructions dismissed; user can continue from APK availability if still relevant). No stuck overlay. |
| Helper trust confirmation inside settings subpage | Back dismisses confirm dialog/page as expected. |

## #1389 — Installed app version on APK download screen

| Case | Expected |
|---|---|
| App installed, download instructions shown (home flow) | Dialog description includes installed version label/value alongside app/requested version when available. |
| App installed, batch download instructions shown | Same as home flow. |
| App not installed | No installed-version segment shown; dialog still renders correctly. |

## #1379 — APKCube download button

| Case | Expected |
|---|---|
| APKCube source in download instructions | Blue "Download APK" button with download icon renders (no old badge drawable). |
| Tap fake button | Existing nudge toast behavior still works; no actual download from in-app button. |
| Light/dark theme | Button remains readable (brand blue, white content). |

## #1391 — Dropdown menu spacing

| Case | Expected |
|---|---|
| Sort/selection menus (home, pickers, row actions, exposed dropdowns using shared menu) | First/last item spacing matches side spacing; item corners align concentrically with menu container. |
| Long menus / scrollable menus | No clipping/regression in item selection or hit targets. |

## #1386 — Check for patch updates on startup

| Case | Expected |
|---|---|
| Settings > Notifications/Updates: "Check for patch updates on startup" enabled (default) | Opening app performs startup bundle update check as before. |
| Disabled | No automatic bundle update check on startup/open. |
| Manual refresh (pull-to-refresh / refresh action) while disabled | Manual bundle update still works. |
| Background update notifications / worker | Unaffected by startup toggle (separate path). |

## #1396 — Saved original APK replacement

| Case | Expected |
|---|---|
| Existing bulky saved split-archive original, later non-mount patch produces merged/non-split original | Saved original can be upgraded/replaced by repatchable non-split original (storage optimization path). |
| Mount-related flows needing split archive input | Not regressed: mount builds keep/archive behavior unchanged; root mount restore still uses installed stock app. |
| Incoming split archive over non-split saved original | Split archive remains preferred when incoming is split. |

## CI commands (when toolchain available)

```bash
./gradlew :app:testDebugUnitTest
./gradlew :app:lint
./gradlew assembleDebug
```
