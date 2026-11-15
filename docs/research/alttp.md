# A Link to the Past — Design and Engine Research

Status: research, November 2026. Purpose: extract the design principles and engine structure behind *The Legend of Zelda: A Link to the Past* (SNES, 1991) — the game that inspired Graal — to shape our default action kit and engine. **We do not recreate the game**: no names, characters, art, maps, music or levels. Only genre mechanics and design principles carry over.

**Source quality.** Only Wikipedia and the MediaWiki API of zeldawiki.wiki / zelda.fandom.com / strategywiki.org were reachable; Data Crystal and TCRF (the technical RAM/ROM documentation) were behind Cloudflare challenges. Tags:
- **[S]** sourced — URLs at the end
- **[BG]** background knowledge from the disassembly community (zelda3 C reimplementation, ZScream, Hyrule Magic), not verified here
- **[BG?]** uncertain background knowledge — verify before relying on exact numbers
- **[INF]** inference

---

## 1. Development and intent

- Directed by Takashi Tezuka, produced by Shigeru Miyamoto, script by Kensuke Tanabe, music by Koji Kondo. Began on NES in 1988, moved to Super Famicom; 8 Mbit cartridge. [S1][S2]
- Cut ideas: a party of three characters, spreading grass fires, freely combinable weapons. Miyamoto required the **sword always equipped** — the sword is the permanent primary verb, items a secondary slot. [S1]
- New versus the 1986 original: 8-way movement, arc sword swing, charged spin attack, hookshot, multi-floor dungeons, two parallel overworlds, layered graphics (weather/FX/HUD). [S1][S2]
- Miyamoto: Zelda as "a real-time adventure game", a live feeling rather than outcomes "decided by stats and numbers". [S3]
- ~4.61M SNES copies; set the series structure for ~20 years; "Zelda-like" is now a genre label. [S1][S2][S4]

## 2. Structure and core loop

- Overworld hub + towns/caves/shops + dungeons. Each dungeon holds one major item that solves its puzzles and usually its boss. [S3]
- Loop: explore → dungeon → big-chest item → boss → heart container → new item opens more of the world. [S1][S3]
- **Two parallel worlds on one geography**: same layout, different enemies/NPCs/dungeons; a mirror item warps from the second world back to the same coordinates in the first. The Dark World was stored as an overlay of the Light World layout. One map → two worlds of content. [S1]
- Linear guided opening (castle at night, unarmed) → progressively more freedom; item gates impose partial order. [S2][BG]
- Teaching without tutorials: introduce an item in a safe room, drill it, test it on the boss. Consistent visual grammar for secrets (cracked walls, odd tiles). [INF]
- Dense small rewards: heart pieces (4 = 1 heart), hidden caves, bombable walls, dig spots, fairy fountains. [S1][S3][BG]

## 3. Combat feel

- 8-direction movement; sword arc; spin attack charged by holding the button ~1 second, no attacking while charging (risk/reward). [S1][S2][S5]
- Shield blocks projectiles passively when facing them; upgrades block more types. [BG]
- On hit: knockback away from the source, flashing invulnerability; small enemies recoil when struck (hits feel solid). [BG]
- Enemies: readable patterns (chargers, wanderers, shooters), wind-up telegraphs, armoured types needing a flank or a specific item. Bosses are usually beaten with the dungeon's item. [S3][BG]
- Resources: hearts (start at 3, containers from bosses and pieces), magic meter, currency in 1/5/20 denominations, bottles holding potions/fairies (fairy revives on death). [S1][S3][BG]

## 4. Item design — every item is also a key

| Function | Examples (generic description) |
|---|---|
| Traversal | grapple to targets across gaps; dash boots (also break rock piles, shake trees); swim ability for deep water; world-switch mirror; light source for dark rooms; fast-travel item |
| Strength | glove tiers to lift light / heavy rocks; hammer for pegs |
| Combat | bow, boomerang (stun + fetch items), bombs (also open cracked walls), fire/ice rods, screen-wide spells (also open specific entrances), block-creating cane, barrier cane, invisibility cape |
| Utility | shovel, net, translation book, transformation powder, bottles |

No jump item in this game; ledges are one-way auto-hops. [BG]

## 5. World interactions

Lift/throw bushes, pots, rocks (weight tiers) — thrown objects break and deal damage; cut grass for random drops (spin clears an area); dash into trees/piles for drops; push blocks (sometimes one-way); floor switches; crystal switches toggling two colours of raised/lowered pegs; small chests and big chest (big key); small keys consumed by locked doors; shutter doors open on room clear or switch; bombable cracked walls; one-way ledges as shortcuts; stairs and pits dropping to the floor below; two-layer rooms (bridges over lower floors); shallow vs deep water; NPC dialogue, shops, mini-games, healing fountains. [S1][S3][BG]

## 6. Dungeon patterns

