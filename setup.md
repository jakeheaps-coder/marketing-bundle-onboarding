# Marketing Bundle Setup — Run Me

You are Claude Code, helping a marketing teammate at Domo set up their laptop on the **Domo Marketing Bundle** (~560 AI skills + agents + hooks for marketing work). The user has limited shell experience. Follow this file step-by-step, top to bottom. Do all of the work yourself via your Bash tool *except* the one Terminal step in Phase 1 — Domo's MDM blocks fully-automated Homebrew install, so the user pastes one command into Terminal once, then comes back. The user's direct interactions are:

1. **One Terminal paste** — Phase 1 Homebrew install. Open Terminal, paste one command, type their laptop password once, wait. The only terminal step in the whole flow.
2. **One browser tab** — to authorize GitHub via the device-code flow (Phase 3)
3. **One VS Code keystroke** — `Cmd+Shift+P` → "Developer: Reload Window" at Phase 8

If you hit a step you can't complete, **STOP** and tell the user exactly what failed and what you need from them. Do not improvise around environment issues. The design assumes the happy path; deviations from it are the user's signal to message Jake (jake.heaps@domo.com).

Announce each step in plain language before running it. Show short progress updates. Don't dump command output unless something fails. Use plain English; the user is not technical.

After each phase succeeds, print a single line: `✓ Phase N done`.

**Important about flexibility:** **Phase 0.5 below is a short discovery section that adapts the install to the user.** Some teammates are already using Claude Code for other work; some have an existing dev folder; some want a different workspace location. Don't skip Phase 0.5 — it sets the variables that later phases use (especially `$WORKSPACE`, the bundle workspace folder). If the user says "just use defaults" → fine, use `~/ai_projects/` and move on, but at least *ask*.

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

## Phase 0.5 — Discovery (a few questions before I touch anything)

> Tell the user: "Before I change anything on your laptop, three quick questions so I can adapt to your existing setup."

### Question 1 — Are you already using Claude Code?

Detect existing Claude Code state. Run all of these and capture results:

```bash
[ -d ~/.claude ] && echo "claude_dir=present"
[ -f ~/.claude/CLAUDE.md ] && echo "personal_md=present"
[ -L ~/.claude/CLAUDE.shared.md ] && echo "bundle_shared_md=symlinked_already"
[ -f ~/.claude/settings.json ] && jq -r '._bundle_settings_version // "user-personal"' ~/.claude/settings.json 2>/dev/null
[ -d ~/.claude/skills ] && ls ~/.claude/skills 2>/dev/null | wc -l
```

**Branch on what you find:**

- **Bundle already installed** (`bundle_shared_md=symlinked_already` AND skill count > 600) → Switch to update mode. Run `bash ~/.local/share/marketing-bundle-cache/scripts/install-on-laptop.sh --update`. Skip Phases 1-7 entirely. Jump to Phase 9 sanity check. Tell the user: "You already have the bundle. I just refreshed it. Skipping ahead to verification."

- **Existing Claude Code user with NO bundle** (`personal_md=present` and/or skills > 0, but no `bundle_shared_md`) → surface what's there:

  > "I see you've already been using Claude Code:
  > - Personal `~/.claude/CLAUDE.md`: present
  > - Skills in `~/.claude/skills/`: `<N>` skills
  > - Settings: `<bundle-version | user-personal | none>`
  > - MCP servers: `<read from settings.json mcpServers keys>`
  >
  > The bundle install will:
  > 1. Symlink ~560 new skills alongside your existing ones (custom skills preserved)
  > 2. Symlink agents/hooks/rules into `~/.claude/`
  > 3. Render a new `~/.claude/settings.json` with the bundle's allow-list + MCP servers — your current one gets backed up to `settings.json.local-backup-<timestamp>.json`
  > 4. Add an `@~/.claude/CLAUDE.shared.md` import to your existing `~/.claude/CLAUDE.md` (your existing content is preserved above the import line)
  >
  > **Three options:**
  > - **(a) Standard install** (recommended) — bundle's settings + your existing skills both work
  > - **(b) Skip settings.json** — keep your existing settings; you'll need to manually add the bundle's allow-list + MCP servers later from `~/.local/share/marketing-bundle-cache/claude-config/settings.template.json`
  > - **(c) Cancel** — don't install the bundle right now
  >
  > Which? (a/b/c)"

  Save the answer as `INSTALL_MODE` (`standard` | `no-settings` | `cancel`).
  - If `cancel` → STOP cleanly: "No problem, no changes made. Re-run setup.md any time."
  - If `no-settings` → Phase 6 will pass `--no-settings` to install-on-laptop.sh.
  - If `standard` → continue.

