# maestro plugin

A PM Bridge plugin for device verification with Maestro: list devices, start an emulator, build and install the app under test, run a
flow and get the verdict, the failing step and the screenshots taken during that run, take a
screenshot, read the UI tree. It is a standard Claude Code plugin: one MCP stdio server plus one
skill.

```
.claude-plugin/plugin.json      name, version, description, author, userConfig (+ metadata.pmb: account book, timeouts)
.mcp.json                       server "maestro": node ${CLAUDE_PLUGIN_ROOT}/dist/server.js, env from ${user_config.KEY}
skills/device-verify/SKILL.md   the skill maestro:device-verify
src/ test/                      the server and its tests
```

It runs unsandboxed as the owner's user and calls `adb`, `emulator`, `gradlew`, `maestro` and
`xcrun` with `execFile` (never a shell), from configured paths.

## Build and run

```sh
npm install
npm run build   # tsc -> dist/server.js (not committed)
npm test        # builds, then runs unit tests and the stdio end-to-end test
```

The host runs the server from `.mcp.json`. `dist/` is not committed: run
`npm install && npm run build` once after checking out.

Tests need no device, emulator, Gradle or maestro: the unit tests inject the command runner and a
fake environment and use recorded outputs (`test/fixtures/`). The end-to-end test spawns the built
server from `.mcp.json` with its `${user_config.KEY}` placeholders filled like the host does, with
`maestro_path` and `android_sdk` pointing at fake shell scripts in a temp folder; it also checks
that the `${user_config.*}` keys `.mcp.json` uses are exactly the `userConfig` keys, that `accounts`
is a sensitive account book, and that the server `timeout` covers every tool timeout.

## Configuration

Read once at startup from the process environment. `.mcp.json` maps each variable to a
`${user_config.KEY}` placeholder that the host (or Claude Code) fills from the owner's settings;
`plugin.json` `userConfig` declares each KEY (title, description, `required`, `sensitive`),
and `metadata.pmb.variables.accounts.kind = "accountBook"` makes `accounts` an account book (host only). All are optional. A placeholder left unexpanded
(`${user_config.maestro_path}`) counts as unset.

| userConfig key → variable | kind | default | |
|---|---|---|---|
| `allowed_roots` → `MAESTRO_ALLOWED_ROOTS` | text | none | absolute folders (comma or newline separated) that `projectDir` and `flowPath` must be inside, after symlinks; with none, every project path is refused |
| `android_sdk` → `ANDROID_SDK` | text | `$ANDROID_HOME`, else `$ANDROID_SDK_ROOT`, else `~/Library/Android/sdk` | `adb` = `platform-tools/adb`, `emulator` = `emulator/emulator` |
| `maestro_path` → `MAESTRO_PATH` | text | `~/.maestro/bin/maestro` | |
| `java_home` → `JAVA_HOME_OVERRIDE` | text | `$JAVA_HOME`, else Android Studio's bundled JBR | the JDK for Gradle and Maestro |
| `flow_timeout_sec` → `MAESTRO_FLOW_TIMEOUT_SEC` | text | 300 | whole seconds, 10 to 7200 |
| `build_timeout_sec` → `MAESTRO_BUILD_TIMEOUT_SEC` | text | 1200 | whole seconds, 10 to 7200 |
| `accounts` → `TEST_ACCOUNTS` | accountBook | none (flows run without a login) | the account book for login flows, delivered as a JSON array string; choose the same book as for test-accounts |

The defaults (`ANDROID_HOME`, `ANDROID_SDK_ROOT`, `JAVA_HOME`) come from the same process
environment. An invalid value (a relative path, a timeout out of range, a `TEST_ACCOUNTS` that is
not a JSON array) does not stop the server: every tool call returns an error that names the
variable and never echoes the value.

