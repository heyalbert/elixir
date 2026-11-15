# GraalOnline — Research Notes

Status: research, November 2026. Sources: graalonline.net wiki (107 developer/operations pages read as raw wikitext), the GraalScript community wiki (graalonline.net/gscript), forums.graalonline.com, next.graalonline.com, home.graalonline.com. Wiki pages are mostly 2005–2013 era; "today" claims are marked where they come from older pages.

Purpose: understand what made Graal work as a *creation platform* so we can build an original, modern equivalent (browser + mobile). We do not reuse Graal's name, assets, file formats or terminology.

---

## 1. Product overview

- Created 1997 by Stefan Knorr (Unixmad). Operated by Cyberjoueurs (Managing Director Stéphane Portha handles payments; "Stefan" is Technical Director).
- Official worlds today: **Era, Classic, Zone, Ol'West, Delteria** (wiki main page). Official worlds have historically been built largely by volunteer players (wiki: Graal).
- Platforms: Windows / Mac / Linux desktop client (C++), iOS (since 2008), Facebook (2011), Android (2013). A Unity-based client ("Quattro3D", "Worlds Client") was in development 2014–2022 and is described as alpha.
- Current homepage (home.graalonline.com) links: App Store, Google Play, **Steam** (Era: app 2358050; Classic/Zone/Ol'West under "GraalOnline Worlds": app 2698140), Discord (discord.gg/graal), and "Play on Web" links (`<world>.graalonline.com/play`, not verified working).
- Claimed scale (next.graalonline.com): 30M+ registered players across 5 worlds, 6,449 production scripts, 25 years.

## 2. World model

| Concept | Details |
|---|---|
| **Level** | 64×64 tiles × 16px = 1024×1024 px, file `.nw` (older `.graal`; `.zelda` no longer supported). Contains tiles, links (warp to another level on touch), signs, NPCs with embedded scripts; built-in baddies and chests are **deprecated** in favour of scripted NPCs. Tile **layers** supported in newer clients. Scripts can modify tiles and persist them (`updateboard2`, `savelevels`). |
| **Tileset** | Default `pics1.png`; scripts can replace/overlay tilesets per level-name prefix (`addtiledef`, `addtiledef2`). Two tileset layouts (type 0 classic, type 1 newer/Era). Tile type per tile index drives collision and behaviour; scripts query `tiletype(x,y)`, `onwall`, `onwater`. The full tile-type list exists only as a legend image. |
| **GMap** | `.gmap` stitches levels into one seamless map; listed in server options `gmaps=`. `loadmap()` can preload a whole gmap client-side. Player sync radius `syncdistancex/y` (default 192×192 tiles). Older "bigmap"/minimap system is legacy. |
| **Z-axis** | `player.z` exists (can be locked); projectiles have `zspeed`; global `gravity`. |

Built-in player properties and mechanics: hearts/fullhearts, swordpower, shieldpower, glovepower, bombs, darts (arrows), gralats (currency), AP (alignment points), MP, HP, spin attack; carrying bush/stone/vase/blackstone/sign/NPC; `canbepushed/pulled/carried`; horses; bush and baddy item drops with configurable rates; `putexplosion`; `shoot()` projectiles.

## 3. Characters and animation

- Character = head image, body image, sword, shield, `attr[]` (attr1 = hat), five recolourable parts (skin, coat, sleeve, shoe, belt). `gif2mng` tool fixes body palettes for recolouring.
- **GANI** animation files (text, header `GANI0001`): sprite sources reference placeholders (`BODY`, `HEAD`, `SHIELD`, `ATTR1`, `SPRITES`) or literal images, so one animation works for every outfit. 4 directions unless `SINGLEDIRECTION`; `SETBACKTO`, `DEFAULT*` parts, `COLOREFFECT`, per-frame `WAIT` and `PLAYSOUND`. Loop/continuous are editor options. **GANIs can contain scripts.**
- Default ganis named in docs: idle, walk, swim, grab, sword, bow, bomb, sit.

## 4. Scripting (GS1 → GS2)

- **GS1**: event checks (`if (created)`), flags with `set`/`setstring`.
- **GS2** (Graal 4+): `function onCreated()` style, "ECMAScript syntax", partially Torque-script compatible, true object orientation, variant types, multi-dimensional arrays, GUI system, particle engine. Strings join with `@`, `SPC`, `NL`, `TAB`. Loop cap 10,000 iterations.
- **One file, two sides**: code after `//#CLIENTSIDE` runs on the client. Bridge: `triggerserver("gui"|"weapon", name, …)` → `onActionServerSide`; `player.triggerclient(…)`; legacy `triggeraction(x,y,…)`; `obj.trigger(event, …)`; `scheduleevent`.
- **Where scripts live**:
  - Level NPCs (inside level files)
  - Database NPCs (global, persistent, ID ≥ 1000, `this.` vars persist)
  - `putnpc2` dynamic NPCs
  - **Weapons**: scripts in the player's "invisible backpack"; in practice the client-side systems (movement, HUD, menus, guns)
  - **Classes**: mixins via `join()`, stored in `scripts/`; `player.join()` with `public function`
  - Control-NPC (server-wide events), scripts inside GANIs
- **Variable scopes**: `this.`, `thiso.`, `temp.`, `player.`, `playero.`, `client.` (client-writable), `clientr.` (server-written, client read-only; client changes are local and overwritten), `server.`, `serverr.` (pushed to every client — bandwidth cost), `level.`, globals, legacy flags.
- **Async & IO**: `settimer`/`onTimeout` (server min 0.1 s), `scheduleevent`, `cancelevents`, `sleep`, `waitfor(obj, event, timeout)`; `requesturl`, `requesthttp`; `requestsql`/`requestsql2` (SQLite via `database=` server option), `requestmysql`; `savevars`/`loadvars`/`loadlines`.
- **Client UI**: Torque-style `GuiControl` tree (~30 controls: GuiWindowCtrl, GuiScrollCtrl, GuiMLTextCtrl, GuiTabCtrl, …) styled by `GuiControlProfile`. `showimg` drawing layers: index < 4 world coordinates, ≥ 4 screen; index < 200 visible to others in the level, ≥ 200 local only. Any showimg can be a particle emitter.
- **Events (sample)**: onCreated, onTimeout, onPlayerEnters, onPlayerChats, onActionServerSide, onWeaponFired, onPlayerTouchsMe, onActionProjectile.
- API size: several hundred functions/properties, documented across a stale community wiki (last edit Dec 2015), MediaWiki pages, and in-engine `/scripthelp` / `-listscriptfunctions`.
- **Graal3D**: (a) an older Torque-based 3D project, cancelled; (b) a Unity-based "Quattro3D" binding letting GS2 call Unity-like APIs and load Unity AssetBundles (no scripts inside bundles); alpha status.

## 5. Server architecture

| Component | Role |
|---|---|
| Login server | Server list, account and subscription status, news, official install packages |
| GServer | Levels, players, props sync, temporary level state (bombs, cut bushes, items), PMs, game time, server flags, file transfer |
| NPC-Server | Separate process that **logs into the GServer like a normal user**; loads levels and DB NPCs; runs all scripts in a **0.1 s loop**; sends changed NPC/player attributes back |

- Playerworld owners start/stop GServer and NPC-Server separately from a web panel showing CPU/memory.
- Content delivery: files download individually on demand (`fileupdate()`, `isdownloadingfiles()`), plus `.gupd` **update packages** (header `GRPKG001`; NAME, VERSION, PLATFORM, SUBPACKAGE, MODE base/required/optional, FILE lines).
- Folder configuration: typed glob lines (`head heads/*`, `level *.nw`, `file *.png`) controlling what clients can download.
- **Historic flaw**: one client file cache shared across all servers, so files from one world could overwrite another's (including default heads/ganis). Official advice: prefix files with the server name.

### Server options (selected — reveal engine features and limits)
Start/unstick location, `heartlimit`, `swordlimit`, `bushitems`, `baddyitems`, item/spawn rates, `respawntime`, `horselifetime`, `jaillevels`, AP system (`apsystem`, `aptime0-5`), `gmaps=`, `syncdistancex/y`, `savelevels`, `newnpcstorage`, `protectdbnpcs`, `database=`, translations (`translatedlanguages`, `serverlanguage`), `sleepwhennoplayers`, `weaponorder`, `protectedweapons`, `speedhacktolerance`, `onlystaff`, `staffguilds`, `profilevars`, `playerlisticons`, `globalguilds`. No documented player cap.

Abuse-control options (each marks a past exploit): `putbombenabled`/`bomballowedtypes`, allowed `showimg` types/images/ganis with logging, `noexplosions`, `puthorseenabled`, `sendplayertriggers`, `dontaddserverflags`, `dontchangekills`, `lockplayerz`, `speedhacktolerance`.

## 6. Admin tooling (RC / NC)

- **RC (Remote Control)**: external desktop client (Windows/Linux; RC3 beta) plus a scripted in-client RC (F6). File manager with **per-account folder rights** (`rw levels/*`, `r levels/x/*`), folder configuration, server options, server flags, player attributes view/set/reset, comments, bans with reasons, admin messages, mass PM, RC chat, logs (`scripterrors.txt`, `scriptfunctionslog.txt`), `/staffactivity`, `/scriptscan`, function profiler, `/updatelevel`, `/savenpcs`.
- **Rights** are named checkboxes (Warpto, Warp players, Update level, Disconnect, View/Set attributes, Admin message, Set server flags, Change rights, Ban, Change comments, Change staff accounts, Change server options, Edit folder config/rights, NPC-Control).
- **NC (NPC-Control)**: edit Classes, Weapons, Database NPCs, level NPC lists; rights use the same glob format (`rw WEAPONS/name/*`).
- Staff login restricted by IP range or PC ID. DB NPCs are the most common attack target and have no quick backup.
- Offline **Graal Editor** (outdated: no GS2, no layers, 256 KB tileset limit), GraalShop (ganis), generators, GraalSuite.

## 7. Staff and community model

- Per-world teams: Owner/Manager/Co-Manager; **NAT** (NPC Administration Team = scripters), **LAT** (levels), **GAT** (graphics); on Classic also Graal Police, FAQ, Events Team.
- Global: Playerworld Administration (PWA), Global Scripting Team, Global News Team, Global Development Team (disbanded 2011).
- Word filter: `wordfilter/rules.txt`; `wordfiltertext(text, type)` returns a bitmask (log, tell admins, replace, warn, jail, ban) per channel (chat, pm, toall, nick).
- Social: PMs, friends, guild tags shown as `nick (TAG)` from server-side guild files, global guilds, profiles, toall/toguild chat, offline PMs, global player list (cross-world PM requires Gold/VIP), Era world chat and in-game forums, Graal 4 IRC-style chat.

## 8. Business model

- **Playerworlds**: rent a dev server via **Developer Gold — €49 / 6 months or €79 / year** (earlier $99; before that, by application). Server appears as "Dev <account>".
- **Status ladder**: Dev (only Gold accounts can enter) → **Hosted** (public trial; requires quality, playable content, stable staff, original concept; revoked if stagnant) → **Classic** (requires player counts, unique concept, GS2 compliance, competent admin). Promotion decided by PWA/Stefan. Scripting **inspection** required before Hosted.
- Every staff member needs Gold to use RC. Free PC accounts: 5 hours/month without Gold.
- Historic: Gold subscription purchased with **Gelats** (cross-world purchase currency; PayPal, SMS, phone).
- Today (per-world): free-to-play mobile, **gralat packs** via app stores (Classic: 2,500 / 8,000 / 27,500), one-time unlock packs (customisation, house, character slots, guild house), VIP bought with gralats (Classic 7,500/month), real-money trading banned. No creator revenue share documented.

### Playerworld scripting inspection — what it forbids
Client-side staff checks; staff flags the client can write (must be `clientr.`); open-ended triggers accepting client-chosen targets; server-side timeouts < 0.5 s; flooding triggers/client var updates (~1/s suggested); overuse of `serverr.`; mixing GS1 and GS2. **These are security rules enforced by human review that a modern engine can enforce by API design.**

## 9. Community state (forums)

- vBulletin forum: 88k threads, 1.55M posts. **Effectively dormant**: dev subforums stopped 2015–2019 (Future Improvements last post 06-2017; NPC Scripting last post 03-2025); ~3–4 active posters. Community appears to have moved to Discord.
- No forum discussion of Graal Next.

### Recurring developer pain points (evidence: forum threads, showthread.php?t=…)
1. No version control / backup / change history — "Request - GIT" (134268274), "SVN Access for Code and Levels" (78791).
2. Weak RC code editor — "Intellisense in the RC script windows" (134267142); community-built scripted RCs (73833, 134269574).
3. Poor debugging/observability — "'echo' as a log-able function" (134267946).
4. No offline/local dev, no free sandbox — "Offline GS2 Editor" (134266707), "Test bed server revival" (134269386).
5. Thin documentation — "Documentation Request" (65778).
6. Stagnant language — GS3 proposal (134268072, 135 replies); requests for Lua/mainstream language.
7. Hosting cost, no self-hosting, opaque listing gatekeeping — "Server hosting" (134270175), "Where are the UC worlds at?" (134270394), "websockets?" (134270388).
8. Asset and script theft — "Protecting Server Content?" (134270307), "Stolen graphics" (134270439), "Protected Files" (134265188).
9. Platform fragmentation PC/mobile/Facebook — "Open mobile servers or open-source" (134270161).
10. Company neglect, vaporware new client — "We need more transparency" (134270436), "Where's the New Client?" (134270520), "New 'Unity Client'" (134270632).

## 10. Graal Next (next.graalonline.com, 2026)

Modernisation programme: "20 projects transforming a 25-year-old MMO into a modern AI-powered platform … to compete in the 2D UGC space". 3 live, 1 building, 16 planned.

| # | Project | Stack | Status |
|---|---|---|---|
| 01 | GraalScript AI Assistant (GS2 codegen/debug) | TypeScript, MCP, Claude | Live |
| 02 | Browser Level Editor (collaborative) | Pixi.js, Yjs, Next.js | Live |
| 03 | Modern RC (replaces GTK2 RC) | Tauri 2.0, Monaco, Rust | Building |
| 04 | AI NPC Engine | Go, LiteLLM, SQLite, Claude Haiku | Live |
| 05 | No-Code NPC Creator | React Flow, Monaco | Planned |
| 06 | Container Game Servers | Go, Incus, ZFS | Planned |
| 07 | ZFS World Cloning ("git-like branching") | Go, ZFS, MySQL | Planned |
| 08 | AI Playtesting Bots | Go, Graal protocol, Claude | Planned |
| 09 | AI Sprite Generator | Python, Stable Diffusion | Planned |
| 10 | WebGL/WASM Client | Emscripten, C++, WebGL2 | Planned |
| 11 | Dead Code Cleanup (16K lines, "Quattro" engine) | C++ | Planned |
| 12 | Security Hardening (TLS 1.2+, AES-256-GCM) | C++, OpenSSL 3 | Planned |
| 13 | Web Gani Editor | Pixi.js, TS | Planned |
| 14 | Web GMap Editor | Pixi.js, TS | Planned |
| 15 | Asset Pipeline (versioning, CDN, search) | Go, MinIO, libvips | Planned |
| 16 | Graal Studio portal ("our Roblox Studio") | Next.js, Zustand | Planned |
| 17 | GraalScript LSP v2 | Rust, tree-sitter, WASM | Planned |
| 18 | Cross-Server Events / economy | Go, Redis, MySQL | Planned |
| 19 | PlayerWorlds Platform ("the 2D Roblox vision", 1000s of sandboxed instances) | Go, Incus, ZFS | Planned |
| 20 | Re-Engagement Engine (30M dormant accounts) | Go, AWS SES, FCM | Planned |

Roadmap 2026: Phase 1 (May) cross-server messaging, clone pipeline; Phase 2 (Jun) AI Clone Arena, cross-server quest; Phase 3 (Jul–Aug) PlayerWorlds revival, world discovery, trading marketplace, battle pass; Phase 4 (Sep–Oct) re-engagement campaign; Phase 5 (Q4 2026+) creator revenue share, server federation.

**Why their stack differs from ours**: every choice is constrained by keeping the existing C++ engine, GS2 language, 5 live worlds and MySQL account base compatible (Emscripten to port the C++ client, tree-sitter for GS2 tooling, MySQL/Redis for existing services). We have no legacy.

## 11. Implications for our project

| Graal lesson | Our requirement |
|---|---|
| Live-editable world + content streaming = the platform | Hot reload of scripts/levels; assets streamed by content hash; no client updates for content |
| One file, server + client sections | Keep: single script module with typed server/client halves and a typed message bridge |
| Security enforced by human inspection | Enforce in the API: client-writable data typed untrusted; permission checks server-only; rate-limited client→server messages |
| Shared global file namespace caused overwrites/theft | Per-world asset namespaces, provenance, licensing |
| Gameplay moved from engine to scripts (baddies/chests deprecated) | Engine core minimal; default action kit written in the public scripting API |
| No version control, poor editor, poor debugging, no local dev | MVP includes history/rollback, Monaco + types, logs with stack traces, free tier, local CLI |
| Paid hosting + opaque promotion | Free start; metrics-based discovery with published criteria |
| Platform fragmentation | One web client everywhere; every world cross-platform |
| 0.1 s NPC-Server loop proven adequate | Server script tick 10 Hz; network 15–20 Hz |
| Named rights + glob folder rights | Role-based rights with path-scoped permissions for assets and scripts |
| Word filter with action bitmask | Per-world moderation rules with graded actions |
