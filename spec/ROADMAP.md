# ROADMAP

Every spec, by area. Status: `current`/`accepted` unless noted. Lessons files are the place to start.

## save-exporter (0001-0019)

| id | type | title | status |
|---|---|---|---|
| [SPEC-0001](snapshot/SPEC-0001-save-decoding.md) | snapshot | Finding and decoding the save | current |
| [SPEC-0002](snapshot/SPEC-0002-exporter-service.md) | snapshot | The exporter service: snapshot, HTTP API and metrics | current |
| [SPEC-0003](snapshot/SPEC-0003-projection.md) | snapshot | Projecting the save forward on the wall clock | current |
| [SPEC-0004](snapshot/SPEC-0004-refiners.md) | snapshot | Refiners: recipe matching and where each refiner is | current |
| [SPEC-0005](snapshot/SPEC-0005-extraction-farms.md) | snapshot | Extraction farms: pipe networks, fill, rate, and the farm check | current |
| [SPEC-0006](snapshot/SPEC-0006-planner-and-crafting.md) | snapshot | Planner and crafting: what the stock makes now and after a harvest | current |
| [SPEC-0007](snapshot/SPEC-0007-save-sections.md) | snapshot | Player sections of /json: inventory, plants, fleet, missions, technology, glyphs, tidy | current |
| [SPEC-0008](snapshot/SPEC-0008-grafana-dashboards.md) | snapshot | Grafana dashboards: status and crafting | current |
| [SPEC-0009](snapshot/SPEC-0009-conky-wall-screens.md) | snapshot | Conky wall screens: traveller and bases | current |
| [SPEC-0010](snapshot/SPEC-0010-save-backup-s3.md) | snapshot | Save backup to an S3-compatible bucket | current |
| [SPEC-0011](snapshot/SPEC-0011-install-and-service.md) | snapshot | Install, systemd units and the dev-deploy loop | current |
| [SPEC-0012](decisions/SPEC-0012-standard-library-only.md) | decision | Standard library only on the save side | accepted |
| [SPEC-0013](decisions/SPEC-0013-save-as-anchor.md) | decision | The save is an anchor, not a feed: project forward from its timestamps | accepted |
| [SPEC-0014](decisions/SPEC-0014-backup-versioned-keys.md) | decision | Save backup as mtime-keyed versions plus a fixed current/ key | accepted |
| [SPEC-0015](decisions/SPEC-0015-networks-from-geometry.md) | decision | Extraction networks come from pipe geometry alone, not from fill levels | accepted |
| [SPEC-0019](lessons/SPEC-0019-save-exporter-lessons.md) | lesson | Save-exporter lessons | current |

## game-data (0020-0039)

| id | type | title | status |
|---|---|---|---|
| [SPEC-0020](snapshot/SPEC-0020-game-pak-access.md) | snapshot | Game pak access: find, unpack, convert, grep and cite packed files | current |
| [SPEC-0021](snapshot/SPEC-0021-game-tables.md) | snapshot | Game tables: building nms_tables.json from the installed build | current |
| [SPEC-0022](snapshot/SPEC-0022-game-reference-values.md) | snapshot | Game reference values: hotspots, gases, power, storms, crops, refiners | current |
| [SPEC-0023](snapshot/SPEC-0023-hud-icons-and-portal-glyphs.md) | snapshot | HUD icons and portal glyphs: extracting game art locally | current |
| [SPEC-0024](snapshot/SPEC-0024-part-footprints.md) | snapshot | Base part footprints from placement scenes | current |
| [SPEC-0025](snapshot/SPEC-0025-harvest-all-mod.md) | snapshot | Harvest All radius: a loose-file mod built from the installed game | current |
| [SPEC-0026](snapshot/SPEC-0026-mechanics-compendium.md) | snapshot | Mechanics compendium: questions, stored sources, evidence levels, concepts | current |
| [SPEC-0030](decisions/SPEC-0030-numbers-from-the-installed-build.md) | decision | Read game numbers from the installed build, not a wiki or a community dataset | accepted |
| [SPEC-0031](decisions/SPEC-0031-collision-shapes-over-mesh-aabbs.md) | decision | Part footprints: collision shapes first, mesh bounding boxes as a fallback | accepted |
| [SPEC-0032](decisions/SPEC-0032-game-art-stays-out.md) | decision | Game art and game files stay out of the repository; ship extraction recipes | accepted |
| [SPEC-0039](lessons/SPEC-0039-game-data-lessons.md) | lesson | game-data lessons | current |

## live-hook (0040-0059)