The child tools get the host's environment **without the seven variables above** (so no child,
not even Maestro, sees `TEST_ACCOUNTS`), with the JDK, SDK and maestro folders and the usual
system folders put on `PATH` (the packaged app's `PATH` is short), plus `ANDROID_HOME`,
`JAVA_HOME` and `MAESTRO_CLI_NO_ANALYTICS=1`. The one chosen account reaches only `maestro test`,
as described under Credential handoff.

### Call timeouts (`metadata.pmb.timeouts`)

The host's default per-call timeout is 60 s. Tools whose worst case with the default config is
longer get their own, in seconds: the sum of the waits in `src/tools.ts` / `src/device.ts`, plus a
margin.

| tool | timeout | worst case with defaults |
|---|---|---|
| `build_and_install` | 1800 | Gradle 1200 + ABI 8 + 2 x lastUpdateTime 10 + `adb install` 300 + warm-up (monkey 30 + up to ~100 wait + 3 + force-stop 15) = ~1676 |
| `run_flow` | 360 | `maestro test` 300 + foreground check 8 = ~308 |
| `start_emulator` | 120 | `-list-avds` 15 + `adb devices` 15 + 5 per running emulator + boot wait 45 (one poll can overrun by ~25) = ~100 plus 5 per emulator |
| `list_devices` | 90 | `adb devices` 15 + `-list-avds` 15 + `simctl` 20 + 5 per running emulator = 50 plus 5 per emulator |

`screenshot` (20 s) and `ui_tree` (50 s) stay under the default. `.mcp.json` also sets the official
per-server `timeout` to 1,800,000 ms (30 minutes, the longest tool) for Claude Code, which has no
per-tool setting. These timeouts are fixed in `plugin.json` / `.mcp.json`: they do **not** follow an owner-raised `MAESTRO_FLOW_TIMEOUT_SEC` or
`MAESTRO_BUILD_TIMEOUT_SEC`, so raising either past the default also needs the matching timeout
raised here.

## Tools

| tool | read/write | what it does |
|---|---|---|
| `list_devices` | read | Android devices / emulators (booted, offline) and bootable AVDs; iOS simulators on macOS |
| `start_emulator` | write | `avd`: boot it detached and wait up to ~45 s for `sys.boot_completed`; returns `booted` or `booting` |
| `build_and_install` | write | `projectDir`, `variant?`, `appId?`, `device?`: `gradlew assemble<Variant>`, choose the APK, `adb install -r -g`, prove the replace, warm up |
| `run_flow` | write | `flowYaml` or `flowPath`, `device?`, `env?`, `accountId?`: run `maestro test`; verdict, phase, failing step, assertion, distilled output, screenshots from this run as images |
| `screenshot` | read | `device?`: the current screen as one PNG image |
| `ui_tree` | read | `device?`, `maxBytes?`: `maestro hierarchy`, compacted to one line per useful node, truncated |

### run_flow result

```json
{ "verdict": "fail", "phase": "assert", "summary": "1 tests, 1 failures, 0 errors",
  "failingStep": "assertCondition textRegex=\"Home\"", "assertion": "Assertion is false: \"Home\" is visible",
  "foregroundActivity": "pkg/.MainActivity", "hint": "...", "account": { "id": "acct-...", "username": "...", "loginType": "..." },
  "flow": "...", "device": null, "runDir": "/tmp/pmb-maestro-run-XXXX",
  "screenshots": { "total": 3, "returned": ["..."], "notReturned": [] }, "output": "<signal lines>" }
```

followed by up to 4 `image/png` content blocks. `verdict`: `pass` (exit 0 and a clean JUnit
report), `fail` (the flow ran and a step failed; `phase` is `assert` when the failing step is an
assertion, else `arrange`), `error` (the run could not complete: no device, install failure,
missing tool, timeout). The failing step comes from Maestro's debug `commands-*.json`, else the
console's failed line; the assertion from the JUnit report, else the console.

Image caps (`IMAGE_LIMITS`): at most 4 images per result, 3 MB each, 8 MB in total; the latest
screenshots win, the others are listed under `notReturned` with the reason.

## Hard rules (enforced in the plugin)

- **Evidence from this run only.** Every run gets a new empty folder (`cwd` of maestro, so
  `takeScreenshot` lands there) and only PNGs in it are returned.
- **No verdict without an assertion.** A flow (with its subflows) without `assertVisible`,
  `assertNotVisible`, `assertTrue`, `assertWithAI` or `assertNoDefectsWithAI` is refused before
  anything runs. A subflow that does not exist is an error.
- **Project paths are confined** to `MAESTRO_ALLOWED_ROOTS` (real paths, symlinks resolved).
- **Credentials never reach the persona.** See below.
- **Install is proven.** With `appId`, the package's `lastUpdateTime` must change across the
  install; APK choice follows the variant the task named, then the device ABI, then the universal
  APK, and refuses to guess otherwise.

### Credential handoff

The owner chooses an account book for `TEST_ACCOUNTS` in the plugin settings: the same book as for
the test-accounts plugin (several plugins can choose one book). The host delivers the whole
book at startup as a JSON array string `[{ id, username, password, loginType, environment, purpose,
totpSecret? }]`. The persona
passes only `accountId`: the `id` that test-accounts' `pick_account` returned, which names the same
account here because both plugins get the same book. An id that is not in this plugin's book (or no
book chosen) is an error that says to choose the same book, and nothing runs. Flows keep writing
`${ACCOUNT}`, `${PASSWORD}` and `${LOGIN_TYPE}` (the names existing flows already use); the
plugin maps them. With an account, `run_flow` copies the flow and its YAML subflows into
`<run folder>/flows/<n>/<file name>`, rewrites those three references to `${MAESTRO_PMB_ACCOUNT}`,
`${MAESTRO_PMB_PASSWORD}` and `${MAESTRO_PMB_LOGIN_TYPE}` and every subflow reference to the
matching copy (resolved against the original file's folder), runs the copy, and hands the values
to Maestro only as those environment variables of the `maestro` process, never on its argv. The
persona's own files are never changed, and only YAML subflows are copied: another file a flow
points at by a relative path (media, scripts) does not resolve from the copy. Maestro copies shell
environment variables whose name starts with `MAESTRO_` into flows under that same name (checked in
Maestro 2.6.1, `maestro.orchestra.util.Env.withInjectedShellEnvVars`: prefix kept, `-e` wins over
it, empty values dropped). The persona cannot set `ACCOUNT`, `PASSWORD`, `LOGIN_TYPE` or any
`MAESTRO_PMB_*` name through `env` (which only ever becomes `-e` flow variables, so it cannot
change `PATH` or other process environment), and with an account a flow may not mention
`MAESTRO_PMB_` itself. Without an account the flow runs as written and any inherited
`MAESTRO_PMB_*` value is removed from Maestro's environment. With an account, the flow and its
subflows may use `${PASSWORD}` only as `- inputText: ${PASSWORD}`
and may not use `runScript`, `evalScript`, `http.` or `${...}` expressions, so a model-written flow
cannot print or post the password. Every text in results and errors is scrubbed of the password
(plain, JSON-escaped, URL-encoded) to `[secret]`; the host's known-value redaction is a second
layer. Limits: the password is in the `maestro` process's environment for the run (readable by
the same user, e.g. via `ps eww` on some systems, but not in the plain `ps` listing), and a
screenshot of an unmasked password field cannot be scrubbed.

Run folders stay under the OS temp dir (one fresh 0700 `mkdtemp` folder per run): the result
names `runDir` and its screenshots, so they must be somewhere the host's file tools can read.

## Skill

- `skills/device-verify/SKILL.md` (`maestro:device-verify`) - arrange vs assert, the measured traps,
  evidence only from this run, the tester's report format.
