# NAVI — AI Voice Advisor for Luxury Real Estate
 
NAVI is a conversational AI voice agent built for a luxury waterfront condominium development in Mexico. Prospective buyers can talk to it directly: ask about unit types, ocean-view availability, ownership models, amenities. The interface responds by showing relevant property imagery in sync with the conversation, working like a virtual sales rep standing next to a screen.
 
**Talk to NAVI:** [va-navi-showcase.netlify.app]
 
There's no source code in this repo. NAVI was built for a real paying client, so the production system prompt, business logic, brand assets, and API credentials belong to them and aren't public. This README documents the architecture, the features, and the engineering decisions in detail instead. Everything below reflects the real, shipped implementation.
 
## What it does
 
A visitor lands on a welcome screen, taps "Start conversation," and starts talking. As the conversation develops, a few things happen automatically. The interface swaps the displayed image or video to match whatever's being discussed: mention the pool and the pool appears, ask about the units and interior renders show up, with no button presses or manual navigation involved. A small set of suggested-question chips let a visitor tap instead of talk, for anyone who'd rather not use voice. A live transcript panel shows what NAVI has said, scrollable back through the conversation. An animated orb visualizes NAVI's current state (idle, listening, or speaking) at a glance.
 
## Core features
 
**Real-time voice conversation.** Built on the ElevenLabs Conversational AI platform (`@elevenlabs/client` SDK) over WebRTC. The LLM generating responses is configured at the agent-platform level rather than called through a separate API.
 
**Conversation-synced visual staging.** This is the centerpiece of the demo. The agent has access to a client-side tool (function calling) called `show_image`, with a `categoria` parameter covering six visual categories: facade, pool, amenities, interior, location, and a default welcome state. The LLM decides on its own, based on what it's currently talking about, when to call this tool. The frontend just listens and swaps the displayed media. There's no hardcoded "if user says X, show Y" logic behind it; it's genuine function-calling driven by conversational context.
 
**State-aware animated orb.** A custom SVG orb with an ocean-wave motif that visibly changes behavior depending on conversation state: a slow idle drift, a sonar-ping ripple while listening, a fast bob while NAVI is speaking. It's built with CSS transform animations rather than animating raw SVG geometry attributes (`r`, `d`) directly, after running into inconsistent cross-browser rendering with that first approach.
 
**Live transcript with experimental real-time sync.** A scrollable transcript box logs NAVI's spoken responses. By default, the ElevenLabs SDK delivers each response as one complete block once generation finishes. A secondary sync path was added using the SDK's `onAudioAlignment` event, which provides character-level timing data tied to actual audio playback, revealing the text letter by letter as NAVI actually says it instead of in one late dump. The exact payload shape isn't fully documented for this SDK version, so the implementation is defensive: it logs unexpected formats and falls back cleanly to full-block display if the alignment event isn't available.
 
**Deterministic date handling.** Rather than letting the LLM infer relative dates ("next Tuesday," "in two weeks") on its own, which is a common source of hallucinated dates in LLM agents, a small JS function pre-computes a rolling calendar reference and injects it as a dynamic variable at session start. NAVI reasons over real, correct dates instead of guessing.
 
**Fail-quiet connection handling.** Early versions revealed the entire dashboard interface the instant "Start" was clicked, so a failed connection meant flashing the full UI just to show an error message on top of it. The flow was restructured so the welcome screen shows a small inline status ("Connecting…") and, on failure, a soft error badge. The full interface only reveals itself once a connection is confirmed, never before.
 
**Mic mute that actually works.** An early mute implementation disabled the local browser media track directly. It looked muted in the UI but audio kept reaching the agent, because the SDK manages audio through its own WebRTC pipeline, which doesn't necessarily respect the raw track state. The fix uses the SDK's own `conversation.setMicMuted()` method as the source of truth, with the local track toggle kept only as a redundant safety layer.
 
**Responsive two-panel layout.** A sidebar (voice controls, orb, transcript, suggested questions) alongside a full-bleed image or video panel, collapsing to a single column on mobile.
 
## Architecture
 
The frontend is a single-page vanilla HTML/CSS/JS app. No framework and no bundler are required to run it locally.
 
The voice and AI layer runs on the ElevenLabs Conversational AI Agents platform, through the `@elevenlabs/client` SDK over WebRTC. The LLM is selected and configured within the agent platform itself, not called directly through a separate API.
 
Hosting is a static deploy on Netlify, with continuous deployment from GitHub. Netlify runs `npm run build`, which pipes the HTML source through `html-minifier-terser` for HTML and inline CSS/JS minification and JS variable mangling before publishing. There's no framework build step involved; this exists purely to avoid shipping readable source to a public URL, since the page itself is publicly reachable even though the repo isn't.
 
## Engineering notes worth mentioning
 
A few things came up while building this that felt worth documenting, since they're the kind of debugging that doesn't show up in a plain feature list.
 
An unpinned `@latest` SDK import silently broke functionality mid-project. A major version release changed the SDK's internals, and because the import wasn't version-pinned, the page picked up the change automatically. A state-tracking callback stopped firing as a result. It was fixed by pinning to a known-good version, and by pinning going forward.
 
WebRTC audio doesn't behave like a normal media track. Both the mute bug and the live-transcript sync issue traced back to the same root cause: ElevenLabs' WebRTC transport handles audio through its own pipeline, separate from the raw browser MediaStreamTrack. Assumptions that normally hold for `getUserMedia()` code don't always apply here.
 
SVG geometry-attribute animation has inconsistent browser support compared to transform-based animation. That's why the orb was rebuilt around `transform: scale()` and `translateY()` instead of animating `r` or `d` directly.
 
## Known limitations and what's next
 
Property media is currently gradient placeholders standing in for real photography. Swapping in actual photos and video is the next visual pass.
 
The six image categories work well enough, but could go more granular: more specific material per unit model instead of one generic set of interior shots for all of them.
 
There's no visual date picker yet. Availability and scheduling currently happen conversationally by voice. A pop-up calendar UI for date selection is planned, likely paired with a real booking integration.
 
Floor plans aren't wired up per unit model. The `show_image` tool would need a second parameter, something like `modelo`, added to its schema once real floor-plan assets exist for each unit type, so the agent only asks "which model?" when it actually needs to.
 
There are no automated tests. The build has been verified manually against a working reference implementation. Basic smoke tests around the connection flow and tool-calling would be worth adding before reusing this pattern for another client.
 
## Why this project
 
This was built as real, shipped client work, not a tutorial project, which meant dealing with real constraints. A WebRTC transport that doesn't behave like plain browser APIs. An SDK that changed under my feet mid-project. Cross-browser animation quirks. A client who cared about polish, down to never flashing the whole UI just to show an error. The notes above are meant to reflect that: what shipped, what broke, and what's next, not just a list of features.
