# Spec: Rewrite Alias Subtitle Message

**Date:** 2026-03-16
**Context:** Clarify to players that their 5-character alias is their identity, non-recoverable, and required for account continuity on shared devices.

---

## Problem

The current alias subtitle message ("This alias tracks your score — you can change it anytime from the title screen") emphasizes flexibility and score tracking. This creates two misconceptions:

1. Players believe aliases are recoverable or account-linked (they're not — no recovery mechanism exists; lost alias = lost scores)
2. Players may not understand that on a shared device, entering a *different* 5-character alias = different player + separate score

The current messaging prioritizes convenience over clarity and doesn't address the critical "remember your alias or lose your scores" constraint.

---

## Goals

- **Communicate identity persistence:** The 5-character alias *is* the player identity. No account system, no recovery, no backup.
- **Warn about score loss:** If you forget the alias, your scores are permanently inaccessible.
- **Document device sharing UX:** Explain how to switch players (different alias = different player).
- **Maintain light, Minecraft-appropriate tone** without doom-mongering.
- **Fit existing UI:** 2–3 lines max, `<br>` line breaks, text-only change.

---

## Non-Goals

- Modify UX/flow (no modal, no checkbox, no changes to input validation)
- Add leaderboard branding or mentions
- Link to external help docs or account recovery flows (none exist)
- Change input field appearance, placeholder, or constraints
- Add storage persistence or auto-recovery mechanics

---

## Constraints

- **Text-only change** — modify only the `.alias-tip` paragraph text in `index.html` line 41
- **No HTML structure changes** — keep `<br>` line breaks as-is (they're mobile-responsive)
- **Tone:** Light, concise, Minecraft-appropriate; avoid corporate/legalistic language
- **Length:** Max ~80 words per line; 2–3 visual lines acceptable
- **Do not mention "leaderboard"** — context is implicit; focus on the alias identity itself

---

## Approach

### Candidate 1: Identity-Focused ("Remember It")
**Emphasis:** Your alias is *your* identity; losing it = losing scores.

```
This alias is YOU — remember it!
No recovery if lost; write it down to be safe.
Switch players on this device by entering a different alias.
```

**Rationale:**
- Directly addresses the "I can always recover it" misconception
- "Write it down" is concrete and actionable
- "Switch players" explains the shared-device scenario upfront
- Shortest version; fits 3 tight lines

**Word count:** ~30 words

---

### Candidate 2: Risk-Centered ("You're Responsible")
**Emphasis:** Your alias is your responsibility; own it.

```
Your alias is your identity — don't forget it!
Lose it and your scores vanish forever.
Different alias = different player (switch folks on this device anytime).
```

**Rationale:**
- Blunt about consequences ("vanish")
- Emphasizes personal ownership
- Explicitly covers the shared-device case
- Slightly longer; three distinct ideas

**Word count:** ~35 words

---

### Candidate 3: Balanced ("Identity + Instructions")
**Emphasis:** Practical and reassuring; alias is your identity, here's how to use it.

```
This alias is YOUR identity — remember it!
No account recovery; forget it and your scores are gone.
Enter a different alias to play as someone else on this device.
```

**Rationale:**
- Opens with direct identity claim ("YOUR identity")
- "Remember them!" is lighter in tone than "don't lose"
- Explains both solo play and shared-device switching
- Longest, but all three points are essential for clarity
- Exclamation mark adds Minecraft/gaming energy

**Word count:** ~45 words

---

## Files to Change

| File | Change |
|------|--------|
| `index.html` line 41 | Replace `.alias-tip` inner text with new subtitle (3 candidates above) |

---

## Overlap & Worktrees

- **Active worktrees:** None (`git worktree list` returns empty)
- **Conflicting specs:** None in `.claude/specs/`
- **Risk of merge conflicts:** Minimal — single-file, single-element text change

---

## Phases

**Phase 1: Approve Copy**
- User selects one of the three candidate rewrites (or proposes a different version)
- Builder replaces `.alias-tip` text in `index.html`

**Phase 2: Verify & Test**
- Visual check in browser (no functional tests needed; text-only)
- Confirm line breaks render correctly on mobile (existing `<br>` tags handle this)
- Optionally: run any pre-deploy checks (linting, etc.) if they exist

**Phase 3: Merge & Deploy**
- Commit: `squash(alias-subtitle): rewrite alias subtitle to emphasize identity persistence`
- Push to `main`

---

## Done Criteria

- [ ] Spec approved; candidate copy selected by user
- [ ] `index.html` line 41 updated with new `.alias-tip` text
- [ ] Visual verification: text renders correctly in browser at desktop and mobile widths
- [ ] No other files modified
- [ ] Commit message follows `squash(alias-subtitle):` format
- [ ] Changes pushed to `main` branch

---

## Decisions

1. **Three candidates instead of one:** Each emphasizes a different priority (identity, consequences, balance). User's selection reveals which message resonates with the game's tone and audience.

2. **No UX changes:** Considered adding a "write it down" prompt or copy-to-clipboard button, but constraints forbid flow/UX modifications. Text-only is sufficient to set expectations.

3. **Dropped "leaderboard" mention:** Current message implies scores are tracked; new message emphasizes ownership instead of tracking. Leaderboard context is clear once user sees the Leaderboard button on title screen.

4. **No account recovery feature:** Spec does not propose adding recovery mechanics. Message simply reflects current state: alias = identity, no backup, no recovery.

---

## Risks

- **Tone mismatch:** If the chosen text feels too ominous ("scores are gone"), it may discourage casual play. Candidate 1 is lightest; Candidate 2 is heaviest. User should review for game culture fit.
- **Mobile layout:** If any candidate exceeds ~40 chars per line, mobile rendering may wrap awkwardly. Candidate 3 is longest; recommend testing on a mobile viewport.
- **Retention vs. clarity tradeoff:** More explicit warnings about score loss may improve alias recall but could also frustrate players who genuinely forget. No solution; inherent tradeoff.
- **Shared-device confusion:** Explaining the "different alias = different player" flow in subtitle alone may not be sufficient; some players may still expect cross-alias score pooling. Spec text clarifies; UX does not.

---

## Recommendation

**Candidate 3 (Balanced)** is recommended.

**Rationale:**
- Hits all three critical points: identity ownership, consequence clarity, shared-device instruction
- Tone is energetic (exclamation) without being preachy
- Word count is higher, but each sentence serves a distinct purpose; nothing is fluff
- Longest candidate gives best chance of being *read and understood* before submission
- Fits the game's audience: younger players, casual/competitive mix; Minecraft tone is celebratory, not cautious

**Alternatives if user wants different tone:**
- Choose **Candidate 1** if you want the lightest/most confident message; risk is that "write it down" may be missed
- Choose **Candidate 2** if you want to emphasize consequences over positivity; risk is tone feels too stern for a quiz game

---

## Implementation Notes

- Replace only the inner text of the `<p class="alias-tip">` element
- Preserve the `<br>` tags for line breaks (mobile-responsive)
- No CSS changes; text will inherit existing `.alias-tip` styling
- If line count changes (2 vs 3 `<br>` breaks), visually verify spacing on mobile

