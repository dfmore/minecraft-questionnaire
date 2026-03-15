# Spec: Minecraft QuizzMaster

**Status:** Draft — awaiting approach selection
**Date:** 2026-03-15
**Project dir:** `/home/daniel/minecraft-questionnaire`
**Deploy target:** GitHub Pages (static) + Cloudflare Worker (edge proxy)

---

## Problem

There is no standalone, aesthetically faithful Minecraft quiz game on the web that uses an LLM to generate fresh, validated questions on demand. Existing quiz sites (TriviaCreator, Sporcle, FunTrivia) serve static question banks with no Minecraft theming in the UI. The goal is a self-hosted, GitHub Pages-deployable quiz game that looks and feels like Minecraft's UI, generates novel questions at runtime via Mistral, and provides enough clean state structure to extend toward leaderboards and multiplayer later.

---

## Goals

- Ship a playable quiz game on GitHub Pages with zero backend infrastructure except a Cloudflare Worker proxy
- Deliver authentic Minecraft pixel-art aesthetic: blocky typography, dirt/stone/grass palette, pixel borders, earthy UI chrome
- Generate 20 questions per round at runtime using Mistral API (LLM validates its own answers via thinking/reasoning mode)
- Support 6 difficulty tiers: Easy, Normal, Hard, Legendary, Insane, Demon
- Show post-answer explanations with correct answer rationale
- Display final score as a Minecraft material rank: Wood, Stone, Copper, Iron, Gold, Diamond, Netherite
- Keep API key invisible to the browser by routing through a Cloudflare Worker
- Structure game state to cleanly support leaderboard and multiplayer extensions (don't build, but don't block)

---

## Non-Goals

- No leaderboard (future)
- No multiplayer sessions (future)
- No user accounts or auth
- No frameworks (React, Vue, etc.)
- No build toolchain (no Webpack, Vite, bundler)
- No backend beyond the Cloudflare Worker proxy
- No custom server-side session management
- No mobile-native features (PWA, app store)
- No question caching or persistence between rounds

---

## Constraints

- **Display name:** Minecraft QuizzMaster
- **Working directory:** `minecraft-questionnaire`
- **File count target:** index.html, style.css, app.js, worker/index.js, worker/wrangler.toml
- **No framework, no bundler** — vanilla ES6 modules via `<script type="module">` is acceptable
- **GitHub Pages** requires everything to be static files at repo root or `/docs`
- **Cloudflare Worker free tier:** 100,000 requests/day — sufficient for personal/demo use
- **Mistral API key** stored as a Cloudflare Worker secret (`wrangler secret put MISTRAL_API_KEY`), never in the repo
- **JSON response format** required from Mistral for reliable parsing

---

## Approach

### Research Findings

