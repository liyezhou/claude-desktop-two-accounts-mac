# Two Claude desktop accounts on one Mac, with their own Dock icons (follow-up guide)

This is a follow-up to Chad's "Run a Separate Claude Desktop App for Each Claude Account" guide (chadtimbl.in/writings/files/multi-account-claude-setup-guide.md). That guide gets you two instances of the Claude desktop app, one per account, each with its own profile folder via `--user-data-dir`, started through guarded AppleScript launchers. It works. This post covers what came next when I lived with it for a day, and the fixes. Everything below was verified on macOS 27.2 with Claude desktop 4.23118.0 and Claude Code 2.1.302 on 2026-11-09. Nothing here needs sudo, and nothing touches `/Applications/Claude.app` itself.

**TL;DR**

1. Your Code-tab sessions do not follow the account. The sidebar list is local, per profile folder *and* per account. A fresh profile shows nothing; sessions of the other account never show. Moving session files between profiles works for the same account.
2. The two running instances get identical "Claude" tiles in the Dock. Badging the AppleScript launchers helps a bit, but the running tiles stay generic.
3. Replacing the launchers with a wrapper app bundle whose executable is a one-line script that `exec`s the real Claude binary gives each instance its own badged tile and hover name, with the process still being the real signed app. Symlinking or hard-linking the binary does not work.
4. `claude://` links (session links, the sign-in callback) all go to one running instance. You cannot target an instance with a link.
5. Bonus: polling each account's 5-hour / 7-day usage meter for free needs one `CLAUDE_CONFIG_DIR` login per account; the status-line route does not work inside the desktop app.

---

## 1. Where the Code sessions actually live

After signing the second instance in, my Code tab was empty. Nothing was lost. The session list is stored on disk inside the profile, keyed by account:

```
~/Library/Application Support/<profile>/claude-code-sessions/<account-uuid>/<org-uuid>/local_<id>.json
```

plus an `archived-sessions.idx` in the same folder. Each JSON file is the sidebar entry (title, cwd, model, archived flag); the transcripts are elsewhere (`~/.claude/projects`, shared by both instances). Nothing about the list syncs through claude.ai.

Consequences:

- A fresh profile starts with no sessions, whichever account signs in.
- Sessions of the other account never appear, even in the same profile (different `<account-uuid>` folder).
- Sessions you made in the default profile *while briefly signed in as the second account* sit under that account's UUID in the default profile. You can move those: quit the source instance, `mv` the `local_*.json` files into the matching `<account-uuid>/<org-uuid>/` folder of the other profile, carry any archived IDs across in `archived-sessions.idx`, then relaunch. Move, never copy: two instances listing one session can both open it.
- Which UUID is which account: `lastKnownAccountUuid` in `<profile>/config.json`. Don't trust the e-mail a Claude Code session reports about itself; inside the second instance mine still said the first account's address, because the CLI's shared config file holds the first login. The real identity is in the session's environment (`CLAUDE_CODE_USER_EMAIL`, `CLAUDE_CODE_ACCOUNT_UUID`); the app hands each session its own token.

## 2. Why the Dock shows two identical tiles, and the fix

Both instances are `/Applications/Claude.app`, so macOS registers both as "Claude" with the same icon. You can't change a running tile per process.

**Half-fix:** badge the launchers' icons and keep *them* in the Dock. Clicking one focuses its instance. But the two generic running tiles are still there, and that's what you see most.

**Full fix:** make each launcher a real app bundle whose executable is a script that `exec`s the real Claude binary with the right `--user-data-dir`. The Dock ties a running app to the bundle it was launched from, so the instance runs under the launcher's tile: badged icon, "Claude Work" on hover, pinned and running tiles merged. Because it's `exec`, the process *is* the real signed binary with the real bundle: sign-in, Keychain, permission prompts (Files & Folders checked), and auto-update are unaffected. Verified first with a throwaway profile.

What did **not** work, so you don't repeat it:

- Symlinking `Contents/MacOS/Claude` into a wrapper: LaunchServices resolves the symlink and attributes the process to the real app (the badge showed, but the process was registered as Claude.app).
- Hard-linking the binary: Electron fails at launch with "Unable to find helper app".
- Swapping the AppleScript applet's `applet.icns`: ignored, because an `osacompile` applet ships an asset catalog (`Resources/Assets.car`) and `CFBundleIconName` in its plist, which macOS prefers. You'd have to remove that key and set the icon aside.
- Even after that, the Dock kept its cached icon. What flushed it: set the icon as a Finder custom icon too (see `seticon.swift` below) and delete `$(getconf DARWIN_USER_CACHE_DIR)com.apple.iconservices` and `...com.apple.dock.iconcache`, then `killall Dock`.

### 2a. Make two badged icons

Any icon tool works. I took the app's own icon (`iconutil -c iconset /Applications/Claude.app/Contents/Resources/electron.icns`), added a coloured circle with a letter in the bottom-right corner with Python/Pillow, and packed it back with `iconutil -c icns`. Save them as `icons/Claude Personal.icns` and `icons/Claude Work.icns` next to the script below.

### 2b. `seticon.swift` (stamps a Finder custom icon; the Dock honours it)

```swift
import AppKit
let args = CommandLine.arguments
let img = NSImage(contentsOfFile: args[1])!
let ok = NSWorkspace.shared.setIcon(img, forFile: args[2], options: [])
print(ok ? "set custom icon on \(args[2])" : "FAILED \(args[2])")
```

Compile once: `swiftc -O -o icons/seticon icons/seticon.swift` (needs Xcode Command Line Tools).

### 2c. `make-wrappers.sh`

Edit the two `build` lines at the bottom for your names and profile folder. "Personal" runs the default profile (no `--user-data-dir`); "Work" runs `~/Library/Application Support/Claude-Work`. Re-run after changing an icon, then quit and relaunch the instance from its tile.

```zsh
#!/bin/zsh
set -e
cd "${0:A:h}"
REAL=/Applications/Claude.app/Contents/MacOS/Claude

build() {
  local label=$1 bundle_id=$2 udd=$3
  local app="/Applications/Claude $label.app"
  rm -rf "$app"
  mkdir -p "$app/Contents/MacOS" "$app/Contents/Resources"
  cp "icons/Claude $label.icns" "$app/Contents/Resources/Claude $label.icns"

  cat > "$app/Contents/Info.plist" <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>CFBundleExecutable</key><string>Claude $label</string>
  <key>CFBundleIdentifier</key><string>$bundle_id</string>
  <key>CFBundleName</key><string>Claude $label</string>
  <key>CFBundleDisplayName</key><string>Claude $label</string>
  <key>CFBundlePackageType</key><string>APPL</string>
  <key>CFBundleIconFile</key><string>Claude $label.icns</string>
  <key>CFBundleShortVersionString</key><string>2.0</string>
  <key>CFBundleVersion</key><string>2</string>
  <key>LSMinimumSystemVersion</key><string>13.0</string>
  <key>NSHighResolutionCapable</key><true/>
</dict>
</plist>
EOF

  # Guard: if this profile's instance is already running, bring it forward and exit.
  # (Normally the Dock activates the associated instance itself and this never runs.)
  if [[ -n $udd ]]; then
    local match="grep -F -- '--user-data-dir=$udd'"
    local args="--user-data-dir=\"$udd\""
  else
    local match="grep -v -e '--user-data-dir'"
    local args=""
  fi
  cat > "$app/Contents/MacOS/Claude $label" <<EOF
#!/bin/zsh
pid=\$(ps axo pid=,command= | grep -e '/Applications/Claude\\.app/Contents/MacOS/[C]laude' | grep -v -e '--type=' | $match | awk '{print \$1}' | head -n1)
if [[ -n \$pid ]]; then
  osascript -e "tell application \\"System Events\\" to set frontmost of (first process whose unix id is \$pid) to true" >/dev/null 2>&1
  exit 0
fi
exec "$REAL" $args
EOF
  chmod +x "$app/Contents/MacOS/Claude $label"
  plutil -lint "$app/Contents/Info.plist" >/dev/null
  echo "built $app"
}

build Personal com.example.claude-personal ""
build Work     com.example.claude-work     "$HOME/Library/Application Support/Claude-Work"

[[ -x icons/seticon ]] || swiftc -O -o icons/seticon icons/seticon.swift
for L in Personal Work; do icons/seticon "icons/Claude $L.icns" "/Applications/Claude $L.app"; done
C="$(getconf DARWIN_USER_CACHE_DIR)"
rm -rf "$C"com.apple.iconservices "$C"com.apple.dock.iconcache
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -f "/Applications/Claude Personal.app" "/Applications/Claude Work.app"
killall Dock
echo "done: quit each Claude instance and relaunch it from its Dock tile"
```

