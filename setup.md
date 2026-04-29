# Marketing Bundle Setup — Run Me

You are Claude Code, helping a marketing teammate at Domo set up their laptop on the **Domo Marketing Bundle** (~619 AI skills + agents + hooks for marketing work). The user has zero shell experience. Follow this file step-by-step, top to bottom. Do all of the work yourself via your Bash tool. **Do not ask the user to open a terminal at any point.** The only direct interactions the user should have are:

1. **One native macOS admin password dialog** — when `osascript` pops it for the Homebrew install (this is a system dialog, not Terminal.app)
2. **One browser tab** — to authorize GitHub via the device-code flow
3. **One VS Code keystroke** — `Cmd+Shift+P` → "Developer: Reload Window" at Phase 8

If you hit a step you can't complete, **STOP** and tell the user exactly what failed and what you need from them. Do not improvise around environment issues. The design assumes the happy path; deviations from it are the user's signal to message Jake (jake.heaps@domo.com).

Announce each step in plain language before running it. Show short progress updates. Don't dump command output unless something fails. Use plain English; the user is not technical.

After each phase succeeds, print a single line: `✓ Phase N done`.

---

## Phase 0 — Platform sanity check

> Tell the user: "Quick platform check first."

Run these checks. STOP and surface a clear message if any fails:

```bash
uname -s
uname -m
sw_vers -productVersion
```

- `uname -s` must be `Darwin`. If not → STOP: "This bundle is Mac-only. You're on `<their OS>`. Message Jake before continuing."
- `uname -m` must be `arm64` or `x86_64`. If something else → STOP.
- `sw_vers -productVersion` must be `13.x` or higher. If lower → WARN, ask user to confirm continue.

Also confirm Claude Code is running inside VS Code:

```bash
echo "TERM_PROGRAM=${TERM_PROGRAM:-} VSCODE_PID=${VSCODE_PID:-}"
```

If neither contains `vscode`, surface: "It looks like you're running Claude Code outside VS Code. The bundle is designed to install into VS Code. Open VS Code, install the Claude Code extension (Step 4 of the landing page), and re-run me from there."

`✓ Phase 0 done`

---

## Phase 1 — Homebrew install + PATH register (~3 min, with admin password)

> Tell the user: "Now I'm setting up Homebrew, the macOS package manager. Everything else installs through it."

First, check if Homebrew is already installed:

```bash
[ -x /opt/homebrew/bin/brew ] && echo "found_arm" || \
[ -x /usr/local/bin/brew ] && echo "found_intel" || \
echo "not_installed"
```

### If `found_arm`:

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
brew --version
```

Persist for future shells (idempotent — only adds if not already present):

```bash
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

Skip to Phase 2.

### If `found_intel`:

```bash
eval "$(/usr/local/bin/brew shellenv)"
brew --version
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zprofile
```

Skip to Phase 2.

### If `not_installed` (the fresh-Mac path):

> Tell the user, in **bold**: "**I'm about to install Homebrew. macOS will pop a password dialog asking for your laptop password. Type it and click OK — that's the only password we'll need. Your terminal will not open. Just look for the dialog from System Authentication.**"

**Step 1.1 — Pre-cache sudo with osascript GUI dialog.** This pops a native macOS authentication dialog (not Terminal). User types password, clicks OK. Sudo is cached for ~5 min:

```bash
osascript -e 'do shell script "/usr/bin/true" with administrator privileges with prompt "Marketing Bundle Setup needs your laptop password to install Homebrew (the macOS package manager). One-time install."'
```

If this errors with "User cancelled" → STOP and surface: "You cancelled the password dialog. I can't install Homebrew without your password. Re-run me when ready."

**Step 1.2 — Run brew installer with `NONINTERACTIVE=1`** (uses cached sudo so no terminal prompt):

```bash
NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

This takes 1–2 minutes. Show user a "Installing Homebrew (1–2 min)..." update.

**Step 1.3 — Eval brew into the current session** (the fix for the "brew not on your path" loop):

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

**Step 1.4 — Persist for future shells:**

```bash
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

**Step 1.5 — Verify:**

```bash
which brew && brew --version
```

If this fails → STOP and surface: "Homebrew didn't install cleanly. Message Jake with the output above."

`✓ Phase 1 done`

