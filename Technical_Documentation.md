# MEMORA — Cross-AI Memory Layer

> A Chrome extension that preserves our conversation context when switching between AI tools.

---

## 1. The Problem

People use multiple AI tools — ChatGPT, Claude, Gemini, Perplexity — because no single AI does everything well, and because usage limits force frequent switching. Every time a user switches platforms, their entire conversation history, project context, goals, and decisions are lost. There is no memory layer that works *across* AI platforms — users are forced to re-explain everything from scratch every time they switch, wasting time and breaking their workflow.

## 2. What MEMORA Does

MEMORA is a lightweight Chrome extension that sits silently on top of AI chat platforms. When enabled, it:

1. Reads your ongoing conversation on a supported AI tool (ChatGPT, Claude, Gemini, or Perplexity)
2. Sends it to the Gemini API for summarization (project, goals, decisions, progress)
3. Saves that summary to a local memory history, on-device
4. When you open a different supported AI tool, automatically types the latest summary into the new chat's input box — so the new AI is instantly caught up
5. Detects ChatGPT's own usage-limit message and immediately forces a fresh memory save the moment that happens, so context is never lost right when a user is forced to switch

The user stays in full control:
- A single ON/OFF toggle in the popup
- A searchable memory history — type a keyword to instantly filter past summaries
- A manual **"Inject"** button on any saved memory (not just the latest one), to push that specific context into whichever supported AI tab is currently open
- The ability to delete any individual memory or clear everything at once

## 3. How It Works (Flow)

```
User chats on ChatGPT
        ↓
content-chatgpt.js reads the conversation from the page (DOM),
also watching for ChatGPT's own "usage limit reached" message
        ↓
background.js sends the text to the Gemini API for summarization
        ↓
Gemini returns a structured summary (project, goals, decisions, progress)
        ↓
Summary is added to a history list in chrome.storage.local (on-device, not on any server)
        ↓
User opens Claude in a new tab
        ↓
content-claude.js detects an empty input box, retrieves the LATEST saved summary,
and types it in automatically
        ↓
User reviews and sends — Claude now has full context

  (Alternatively: user searches their memory history in the popup,
   picks an OLDER memory, clicks "Inject" — that specific summary is
   sent to whichever supported AI tab is currently active)
```

## 4. Tech Stack (What Is Actually Built)