Then pin both to the Dock (drag from Applications, or `defaults write com.apple.dock persistent-apps -array-add ...` and `killall Dock`).

Rules that still apply from the original guide, plus two new ones:

- Sign in with only one instance running.
- Never duplicate, move or rename `/Applications/Claude.app`; both wrappers point at it, so one auto-update serves both.
- **New:** an instance only wears its badge if it was launched from its tile. After a Claude auto-update (or if a login item points at plain `Claude.app`), quit each instance and relaunch from the tile. Point login items at the wrappers.
- **New:** `claude://` links can't pick an instance. Every one (session links from tools, the sign-in callback) goes to a single running instance; a link to the other account's session shows "This session couldn't be found". In my tests the instance launched first got them, so launch the account you want links to land in first. The other account's sessions are reached from its own sidebar. No third instance was ever spawned by a link, which was the failure mode to watch for.

## 3. Bonus: poll each account's usage meters for free

If you want the 5-hour / 7-day meters of both accounts logged without opening either app:

- The documented route (Claude Code's status line JSON has `rate_limits.five_hour` / `seven_day`) **does not run inside the desktop app**. I tested it: a fresh desktop session completed a reply and the status-line command never executed. It only fires in a terminal `claude` session, and only reports on that session's own responses.
- What works is the community recipe: `GET https://api.anthropic.com/api/oauth/usage` with `Authorization: Bearer <Claude Code OAuth access token>` and `anthropic-beta: oauth-2025-04-20`. Undocumented, could vanish, costs zero tokens.
- The desktop app does not expose its token, and on my Mac the CLI had no login of its own at all (`claude auth status` → `loggedIn: false`, no Keychain item). So give the CLI one login **per account**, each in its own config folder:

  ```
  CLAUDE_CONFIG_DIR=~/.claude-poll-personal claude auth login --email you@personal
  CLAUDE_CONFIG_DIR=~/.claude-poll-work     claude auth login --email you@work
  ```

  Per the docs, each config dir gets its own Keychain item, so they coexist. The item is named `Claude Code-credentials-<first 8 hex of sha256(config dir path)>`. Gotcha: `--email` only pre-fills the form; the browser's existing claude.ai session wins, so do the second login in a private window or you'll silently log both folders into the same account (I did).
- A ~150-line stdlib Python script reads the item with `security find-generic-password -s <name> -w`, keeps the token in memory only, calls the endpoint (host pinned, redirects refused), and appends a JSON line per reading. When the token is within 5 minutes of expiry it runs `CLAUDE_CONFIG_DIR=<dir> claude -p ok --model haiku --max-turns 1` once, which makes Claude Code refresh its own credentials (never touch the refresh token yourself). Two launchd agents at `StartInterval` 120 have run for a day with no 429s.
- Two macOS gotchas: a launchd job cannot read anything under `~/Documents` (TCC), so run the script from `~/.local/bin` or similar; and the first `security` read from a new process may show a Keychain dialog, so click "Always Allow" once or the timer stalls silently.
- Payload notes: `five_hour` is `null` when no 5-hour window is active; `seven_day_opus` / `seven_day_sonnet` exist (null for me); ~30 other obfuscated keys. Keep the raw reply per reading.

---

*Everything above was worked out with Claude Code in the desktop app itself, which is why some steps (loading launchd agents, editing its own settings) had to be run by hand: its auto mode refuses to persist timers or edit its own config. Credit for the base technique: Chad's guide and the r/ClaudeAI thread it cites.*
