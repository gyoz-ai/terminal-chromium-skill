# terminal-chromium (Claude Code Skill)

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that gives Claude a full Chromium browser in your terminal. Browse, inspect, and interact with web pages using CDP (Chrome DevTools Protocol) -- all from within a Claude Code session.

Built on [carbonyl](https://github.com/fathyb/carbonyl), which renders Chromium using unicode half-block characters. Works in any terminal, including tmux and over SSH.

## Demo

<table>
  <tbody>
    <tr>
      <td>
        <video src="https://github.com/gyoz-ai/terminal-chromium-skill/raw/main/demo.mp4">
      </td>
    </tr>
  </tbody>
</table>

## What it does

When you invoke `/terminal-chromium`, Claude will:

1. Launch a Chromium browser in a tmux split pane next to your conversation
2. Navigate to any URL you specify
3. Interact with pages programmatically via CDP (click, type, evaluate JS, get page content)

### CDP Commands

Once the browser is running, Claude uses these commands to control it:

| Command | Description |
|---------|-------------|
| `list` | List open tabs |
| `nav <target> <url>` | Navigate to a URL |
| `eval <target> <expr>` | Evaluate JavaScript |
| `click <target> <selector>` | Click an element by CSS selector |
| `clickxy <target> <x> <y>` | Click at coordinates |
| `type <target> <text>` | Type text |
| `html <target> [selector]` | Get page HTML |
| `snap <target>` | Get accessibility tree |
| `shot <target> [file]` | Take a screenshot (PNG) |
| `open [url]` | Open a new tab |
| `evalraw <target> <method> [json]` | Send raw CDP command |

### Example conversation

```
You: /terminal-chromium open https://news.ycombinator.com

Claude: [launches browser, lists tabs]
  A1B2C3D4  Hacker News  https://news.ycombinator.com/

You: get me the top 5 story titles

Claude: [runs eval to extract titles]
  1. Show HN: I built a thing
  2. Why Rust is taking over
  ...
```

## Installation

```bash
# Copy skill files
mkdir -p ~/.claude/skills/terminal-chromium
cp skills/terminal-chromium/SKILL.md ~/.claude/skills/terminal-chromium/
cp skills/terminal-chromium/cdp.mjs ~/.claude/skills/terminal-chromium/
chmod +x ~/.claude/skills/terminal-chromium/cdp.mjs
```

Or symlink to stay updated:

```bash
git clone https://github.com/gyoz-ai/terminal-chromium-skill.git ~/terminal-chromium-skill
ln -s ~/terminal-chromium-skill/skills/terminal-chromium ~/.claude/skills/terminal-chromium
```

The skill automatically checks for `tmux`, `carbonyl`, and `Node.js 21+` at runtime and will guide you through installing anything missing.

## Usage

Start a Claude Code session inside tmux, then:

```bash
# Open a page
/terminal-chromium open https://example.com

# Or just invoke it and tell Claude what to browse
/terminal-chromium go to the Rust documentation
```

Claude handles the browser launch, CDP interaction, and pane management automatically.

## How it works

```
+---------------------------+---------------------------+
|                           |                           |
|    Claude Code            |    Carbonyl (Chromium)    |
|    conversation           |    rendering in terminal  |
|                           |                           |
|    Uses cdp.mjs to        |    Exposes CDP on         |
|    send CDP commands  --> |    localhost:PORT          |
|                           |                           |
+---------------------------+---------------------------+
                    tmux split pane
```

- **carbonyl** renders real Chromium in the terminal using unicode half-block characters
- Claude launches it in a tmux split pane with `--remote-debugging-port`
- **cdp.mjs** connects via WebSocket to the CDP endpoint to control the browser
- Multiple browser instances can run on different ports (9222, 9223, ...)

## Notes

- Renders at 60 FPS with 0% idle CPU
- Supports WebGL, WebGPU, audio, and video
- Works over SSH
- Heavy JS sites (YouTube, Twitter) may crash carbonyl -- simpler pages work best
- Terminal shaders (CRT, bloom, etc.) can reduce contrast of the browser rendering
- Node.js 21+ required for built-in `WebSocket` (no npm dependencies needed)

## Credits

- [carbonyl](https://github.com/fathyb/carbonyl) by Fathy Boundjadj -- the terminal Chromium renderer
- [gyoz-ai/terminal-chromium](https://github.com/gyoz-ai/terminal-chromium) -- fork with pre-built binaries

## License

MIT