---

## Phase 2 — Install gh + git + jq + python (~2 min)

> Tell the user: "Installing the four tools the bundle needs: GitHub CLI, git, jq (JSON tool), Python 3.12."

```bash
brew install gh git jq python@3.12
```

Verify each:

```bash
which gh && gh --version | head -n 1
which git && git --version
which jq && jq --version
which python3 && python3 --version
```

If any tool is missing → STOP and surface: "Couldn't install `<tool>`. Output above. Message Jake."

`✓ Phase 2 done`

---

## Phase 3 — GitHub authentication (~2 min, browser only)

> Tell the user: "Now signing you into GitHub so the bundle can clone."

First, check if already authed:

```bash
gh auth status 2>&1
```

### If already authed (output contains "Logged in to github.com"):

Verify the active account is the Domo EMU account (username ends in `_domo`):

```bash
gh api user --jq .login
```

If it ends in `_domo` → skip to Phase 4. If not → surface to user: "You're signed in as `<username>` but your Domo EMU account should end in `_domo`. Do you want to (a) log out and re-auth as your `_domo` account, or (b) continue with this account anyway?" — wait for user choice.

### If not authed (the fresh-laptop path):

> Tell the user: "**A browser tab will open in a moment. You'll see a code on this side — copy it, paste it into the browser tab, and click Authorize. Then come back here.**"

Run gh auth login in the background and read its stdout to get the device code:

```bash
gh auth login --hostname github.com --git-protocol https --web --scopes "repo,read:org,workflow,gist" 2>&1 | tee /tmp/gh-auth.log &
```

Wait 3 seconds, then read `/tmp/gh-auth.log` and parse the device code (gh prints a line like `! First copy your one-time code: XXXX-XXXX`):

```bash
sleep 3
grep -E '^! First copy your one-time code:' /tmp/gh-auth.log | sed 's/.*: //'
```

Display the code prominently to the user:

> "**Your one-time code: `XXXX-XXXX`**
>
> Paste it at https://github.com/login/device — I'm opening that page now."

Open the browser:

```bash
open "https://github.com/login/device"
```

Now poll `gh auth status` every 3 seconds until success, max 5 minutes:

```bash
for i in $(seq 1 100); do
  if gh auth status >/dev/null 2>&1; then
    echo "authed after ${i} polls"
    break
  fi
  sleep 3
done
```

Verify:

```bash
gh api user --jq .login
gh auth status 2>&1 | grep -E "Token scopes"
```

The username should end in `_domo`. The scopes should include `repo`, `read:org`, `workflow`. If either check fails → STOP and surface what's wrong.

`✓ Phase 3 done`

---

## Phase 4 — Repo access verification (~10s)

```bash
gh repo view jake-heaps_domo/marketing-bundle --json name,visibility 2>&1
```

If output contains `"name":"marketing-bundle"` → success, continue.

If error mentions "Could not resolve to a Repository" or "permission" → STOP with this exact message:

> "**Your `<username>_domo` GitHub doesn't have collaborator access to the bundle yet.** Message Jake (jake.heaps@domo.com) and paste this exact line:
>
> > Add `<username>_domo` to `jake-heaps_domo/marketing-bundle` as collaborator with push access.
>
> I'll wait — re-run me once Jake confirms (he'll reply when it's done)."

Replace `<username>_domo` with their actual username (from `gh api user --jq .login`).

`✓ Phase 4 done`

---

## Phase 5 — Create `~/ai_projects/` workspace (~5s)

> Tell the user: "Setting up your workspace folder. Everything you build will live in `~/ai_projects/`."

**Step 5.1 — Folder casing precheck.** If `~/AI_projects` or `~/Ai_Projects` exists but `~/ai_projects` doesn't, surface and ask:

```bash
[ -d ~/AI_projects ] && [ ! -d ~/ai_projects ] && echo "uppercase_exists"
```

If `uppercase_exists` → ask user: "I see `~/AI_projects` (capital I). The bundle conventions expect `~/ai_projects` (all lowercase). Rename it? **[y/n]**"

- If `y`: `mv ~/AI_projects ~/ai_projects`
- If `n`: STOP and surface: "Bundle conventions need lowercase. Either rename manually and re-run me, or message Jake."

