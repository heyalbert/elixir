# Plan — Modern 2D MMORPG Creation Platform

Status: planning, no code yet. Working name: TBD.

Goal: a modern, browser- and mobile-first platform in the spirit of GraalOnline: a 2D top-down action MMORPG world that creators build live, with custom scripts and assets, and that players can use as a base to build their own games. Original name, art, default world and terminology; nothing from Graal or Nintendo.

Background research: [research/graal.md](research/graal.md), [research/graal-141.md](research/graal-141.md), [research/alttp.md](research/alttp.md).

---

## 0. Principles

Filter for every spec and feature decision. Derived from Graal 1.41 (2000), modern Graal, and the forum pain points (see research docs).

**Keep (what Graal got right)**
1. **Create inside the world.** Edit the level you stand in and play-test instantly — no separate studio app, no build or publish wait.
2. **Tiny first step.** A door, sign, shopkeeper or guard is a few lines. Depth sits behind the simple path, never in front of it.
3. **Readable, remixable content.** Creators learn by reading others' work. Open and forkable with attribution by default; private as an option.
4. **Small ownable units.** Chunks of 32×32 tiles that one person can understand and own; worlds composed from them.
5. **Social life is the core game.** Identity, cosmetics, guilds, chat, PvP and social rules are first-class.
6. **Instant start.** A world is a URL; plays in the browser with no install.

**Change (what we do differently today)**
7. **Server authority.** Shared logic runs on the server; client code is presentation and input. Client-writable data is typed untrusted.
8. **Modern, typed scripting.** TypeScript with named event handlers, async/await, typed scoped state that syncs and persists automatically.
9. **Data-driven content.** Items, enemies, damage rules, animations and PvP/social rules are definitions plus script hooks. The default kit is a forkable package, not engine code.
10. **Touch-first, every device.** One web client; 32px tiles; touch, keyboard and controller; any screen size.
11. **Versioned, collaborative tools.** Every change has history, rollback and branching; several creators edit live together.
12. **Security by construction.** Rate limits, quotas, server-only permission checks, sanitised rich text, no device access, namespaced assets with provenance — enforced by the platform, not by human inspection.
13. **Free start, transparent discovery.** Idle worlds cost nothing; discovery is metrics-based with published criteria.

