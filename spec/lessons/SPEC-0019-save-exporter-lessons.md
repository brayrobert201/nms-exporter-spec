---
id: SPEC-0019
title: 'Save-exporter lessons'
description: 'What reading the save, projecting it, and putting it on Grafana and conky wall screens cost to learn: signed wallets, phantom networks, assumed rates, dpi, clicks, label escaping.'
type: lesson
area: save-exporter
author: bob
created: 2026-10-23
last_updated: 2026-10-23
status: current
related_specs: [SPEC-0001, SPEC-0002, SPEC-0003, SPEC-0004, SPEC-0005, SPEC-0006, SPEC-0007, SPEC-0008, SPEC-0009, SPEC-0010, SPEC-0011, SPEC-0015]
---

# SPEC-0019: Save-exporter lessons

Ordered by how much time each would save a second builder.

## Lesson: fill levels are not network membership

**What happened.** Extractors on one network showed the same fill on every base first looked at,
so a pass split each traced pipe component wherever extractor fills differed, and the farm check
flagged a piped extractor with a unique fill as "split off". On the largest base (Cobalt Oxygen)
two saves 204 s apart showed three traced networks in which every unit rose by exactly the same
amount while keeping different absolute fills. The split had invented about a dozen
one-extractor "networks", each with its own wrong rate and time-to-full ("Cobalt 2", "Oxygen 3":
250/250 items, 250/h assumed). The bases screen listed 19 farm rows; traced by geometry alone the same bases give 17.

**The rule.** Networks come from pipe geometry only (ends within 3.2 units). Units on a network
gain in lockstep but keep the offsets they started with. Co-movement across two saves is a check,
never a single-save fill comparison. See SPEC-0015.

## Lesson: the wallet is uint32 written signed

**What happened.** After selling 89 devices the balance went from 788,126,648 to 2,178,378,504
units; the save wrote it as -2,116,588,792 and every screen and the Grafana history showed a
negative balance. Fixed by adding 2^32 to negative `Units`, `Nanites`, `Specials`
(commit "Read the wallet as unsigned so balances past 2.1B stay positive"). The lifetime counters
were not fixed at the same time: `nms_stat{stat="MONEY_EVER"}` reads -617,290,418 today
(3,677,676,878 unsigned), and the traveller screen prints "-617.3M earned ever".

**The rule.** Any 32-bit currency field, current or lifetime, is unsigned. Grep the stats block
for `MONEY*` and `NANITES*` and apply the same correction. The wallet cap (presumably
4,294,967,295) and what the game does there are not observed.

## Lesson: the save does not say what a hotspot gives, or how strong it is

**What happened.** Farm rows were first named by guessing the resource, and rates assumed class
C (250 items/h per extractor). The save stores neither class nor substance. A cross-section of
14 measured networks showed totals of ~870-1,570 items/h whether a network had 2 or 13
extractors; the assumption put a 13-extractor gas network at 3,250/h against 1,270/h measured,
and an 8-extractor one at 2,000/h. A class-B mineral hotspot on one planet gave Silver (the rare
resource), on another the common one, so even "mineral hotspot = common resource" is wrong.

**The rule.** Name rows from the substances the player wrote in the base name (longest match
first, so "Magnetised Ferrite" is not "Ferrite"). Measure rates from consecutive saves and label
the fallback "assumed C-class" on every surface. Ignore a pair whose later fill is at capacity.

## Lesson: conky scales pixels by dpi but not fonts

**What happened.** On KDE under XWayland, `Xft.dpi` is 124. conky multiplies gaps, sizes,
`${goto}`, `${voffset}` and graph sizes by dpi/96 but not font pixel sizes, so the 2160-wide
layout landed ~29% past the right edge. Xvfb defaults to 100 dpi, so an unqualified test render
is itself scaled 4% and does not show what the real screen shows.

**The rule.** Read `Xft.dpi`, divide every pixel value by dpi/96 in the config and in the
renderer's markup, and always render tests with `xvfb-run -s "-screen 0 2160x3840x24 -dpi 96"`.

## Lesson: conky 1.22 never sees clicks under XInput2

**What happened.** The click-to-search feature used conky's mouse hook. Motion events arrived;
button presses never did: conky resolves the window under the pointer with the wrong device id,
then drops the duplicate event as already handled.

**The rule.** Read raw button events on the root window with a small ctypes watcher (Xlib +
libXi), act when the window under the pointer has the screen's window class, and tie the
watcher's life to conky's.

## Lesson: KWin ignores where an X11 window asks to be