- Map, compass, small keys (tight economy ≈ one per lock), big key for boss door and big chest. [S3]
- Multiple floors with stairs and drop holes. [S1]
- Mid-dungeon item used to beat the boss; introduced first in safe rooms. [S3]
- Puzzle types: block pushing, switch order, dark rooms, conveyor floors, kill-all-to-open shutters, colour-peg toggles. [BG]

## 7. Engine structure [BG unless noted]

**Display.** SNES 256×224, 64 KB VRAM, 256-colour palette from 32,768 [S6]. Mode 1: two 4bpp background layers (world, upper/lower floor) + one 2bpp layer (HUD, text, weather). 128 hardware sprites, 8×8/16×16, max ~32 per scanline (flicker rotation). Tiles: 8×8 hardware → 16×16 "map16" metatiles (per-quadrant flip/palette) → 32×32 "map32" blocks for overworld compression. A per-tile priority bit lets canopies/arches draw over sprites.

**Overworld.** Each world = 8×8 grid of 512×512 px areas (4096×4096 px); "large" areas are 2×2. Camera follows freely inside an area, clamps at edges, scroll transition into the neighbour (input frozen). Each area has its own tileset, palette, music, sprite set. Secrets are per-area overlays toggled by save flags.

**Dungeons.** ~296 rooms on a 16-wide grid. Room = 512×512 px split into four 256×256 quadrants; a layout setting joins quadrants into one scrollable space. Room header: BG2 mode (normal/parallax/dark/translucent), palette, tileset, sprite set, effect, two tag routines (e.g. "kill all → open door", "push block"), destinations for holes/stairs. Rooms are stored as **object lists** (wall runs, floors, chests, stairs) rendered into tilemaps at load, plus door list, sprite list, and secrets-under-pots list. Two floor planes (BG1/BG2); the player has a current-layer flag flipped by stairs and some doors; collision reads that layer's attribute map.

**Tile attributes [BG?].** One attribute byte per 8×8 cell, separate map per layer. Categories: floor; walls; diagonal walls; deep water; shallow water; pits; one-way ledges (N/S/E/W + diagonals); spikes/damage floor; ice; conveyors ×4; slow stairs; layer-switch stairs; thick grass (slow, cuttable); liftables (bush, light rock, heavy rock, sign, pot); chests; diggable ground; damaging plants; warp tiles; colour pegs; torches; hook targets; doors (normal, key, big key, bombable, shutter, layer toggle); entrance/exit triggers. Movement tests a few points around the feet, with **corner sliding** (nudge sideways when only one test point is blocked).

**Player numbers (60 fps) [BG?].**
| Quantity | Approx. value |
|---|---|
| Walk | ~1.5 px/frame (~90 px/s); slow terrain ~1 px/frame |
| Dash | ~0.5 s wind-up, then ~2.5–3× walk |
| Spin charge | ~1 s [S5] |
| Sword active window | ~8–12 frames, 3–4 hitboxes per swing arc |
| Invulnerability after hit | ~58 frames, blinking |
| Knockback | ~16–24 px with a small hop, input locked ~16–20 frames |
| Player art vs hitbox | ~16×24 visible (16×16 head + 16×16 body overlapping); hitbox ~12–16 px square at the feet, so the head can overlap walls (¾-view depth) |

**Sprites/AI.** 16 general sprite slots, ~10 projectile/effect slots, 8 invisible spawner slots. Per-type property table (hitbox, health, contact damage, palette, deflect flags). AI = per-type routine indexed by a state value with per-slot timers, plus shared recoil/stun/fall routines. **Damage table**: enemy damage class × weapon → damage amount or special effect (immune, stun, freeze, burn, transform). Damage to the player = contact class scaled by armour tier.

**Z-axis.** Entities carry z and vz; constant gravity; drawn at y − z with a ground shadow. Throws: fixed ~4-tile arc, break at z = 0. Ledge jumps and pit falls are scripted motions (pit: shrink animation, respawn at room entry, lose ~½ heart). Collision resolved in XY; z gates hit eligibility.

**Save model.** 3 slots × ~0x500 bytes with mirror + checksum. Contents: 16-bit flag word per dungeon room (chests, doors, boss, bombed walls, visited quadrants), one byte per overworld area (overlay revealed, heart piece taken), inventory with item levels, currency, ammo, magic, hearts, heart-piece count, per-dungeon keys, map/compass/big-key bitfields, progress stage, event flags, respawn point.