**Step 5.2 — Create the workspace folder + skeleton subfolders:**

```bash
mkdir -p ~/ai_projects/{automations,gemini,knowledge_graphs,front-end-sites,other,shared}
```

These are placeholders. They'll fill in as the user adopts skills. Don't create files yet — Phase 6's bundle install drops `CLAUDE.md` and `.gitignore` after cloning.

`✓ Phase 5 done`

---

## Phase 6 — Clone bundle + run install-on-laptop.sh (~3 min)

> Tell the user: "Cloning the bundle (~50 MB shallow clone) and running the install script. This adds 619 skills, hooks, agents, and rules into your Claude Code config. Sit tight — about 2-3 minutes."

**Step 6.1 — Clone the bundle (shallow, ~50 MB):**

```bash
mkdir -p ~/.local/share/marketing-bundle-cache
git clone --depth 1 https://github.com/jake-heaps_domo/marketing-bundle.git ~/.local/share/marketing-bundle-cache
```

If clone fails with auth error → STOP. The Phase 4 access check should have caught this; surface "Clone failed even though access check passed. Output above. Message Jake."

**Step 6.2 — Run the install script:**

```bash
bash ~/.local/share/marketing-bundle-cache/scripts/install-on-laptop.sh
```

This script handles:
- Skill symlinks (~619 → `~/.claude/skills/`)
- Agent / hook / rule symlinks
- `~/.claude/CLAUDE.md` two-layer setup (your personal layer + `@~/.claude/CLAUDE.shared.md` import)
- `~/.claude/settings.json` rendering from the bundle's `settings.template.json` (this is what gives Claude Code the allow-list so it stops asking permission on every Bash command)
- `launchctl` job for 15-min auto-sync

**Step 6.3 — Verify the install:**

```bash
test -d ~/.claude/skills && ls ~/.claude/skills/ | wc -l
test -L ~/.claude/CLAUDE.shared.md && echo "shared md ok"
test -f ~/.claude/settings.json && jq -r '._bundle_settings_version' ~/.claude/settings.json
```

Skill count should be > 600. `_bundle_settings_version` should be `1.0.0` or higher. If either fails → STOP, surface output, message Jake.

**Step 6.4 — Drop the workspace `CLAUDE.md` and `.gitignore` from the bundle's vendored templates:**

```bash
cp ~/.local/share/marketing-bundle-cache/templates/ai_projects/CLAUDE.md ~/ai_projects/CLAUDE.md
cp ~/.local/share/marketing-bundle-cache/templates/ai_projects/.gitignore ~/ai_projects/.gitignore
```

`✓ Phase 6 done`

---

## Phase 7 — Set up `~/ai_projects/.env` (~30s)

> Tell the user: "Setting up your environment file with the public Knowledge Graph key pre-filled. The skill we hand off to next will fill in your team-specific keys (Domo, Apollo, Figma, Eloqua, etc.) based on what your team uses."

**Step 7.1 — Copy the bundle's pre-populated env template:**

```bash
cp ~/.local/share/marketing-bundle-cache/templates/.env.example ~/ai_projects/.env
chmod 600 ~/ai_projects/.env
```

**Step 7.2 — Verify the pre-filled `KG_API_KEY` works** (should return JSON from the Knowledge Graph API):

```bash
KG_KEY=$(grep ^KG_API_KEY ~/ai_projects/.env | cut -d= -f2)
curl -sf -H "X-API-Key: $KG_KEY" https://knowledge-graph-api-1053548598846.us-central1.run.app/api/products | head -c 200
```

If you get JSON back (starts with `[` or `{`) → great, the key works.
If 401/403 → STOP: "The pre-filled KG key didn't work. Message Jake — he may need to rotate the bundle's KG key."
If timeout/network → WARN but continue (KG might be down; not a blocker).

`✓ Phase 7 done`

---

## Phase 8 — Reload Claude Code window

> **This is the only step the user does themselves. Stop and surface the instruction prominently.**

Print this to the user, in **bold**, formatted as the most important thing on screen:

> ## 🔄 One thing for you to do
>
> **Press `Cmd + Shift + P`, type `Developer: Reload Window`, and press Enter.**
>
> This applies the new permissions and MCP servers I just installed. Without this, you'd see permission prompts on every command going forward.
>
> After it reloads, **drag this same `setup.md` file back into Claude Code** and type:
>
> ```
> continue from Phase 9
> ```