- **Fresh Claude Code, just signed in** (`claude_dir=present` but empty/minimal) → tell user: "Looks fresh. I'll do a standard install." Set `INSTALL_MODE=standard`.

- **Claude Code not installed at all** (`claude_dir` missing) → STOP: "I don't see Claude Code installed (`~/.claude` doesn't exist). Finish landing-page Step 4 first (install the Claude Code extension in VS Code + sign in), then re-run me."

### Question 2 — Where do you want your bundle workspace folder?

> Tell the user: "Where do you want your bundle workspace? This is where bundle skills will save outputs and where `mkt-bundle-onboard-me` will scaffold your first projects."
>
> "Default: `~/ai_projects/` (mirrors Jake's setup, recommended).
> Or type a custom path like `~/work/marketing/`, `~/Documents/domo/`, `~/projects/ai/`, etc.
>
> Press Enter for default, or type the path."

Save the answer as `WORKSPACE`. Defaults:

```bash
# If user hits Enter / says "default" / says nothing
WORKSPACE="$HOME/ai_projects"

# Otherwise expand ~ if user typed it
WORKSPACE="${USER_INPUT/#~/$HOME}"
```

Validate the path is sane:

```bash
mkdir -p "$WORKSPACE" 2>&1 && echo "workspace_creatable" || echo "FAIL: cannot create $WORKSPACE"
```

If creation fails → STOP, surface the error, ask user for a different path.

If the chosen folder **already exists with content** (`ls "$WORKSPACE" 2>/dev/null | wc -l` > 0), warn:

> "I see existing files in `$WORKSPACE`. I'll add a `CLAUDE.md` (Claude Code's project context), a `.gitignore`, and a few empty subfolders (automations/, gemini/, etc.). I won't touch any of your existing files. OK? (yes/no)"

If `no` → ask for a different path; loop back.
If `yes` → continue.

**Remember `$WORKSPACE` for Phases 5, 6, 7, 9.** All references to `~/ai_projects/` in those phases should use `$WORKSPACE` instead.

### Question 3 — Anything else I should know?

> Tell the user: "Last question. Anything I should know before I start? For example:
> - An existing dev folder elsewhere you'd like Claude Code to know about
> - A specific git identity you use (`git config user.email`)
> - Particular skills you don't want overwritten
> - Anything I should avoid touching
>
> Or just say `nothing` to continue."

If user mentions a path, file, or setup detail:
- Save it to a variable `USER_NOTES`
- After Phase 6 (when `~/.claude/CLAUDE.md` exists), append a section to it:

  ```bash
  cat >> ~/.claude/CLAUDE.md <<EOF

  ## User context (from setup.md Phase 0.5)

  $USER_NOTES
  EOF
  ```

  This way future Claude Code sessions know the context without the user having to re-explain.

If user says `nothing` → skip the note.

### Recap

Tell the user, before continuing: "Got it. Here's what I'll do:
- Install mode: **`<standard | no-settings>`**
- Workspace folder: **`$WORKSPACE`**
- Extra notes: **`<USER_NOTES | none>`**

Starting now. The next ~10 minutes are mostly hands-off, except for: one macOS password dialog (Phase 1), one GitHub authorize tab (Phase 3), and one VS Code reload (Phase 8)."

`✓ Phase 0.5 done — discovery complete`

---

## Phase 1 — Homebrew install + PATH register (~5 min, ONE Terminal step)

> Tell the user: "Now I'm setting up Homebrew, the macOS package manager. **This phase has the only Terminal step in the entire setup.** Domo's enterprise security tooling blocks the fully-automated install, so we do this one step manually — then I take over again for everything else."

First, check if Homebrew is already installed:

```bash
[ -x /opt/homebrew/bin/brew ] && echo "found_arm" \
|| [ -x /usr/local/bin/brew ] && echo "found_intel" \
|| echo "not_installed"
```