**What happened.** conky's `xinerama_head` and gaps put the window on the right monitor in plain
X, but KWin ignored the requested position and clipped the window to `_NET_WORKAREA` - one
rectangle for the whole X screen that a panel on any monitor shortens. Rules keyed by screen
index broke after a reboot because KWin renumbers outputs.

**The rule.** Force position (a point inside the target monitor, logical coordinates) and
fullscreen with window rules matched by window class; never by screen index.

## Lesson: procedural item ids break the scrape

**What happened.** Procedural item ids are `<type>#<seed>` where the type can be a binary blob.
Some contain bytes a text-format parser reads as line breaks; one such label value invalidated
the entire `/metrics` response. Names showed as `PROC_BIO#08282`, and base parts and corvette
parts (`BUILD_REFINER1`, `B_SHL_A`) as raw ids because they live in three product tables of their
own.

**The rule.** Replace every unprintable character in label values with `?` before escaping;
decode the save with `errors="replace"`; name procedural items "Procedural item #seed" when the
type is not printable; include the base-part, modular-part and procedural product tables in the
extract.

## Lesson: Grafana label matchers and display names

**What happened.** Three fixes in one afternoon. Item variables used the display name as value;
names containing quotes broke `{item="$item"}`. Crafting joins used the `recipe` display name,
which several recipes share, so a join could match the wrong recipe's rows. The variable regex assumed `item`
came right before `job`, which broke when labels were reordered.

**The rule.** Every metric that names something carries a stable id label (`id`, `recipe_id`,
`ingredient_id`); variables use the id as value and the name as text; join on ids; write variable
regexes that do not depend on label order.

## Lesson: refiner positions are in a planet-aligned frame

**What happened.** `RefinerBufferKeys` positions for location 3 (planetary base) did not match
any refiner object in the base's own frame. They are relative to the base origin but on the
planet's axes (up = base position from planet centre, forward = base Forward made orthogonal).
The `Location` integer has no enum in the game's metadata; the codes (1 surface, 2 freighter,
3 base, 5 space, 6 frigate, 8 corvette) were read off a save. The same list also holds extractor
rooms (`^MAINT_HOOVER`), damaged machinery, drop pods and frigate repair slots.

**The rule.** Rotate each base's refiner objects into the planet-aligned frame and match within
2 units; filter the list by slot contents and staleness (7 days), not by location alone.

## Lesson: mission progress is a number, not a stage index

**What happened.** `MissionProgress.Progress` looks like a stage index and is not: read that way
it points at the wrong objective. It is a progress number; each stage's `Versions` table maps it per mission version, the finished
value comes from `FinalStageVersions`, and `2147483647` also means complete.

**The rule.** The current objective is the last stage whose number is at or below the saved
progress; rewards are what finishing the whole mission pays.

## Lesson: substances can be crafted too

**What happened.** The crafting tree treated every substance as gathered. Star Bulb, Cactus
Flesh, Solanium, Gamma Root, Faecium and Fungal Mould all refine from common substances, so
"blocked" was overstated. Separately, refiner and cooking recipes were required to be
"discovered", but the machines run any recipe.

**The rule.** Only items with no route are leaves; refine and cook routes need no discovery;
guard loops for recipes that need their own output as a seed.

## Lesson: wait for the game to finish writing

**What happened.** The game writes the save in place, and the exporter refreshes on every scrape,
so sooner or later a scrape lands mid-write. A read then sees a truncated chunk chain, and without
a "do not retry this mtime" rule the same broken file would be re-decoded on every scrape.

**The rule.** Skip files younger than 2 s (decoder and backup alike); remember `(path, mtime)`
of failed attempts.

## Lesson: kill test processes by port, never by pattern

**What happened.** A test exporter started from the working tree looks, to `pkill -f`, much like
the live service. Rendering the screens for this spec, `pgrep -f "nms_search.py --watch" | xargs
kill` also matched the shell running that very command line and killed it.

**The rule.** Stop a test server by the PID listening on its port (`ss -ltnp`), and a helper by
the PID you started.

## Lesson: the corvette and freighter are bases

**What happened.** `PersistentPlayerBases` includes the freighter (`FreighterBase`) and the
corvette (`PlayerShipBase`), positioned wherever they are. Plants on the freighter and refiners
on the corvette are real, but anything that assumed "base = planet" (planet lists, frames) went
wrong when the corvette was parked on a base-less planet.

**The rule.** Name them "Freighter <name>" and "Corvette"; exclude them from anything
planet-related; include them for refiners and plants.