| id | type | title | status |
|---|---|---|---|
| [SPEC-0040](snapshot/SPEC-0040-hook-attach-under-proton.md) | snapshot | Attaching pyMHF / NMS.py to the running game under Proton | current |
| [SPEC-0041](snapshot/SPEC-0041-executor-hot-patching.md) | snapshot | In-game executor and hot-patching mods without a restart | current |
| [SPEC-0042](snapshot/SPEC-0042-live-position-mod.md) | snapshot | live_position mod: player position at 4 Hz, trail, galactic address | current |
| [SPEC-0043](snapshot/SPEC-0043-planet-frame-heading-radius-address.md) | snapshot | Planet frame, compass, heading, nominal radius, sea level and universe address | current |
| [SPEC-0044](snapshot/SPEC-0044-weather-storms-planet-data.md) | snapshot | Weather, storms and the current planet''s generation data | current |
| [SPEC-0045](snapshot/SPEC-0045-visor-log-markers-chat.md) | snapshot | visor_log mod: analysis visor, game markers, hotspots and chat commands | current |
| [SPEC-0046](snapshot/SPEC-0046-galaxy-log-systems-names-portal.md) | snapshot | galaxy_log mod: star systems, system and planet names, portal codes | current |
| [SPEC-0047](decisions/SPEC-0047-attach-once-hot-patch-over-restart.md) | decision | Attach once per game run; change mods by hot-patching through the executor | accepted |
| [SPEC-0048](decisions/SPEC-0048-hook-files-as-raw-record.md) | decision | The mods write raw JSONL day files; interpretation lives in the exporter | accepted |
| [SPEC-0049](lessons/SPEC-0049-live-hook-lessons.md) | lesson | live-hook lessons | current |

## map-ui (0060-0079)

| id | type | title | status |
|---|---|---|---|
| [SPEC-0060](snapshot/SPEC-0060-map-page-and-frames.md) | snapshot | Map page: views, coordinate frames and projections | current |
| [SPEC-0061](snapshot/SPEC-0061-map-live-layer.md) | snapshot | Map live layer: position, heading, trail, follow, nearby and go-to | current |
| [SPEC-0062](snapshot/SPEC-0062-map-heights.md) | snapshot | Map heights: ground elevation from the trail, terrain raster, hillshade and contours | current |
| [SPEC-0063](snapshot/SPEC-0063-map-finds.md) | snapshot | Map finds: hotspots, deposits and marks, with player-recorded class and resource | current |
| [SPEC-0064](snapshot/SPEC-0064-map-interaction.md) | snapshot | Map interaction: tap targets, tooltip, selection, legend, Show menu, zoom, grid and declutter | current |
| [SPEC-0065](snapshot/SPEC-0065-map-globe.md) | snapshot | Map globe: orthographic planet view on a canvas | current |
| [SPEC-0066](snapshot/SPEC-0066-map-base-footprints-and-power.md) | snapshot | Map base view: building footprints, segments and the power card | current |
| [SPEC-0067](snapshot/SPEC-0067-map-sectors-and-places.md) | snapshot | Map sectors and the Places sheet | current |
| [SPEC-0068](snapshot/SPEC-0068-systems-page.md) | snapshot | Systems page: every star system seen, with portal addresses | current |
| [SPEC-0069](snapshot/SPEC-0069-map-design-rules.md) | snapshot | Map and systems pages: design rules, palette tokens and degraded states | current |
| [SPEC-0070](decisions/SPEC-0070-browser-map-over-kiosk-dashboard.md) | decision | An interactive browser page served by the exporter, not a kiosk dashboard | accepted |
| [SPEC-0071](decisions/SPEC-0071-svg-world-canvas-globe-no-libraries.md) | decision | SVG world in metres plus a canvas globe, with no mapping or charting libraries | accepted |
| [SPEC-0072](decisions/SPEC-0072-player-records-hotspot-class.md) | decision | Hotspot class and resource are recorded by the player, never guessed | accepted |
| [SPEC-0079](lessons/SPEC-0079-map-ui-lessons.md) | lesson | Map UI lessons | current |

## storage-deploy (0080-0099)

| id | type | title | status |
|---|---|---|---|
| [SPEC-0080](snapshot/SPEC-0080-history-store-and-ingest.md) | snapshot | History store and ingest | current |
| [SPEC-0081](snapshot/SPEC-0081-postgres-only-degraded-mode.md) | snapshot | Postgres only: behaviour when the store is down | current |
| [SPEC-0082](snapshot/SPEC-0082-environment-install-and-service.md) | snapshot | Python environment, install and the systemd service | current |
| [SPEC-0083](snapshot/SPEC-0083-managed-postgres.md) | snapshot | The managed Postgres cluster | current |
| [SPEC-0084](snapshot/SPEC-0084-machine-identity-for-secrets.md) | snapshot | Machine identity for secrets | current |
| [SPEC-0085](snapshot/SPEC-0085-exposing-the-map.md) | snapshot | Exposing the map on the LAN | current |
| [SPEC-0090](decisions/SPEC-0090-postgres-for-history.md) | decision | Postgres for the map''s history, with no file fallback | accepted |
| [SPEC-0091](decisions/SPEC-0091-uv-environment.md) | decision | A locked uv environment for the one dependency | accepted |
| [SPEC-0092](decisions/SPEC-0092-address-bound-machine-roles.md) | decision | Address-bound, non-expiring machine roles for host secrets | accepted |
| [SPEC-0099](lessons/SPEC-0099-storage-deploy-lessons.md) | lesson | Storage and deploy lessons | current |