### If `found_arm`:

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
brew --version
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
```

Skip to Phase 2 with `✓ Phase 1 done (already installed)`.

### If `found_intel`:

```bash
eval "$(/usr/local/bin/brew shellenv)"
brew --version
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zprofile
```

Skip to Phase 2 with `✓ Phase 1 done (already installed)`.

### If `not_installed` — Terminal step (the only one in setup.md)

**Important — do NOT attempt to install Homebrew yourself via your Bash tool.** Past sessions tried `osascript` admin dialogs, `NONINTERACTIVE=1`, and writing temporary `/etc/sudoers.d/` entries — all blocked by Domo's MDM sudo policy plugin (you'll see custom `sudoabr.log` and `sudo_lecture` defaults; multiple `(ALL) ALL` entries override any `NOPASSWD: ALL` you write). Each retry creates another password dialog without progress. The clean working path is: surface Terminal instructions, STOP, wait for the user.

> Surface this to the user, formatted as the most important thing on screen, **in bold**:
>
> ## 🖥️ One Terminal step (~5 min)
>
> **Homebrew install needs to run in your Terminal once.** Domo's enterprise security tooling blocks fully-automated install, so you'll do it manually — then I take over again.
>
> 1. Press **`Cmd + Space`**, type `Terminal`, press Enter. *(That opens macOS's built-in Terminal app. You can leave VS Code open in another window — both can be open at once.)*
> 2. **Paste this command** into the Terminal window and press Enter:
>    ```
>    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
>    ```
> 3. When it asks for your **laptop password**, type it and press Enter. *(You won't see characters as you type — that's normal macOS behavior. Just type and press Enter when done.)*
> 4. When you see **`Press RETURN/ENTER to continue or any other key to abort:`**, press **Enter**.
> 5. Wait **2–5 minutes** while it downloads. You'll see lots of `Receiving objects...` and `Resolving deltas...` lines — that's normal.
>
> **If it asks to install Xcode Command Line Tools first**, click **Install** in the popup and wait ~5 min for that to finish. Then re-run the same paste command.
>
> When you see **`==> Next steps:`** or **`Installation successful!`** at the bottom, you're done. Come back here and tell me **"brew is installed"** (or just paste the last few lines of Terminal output).

**Then STOP.** Do not run any commands. Wait for the user to confirm.

### When the user confirms (or pastes output containing "Installation successful" / "Next steps")

Verify:

```bash
[ -x /opt/homebrew/bin/brew ] && echo "found_arm" \
|| [ -x /usr/local/bin/brew ] && echo "found_intel" \
|| echo "still_not_installed"
```

If `still_not_installed` → ask the user (don't troubleshoot blindly):

> "I don't see brew yet. A couple things to check:
> 1. Did the Terminal output end with `==> Next steps:` or `Installation successful`? If not, can you paste the last 10 lines?
> 2. In Terminal, run `which brew` — what does it say?
> 3. The install can take longer than 5 min on slow connections. If it's still running, just wait."

If `found_arm` (Apple Silicon — the standard Domo laptop):

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
which brew && brew --version
```

If `found_intel` (older Mac):

```bash
eval "$(/usr/local/bin/brew shellenv)"
grep -q 'brew shellenv' ~/.zprofile 2>/dev/null || echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zprofile
which brew && brew --version
```

The eval-into-current-session step makes brew available to subsequent phases without requiring another shell reload.

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

## Phase 5 — Create the workspace folder (~5s)

> Use the `$WORKSPACE` variable from Phase 0.5. Default is `$HOME/ai_projects`.

> Tell the user: "Setting up your workspace folder at **`$WORKSPACE`**."

**Step 5.1 — Folder casing precheck (only when `$WORKSPACE` is the default `~/ai_projects/`).**

If `$WORKSPACE` is `$HOME/ai_projects` (the default), check for capital-I variants:

```bash
if [ "$WORKSPACE" = "$HOME/ai_projects" ]; then
  [ -d "$HOME/AI_projects" ] && [ ! -d "$HOME/ai_projects" ] && echo "uppercase_exists"
fi
```

