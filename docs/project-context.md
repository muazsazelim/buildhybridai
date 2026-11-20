# #BuildHybridAI 2026: project context and lessons learned

Notes from the first week of exploration (Nov 13–20, 2026): the hackathon rules, what's set up, what we verified about Firebase AI Logic hybrid inference, what we built and threw away, and the ideas we're considering next. Read this before starting the next build.

---

## 1. Hackathon at a glance

- **Event:** #BuildHybridAI 2026 on Devpost. **Deadline: Nov 30, 2026, 2:00 pm PST.**
- **Core requirement:** use **Firebase AI Logic hybrid inference**: run prompts on-device when a local model is available, and fall back to the cloud otherwise.
- **All AI inference must go through Firebase AI Logic.** No direct calls to any other model API.
- **Platform:** Web (Chrome) first. iOS (Apple Foundation Models via Firebase AI Logic) is a maybe-later; not now.
- **App Check is required.** Debug tokens are accepted for the submission.
- **Repo:** must be public and open source (Apache 2.0 is already in place), and must show Firebase AI Logic actually imported and called at runtime, not just mentioned in the README.
- **Deliverables:** a working app (hosted URL encouraged), the public repo, a 3-minute demo video, and a text description.
- **Judging:** technological implementation (effective use of hybrid inference), design (a complete, coherent product), potential impact, and quality of the idea. **The on-device vs cloud split should be clear and visible.**

## 2. Current setup

