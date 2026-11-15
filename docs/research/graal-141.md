# Graal 1.41 (2000) — Analysis of the Original Client Files

Status: research, November 2026. Source: the Graal 1.41 offline client archive supplied by the project owner (readme says v1.39 release notes; Graal.exe dated Nov 2000). Analysed read-only; no executables were run and **no Graal assets, levels or scripts are copied into this repository**. Supporting docs inside the archive: `readme.txt`, `newfeatures139.txt`, `commands.rtf` (command reference), `npcprogramming.doc` (Stefan Knorr's NPC programming guide, © 2000), `guide.txt` (fan FAQ by "Dacium" for v1.41rev1 — community claims), `languages/languageEnglish.txt`, `headsconfig.txt`, `graalconfig.txt`, `bigmap.txt`.

Complements [graal.md](graal.md) (modern Graal, GS2 era) — this file shows what the platform looked like at the moment it took off.

---

## 1. Archive contents

| Content | Count / size |
|---|---|
| Levels (`.graal`, few `.zelda`) | 2,951 decodable levels (GR-V1.03: 2,779; GR-V1.02: 141; GR-V1.01: 28; Z3-V1.04: 2) |
| Images in `levels/gifs` | 3,208 |
| Heads / shields / swords / bodies | 808 / 482 / 352 / 6 |
| Weapon scripts (`levels/weapons/*.txt`) | 7 GS1 scripts + sprites |
| MIDI / WAV | 47 / 42 |
| Tileset `pics1.png` | 1856×512 = 116×32 tiles of 16 px |
| Total | 8,179 files, 49 MB |

## 2. Level file format (decoded)

Binary, header `GR-V1.0x` (8 bytes), then:

1. **Board**: 64×64 tile indices, bit-packed LSB-first; 13-bit codes for V1.02/1.03 (12-bit for older). A code with the top bit set is a run: low 8 bits = count; if bit 0x100 is also set, the next two codes alternate (double run), else the next code repeats. Tile index → tileset position: `tx = t%16 + (t/512)*16`, `ty = (t/16)%32` (tileset arranged in 16-tile-wide columns).
2. **Links**: text lines `dest.graal x y w h newx newy` (newx/newy may be `playerx`/`playery` to preserve position), terminated by `#`.
3. **Baddies**: binary records (x, y, type) plus verse strings, terminated by `\xff\xff\xff`.
4. **NPCs**: one per line: x+32, y+32 as bytes, image filename, `#`, script with newlines encoded as `§` (0xA7); section ends with `#`.
5. **Chests** (x, y, item, sign index) and **signs** (x, y, encoded text) follow.

The decoder (kept outside the repo) renders levels correctly against `pics1.png`; the 18×15 `bigmap.txt` grid stitches into one continuous overworld.

`bigmap.txt` / `minimap.txt`: CSV grids of level names — the predecessor of GMap. Map cells are 56 px (map) and 14 px (minimap) per level.

## 3. Corpus statistics (all 2,951 levels)

| Metric | Value |
|---|---|
| Links (warps) | 6,923 |
| NPCs | 16,220 in 2,464 levels (max 104 in one level) |
| Script size per NPC | median 122 chars, p90 439, max 16,723 |
| Total script text | 4.9 MB |
| Distinct tiles used | 3,448 |

Most frequent script identifiers: `if`, `this`, `timeout`, `playerenters`, `playertouchsme`, `onwall`, `setcharprop`, `playerx/y`, `message`, `say`, `sleep`, `hearts`, `exploded`, `washit`, `dontblock`, `setgif`, `playerendsreading`, `playerlaysitem`, `compusdied`, `random`, `waspelt`, `showcharacter`, `putleaps`, `strequals`, `hidelocal`, `timereverywhere`, `putexplosion`, `showimg`, `isleader`, `setlevel`, `aimplayer`, `putcomp`.

**Reading**: most content was tiny scripts — doors, signs, shopkeepers, decorations, simple patrolling/attacking characters, warps with conditions. Many NPCs carry "NPC made by <name>" credit comments: authorship mattered to creators.

## 4. GS1 scripting model (v1.34–1.41)

**Execution**: each NPC has one script. On any event the *whole script reruns from the top* with that event's flag true; logic is `if (event) { … }` blocks. `timeout = secs` schedules a timeout event (tick 0.05 s); `sleep` builds on it. Loops capped at 10,000 iterations unless they sleep.

**Online authority problem**: only the *leader* (first player in the level) receives timeouts. Workarounds: `timereverywhere` (all players run timers), `isleader` checks. Only `save[0..9]` (integers 0–220) synchronised between players. Shared, multi-player NPC behaviour was hard — the root of later NPC-Server/GS2 design.

**Events**: playerenters, playertouchsme, playertouchsother, playerlaysitem, playerchats, playerendsreading, weaponfired, firedonhorse, compusdied, exploded, timeout, washit, wasshot (shotbyplayer/shotbybaddy), waspelt (by bush/sign/vase/stone/blackstone/npc), wasthrown, created.

**Commands (by category)**:
- Flow and state: `if/else`, `for`, `while`, `sleep`, `set`/`unset` flags, `setarray`, `setstring`, `timereverywhere`.
- NPC display: `setgif`, `setgifpart`, `show/hide`, `showlocal/hidelocal`, `dontblock`, `drawoverplayer/underplayer`, `canbecarried/pushed/pulled`, `showcharacter`, `setcharprop`, `carryobject`, `throwcarry`.
- Drawing: `showimg i,file,x,y`, `changeimgpart`, `hideimg`.
- Text: `message` (chat bubble), `say n` (sign text).
- Lifecycle: `putnpc img,script.txt,x,y`, `destroy`.
- Items/weapons: `lay`, `take`, `toweapons name` (NPC becomes a player weapon), `toinventory`.
- Projectiles/combat: `shootarrow`, `shootfireball`, `shootfireblast`, `shootnuke`, `shootball`, `hitplayer`, `hitnpc`, `hurt`.
- Player control: `setlevel`, `setplayerdir`, `freezeplayer`, `hideplayer`, `hidesword`, `sethead`, `setsword`, `setshield`, `setbow`, colour setters, `disableweapons`.
- Board: `board[x+64*y]` read/write + `updateboard x,y,w,h`; `putbomb`, `putexplosion`, `putleaps`, `puthorse`; `setbackpal`, `setmap`, `setminimap`, `noplayerkilling`.
- Baddies: `putcomp`, `putnewcomp` (reskin/power only — no new AI), `hitcompu`, `removecompus`.
- Media: `play` (wav/midi), `playlooped`, `stopsound`, `openurl` (asks user).

**Variables**: all floats. Unprefixed variables are shared by every NPC in the level; `this.` is per NPC. Flags persist in save games / online accounts; `server.` flags are server-wide. Strings are stored as `name=value` flags. No `client.` scope yet. Built-ins: NPC `x, y, dir, sprite, hearts, rupees, …`; player `playerx, playery, playerdir, playersprite, …` (aliases of `players[0]`); lists `npcs[]`, `players[]`, `compus[]`, `bombs[]`, `arrows[]`, `items[]`, `explos[]`, `horses[]`. Message codes `#a #n #g #c #v() #s() #C0–#C4` interpolate account, nick, guild, chat, variables, strings, colours.

**Units**: positions in fractional tiles (0–63); directions 0 up, 1 left, 2 down, 3 right. Items vanish after 8.2 s; bombs explode after 3 s.

**Weapons were just NPC scripts**: the hammer weapon animates via `playersprite`, draws with `showimg`, performs its own hit tests on `board[]`, `compus[]`, `players[]`, and spawns projectile NPCs via `putnpc`. Players' tools were fully user-scriptable from the start — and therefore client-trusted.

## 5. Built-in game rules

- 25 item ids: rupees (green 1, blue 5, red 30, gold 100), bombs, darts, heart, fullheart, glove1/2, bow, superbomb, joltbomb, shield/mirror/lizard shield, sword/battleaxe/lizardsword/goldensword, fireball, fireblast, nukeshot, spinattack.
- Sword power 1–4 = 0.5–2 hearts damage. Shields: block / reflect / reflect + speed. Start with 3 hearts. Gloves lift light rocks, then dark stones.
- Liftables: bush, sign, vase, stone, blackstone — throw or hide under. Bushes respawn and drop currency, bombs or hearts.
- Horses (eat bushes for speed; become boats on water), beds heal.
- 10 baddy types (soldiers ×5, frog, octopus, golden warrior, lizardon, dragon) with modes walking/looking/hunting/hurt/bumped/dying/shooting/jumping/dead; editor lets each baddy say lines when attacking/hurt.
- Chests: one item + optional sign.
- **Alignment (AP 0–100)**: below 40 cannot heal; respawn hearts depend on AP; killing costs AP; regenerates faster when low; at 100 cannot be hurt; nickname colour follows AP. **Sparring** levels: no AP change, no loss on death.
- Death online costs arrows, bombs and up to 1,000 currency (fan guide).
- Guilds as name tags (join after 100 online hours, found after 200 — fan guide). Emote icons via Ctrl+letter.

## 6. Client, editor and online model

- Modes: single player, single player online, 2–4 player local split-screen.
- Offline-only tools: **F4 level editor**, F5 load level, F3 save, **F6 debugger** (shows this./vars/flags/attributes, step/run, click-to-set breakpoints).
- Editor: predefined multi-tile object stamps (`objects.dat`, extendable), single-tile painting with drag, background tile, move/copy selections, undo, warp fields (dest level, rect, new position or `playerx`), sign list with graphic codes, NPC editor with image + script + **Test** (syntax check highlighting the bad argument), **character NPC wizard** generating scripts from options (walk mode, speed, attack mode, hunt speed, power, respawn, carryable, damage rules), flags editor, drag-and-drop images/scripts onto the level, play-test with P, world map from `bigmap.txt`, level snapshots, **player path recording** to drive NPC routes.
- Online server: accounts and saves, server-checked currency pickup (anti-dupe), `server.` flags, synced `save[]`, board-change respawn timers, downloads of levels/images/heads, MOTD, player list (GPs/buddies/guild/ignore), HTML PMs, toall/toguild, profiles, jail, RC. UDP transport. Online restrictions: `putobject` and URL warps disabled, some effects local-only, `followplayer` unsupported.

## 7. Character model

| Sheet | Size | Layout |
|---|---|---|
| Head | 32×560 | 32×32 frames, by direction and pose |
| Body | 128×720 | 4 columns (up/left/down/right) × rows of poses, 32×32 cells; fixed key colours swapped at runtime for 5 colour slots |
| Sword | 128×96 | fixed multi-frame layout |
| Shield | ≤76×40 | small multi-frame |
| Horse | 96×576 | 3 frames × 4 directions |

Player pose indices: 0 idle, 1–8 walk, 9–13 sword, 14–18 push, 19–22 pull, 23 lift, 24–32 carry, 33 bow, 34–36 ride, 37 sit, 38 sleep, 39 hurt, 40 dead. Head presets (`headsconfig.txt`): `Name=head.gif,gender,skin,coat,sleeve,shoe,belt` from a 19-colour named palette.

This is the pre-GANI system: fixed sheets with numbered poses. GANI later made poses data-driven and outfit-independent.

## 8. Lessons for our platform

1. **Event-driven tiny scripts built the world.** A median NPC script was 122 characters. The API must make a door, a sign, a shopkeeper or a patrolling guard a few lines — then scale up to systems.
2. **No-code on-ramp.** The character NPC wizard generated scripts from form options. Equivalent: templates/wizards that emit editable TypeScript (Graal Next plans a "No-Code NPC Creator" for the same reason).
3. **Authority must be designed, not patched.** Leader-only timers, `timereverywhere`, `isleader`, `save[]` 0–220 sync were all symptoms of client-run shared NPCs. Our model: shared NPC logic runs on the server; client code is presentation and input.
4. **Typed scoped state.** Floats-only variables and strings-as-flags were limits; provide typed per-player, per-NPC, per-chunk, per-world and per-guild stores.
5. **Weapons/tools as scripts — sandboxed.** Keep player-held script systems, but hit detection, projectiles and damage go through engine APIs validated by the server.
6. **The creator loop is the product.** Edit → syntax check → play-test → debugger with breakpoints → flags inspector, all in one app, instant. Ship this in the browser from the start; add path recording for NPC routes.
7. **Data-driven content.** Fixed 25 item ids and 10 baddy types became cages (`putnewcomp` could only reskin). Items, enemies and damage rules must be definitions + script hooks.
8. **Authorship and credit.** Thousands of "NPC made by …" comments: provide first-class attribution metadata for scripts, levels and assets.
9. **Cosmetics drove identity.** 808 heads, 482 shields, 352 swords — mostly community-made. Standardised, validated sheet specs plus moderation.
10. **Social systems and PvP rules (alignment, sparring, safe zones) were core gameplay**, not add-ons.
11. **Safety defaults** came from abuse: server-checked pickups, sanitised rich text in PMs, confirmations for external URLs, no arbitrary object spawning by clients.