Then **STOP**. The current Claude Code session is about to die when the user reloads.

---

## Phase 9 — Post-reload sanity check + handoff (~30s)

> *(This phase runs in the new post-reload session.)*

> Tell the user: "Welcome back. Final checks, then I'll hand you off."

**Step 9.1 — Confirm everything is in place:**

```bash
test -d ~/.local/share/marketing-bundle-cache && echo "bundle cache ok"
test -d ~/ai_projects && echo "workspace ok"
test -f ~/ai_projects/.env && echo "env ok"
test -L ~/.claude/CLAUDE.shared.md && echo "shared md ok"
test -f ~/.claude/skills/mkt-bundle-onboard-me/SKILL.md && echo "onboard-me skill ok"
```

All five lines should print "ok". If any fails → surface which one and ask user to retry the reload.

**Step 9.2 — Confirm the allow-list is in effect.** Run a benign command:

```bash
bash -c 'echo allow-list-active'
```

If Claude Code prompts you for permission on this trivial command → the settings didn't apply. Tell the user: "The reload didn't fully take. Press `Cmd+Shift+P` → 'Developer: Reload Window' once more, then re-drag this file and try again."

**Step 9.3 — Confirm the 15-min auto-sync is loaded:**

```bash
launchctl list 2>/dev/null | grep com.domo-marketing.bundle-sync && echo "auto-sync ok"
```

**Step 9.4 — Hand off:**

Print this final summary:

> ## ✓ Setup complete
>
> - **619 skills** installed in `~/.claude/skills/`
> - **`~/ai_projects/`** scaffolded as your workspace
> - **`.env`** ready with public KG key pre-filled
> - **15-min auto-sync** running so you stay current with bundle updates
>
> ## What's next
>
> Now I'm handing you off to the bundle's onboarding skill, which will:
> 1. Figure out which marketing team you're on
> 2. Walk you through the top 5 skills built for your role
> 3. Help you ship your first PR (a friendly typo fix in the sandbox)
> 4. Generate a personalized cheat sheet at `~/Documents/marketing-bundle-<your-team>-cheatsheet.md`
>
> **Type or paste this to start:**
>
> ```
> /mkt-bundle-onboard-me
> ```
>
> Or, in plain English: "onboard me" works too.
>
> When you're done with that, message Jake to schedule a 30-min follow-up — by then you'll have the bundle running and your first PR shipped, so we can spend the meeting on the high-value stuff (which skills to use for your daily work, how to chain them together, what to automate next).

`✓ Phase 9 done — handed off to /mkt-bundle-onboard-me`

---

## If something goes wrong

**STOP** at the failing phase. Tell the user:

1. Which phase failed
2. The exact error (last 5 lines of output)
3. The recovery action: "Message Jake (jake.heaps@domo.com) and paste this entire chat thread."

Don't try to recover. The 9 phases are designed so each builds on the previous; a Phase 1 failure means Phases 2-9 won't work either. Better to surface clearly and let Jake debug than to limp through.

---

## Notes for Claude Code (not for the user)

- This file expects to run in a fresh Claude Code session with permissive defaults. If you hit unexpected permission prompts during Phases 1-7, that's normal pre-Phase-8 behavior — just keep going. The user will Yes-click roughly 10 prompts in those phases. After Phase 8 reload, prompts disappear.
- The user is on a fresh Mac with zero shell experience. Use plain English between commands. Avoid jargon ("PATH", "shellenv", "symlink", "MCP", "launchd") unless you immediately explain it.
- The `osascript` admin dialog in Phase 1 is a native macOS authentication dialog with a system-mediated lock icon. It is NOT Terminal.app and NOT a shell. The user types their laptop password into a system dialog box, clicks OK, and that's it.
- The `gh auth login --web` device code flow is stock OAuth. The user pastes a 8-character code into a github.com page and clicks Authorize. No password is exchanged with Claude Code.
- Phase 8 is a hard cut. The current Claude Code session ends when the user reloads. There is no API to trigger a reload from inside the session.
- Phase 9 is designed to run in a fresh post-reload session with this file dragged in again. The user types "continue from Phase 9" and you skip to that phase.