- **Firebase project `buildhybridai`:** web app registered, Firebase AI Logic enabled with the **Gemini Developer API** (free tier). We want to stay on the free tier, and we don't use Firebase Analytics.
- **App Check:** the debug provider works on localhost. A debug token is registered for the main Chrome profile.
  - Every new browser or profile (incognito, VS Code's built-in browser, cleared site data) generates a **new** token that must be registered again.
  - Never commit tokens. To keep a stable one, put it in the local, git-ignored `.env` as `VITE_APPCHECK_DEBUG_TOKEN`.
  - Don't add `localhost` to the reCAPTCHA allowed domains.
- **Config:** a local `.env`, never committed, holds `VITE_FIREBASE_API_KEY`, `VITE_FIREBASE_AUTH_DOMAIN`, `VITE_FIREBASE_PROJECT_ID`, `VITE_FIREBASE_STORAGE_BUCKET`, `VITE_FIREBASE_MESSAGING_SENDER_ID` and `VITE_FIREBASE_APP_ID`. `.env` and `node_modules` are in `.gitignore`.
- **Chrome:** desktop Chrome with `chrome://flags/#prompt-api-for-gemini-nano-multimodal-input` enabled. Gemini Nano is downloaded and reports `available`.
- **Repo:** contains only `LICENSE` (Apache 2.0), `.gitignore` and this document. All prototype code was removed on purpose (see section 6).
- **Browser testing:** Browser MCP, the Chrome extension plus `@browsermcp/mcp`, is configured for Claude Code. The first connection needs approval in the extension popup with a pairing code.

## 3. Firebase AI Logic hybrid: verified facts (JS SDK `firebase` 13.3–13.4)

The docs are the source of truth; the points below were checked against the installed SDK types and runtime behavior.

**Setup**
- Imports come from `firebase/ai`: `getAI`, `getGenerativeModel`, `GoogleAIBackend`, `InferenceMode`, `InferenceSource`, `Schema`, `AIError`, `ThinkingLevel`.
- Init: `getAI(app, { backend: new GoogleAIBackend() })`, with App Check initialized first (debug provider: set `self.FIREBASE_APPCHECK_DEBUG_TOKEN = true` before `initializeAppCheck`).

**Modes and fallback**
- `getGenerativeModel(ai, { mode, onDeviceParams, inCloudParams })`.
- The modes are `PREFER_ON_DEVICE`, `ONLY_ON_DEVICE`, `PREFER_IN_CLOUD` and `ONLY_IN_CLOUD`.
- The default cloud model is `gemini-3.5-flash-lite`; override it with `inCloudParams.model`.
- **`model.initializeDeviceModel(onProgress?)`** is a method on the model. It takes a progress callback that receives a value from 0 to 1, and must be called from a user gesture.
- In `PREFER_ON_DEVICE`, requests don't wait for the download; they go to the cloud until the model is ready.
- **`PREFER_IN_CLOUD` swallows the cloud error when it falls back to on-device**, triggered by `fetch-error`, `error` or `api-not-enabled`, which includes 429s. If you need to know why the cloud failed, call `ONLY_IN_CLOUD` yourself and then `ONLY_ON_DEVICE`.

**Responses**
- **`response.inferenceSource`** is `"on_device"` or `"in_cloud"`, in **lowercase**, unlike the docs' `ON_DEVICE` / `IN_CLOUD`. Compare against the `InferenceSource` constant instead of a string.

**Structured output**
- In the cloud: `inCloudParams.generationConfig.responseMimeType` / `responseSchema`.
- On-device: **`onDeviceParams.promptOptions.responseConstraint`**, either a schema or plain JSON Schema.
- Enums of your own IDs, via `Schema.enumString` or a JSON-schema `enum`, stop the model from inventing values.
- `propertyOrdering` on `Schema.object` forces key order in the cloud. For Nano, pass plain JSON Schema; its `properties` order is the output order.

**On-device limits:** single-turn only (no chat, no tools), text plus JPEG/PNG input, structured output supported, and about a **9k-token** limit. The SDK creates a new session per request but keeps the previous one alive, so the model stays in memory.

**Requests and errors**
- The SDK's default request timeout is **180 s**. Pass `{ timeout, signal }` as the second argument to `generateContent` to set your own.
- Cloud errors arrive as `AIError` with `customErrorData.status`: 429 for quota, 403 for App Check or permissions, 404 for a bad model name, 500 for an overloaded model.
- `thinkingConfig: { thinkingLevel: ThinkingLevel.MINIMAL | LOW }` is accepted by the Gemini 3.x models we used, and reduces latency.

**App Check**
- Firebase AI Logic **requires App Check enforcement from Nov 2, 2026**.
- `useLimitedUseAppCheckTokens: true` on `getAI` is only needed for replay protection.

## 4. Models, quotas and measured performance (free tier, Nov 19–20)

| Model | Free daily limit | Measured |
|---|---|---|
| **Gemini Nano** (on-device, Chrome) | none | Short JSON turns: **first words about 0.4–0.6 s, done about 1–2.5 s**. Bigger JSON (a small "room"): **about 6–7 s**. A 2-paragraph story: **about 7 s**. Reliable throughout. |
| `gemini-3.8-flash` | **20 requests per day per project** (from the 429 error) | Used up quickly during testing; resets daily. Better writing quality. |
| `gemini-3.5-flash-lite` | Not confirmed (check https://ai.dev/rate-limit) | Short JSON turn: **about 10–12 s**. Room-sized JSON: **10.7 s, 22.7 s, then over 40 s**. Some **500 "experiencing high demand"** errors. Good quality output. |

**Streaming:** `generateContentStream` from `gemini-3.5-flash-lite` returned 500s and "Failed to parse stream" errors in testing, while plain `generateContent` worked. Streaming from Gemini Nano works well. When a stream fails, also handle `result.response` (for example `.catch(() => {})`), or it surfaces as an uncaught error.

## 5. Design lessons for the next idea

1. **Lots of short, frequent work on-device; very few cloud calls.** Nano is fast, private and free. On the free tier, the cloud is limited by daily quota and is often slow.
2. **Aim for 1–2 cloud calls per session.** One idea we liked: the cloud plans the whole session up front, and Nano plays every moment.
3. **Rules and numbers live in code; the AI only writes words.** The app stays fair and still works when every model fails.
4. **Always use structured output with enums** for anything the code acts on, and **sanitize model output** as untrusted data: arrays might not be arrays, enums might be wrong, strings can be too long.
5. **Set hard timeouts on every cloud call** and keep a non-AI fallback (plain text, or a small hand-written content set).
6. **Log why a fallback happened.** Silent fallbacks hid the real cause, the quota, for a long time.
7. **Stream only on-device.** Show results the moment they're known (dice rolls, rule effects) and let narration stream in.
8. **Background pre-generation hides latency but doubles quota use.** Use it deliberately.
9. **Make the hybrid split visible:** a badge (ON-DEVICE / CLOUD) on every AI output, timings, and a "how this session was written" summary. A test switch for hybrid / on-device only / cloud only helps the demo.
10. **Keep Nano prompts short.** It re-reads the whole prompt every call, and every output word costs time.

**Gotchas we hit**
- Writing a CSS `url("…")` inside an HTML `style="…"` attribute breaks the attribute, so every background silently disappeared. Use `url('…')`.
- Vite hot reload creates new module instances, so poking module state from the console can hit a stale copy.
- Avoid `confirm()` / `alert()` dialogs while testing with browser automation; they block the tools.

## 6. What we built and dropped

1. **Phase 1 proof of concept:** Vite + TypeScript, one `ai.ts` module with a local-first model and a cloud model, a download button with progress, source badges and error handling. It worked.
2. **"Tavern Test":** a single scene with stats, three characters, dice in code, and Nano narrating plus suggesting the next actions as JSON. It worked; we tuned it with shorter outputs, streaming and a warm-up.
3. **"Crooked Lantern":** a procedural pixel-art RPG.
   - A hand-made tavern, cloud-designed rooms built only from a code-defined art kit (themes, props, sprites), and procedural placement.
   - Explore actions, turn-based combat with spells and items, a recruitable companion and a boss.
   - A cloud-written end-of-run legend, and design mockups on a private claude.ai design canvas.
   - **Result:** everything worked in Chrome. Nano was excellent for turns, battles and fallback rooms.
   - **Why we dropped it:** the cloud side didn't fit the free tier. One large cloud call per room hit the 20-a-day limit on `gemini-3.8-flash` within a few runs, and `gemini-3.5-flash-lite` was too slow and unreliable for on-demand room design. The scope was also large for the deadline.

## 7. New ideas (shaped by the lessons above)

Each idea has lots of short on-device work and only 1–2 cloud calls per session.

| Idea | On-device (Gemini Nano) | Cloud (rare) | Why it fits |
|---|---|---|---|
| **Study Duel** (current favorite) | Turns pasted notes or a photo of a page into quiz questions, checks answers, explains mistakes, writes monster taunts | One end-of-session study report and plan | Game + students, private notes, high-volume structured on-device work, small scope |
| **Pocket Language Café** | NPC replies, grammar checks and hints every line | New scenario + progress report, once per scenario | Clear education impact. Nano is strongest in English, Spanish and Japanese. |
| **Mystery in a Box** | Every suspect answer and contradiction check | Writes the whole case at the start + the final reveal | Keeps the game feel; uses the "one cloud plan per session" pattern |
| **Snap & Sort** (fridge/pantry) | Recognizes items from photos, recipe ideas | A weekly meal plan on request | Everyday impact; shows on-device image input |
| **Kind Reply** (message tone coach) | Live tone check and calmer rewrites as you type | A deeper conversation-coaching session | Strong privacy story; the fast on-device response is very visible |

## 8. Open decisions and next steps

- [ ] Pick the idea.
- [ ] Choose the cloud model per task with the quota in mind. Check real limits at https://ai.dev/rate-limit. Consider `gemini-3.8-flash` for rare, quality-critical calls and Nano for everything else.
- [ ] Decide on hosting:
  - A public URL needs a **reCAPTCHA Enterprise** App Check provider; debug tokens can't be shipped in a public bundle.
  - Otherwise submit with a video and local setup steps.
- [ ] Rebuild the small `ai.ts` layer from section 3: hybrid models, explicit cloud-then-device routing with logged errors, timeouts, streaming only on-device, source badges.
- [ ] Leave time for the README, the 3-minute video (show the badges, a forced-cloud run and an offline run) and the Devpost text.