**Reimplementations/editors.** zelda3 (C reimplementation reading assets from a user-supplied ROM): fixed 60 Hz update, global state struct mirroring SNES RAM, per-module state machines (overworld, dungeon, transitions, menus). ZScream (C#) and Hyrule Magic: map16/map32, room objects, sprite lists, tile-attribute editing. *For reference only — none of their code or data goes into our project.*

## 8. What Graal borrowed [INF]

Top-down player with sword, shield, hearts, bombs, arrows; liftable/throwable/cuttable bushes and pots with drops; "baddies"; currency in green/blue/red denominations; chests and keys; tile-grid levels linking at edges plus interior levels. Graal added multiplayer, player-made levels and scripting. Graal's level span (64×64 tiles × 16 px = 1024 px) is close to two ALttP areas wide.

---

## 9. Design principles for our default kit

1. One primary verb always available (melee attack on a fixed button); items in secondary slots.
2. Every tool has a combat use, a traversal use and a gate it opens.
3. Introduce → drill → test, without text tutorials.
4. Consistent visual grammar for secrets so players learn to read the world.
5. Dense small rewards on most screens.
6. Risk/reward attacks (charging, committing to a dash).
7. Fair hits: telegraphs, knockback, invulnerability frames.
8. Parallel-world overlays on the same geometry — doubles content cheaply and suits creators.
9. Ability gates over stat gates.
10. One-way ledges, drops and keyed doors as shortcuts to keep backtracking short.
11. Short guided opening, then widening freedom.
12. Self-contained rooms with optional room-locked camera for puzzle focus.

**MMO adaptations [INF]**: chests, keys, heart pieces and secrets are **per-player** flags while bushes, enemies and world events are **shared** state; dungeons as per-party instances; parallel-world overlays as a map-variant feature in the editor.

## 10. Engine translation to 32px

| Area | Recommendation |
|---|---|
| Tile | 32 px art tile (= ALttP map16 at 2×), flip flags per tile |
| Collision resolution | 16 px sub-cells (2×2 per tile) to keep thin walls, ledge lips and diagonals precise |
| Chunk | 32×32 tiles = 1024×1024 px (= one ALttP area at 2×; same span as a Graal level); unit for streaming, server processes and edit locks; world is seamless across chunks |
| Layers | ground, ground detail, objects, overhead (draws above actors; replaces the priority bit), optional second floor plane; each floor plane has its own attribute map; actors carry a `plane` field switched by stair/toggle tiles |
| Tile attributes | FLOOR, WALL, WALL_DIAG_×4, DEEP_WATER, SHALLOW_WATER, PIT, SPIKES, LAVA, ICE, CONVEYOR_×4, SLOW, LEDGE_×8, STAIRS_UP/DOWN_PLANE, LAYER_TOGGLE, CUTTABLE_GRASS, BUSH, ROCK_LIGHT, ROCK_HEAVY, POT, DIGGABLE, CHEST, SIGN, HOOK_TARGET, DAMAGE_FLOOR, NO_PROJECTILE, WARP, DOOR_TRIGGER, LOCKED_DOOR, BOMBABLE, SWITCH_PEG — extensible by scripts; liftables/breakables become entities when touched |
| Simulation | Fixed 60 Hz client simulation with interpolated rendering; server validates and runs scripts at 10 Hz, network 15–20 Hz |
| Movement constants (2×, tune by feel) | walk ~180 px/s (~5.6 tiles/s); slow terrain ~120 px/s; dash ~360–480 px/s after 0.5 s wind-up |
| Combat constants | sword active ~10 frames; spin charge 60 frames; knockback 32–48 px over ~16 frames; invulnerability ~60 frames |
| Player collision | ~24×24 px box at the feet inside a ~32×48 body in a 64×64 frame; corner sliding up to ~8 px |
| Camera | Smooth follow clamped to a region; region = open world (any chunks) or a room (room-locked with optional scroll transition). Phones are wider than 8:7 — reveal more width or letterbox |
| Height | z/vz per entity, constant gravity, draw at y − z with shadow; ledge hops, pit falls, throws (~128 px arc) as scripted motions |
| Damage | Per-entity damage class × damage type table (amount or effect: immune, stun, freeze, burn, transform), exposed to scripts |
| Persistence | Per-player document: inventory + levels, resources, per-chunk/room flag sets, per-dungeon keys, progress/event flags, respawn point. Shared world state stored separately |

Verification still needed for [BG?] values: requires access to Data Crystal / TCRF or reading published disassembly notes.

## Sources

- [S1] https://en.wikipedia.org/wiki/The_Legend_of_Zelda:_A_Link_to_the_Past
- [S2] https://en.wikipedia.org/wiki/Takashi_Tezuka
- [S3] https://en.wikipedia.org/wiki/The_Legend_of_Zelda
- [S4] https://en.wikipedia.org/wiki/Action-adventure_game
- [S5] https://zeldawiki.wiki/wiki/Spin_Attack ; https://zeldawiki.wiki/wiki/Pegasus_Boots ; https://zeldawiki.wiki/wiki/Magic_Mirror
- [S6] https://en.wikipedia.org/wiki/Super_Nintendo_Entertainment_System