**Add (what didn't exist in 2000)**
14. **AI assists creators** (script writing and explaining, style-locked sprite generation, NPC dialogue) — it never replaces authored content.
15. **Share-by-link growth**, spectating and creator-facing analytics (retention, session length, drop-off points).
16. **Creator economy designed in from day one** (ledger and attribution now, payouts later).
17. **Accessibility and localisation from the start.**

**Avoid**
18. No full-engine-editor complexity on the default path.
19. No lock-down that kills learning and remixing.
20. No graphics investment ahead of creation ease.
21. No scale or marketplace work before one world with ~50 players is fun.
22. No AI-generated filler content.

## 1. Non-negotiables (the "spirit")

1. The world is editable while it runs; scripts and levels hot-reload with players inside.
2. One script file holds server and client logic, with a simple typed bridge between them.
3. Assets stream from the world; content changes never require a client update.
4. Anyone can own a world and appoint staff with fine-grained rights.
5. A default top-down action kit (sword, hearts, lifting/throwing, enemies, items) that worlds can use, change or replace.
6. Modular characters with animations shared across outfits.
7. One identity across many worlds, with a world browser.

## 2. Decisions

| # | Topic | Decision | Status |
|---|---|---|---|
| 1 | Tile size | **32px** tiles; character body ~32×48 in 64×64 animation frames; internal render ~640×360, integer scaling | **Decided** |
| 1a | World units | 32×32-tile chunks (1024×1024 px) on a seamless map; 16px collision sub-cells; layers: ground, detail, objects, overhead, optional second floor plane (see research/alttp.md §10) | Proposed |
| 2 | Backend | Elixir/OTP + Phoenix Channels | Proposed |
| 3 | Scripting language | TypeScript, executed in sandboxed QuickJS (server: per-world runtime with CPU/memory quotas; client: QuickJS-WASM in a Web Worker) | Proposed |
| 4 | Client | Phaser (or PixiJS + thin layer), HTML/React overlays for chat, menus and editor panels; PWA, then Capacitor for app stores | Proposed |
| 5 | Editor UI state / collaboration | React + Zustand panels; Yjs CRDT for collaborative level editing, server remains authoritative for published state | Proposed |
| 6 | Map format | Tiled-compatible JSON + our extensions (tile attributes, warps, NPC slots, layers) | Proposed |
| 7 | Graal file importers | None (clean break; 16px assets don't fit 32px) | Proposed |
| 8 | Storage | Postgres (accounts, worlds, versioned levels/scripts); S3-compatible object storage + CDN, content-addressed; per-world SQLite optional for world data | Proposed |
| 9 | Isolation | BEAM process tree per world; container boundary (Incus/Docker) for script runtimes and per-tenant quotas; storage snapshots for world fork/backup | Proposed |
| 10 | AI | Claude for AI NPC dialogue/behaviour and script generation/debugging; gateway for per-world metering | Proposed |
| 11 | Business model | Free creator tier; paid tiers for CCU, storage, script CPU, AI credits; in-world real-money sales deferred | Open |
| 12 | Scale target | CCU per world and overall for first milestone | Open |
| 13 | Team and timeline | Solo + AI vs team; hours/week; target date | Open |

## 3. Architecture

**Server (Elixir/OTP)**
- Supervision tree per world: world supervisor → chunk/instance processes (GenServer per 32×32-tile chunk or interior/dungeon instance) → player session processes → script runtime processes.
- Transport: Phoenix Channels over WebSocket. Fan-out via Phoenix PubSub. Clustering via libcluster/Horde later.
- Interest management: subscribe to current chunk + neighbours.
- Tick rates: client simulation fixed 60 Hz with interpolated rendering; server script loop 10 Hz (Graal's proven 0.1 s); network state 15–20 Hz.
- State split: per-player flags (chests, keys, collectibles, secrets) vs shared world state (bushes, enemies, events); dungeons as per-party instances.
- Authority: client-predicted movement validated by server (speed, collision, warps); combat, items, currency and all server-side script state authoritative.
- Idle worlds hibernate (near-zero cost) → enables a free tier.

**Scripting model**
- Event-driven (`onCreated`, `onPlayerEnters`, `onTimer`, …), one module with server and client sections, typed `triggerServer` / `triggerClient`.
- Scopes mapped from Graal: object, player (server-written, client-readable), client-writable (typed **untrusted**), world, level/chunk.
- Script locations: level objects, world (global persistent) objects, player-attached systems, shared classes/mixins.
- Security by construction: untrusted data cannot reach permission checks; rate limits on client→server messages; minimum server timer interval; per-world CPU/memory quotas.

**Client**
- Phaser + React overlays. Virtual joystick + context action buttons on touch. Responsive script UI framework replacing Torque-style GUI controls.
- Assets loaded on demand by content hash; aggressive caching.

**Creator tools (inside the client, "edit mode")**
- Tile painting with layers and tile-attribute painter, warp tool, object/NPC placement, animation editor, Monaco script editor with types, asset upload with validation (palette, grid, frame size, anchors), logs with stack traces, profiler.
- Version history and rollback for every level and script; world branching/forking.
- Creator loop from Graal 1.x: syntax check with highlighted errors, instant play-test, breakpoint debugger, state/flags inspector, path recording for NPC routes, multi-tile stamps, attribution metadata on every script/level/asset.
- No-code on-ramp: NPC/enemy/item wizards that generate editable TypeScript.
- Items, enemies and damage rules are data definitions with script hooks, never fixed engine enums.
- Admin console (modern RC): roles with named rights + path-scoped permissions, bans, moderation rules, staff audit log.
- Local CLI that syncs a world's scripts/assets to a folder for VS Code + Git users; published TypeScript API types.

**Platform**
- Global accounts, per-world profiles, friends/PMs/guild tags, world browser with metrics-based discovery.
- Per-world asset namespaces, provenance, licensing.
- Trust & safety: age gate (COPPA/GDPR-K), chat filter with graded actions, reports, image moderation (including AI-generated), app-store UGC requirements.

## 4. Assets

- 32px tiles, 64×64 character frames, fixed palette per world (32–64 colours), top-left light, 3/4 top-down perspective.
- Pipeline: style bible → hand-made seed set (base body template + core animations, ~50 tiles) → custom style model (Scenario or LoRA) → bulk generation (PixelLab, Retro Diffusion) → Aseprite cleanup → automated validation on upload.
- Later: constrained AI generation inside the editor (locked to world palette/grid/template), with server-side snap-and-validate.

## 5. Roadmap

| Phase | Deliverable | Proves |
|---|---|---|
| 0 | Specs: script API, map/animation formats, network protocol, OTP process tree, style bible | Decisions locked |
| 1 | One world: seamless 32px map, multiplayer movement, tile collision, warps, chat, browser client | Netcode + rendering |
| 2 | Scripting runtime: level/world/player scripts, server/client bridge, persistence, logs + stack traces | Core platform |
| 3 | In-client level editor + script IDE + asset upload + hot reload + version history | Live world editing |
| 4 | Default action kit in scripts: sword, hearts, lift/throw, enemies, items; animation system; outfits | The action feel |
| 5 | Multi-world: world creation, free tier, staff rights, admin console, world browser, local CLI | Player-owned worlds |
| 6 | Mobile polish, PWA/Capacitor, moderation, monetisation, AI NPCs and AI asset generation | Shippable product |

**MVP** = phases 1–4 for one hosted world, including version history/rollback and script logs (the top unmet needs of Graal creators).

## 6. Competitive position

Graal Next (next.graalonline.com) targets the same "2D UGC platform" goal from the incumbent side: 30M accounts, 5 live worlds, but constrained by a C++ engine and GS2. Live in 2026: browser level editor, AI NPC engine, GS2 AI assistant. Planned: WASM browser client, PlayerWorlds revival, LSP, world cloning, asset pipeline, creator revenue share. Their creator community (forums) is dormant and not engaged with Graal Next.

Our edge: mobile-first web client from day one, mainstream scripting language, creator workflow (history, debugging, local dev, docs) shipped first, free start, owned creator community.