If `uppercase_exists` → ask user: "I see `~/AI_projects` (capital I). The bundle conventions expect `~/ai_projects` (all lowercase). Rename it? **[y/n]**"

- If `y`: `mv "$HOME/AI_projects" "$HOME/ai_projects"`
- If `n`: STOP and surface: "Bundle conventions need lowercase. Either rename manually and re-run me, choose a different `$WORKSPACE`, or message Jake."

(If `$WORKSPACE` is a custom path, skip this check — the user picked it deliberately.)

**Step 5.2 — Create the workspace folder + skeleton subfolders:**

```bash
mkdir -p "$WORKSPACE"/{automations,gemini,knowledge_graphs,front-end-sites,other,shared}
```

These are placeholders. They'll fill in as the user adopts skills. Don't create files yet — Phase 6's bundle install drops `CLAUDE.md` and `.gitignore` after cloning.

`✓ Phase 5 done`

---

## Phase 6 — Clone bundle + run install-on-laptop.sh (~3 min)

> Tell the user: "Cloning the bundle (~50 MB shallow clone) and running the install script. This adds 560 skills, hooks, agents, and rules into your Claude Code config. Sit tight — about 2-3 minutes."

**Step 6.1 — Clone the bundle (shallow, ~50 MB):**

```bash
mkdir -p ~/.local/share/marketing-bundle-cache
git clone --depth 1 https://github.com/jake-heaps_domo/marketing-bundle.git ~/.local/share/marketing-bundle-cache
```

If clone fails with auth error → STOP. The Phase 4 access check should have caught this; surface "Clone failed even though access check passed. Output above. Message Jake."

**Step 6.2 — Run the install script.** Use the install mode from Phase 0.5 Q1:

```bash
if [ "$INSTALL_MODE" = "no-settings" ]; then
  bash ~/.local/share/marketing-bundle-cache/scripts/install-on-laptop.sh --no-settings
else
  bash ~/.local/share/marketing-bundle-cache/scripts/install-on-laptop.sh
fi
```

This script handles:
- Skill symlinks (~560 → `~/.claude/skills/`) — preserves any custom Jake-local-style skills you already had
- Agent / hook / rule symlinks
- `~/.claude/CLAUDE.md` two-layer setup (your existing personal layer is preserved; the `@~/.claude/CLAUDE.shared.md` import is appended if missing)
- `~/.claude/settings.json` rendering from the bundle's `settings.template.json` *(skipped if `--no-settings`)*. The script auto-detects whether your existing settings.json is bundle-managed (sentinel present → safe to re-render) or user-personal (backed up to `.local-backup-<timestamp>.json` first)
- `launchctl` job for 15-min auto-sync

**Step 6.3 — Verify the install:**

```bash
test -d ~/.claude/skills && ls ~/.claude/skills/ | wc -l
test -L ~/.claude/CLAUDE.shared.md && echo "shared md ok"
test -f ~/.claude/settings.json && jq -r '._bundle_settings_version' ~/.claude/settings.json
```

Skill count should be > 600. `_bundle_settings_version` should be `1.0.0` or higher. If either fails → STOP, surface output, message Jake.

**Step 6.4 — Drop the workspace `CLAUDE.md` and `.gitignore` from the bundle's vendored templates** (only if not already present — don't overwrite existing files):

```bash
[ -f "$WORKSPACE/CLAUDE.md" ] || cp ~/.local/share/marketing-bundle-cache/templates/ai_projects/CLAUDE.md "$WORKSPACE/CLAUDE.md"
[ -f "$WORKSPACE/.gitignore" ] || cp ~/.local/share/marketing-bundle-cache/templates/ai_projects/.gitignore "$WORKSPACE/.gitignore"
```

