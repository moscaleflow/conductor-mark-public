# V4 Shell Refinement: Mic Placement

> Coder-3 Spec | D152 | For Coder-1 execution
> Source: D85 lock — "Mic on right side of search bar for voice input"
> Build order: 5 of 6 (deferred — depends on voice input integration)

---

## User-observable behavior

**Before:** SearchBar is a plain text `<input>` with rotating placeholder prompts. No mic icon. No voice input.

**After:** A microphone icon appears inside the SearchBar, right-aligned (to the right of the text input area, inside the input's border radius). Tapping the mic icon:
1. Requests browser microphone permission (first time only)
2. Starts speech-to-text via Web Speech API (`SpeechRecognition`)
3. Mic icon pulses red while listening
4. Transcribed text fills the input field
5. On speech end (silence timeout), the transcribed text auto-submits via `onSubmit`
6. If permission denied or API unavailable, mic icon grays out with a tooltip: "Microphone not available"

**Acceptance criteria:**
- Mic icon is visible on desktop and mobile
- Icon is right-aligned within the input container, vertically centered
- Icon does not overlap typed text (input has right padding)
- Voice transcription populates the input and submits on speech end
- Works in Chrome and Safari (Web Speech API coverage)
- Graceful fallback: mic icon hidden entirely if `SpeechRecognition` is undefined (Firefox, some mobile browsers)
- No new npm dependencies — Web Speech API is browser-native

---

## File-level scope

| File | Change | ~LOC |
|---|---|---|
| `src/components/operator/SearchBar.tsx` | Add mic icon button inside input wrapper. Add `useSpeechRecognition` hook (local to file or extracted to a hook). Adjust input padding-right for icon space. | ~60 |

**Total: ~60 LOC in 1 file.**

---

## Data dependencies

None. Voice input feeds into the existing `onSubmit(query: string)` callback — no new API, no new schema.

---

## Visual requirements

**No HTML design reference exists.** D144 screen inventory never landed (see D152 status section). Visual spec needed from Mark:
- Icon style: outline mic vs filled mic, line weight, color (#636366 idle, #ff453a listening)
- Icon size: suggest 18px to match the input's 15px font
- Pulse animation: suggest CSS `@keyframes` opacity pulse at 1s interval while recording

**Assumption (flagged):** Using SF Symbols-style outline mic icon rendered as inline SVG. Mark can swap the icon path without code changes.

---

## Blast radius

**Minimal.** Changes are contained to SearchBar.tsx. No API changes. No schema changes. No other components affected. The mic button is additive — existing text input behavior is unchanged.

**Risk:** Web Speech API is not universally supported. Firefox and some WebViews lack it. The spec requires hiding the mic entirely when unsupported, so no broken UX.

---

## Build order rationale

Ranked 5 of 6 because:
- Voice input is a nice-to-have, not a workflow blocker
- Depends on Mark confirming the icon style (no design reference)
- Web Speech API behavior varies across browsers — needs manual testing
- Can ship independently at any time without blocking other refinements

---

## Open questions

**Q1 (assumed answer flagged):** Should the mic auto-submit on speech end, or fill the input and wait for Enter? **Assumed: auto-submit** — the voice interaction should feel like talking to Milo, not dictating into a text field. If Mark wants the pause, change `recognition.onresult` to populate the input without calling `onSubmit`.
