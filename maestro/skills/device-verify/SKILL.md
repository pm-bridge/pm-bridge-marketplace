---
name: device-verify
description: How to verify a change on an Android device or emulator, or an iOS simulator, with Maestro - arrange by any deterministic means, decide only by an assertion on the real UI in this run, keep evidence from this run only, know the measured traps, and report steps, verdict and screenshots.
metadata:
  tools: list_devices build_and_install run_flow screenshot ui_tree
---

# Verify a change on a device

A device verification has two segments with different rules:

- **Arrange** - get the app into the state where the behaviour under test can be seen. Any
  deterministic means is fine: another UI path the source code reveals, a shared subflow, a
  self-healing precondition (the list is empty, so create an item first), `am start` to an
  exported activity. The source code is a first-class tool here, not only for reading ids.
- **Assert** - the verdict. It must be an assertion (`assertVisible`, `assertNotVisible`,
  `assertTrue`, ...) on the real UI, in this run, against this build. `run_flow` refuses a flow
  without one: launching and taking screenshots is evidence, not a verdict.

Three invariants; everything else is open:

1. The verdict runs on the real UI, this run, this build.
2. Arrangement may be scripted or take a back door; the verdict may not. Debug-only test hooks
   are ruled out by the owner: verify the real product through real entry points, and if that is
   impossible, stop and say a human is needed.
3. An arrangement failure is **not** a product defect. Report it separately (`phase: "arrange"`
   or `verdict: "error"` in the `run_flow` result).

## Steps

1. **Device.** Get it from PM Bridge, never start or stop one yourself: `start_device` (PM Bridge's
   tool) gives a running, booted device and its serial — an Android emulator's `emulator-5554`, an
   iOS simulator's UDID — and the app opens it with its window when it is not running (the owner is
   asked the first time). Devices are shared: never `emulator -avd`, `simctl boot`/`shutdown` or
   `adb emu kill`. If a layout depends on the form factor (phone vs tablet), pick the matching device.
2. **This build.** Android: `build_and_install(projectDir = the task workspace, variant, appId, device)`.
   iOS: build for the simulator yourself (`xcodebuild -scheme <S> -destination 'platform=iOS
   Simulator,id=<UDID>' build`; it writes `~/Library/Developer/Xcode/DerivedData`, so it needs the
   Full access level) and install with `xcrun simctl install <UDID> <the .app>`.
   Without it the flow runs against whatever APK is already on the device (once two weeks older
   than the fix). Name a single variant (e.g. `CloneDebug`): building every flavour can fail on one
   the change never touched. Pass `appId` so the install is proven to replace the package and the
   app is warmed up once. A build failure says `code` (compiler errors: fix the code) or `config`
   (project setup: check the base branch builds; do not burn rounds on it).
3. **Flow.** Write the flow in the workspace (for example `.maestro/verify_<ISSUE>.yaml`, shared
   setup in `.maestro/subflows/`), and run it with `run_flow(flowPath)`; `flowYaml` is for short
   self-contained flows only. Take screenshots with `takeScreenshot` at the moments that prove the
   result.
4. **Login.** Never write an account or password in a flow. Pick the account with test-accounts'
   `pick_account` and pass its `id` as `accountId` (both plugins use the same account book, so the
   id names the same account); the flow gets `${ACCOUNT}`, `${PASSWORD}` and `${LOGIN_TYPE}`
   (write exactly these names; the plugin maps them to the values it passes Maestro privately, and
   `MAESTRO_PMB_*` names are refused in flows and in `env`). If
   the id is not found, report it to the owner (the two plugins must choose the same account book).
   Type the password only with `- inputText: ${PASSWORD}`; with
   an account the flow may not use `runScript`, `evalScript`, HTTP or `${...}` expressions.
   You never see the password and must not try to. Do not reveal a password field (show-password
   toggles) or screenshot it.
5. **Read the result.** `verdict` is `pass`, `fail` or `error`; `phase` says whether the failing
   step was arrangement or the assertion; `failingStep`, `assertion` and `output` are the signal
   lines without the JVM noise; `foregroundActivity` is where the device really stopped.