If either file already existed (the user's existing workspace), leave it alone and tell the user: "Kept your existing `CLAUDE.md` / `.gitignore` in `$WORKSPACE`. Compare to the bundle's templates at `~/.local/share/marketing-bundle-cache/templates/ai_projects/` if you want to merge anything."

**Step 6.5 — Append USER_NOTES to `~/.claude/CLAUDE.md`** (only if Phase 0.5 Q3 captured anything):

```bash
if [ -n "${USER_NOTES:-}" ]; then
  cat >> ~/.claude/CLAUDE.md <<EOF

## User context (from setup.md Phase 0.5)

$USER_NOTES
EOF
fi
```

`✓ Phase 6 done`

---

## Phase 7 — Set up the workspace `.env` (~30s)

> Tell the user: "Setting up your environment file at **`$WORKSPACE/.env`** with the public Knowledge Graph key pre-filled. The skill we hand off to next will fill in your team-specific keys (Domo, Apollo, Figma, Eloqua, etc.) based on what your team uses."

**Step 7.1 — Copy the bundle's pre-populated env template** (only if no existing `.env` — don't overwrite the user's secrets):

```bash
if [ -f "$WORKSPACE/.env" ]; then
  echo "existing .env found at $WORKSPACE/.env — preserving"
else
  cp ~/.local/share/marketing-bundle-cache/templates/.env.example "$WORKSPACE/.env"
  chmod 600 "$WORKSPACE/.env"
fi
```

If existing `.env` was found, surface to user: "I see you already have `$WORKSPACE/.env`. I left it alone (didn't want to clobber your secrets). The bundle's template is at `~/.local/share/marketing-bundle-cache/templates/.env.example` if you want to compare and merge missing keys." Verify the existing file has `KG_API_KEY` set:

```bash
grep -q ^KG_API_KEY "$WORKSPACE/.env" && echo "kg_key_present" || echo "kg_key_missing"
```

If `kg_key_missing` and existing file → tell user: "Your existing `.env` doesn't have a KG_API_KEY. I'll append the public one (it's not a secret per KG governance):"

```bash
echo "" >> "$WORKSPACE/.env"
echo "# Knowledge Graph (public-tier inside Domo)" >> "$WORKSPACE/.env"
grep ^KG_API_KEY ~/.local/share/marketing-bundle-cache/templates/.env.example >> "$WORKSPACE/.env"
grep ^KG_API_URL ~/.local/share/marketing-bundle-cache/templates/.env.example >> "$WORKSPACE/.env"
```

**Step 7.2 — Verify the `KG_API_KEY` works** (should return JSON from the Knowledge Graph API):

```bash
KG_KEY=$(grep ^KG_API_KEY "$WORKSPACE/.env" | cut -d= -f2-)
curl -sf -H "X-API-Key: $KG_KEY" https://knowledge-graph-api-1053548598846.us-central1.run.app/api/products | head -c 200
```

> Note: `cut -d= -f2-` (with the trailing dash) is intentional. The KG API key is base64-padded with a trailing `=`; using plain `cut -d= -f2` would truncate the key and the request would return 403.

If you get JSON back (starts with `[` or `{`) → great, the key works.
If 401/403 → STOP: "The pre-filled KG key didn't work. Message Jake — he may need to rotate the bundle's KG key."
If timeout/network → WARN but continue (KG might be down; not a blocker).

`✓ Phase 7 done`

---

## Phase 8 — Reload Claude Code window

> **This is the only step the user does themselves. Stop and surface the instruction prominently.**

**Step 8.0 — Persist Phase 0.5 state so Phase 9 can recover it post-reload:**

```bash
cat > ~/.local/share/marketing-bundle-cache/.setup-md-state <<EOF
WORKSPACE="$WORKSPACE"
INSTALL_MODE="${INSTALL_MODE:-standard}"
EOF
```

Then print this to the user, in **bold**, formatted as the most important thing on screen:

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

**Step 9.0 — Restore Phase 0.5 state from disk:**

```bash
[ -f ~/.local/share/marketing-bundle-cache/.setup-md-state ] && source ~/.local/share/marketing-bundle-cache/.setup-md-state
echo "WORKSPACE=$WORKSPACE INSTALL_MODE=$INSTALL_MODE"
```

If state file is missing, default `WORKSPACE="$HOME/ai_projects"` and tell user we're falling back.

**Step 9.1 — Confirm everything is in place** (use the `$WORKSPACE` from Phase 0.5):

```bash
test -d ~/.local/share/marketing-bundle-cache && echo "bundle cache ok"
test -d "$WORKSPACE" && echo "workspace ok"
test -f "$WORKSPACE/.env" && echo "env ok"
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

**Step 9.4 — Confirm the 5 MCP servers are wired:**

```bash
jq -r '.mcpServers | keys[]' ~/.claude/settings.json 2>/dev/null
```

Expect: `asana`, `domoBasic`, `figma`, `github`, `webflow` (5 entries). Tell the user about the OAuth-on-first-use behavior so it doesn't surprise them later:

> "**Five MCP servers are wired up:** `domoBasic`, `figma`, `asana`, `webflow`, `github`.
>
> The first time you use a skill that calls **figma**, **asana**, or **webflow**, a browser tab will pop up asking you to sign in. That's normal — it's OAuth. Sign in with your Domo account; tokens are cached afterward so you only do it once.
>
> **github** uses your `gh` keychain (already set up in Phase 3) — no browser pop-up.
>
> **domoBasic** uses a developer token from your `.env` (DOMO_BASIC_DEV_TOKEN). If that's blank, the next skill we hand off to (`/mkt-bundle-onboard-me`) will help you set it up."

**Step 9.5 — Hand off:**

Print this final summary:

> ## ✓ Setup complete
>
> - **560 skills** installed in `~/.claude/skills/`
> - **`$WORKSPACE`** scaffolded as your workspace
> - **`$WORKSPACE/.env`** ready with public KG key
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

- **Phase 0.5 is the flexibility surface.** It captures three variables — `INSTALL_MODE` (standard | no-settings), `WORKSPACE` (workspace folder path), and `USER_NOTES` (free-form notes appended to `~/.claude/CLAUDE.md`) — that the rest of the file consumes. Don't skip Phase 0.5. If the user wants defaults, walk through it quickly anyway so the variables are set; they'll only be asked things they care about.
- This file may run on a fresh laptop OR on a laptop where the teammate already uses Claude Code for other work. Phase 0.5 detects the difference and Phases 5-7 use existence-checks + skip-if-present semantics so you never overwrite the user's content (their `.env`, their `CLAUDE.md`, their custom skills).
- The user MAY pick a non-default workspace path. Always reference `$WORKSPACE` in subsequent phases, not `~/ai_projects/`. The folder-casing precheck in Phase 5.1 only runs when `$WORKSPACE` is the default.
- During Phase 0.5 Q3, if the user mentions a SPECIFIC path you should know about (e.g., "my real dev folder is at `~/work/`"), add it to `USER_NOTES` so Phase 6.5 appends it to `~/.claude/CLAUDE.md`. That makes future Claude Code sessions aware without the user re-explaining.
- This file expects to run with permissive defaults. If you hit unexpected permission prompts during Phases 1-7, that's normal pre-Phase-8 behavior — just keep going. The user will Yes-click ~10 prompts in those phases. After Phase 8 reload, prompts disappear.
- The user is on a Mac, often with limited shell experience. Use plain English between commands. Avoid jargon ("PATH", "shellenv", "symlink", "MCP", "launchd") unless you immediately explain it.
- Phase 1 is the only Terminal step. Do NOT try to automate the brew install — Domo's MDM sudo policy plugin blocks every workaround (osascript admin dialogs, `NONINTERACTIVE=1`, sudoers.d entries, sudo cache priming). The instructions surface a paste-and-wait flow; you wait for the user to confirm. After confirmation, all you do is `eval "$(brew shellenv)"` to register the binary in the current session.
- The `gh auth login --web` device code flow is stock OAuth. The user pastes an 8-character code into a github.com page and clicks Authorize. No password is exchanged with Claude Code.
- Phase 8 is a hard cut. The current Claude Code session ends when the user reloads. There is no API to trigger a reload from inside the session. **Important:** before Phase 8 ends, save `$WORKSPACE`, `$INSTALL_MODE`, and `$USER_NOTES` somewhere persistent (e.g., write them to `~/.local/share/marketing-bundle-cache/.setup-md-state` as a sourceable bash file) so Phase 9 (which runs in a NEW session after reload) can rehydrate them:

  ```bash
  cat > ~/.local/share/marketing-bundle-cache/.setup-md-state <<EOF
  WORKSPACE="$WORKSPACE"
  INSTALL_MODE="${INSTALL_MODE:-standard}"
  EOF
  ```

  Then Phase 9 starts with `source ~/.local/share/marketing-bundle-cache/.setup-md-state` to recover them.
