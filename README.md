# Game Forge

A custom Agent Skill for the [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery) app that has Gemma 4 (E4B) generate a small playable game on the fly and renders it inline in the chat.

This is deliberately a **capability test**: the skill does no game-specific scaffolding. Every game you play is HTML/CSS/JS that Gemma wrote in that turn, encoded into a `data:` URL and handed back to the app's webview embed.

## How it works

1. You ask Gemma to play / build / make a game.
2. The skill instructions in `SKILL.md` direct Gemma to either pick up a named game or ask once which game you want.
3. Gemma generates a single self-contained HTML document and calls the built-in `run_js` tool with `{ gameHtml, gameTitle, gameDescription }`.
4. `scripts/index.html` runs in a hidden webview, validates the HTML, ensures a mobile viewport, base64-encodes it into a `data:text/html;...` URL, and returns it via the skill's `webview.url` response field.
5. Edge Gallery embeds that URL as an inline panel in the chat. You play.

## Expected reliability on Gemma 4 E4B

| Game                       | Likely outcome                                           |
| -------------------------- | -------------------------------------------------------- |
| Tic-tac-toe                | High. Small state space, deterministic AI fits in <100 lines. |
| Whack-a-mole               | High. Just timers and click handlers.                    |
| Memory match (3x3 / 4x4)   | High. DOM shuffle + flip animation.                      |
| Simon Says                 | Medium. Sequence playback timing is finicky.             |
| Pong                       | Medium. Needs a clean `requestAnimationFrame` loop.      |
| 2048                       | Medium. Tile-merge logic is where small models slip.     |
| Snake                      | Medium. Game loop + growing array + collisions.          |
| Checkers                   | Low. Full ruleset is heavy for a 4B model — expect a stripped-down variant. |
| Chess                      | Don't.                                                   |

Generation time on-device is typically 20–60 seconds per game depending on the device and game size.

## Install

### Option A — push the folder to your Android device

1. Connect your phone to your computer.
2. Push the entire `game-forge/` folder to a known location, e.g. `/sdcard/Download/game-forge/`.
3. Open Edge Gallery, pick **Gemma 4 E4B**, open the **Agent Skills** use case.
4. Tap the **Skills** chip → **+** → **Add skill from local file** → navigate to `game-forge/`.

### Option B — host on a static web server (e.g. GitHub Pages)

1. Put the `game-forge/` folder in a public repo.
2. Add an empty `.nojekyll` file at the repo root so GitHub Pages serves `SKILL.md` raw instead of trying to render it.
3. Verify in a browser that `https://your.host/game-forge/SKILL.md` returns raw markdown.
4. In Edge Gallery: **Skills** → **+** → **Load skill from URL** → enter `https://your.host/game-forge` (note: point at the folder, **not** the `SKILL.md` file).

> Note: GitHub's `raw.githubusercontent.com` hosts files as `text/plain` and the Edge Gallery webview won't execute them as HTML, so you do need a real static host (GitHub Pages, Cloudflare Pages, Netlify, etc.) for option B.

## Use

After the skill is loaded:

- "Make me a game" → Gemma asks what to build.
- "Let's play tic-tac-toe" → Gemma builds and embeds it.
- "Make it harder" / "use a 4x4 board" / "add a score counter" → Gemma regenerates with the change.

## The one thing that might go wrong

This skill returns the generated HTML as a base64-encoded `data:` URL. The Edge Gallery's documented examples only show relative paths under an `assets/` folder; `data:` URLs aren't explicitly mentioned in the skills docs. They *should* work in a stock Android WebView, but if the embedded panel comes back blank, that's almost certainly the cause — the host app's WebView may be configured to refuse `data:` URL navigations.

If that happens, the workaround is a **hybrid skill**:

1. Pre-build a few games as static HTML files in `assets/` (`tic-tac-toe.html`, `snake.html`, …).
2. Modify `SKILL.md` so Gemma's job is to pick a `gameId` from a known list and pass it as `data.gameId`.
3. Modify `scripts/index.html` to map `gameId` → `assets/<gameId>.html` and return that as `webview.url`.

That gives up the dynamic-generation experiment but is reliable. Happy to produce that variant on request.

## Files

```
game-forge/
├── SKILL.md             # frontmatter metadata + LLM instructions
├── README.md            # this file
└── scripts/
    └── index.html       # JS skill runner (defines ai_edge_gallery_get_result)
```

## License

Apache-2.0 (matching the Edge Gallery convention).