6. **On a failure, look before changing anything.** The device is still up. `ui_tree` shows the
   real elements; `screenshot` shows the screen. When an element is not found, grep the source
   for it: the layout XML gives the real id, and whether its parent is a `ScrollView` /
   `RecyclerView` (then it is below the fold: scroll, do not change the assertion).

## Maestro or your own command: whichever is faster and surer

- **The screen goes through Maestro**: tap, type, swipe, wait, check. It finds elements by their
  text or id, waits for the screen to settle and says which step failed. `adb shell input tap x y`
  is blind: no wait, no check, coordinates that break on another device.
- **One flow, many steps.** Each `run_flow` first connects to the driver on the device (a few
  seconds): write the steps that belong together into one flow and run it once, never one run a step.
- **One-step things are faster with your own commands**: install (`adb install -r`, `xcrun simctl
  install`), logs (`adb logcat -d`, `xcrun simctl spawn <UDID> log show`), launching with a deep link
  (`adb shell am start -d`, `xcrun simctl openurl`), a quick look (`adb exec-out screencap -p`,
  `xcrun simctl io <UDID> screenshot`). A screenshot that is evidence still comes from `run_flow`.

## Measured traps (real emulator, each cost a wasted round)

- **Cold start ~23 s.** The first launch after installing a 139 MB APK took 23.1 s (dexopt plus
  first-run init) against a flow that waited 20 s for its first element. `build_and_install` with
  `appId` warms the app once. Otherwise give the first element `extendedWaitUntil` with a timeout
  of 30 s or more instead of failing a healthy app.
- **A modal dialog removes everything else from the tree.** Not hidden behind it: absent. No
  assertion about the screen underneath can pass while a dialog is up (permissions, privacy,
  "keep running in background"). Dialogs may come late or not at all.
- **`runFlow: when: visible:` is instantaneous.** Writing it twice is not a retry. For a late
  dialog: `extendedWaitUntil` with `optional: true` first, then the conditional tap.
- **Overlapping elements.** An element loaded later (for example an async info row) can lie over
  the middle of a button. Maestro taps element centres, so the tap is swallowed: tap a free part of
  the button instead (`tapOn: point:`). `relativePoint` is not valid YAML in Maestro 2.6.1.
- **Side menus confined to a strip.** The pad side menu is a `NestedScrollView` at x = 0..180 of
  2560 px; `scrollUntilVisible` swipes at the screen centre and scrolls the main content. Anchor
  the swipe to an element inside the menu.
- **Build-variant guards.** Code guarded by a package-name or flavour check may be off in a debug
  build (debug often appends a suffix such as `.dev`). Such a change cannot be verified on that
  build: say so instead of reporting a fail. Project-specific traps like this live in the team's
  memory, not in this skill.
- **Fixture accounts.** A test account may have no data or be forced into extra sign-in steps
  after a data wipe. Those are fixture problems, not defects.
- **Cheapest arrangement is none.** `clearState: true` signs the account out and makes every
  round pay the whole login. Clear state only when the login is what is under test.

## Evidence

Screenshots in a report come **only** from the `run_flow` result of this verification run (its
images and its `runDir`). Never use ticket attachments, older runs or a `screenshot` taken
outside the run as evidence of the verdict.

## Report (for the tester)

Always write a report, in English when it goes to YouTrack:

```
Verification: <ticket> on <device> (<build variant>, installed <installedAt>)
Steps:
  1. <arrange step> ...
  2. <assert step>: <assertion>
Verdict: PASS | FAIL (product) | NOT VERIFIED (arrangement / device / tooling)
Failing step: <failingStep>   Assertion: <assertion>   Device stopped at: <foregroundActivity>
Screenshots: <the images returned by run_flow in this run>
Notes: <what is not verified and why>
```

`verdict: error`, `phase: arrange`, or a stop on a sign-in screen after an account was injected
is NOT VERIFIED, never FAIL (product).