| Component | Technology | Purpose |
|---|---|---|
| Extension core | JavaScript, Chrome Extension Manifest V3 | Runs entirely inside the browser |
| Conversation reading | DOM scraping (`content-chatgpt.js`, `content-claude.js`, `content-gemini.js`, `content-perplexity.js`) | Extracts chat text from each supported platform's page |
| Usage-limit detection | Text pattern matching on the ChatGPT page | Triggers an immediate memory save the moment a usage-limit message appears |
| Summarization | Google Gemini API (free tier, user's own key) | Compresses conversation into a structured memory snapshot |
| Storage | `chrome.storage.local` | Stores memory history on-device; no external database |
| Context injection (automatic) | Per-platform content scripts | Auto-fills the new platform's input box with the latest memory |
| Context injection (manual) | `chrome.tabs` messaging from the popup | Lets the user pick any saved memory and push it into the active tab on demand |
| Search | Client-side text filtering in `popup.js` | Instantly filters memory history by keyword |
| Settings | `options.html` / `options.js` | Where the user pastes their own free Gemini API key |
| UI | HTML/CSS with a local Tailwind utility stylesheet (`tailwind-local.css`) | Popup with ON/OFF toggle, memory preview, history list, search box, delete controls |

**Note:** This is a serverless, bring-your-own-key architecture — no backend server, no cloud database, and no user data leaves the browser except for the direct summarization call each user makes to Gemini using their own key.

## 5. Setup & Testing Instructions

1. Unzip the submitted project folder
2. Open Chrome → go to `chrome://extensions/`
3. Enable **Developer Mode** (top right toggle)
4. Click **Load Unpacked** → select the project folder
5. Click the MEMORA icon in the toolbar → click **⚙ Settings (API Key)**
6. Get a free Gemini API key at `aistudio.google.com/apikey` (no card required), paste it in, click **Save Settings**
7. Back in the popup, toggle MEMORA **ON**
8. Open `chatgpt.com`, have a short conversation (a few exchanges, more than ~200 characters)
9. Wait ~15-20 seconds, then open `claude.ai` in a new tab
10. The conversation summary will appear automatically in Claude's input box
11. Click the MEMORA icon → **"📜 View Memory History"** to see saved summaries, with options to delete individual entries or clear all
12. Try typing a keyword into the search box at the top of the history list to filter, and click **"📤 Inject"** on any entry to manually push that specific memory into the currently active AI tab

## 6. Current Limitations (Prototype Scope)

- Supports four platforms: ChatGPT, Claude, Gemini, and Perplexity
- Relies on each platform's current page structure (DOM selectors); if any of these significantly redesign their interface, the relevant selector in that platform's content script would need a one-line update (selectors include fallback chains to reduce this risk)
- Summarization quality depends on the Gemini free tier and conversation length; very short conversations (<200 characters) are skipped intentionally
- No user accounts or cross-device sync in this version — memory history is local to the browser it was created in, capped at the most recent 20 entries
- Subject to Gemini's free-tier daily quota; each user brings their own key, so usage doesn't share a pool across users

## 7. Feasibility

MEMORA is feasible to build and maintain because it deliberately avoids the heaviest parts of typical AI-product infrastructure:

- **No backend server to build, host, or keep alive.** Everything runs as a Chrome extension executing inside the user's own browser. There is no server that can go down during a demo or in production.
- **No database to design or manage.** `chrome.storage.local` is provided by Chrome itself; the only "schema" is a small JSON array of memory entries.
- **Bring-your-own-key model removes cost and billing risk entirely.** MEMORA itself never pays for or manages any API usage — each user supplies their own free Gemini key, so there is no shared cost that scales with user count.
- **DOM-scraping approach is realistic for a hackathon timeline** because it only needs to read visible text, not integrate with any platform's private API (none of ChatGPT, Claude, Gemini, or Perplexity currently offer a public conversation-export API for this use case).
- **The riskiest technical assumption — that this cross-platform injection actually works — has been built and verified**, not just designed on paper. All four platforms have working read/inject logic with fallback DOM selectors to reduce breakage if a site updates its interface.

The main feasibility risk is **selector fragility**: if a platform meaningfully redesigns its chat interface, the relevant one-line selector needs updating. This is a known, bounded maintenance cost, not a fundamental flaw in the approach.

## 8. Scalability

**What scales well as-is:**
- Because there is no shared backend, MEMORA's only "infrastructure" is the Chrome Web Store distribution itself — adding the 1,000th or 100,000th user costs the project nothing extra, since each user's Gemini usage is on their own free key, not a shared resource
- Memory history is capped at 20 entries per user by design, keeping per-user storage small and consistent regardless of how long someone has used the extension

**What would need to change to scale further (see Future Roadmap):**
- **Cross-device sync** is the main current limitation — memory is local to one browser. A lightweight cloud sync layer (e.g. Supabase) would be the natural next step, used only for storing each user's own memory history, not for any shared processing
- **Search would benefit from semantic (vector) search** as a user's history grows beyond what simple keyword matching can usefully filter, particularly if memory caps are raised
- **Platform coverage** can scale to additional AI tools by adding one new content script per platform, following the same pattern already used for the current four — this is a linear, well-understood cost per new platform, not a redesign

## 9. Future Roadmap

These are planned directions beyond the current prototype, not implemented yet:

- **Cloud sync** (e.g. Supabase) — so memory history follows the user across devices and browsers
- **Vector-based (semantic) memory search** (e.g. Pinecone) — for recalling specific past context by meaning, not just exact keyword matches
- **User accounts and authentication** — for multi-device continuity
- **Smarter long-conversation handling** — chunked summarization for very long chats, instead of trimming to the most recent portion
- **Browser support beyond Chrome** — Firefox/Edge equivalents

## 10. Why This Matters

Existing AI memory features (e.g. ChatGPT's built-in memory) work only within their own platform. MEMORA is positioned as the layer that sits *above* individual AI tools, addressing the specific gap of cross-platform context loss — a problem that grows as users increasingly combine multiple AI tools in their daily workflow.

---

*Built for [Hackathon Name] — Round 2 Prototype Submission*