#### Pixel Art CSS Techniques
- **Font:** `Press Start 2P` (Google Fonts) — bitmap font based on 1980s Namco arcade games, designed for 8px/16px multiples; free under SIL OFL. Also consider `VT323` for body text (less aggressive, still pixel-y)
- **Anti-aliasing off:** `-webkit-font-smoothing: none; font-smooth: never;` on pixel font elements
- **Image rendering:** `image-rendering: pixelated` on any scaled images or sprites
- **Pixel borders:** CSS `box-shadow` with `inset` values creates authentic Minecraft button bevels. Pattern: light top-left + dark bottom-right inner shadows = raised block effect
- **Palette:** Dirt (#8B6914), Stone (#888888), Grass top (#5D9E31), Grass side (#8B6914 + green overlay), Coal (#2D2D2D), Oak plank (#C8A96E), Netherrack (#7A3737), Sky (#7EC8E3), Night sky (#1A1A2E)
- **No images needed:** Full aesthetic achievable with CSS gradients, box-shadow pixel art, and the Google Font

#### Mistral API — Question Generation
- **Endpoint:** `POST https://api.mistral.ai/v1/chat/completions`
- **Recommended model:** `mistral-large-latest` (best quality, $0.50/1M input, $1.50/1M output — 20 questions ≈ ~3k tokens ≈ $0.005 per round)
- **Budget model:** `ministral-8b-latest` ($0.15/1M — adequate for Easy/Normal)
- **Reasoning model option:** `magistral-medium-latest` — supports thinking mode; response includes `{"type":"thinking","thinking":[...]}` chunks before `{"type":"text","text":"..."}`. Use for Legendary/Insane/Demon difficulty to ensure validated answers
- **Response format:** Set `response_format: {"type":"json_object"}` — forces valid JSON output for reliable parsing
- **Request body:**
  ```json
  {
    "model": "mistral-large-latest",
    "messages": [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}],
    "response_format": {"type": "json_object"},
    "temperature": 0.7,
    "max_tokens": 4096
  }
  ```
- **Thinking mode (magistral):** Response `content` is an array — filter for `type === "text"` to get the final answer JSON; `type === "thinking"` contains reasoning traces (can be discarded or shown optionally)

#### System Prompt Strategy
The system prompt instructs the model to:
1. Think step by step, validating each answer before writing it
2. Cross-check against known Minecraft version history, wiki facts, and game mechanics
3. Ensure exactly one answer is correct and three plausible distractors
4. Output valid JSON with the schema shown below
5. Mix Minecraft-specific trivia (~70%) with general knowledge (~30%) appropriate to difficulty

**Question JSON schema per round:**
```json
{
  "questions": [
    {
      "id": 1,
      "question": "string",
      "options": ["A", "B", "C", "D"],
      "correct": 0,
      "explanation": "string",
      "category": "minecraft|general",
      "difficulty": "easy|normal|hard|legendary|insane|demon"
    }
  ]
}
```

#### Cloudflare Worker Proxy
- Pattern: Worker receives POST from GitHub Pages frontend, injects `Authorization: Bearer ${MISTRAL_API_KEY}` (from wrangler secret), forwards body to `https://api.mistral.ai/v1/chat/completions`, returns response with CORS headers
- **Origin allowlisting:** Check `request.headers.get('origin')` against allowed GitHub Pages URL to prevent key abuse from other origins
- **CORS headers required:** `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers` — include OPTIONS preflight handler
- **No caching:** Each quiz round needs fresh questions; do not cache responses
- **Wrangler config:** `wrangler.toml` with worker name, compatibility date, and route

#### Existing Quiz Design Patterns (from TriviaCreator, arealme.com, Sporcle)
- Progress bar showing question X of 20
- Immediate feedback on answer click (green correct, red wrong)
- Answer revealed with explanation before next question auto-advances or on button click
- Score accumulation shown live
- Results screen with rank/badge + share option
- Keyboard support (1-4 keys for answer selection)

---

### Three Implementation Approaches

#### Approach A — Surgical (Smallest Blast Radius)
Single HTML file with embedded CSS and JS, worker as a separate file. Near-zero configuration, maximum simplicity.

**Files touched:**

| Path | Description |
|------|-------------|
| `index.html` | All HTML structure + embedded `<style>` + embedded `<script>` — one file ships the game |
| `worker/index.js` | Cloudflare Worker proxy script |
| `worker/wrangler.toml` | Wrangler config for worker deployment |

**Pros:**
- Fewest files, easiest to understand and ship
- No asset loading issues on GitHub Pages; zero path confusion
- Any browser can open index.html locally for development

**Cons:**
- index.html will be 500–800 lines — mixing concerns makes future edits harder
- CSS and JS not separately cacheable by browser
- Harder to do targeted code review or incremental changes

**Risk:** If game expands (leaderboard, multiplayer), untangling embedded CSS/JS from HTML is painful.

---

#### Approach B — Structural (Clean Separation) ✅ RECOMMENDED
Three separate files (index.html, style.css, app.js) with ES6 module-level state, plus worker directory. Best separation of concerns.

**Files touched:**

| Path | Description |
|------|-------------|
| `index.html` | Semantic markup only; links to style.css and app.js |
| `style.css` | All Minecraft pixel-art styles, CSS custom properties for palette/typography |
| `app.js` | Game state machine, API calls, DOM manipulation — ES6 module |
| `worker/index.js` | Cloudflare Worker proxy |
| `worker/wrangler.toml` | Wrangler config |

**Pros:**
- Clean boundaries — CSS changes don't touch JS and vice versa
- Browser caches CSS/JS independently (better for repeat players)
- Future modules (leaderboard.js, multiplayer.js) slot in cleanly
- Standard web project structure — any dev can navigate it

**Cons:**
- 2 more files than Approach A
- Requires correct relative path setup for GitHub Pages

**Risk:** Low. This is the standard vanilla web project structure; no novel patterns introduced.

---

#### Approach C — Pragmatic (Config-Driven Questions)
Same file structure as B, but adds a `config.js` that externalizes game config (difficulty prompts, rank thresholds, worker URL) making the game highly tweakable without touching app.js logic.

**Files touched:**

| Path | Description |
|------|-------------|
| `index.html` | Semantic markup |
| `style.css` | All pixel-art styles |
| `config.js` | Externalized game config: difficulty settings, rank thresholds, worker URL |
| `app.js` | Game logic, imports from config.js |
| `worker/index.js` | Cloudflare Worker proxy |
| `worker/wrangler.toml` | Wrangler config |

**Pros:**
- Tweaking difficulty, ranks, or worker URL requires zero logic changes
- Config file doubles as documentation of game parameters
- Natural place to add leaderboard config, multiplayer settings later

**Cons:**
- One extra file adds marginal overhead
- Config/logic split adds a decision point about where things live
- Slight over-engineering for MVP

**Risk:** Low, but risks scope drift if config grows into something more complex than a static object.

---

### Recommendation

**Approach B — Structural** is the right call for this project. It ships the MVP fast (5 files total including worker), follows standard web conventions, and creates the clean separation that makes leaderboard/multiplayer extensions natural. Approach A saves zero meaningful time while creating future debt. Approach C is one abstraction layer ahead of what's needed right now.

---

## Files to Change

_(Based on recommended Approach B)_

| Path | Type | Description |
|------|------|-------------|
| `index.html` | New | HTML shell: title screen, quiz screen, results screen (3 sections, CSS-toggled visibility) |
| `style.css` | New | Minecraft pixel-art theme: Press Start 2P font, dirt/stone palette, pixel borders, button bevels, screen layout, animations |
| `app.js` | New | Game state machine: `TITLE → LOADING → QUIZ → RESULTS → TITLE`; API call to worker; question rendering; answer handling; score tracking |
| `worker/index.js` | New | Cloudflare Worker: OPTIONS preflight handler, origin allowlist check, Mistral API proxy with injected auth header |
| `worker/wrangler.toml` | New | Worker name, compatibility_date, routes config |

---

## Phases

### Phase 1 — Static UI Shell
- [ ] Create `index.html` with title screen, quiz screen (question + 4 options + progress), results screen
- [ ] Create `style.css` with Press Start 2P font (Google Fonts import), Minecraft color palette as CSS variables, pixel border mixin (box-shadow), difficulty button grid, quiz layout, results layout
- [ ] Hardcode dummy question data in `app.js` to verify UI renders correctly across all screens
- [ ] Verify all 6 difficulty buttons render, all 3 screens toggle correctly, rank display works

### Phase 2 — Cloudflare Worker
- [ ] Create `worker/index.js` with OPTIONS handler, origin check, Mistral proxy
- [ ] Create `worker/wrangler.toml`
- [ ] Deploy worker: `wrangler deploy` from `worker/` directory
- [ ] Store API key: `wrangler secret put MISTRAL_API_KEY`
- [ ] Test worker directly with curl before wiring to frontend

### Phase 3 — LLM Question Generation
- [ ] Write system prompt for question generation (validate facts, cross-check, JSON schema)
- [ ] Implement `generateQuestions(difficulty)` in `app.js` — POST to worker URL, parse response JSON
- [ ] Handle magistral thinking-mode response format (filter `type === "text"` content chunks)
- [ ] Add loading screen state with Minecraft-themed loading message ("Generating world...")
- [ ] Test all 6 difficulty levels end-to-end with real API

### Phase 4 — Game Loop Polish
- [ ] Wire up complete game loop: Title → difficulty select → loading → quiz → results → back to title
- [ ] Implement answer feedback (green/red highlight, disable other options)
- [ ] Show explanation after each answer
- [ ] Progress bar (question X of 20)
- [ ] Live score counter
- [ ] Results screen: final score, rank (Wood→Netherite), rank icon (CSS pixel art)
- [ ] Keyboard support: 1-4 to select answer, Enter/Space to continue

### Phase 5 — GitHub Pages Deploy
- [ ] Push repo to GitHub
- [ ] Enable GitHub Pages from repo root
- [ ] Update worker origin allowlist with actual GitHub Pages URL
- [ ] Smoke test full game flow on live URL

---

## Done Criteria

- [ ] Game is accessible at `https://{username}.github.io/minecraft-questionnaire/`
- [ ] All 6 difficulty buttons navigate to a 20-question quiz
- [ ] Questions are LLM-generated (not hardcoded) — verified by reloading and confirming different questions appear
- [ ] Each answer shows feedback color (green/red) + explanation text
- [ ] Progress shows "Question X / 20" throughout the quiz
- [ ] Results screen shows numerical score + correct rank label (Wood through Netherite)
- [ ] "Play Again" returns to title screen
- [ ] API key is NOT present in any file in the repo (worker secret only)
- [ ] Worker origin check rejects requests from non-GitHub-Pages origins (test with curl from different origin)
- [ ] UI matches Minecraft aesthetic: pixel font, earthy palette, box-shadow pixel borders on buttons

---

## Decisions

| Decision | Rationale |
|----------|-----------|
| Vanilla HTML/CSS/JS, no framework | Matches constraint; no build step = direct GitHub Pages deploy; no framework maintenance |
| Cloudflare Worker for proxy | Free tier (100k req/day) sufficient; simplest way to hide API key from static site |
| `mistral-large-latest` as default model | Best balance of accuracy and cost; ~$0.005/round; good enough for all difficulties |
| `magistral-medium-latest` for Demon/Insane | Thinking mode validates factual accuracy on hard questions; reasoning traces discarded after parsing |
| `response_format: json_object` | Eliminates fragile regex parsing; guaranteed valid JSON from Mistral |
| Press Start 2P (Google Fonts) | Free, SIL OFL licensed, authentic 8-bit look, no font files to host |
| Box-shadow pixel borders (no images) | Zero assets; fully CSS-driven; scales correctly; no sprite loading issues |
| 3 HTML sections, CSS visibility toggle | Simpler than routing; no page reloads; instant transitions between screens |
| Rank thresholds: Wood 0-28%, Stone 29-42%, Copper 43-56%, Iron 57-70%, Gold 71-84%, Diamond 85-94%, Netherite 95-100% | Mirrors Minecraft material progression; Netherite intentionally rare (19/20+) |
| 70% Minecraft / 30% general knowledge | Keeps game accessible to players with broad quiz interest while staying thematic |
| No question caching | Fresh questions per round is core value prop; caching contradicts it |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Mistral returns malformed JSON despite `json_object` format | Low | High — game breaks | Wrap parse in try/catch; show error screen with retry button |
| Mistral hallucinates incorrect "correct" answers | Medium | Medium — bad game experience | System prompt instructs self-validation; explanation text lets players catch errors |
| Cloudflare Worker free tier rate limit hit (100k/day) | Very Low for personal use | Medium | Display friendly error if worker returns 429 |
| Origin allowlist blocks legitimate GitHub Pages URL after repo rename | Low | High — game broken | Document allowlist update step in Phase 5; worker URL configurable |
| Magistral thinking-mode response format changes | Low | Medium — parsing breaks | Isolate response parsing to a single `parseQuestions(response)` function; easy to patch |
| `magistral-medium-latest` too slow for good UX (reasoning takes time) | Medium | Low-Medium — loading feels long | Show animated "Generating world..." screen; only use magistral for Demon/Insane |
| Press Start 2P too small at default size for readability | Low | Low | Use `font-size: 8px` minimum with `line-height: 2`; test at mobile widths |
| GitHub Pages URL changes if repo renamed or user renames account | Low | Medium | Worker allowlist is in `worker/index.js` — one-line change + redeploy |
