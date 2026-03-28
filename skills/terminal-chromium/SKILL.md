---
name: terminal-chromium
description: Terminal-native Chromium browser with CDP remote debugging. Renders real web pages in the terminal using unicode half-block characters (works in any terminal, including tmux). Use for browsing, debugging, testing, and inspecting web pages. Based on carbonyl (https://github.com/gyoz-ai/terminal-chromium).
---

# Terminal Chromium

A full Chromium browser that renders natively in your terminal. Uses CDP (Chrome DevTools Protocol) for programmatic control.

## How it works

- **carbonyl** renders Chromium directly in the terminal using unicode half-block characters — no Kitty graphics needed, works everywhere including tmux
- When launched with `--remote-debugging-port=PORT`, it exposes a CDP endpoint at `http://127.0.0.1:PORT`
- The `cdp.mjs` script connects to this endpoint to control the browser
- Multiple instances can run on different ports (9222, 9223, 9224, ...)

## First-time setup

Install carbonyl from the gyoz-ai fork's pre-built binaries:

```bash
# Detect platform (carbonyl uses "macos" not "darwin")
ARCH=$(uname -m)
OS=$(uname -s)
case "$OS" in
  Darwin) PLATFORM_OS="macos" ;;
  Linux)  PLATFORM_OS="linux" ;;
  *)      echo "Unsupported OS: $OS"; exit 1 ;;
esac
case "$ARCH" in
  arm64|aarch64) PLATFORM_ARCH="arm64" ;;
  x86_64)        PLATFORM_ARCH="amd64" ;;
  *)             echo "Unsupported arch: $ARCH"; exit 1 ;;
esac
PLATFORM="${PLATFORM_OS}-${PLATFORM_ARCH}"

# Download and install
INSTALL_DIR="$HOME/.local/share/terminal-chromium"
mkdir -p "$INSTALL_DIR"
curl -fSL "https://github.com/gyoz-ai/terminal-chromium/releases/latest/download/carbonyl.${PLATFORM}.zip" -o /tmp/carbonyl.zip
unzip -o /tmp/carbonyl.zip -d "$INSTALL_DIR"
# The zip extracts into a subdirectory — move files up
if [ -d "$INSTALL_DIR/carbonyl-"* ]; then
  mv "$INSTALL_DIR"/carbonyl-*/* "$INSTALL_DIR/"
  rmdir "$INSTALL_DIR"/carbonyl-*/
fi
rm /tmp/carbonyl.zip
chmod +x "$INSTALL_DIR/carbonyl"

# Add to PATH if not already
if ! echo "$PATH" | grep -q "terminal-chromium"; then
  echo 'export PATH="$HOME/.local/share/terminal-chromium:$PATH"' >> ~/.zshrc
  export PATH="$HOME/.local/share/terminal-chromium:$PATH"
fi

echo "carbonyl installed: $($INSTALL_DIR/carbonyl --version)"
```

## Launch browser (when /terminal-chromium is invoked)

When this skill is invoked, run through these prerequisite checks first, then launch.

### Step 1: Check prerequisites

Run all checks in a single bash command:

```bash
echo "=== Prerequisite Check ===" && \
echo -n "tmux: " && (command -v tmux >/dev/null 2>&1 && echo "OK" || echo "MISSING") && \
echo -n "carbonyl: " && (test -x "$HOME/.local/share/terminal-chromium/carbonyl" && echo "OK" || echo "MISSING") && \
echo -n "node: " && (node -v 2>/dev/null || echo "MISSING") && \
echo -n "tmux session: " && ([ -n "$TMUX" ] && echo "OK" || echo "NOT IN TMUX")
```

**If tmux is MISSING**: Tell the user to install it (`brew install tmux` on macOS, `apt install tmux` on Linux).

**If carbonyl is MISSING**: Run the first-time setup section above to install it automatically.

**If NOT IN TMUX**: Tell the user: "The browser needs tmux to open in a split pane. Run `/exit`, then `tmux new-session "claude --resume"` to relaunch inside tmux."

**If all OK**: Continue to Step 2.

### Step 2: Ensure tmux config

Check and apply required tmux settings for carbonyl:

```bash
# Apply settings for this session (idempotent)
tmux set -g mouse on 2>/dev/null
tmux set -g allow-passthrough all 2>/dev/null
tmux set -g extended-keys on 2>/dev/null
echo "tmux settings applied (mouse, passthrough, extended-keys)"
```

Also check if `~/.tmux.conf` has these settings persisted. If not, append them:

```bash
TMUX_CONF="$HOME/.tmux.conf"
touch "$TMUX_CONF"
grep -q "set -g mouse on" "$TMUX_CONF" || echo -e "\n# terminal-chromium settings\nset -g mouse on" >> "$TMUX_CONF"
grep -q "allow-passthrough" "$TMUX_CONF" || echo "set -g allow-passthrough all" >> "$TMUX_CONF"
grep -q "extended-keys" "$TMUX_CONF" || echo "set -g extended-keys on" >> "$TMUX_CONF"
echo "tmux.conf updated"
```

### Step 3: Launch browser

```bash
# Find available CDP port
PORT=9222
while curl -s "http://127.0.0.1:$PORT/json/version" >/dev/null 2>&1; do
  ((PORT++))
done

# Launch carbonyl in a tmux split pane
URL="${1:-https://google.com}"
CARBONYL="$HOME/.local/share/terminal-chromium/carbonyl"
PANE_ID=$(tmux split-window -h -p 50 -P -F '#{pane_id}' \
  "$CARBONYL --remote-debugging-port=$PORT '$URL'")
sleep 3

# Verify CDP connection
curl -s "http://127.0.0.1:$PORT/json/version" | head -3
```

### Step 4: Use CDP

After launch, use the CDP commands below to interact with the browser programmatically.

## CDP Commands

All commands use the CDP script at `~/.claude/skills/terminal-chromium/cdp.mjs`.

Set `CDP_PORT=<port>` before every command. The `<target>` is a targetId prefix from `list`.

```bash
# List open pages
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs list

# Navigate
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs nav <target> <url>

# Click element by CSS selector
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs click <target> <selector>

# Click at coordinates
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs clickxy <target> <x> <y>

# Type text
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs type <target> <text>

# Evaluate JavaScript
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs eval <target> <expr>

# Get page HTML
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs html <target> [selector]

# Accessibility tree
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs snap <target>

# Screenshot
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs shot <target> [file]

# Open new tab
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs open [url]

# Raw CDP command
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs evalraw <target> <method> [json]
```

## Typical workflow

```bash
# 1. List tabs to get target ID
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs list

# 2. Interact
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs eval <target> "document.title"
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs click <target> "a.nav-link"
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs nav <target> "https://other-site.com"

# 3. Get page content
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs snap <target>
CDP_PORT=9222 ~/.claude/skills/terminal-chromium/cdp.mjs eval <target> "document.body.innerText.slice(0, 3000)"
```

## Close

```bash
# Close the browser pane
tmux kill-pane -t "$PANE_ID" 2>/dev/null
```

## Notes

- Renders using unicode half-block characters — works in any terminal, no Kitty graphics needed
- Works natively in tmux without passthrough or special configuration
- Starts in under 1 second, runs at 60 FPS, 0% idle CPU
- Supports WebGL, WebGPU, audio, and video
- Works over SSH
- tmux is configured with solid background (`bg=#1e1e2e`) for better contrast
- Source: https://github.com/gyoz-ai/terminal-chromium
