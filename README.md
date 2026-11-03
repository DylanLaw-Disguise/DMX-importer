<div align="center">

# DMX Importer

**Patch, visualise and program DMX lighting fixtures in disguise Designer — from a single HTML file.**

[![single file](https://img.shields.io/badge/single%20file-index.html-5b6176)](index.html)
[![no build step](https://img.shields.io/badge/build%20step-none-4c9a5e)](#quick-start)
[![dependencies](https://img.shields.io/badge/dependencies-zero-4c9a5e)](#quick-start)
[![fixtures](https://img.shields.io/badge/fixtures-647%20·%202%2C878%20modes-3b6ea5)](#fixture-libraries)
[![GDTF](https://img.shields.io/badge/GDTF-supported-3b6ea5)](#gdtf)

</div>

---

Designer has no fixture library importer and no GDTF support. This fills that gap: open one HTML file in a browser, browse 647 fixtures from the [Open Fixture Library](https://open-fixture-library.org/) or drop in a `.gdtf`, lay the rig out on a DMX universe map, and push it straight into the open Designer session over its HTTP Session API.

It then goes further than patching — it reads back what is already in the scene, draws the rig in 3D, and writes keyframes onto Designer's timeline so a look programmed in the browser is cross-programmable in Designer.

<!-- Drop a capture at docs/screenshot.png and this will render. -->
<p align="center"><img src="docs/screenshot.png" alt="The Patch tab: universe map, fixture browser and detail panel" width="900"></p>

## Quick start

```bash
# no install, no build - just serve the folder (or open index.html directly)
python -m http.server 8766
```

1. Start **Designer** and open a project.
2. Open `index.html`.
3. Set **IP** and **Port** in the header — `127.0.0.1` and `81` for the local machine.
4. Hit **Fetch scene** to confirm the connection and pull in anything already patched.

> [!NOTE]
> Designer publishes the API through nginx, which listens on port **80** on some machines and **81** on others. If the port is wrong the tool retries the other one and keeps whichever answers.

> [!IMPORTANT]
> Nothing this tool writes is durable until you **save the project in Designer**. The Python API cannot create a save point, and an unsaved session can be rolled back — see [Gotchas](#gotchas-worth-knowing).

## The three tabs

### Patch

| | |
|---|---|
| **Browse** | Manufacturers bottom-left, models and their modes bottom-right. Picking a mode fills the detail panel — channel count, physical size, inferred driver, and the full channel map with the disguise slot each channel resolves to. |
| **Address** | Drag a fixture onto the universe map. Each block is exactly as wide as the mode's real footprint, overlaps are refused, and valid landing channels are highlighted while you drag. Add universes as you need them. |
| **Name** | Names auto-increment as you place, so a rig of 24 pars does not need 24 manual edits. |
| **Import** | **Import all to Designer** sends them one fixture at a time with a per-fixture log, so a failure stays contained and tells you exactly which one. |

Channels with no disguise equivalent — Lime, Indigo, UV, Deep Red, Mint — are flagged **unsupported** rather than silently aliased onto the nearest colour, and forcing one anyway shows a warning.

### Visualiser

A dependency-free canvas 3D view of the rig, read live from the session.

- Fixtures draw as lamp icons with an aim arrow taken from each instance's world matrix, plus a beam cone once a colour is set.
- Click to select; the console on the right is laid out like an ETC Eos — intensity, RGB **or** CCT with tint, pan and tilt.
- **FLASH** writes full white at the playhead so you can pick a fixture out of the rig, and a second press removes exactly what it wrote.

### Timeline

- Shows the layers the tool created per fixture, with keyframes at their real times.
- Dial a look on the console, then **Record** it at the playhead. Drag keyframes along a layer, scrub by dragging the time readout, resize layers, and play the whole thing back in the browser.
- Keyframes are written as real field sequences, so they appear in Designer's own timeline and can be programmed from either side.

## How it talks to Designer

Everything goes through one endpoint:

```
POST http://<host>:<port>/api/session/python/execute
```

The important part is that **Designer does not let you write DMX channel values directly** — a DMX light samples rendered content. So lighting a fixture means building content that happens to be the colour you want:

| What you want | What the tool actually creates |
|---|---|
| Colour and intensity | An `RGBColourModule` layer whose `mapping` targets the light's auto-created `DirectProjection` at `objects/dmxlights/<name> (direct).apx` |
| Raw channels — Mix, CCT, tint, pan, tilt | A `DmxControlModule` layer holding up to eight `DmxCommand` resources, each with a `channel`, `min_value` and `max_value` |
| Keyframes | `field.sequence.setFloat(localT, v)` on the layer's fields, with `disableSequencing` cleared |

### Mix: RGB or CCT

Fixtures with a **Mix** channel ignore their RGB channels unless Mix is at full, so the choice is explicit:

- **RGB** — Mix is driven to `255`, and the colour layer carries the look.
- **CCT** — Mix goes to `0` to put the fixture on its studio engine, the colour layer is pinned to white so content is not black, and colour temperature, tint and intensity come from the DMX control layer.

### Driver model

| Driver | Channel offsets | Used for |
|---|---|---|
| `FixtureDriver` | none | Modes where nothing maps to a disguise slot — a passthrough |
| `DmxDriver` | **none at all** — no `.lamp` exists | Designer's own stock drivers (`rgb.apx`). Readable layout: none; it is whatever Designer applies internally for the colour mode |
| `GenericLampDriver` | every slot, explicitly | Everything this tool patches |

> [!TIP]
> `GenericLampDriver` derives from `DmxDriver`, so nothing is lost by always using it — and every offset becomes readable *and* editable.

### Reading and editing an existing rig

**Fetch scene** reads each light's driver in full: colour mode, all eleven lamp offsets, the four pan/tilt offsets with their ranges, and any default `DmxCommand`s. The detail panel shows one row per channel saying what controls it, and for a `GenericLampDriver` each row is a dropdown you can reassign and write back.

- Rows read straight off the driver are shown as fact; rows merely implied by a bare `DmxDriver`'s colour mode are labelled **(implied)**.
- A driver used by more than one fixture is flagged, because editing the map changes all of them.
- A bare `DmxDriver` light can be given its own `GenericLampDriver` — seeded from the implied layout — so its map becomes editable. The old driver is left in place, since stock drivers are shared.

You can also re-address a fetched light by dragging its block on the map; only the `DmxString`'s universe and channel change, never its fixture, driver or channel map.

## Fixture libraries

### Open Fixture Library

647 fixtures across 134 manufacturers and 2,878 modes ship inline as a ~895 KB JSON index. **Rescan Fixtures folder** rebuilds it in-browser from a local OFL export.

The resolver handles the awkward parts of the OFL schema: `fineChannelAliases`, `switchChannels`, `templateChannels` registered under every pixel key they can be stamped with, and matrix inserts expanded to *pixels × per-pixel* so a tube's channel count is right.

### GDTF

Drop in a `.gdtf` and it is read entirely in the browser — the ZIP is inflated with `DecompressionStream('deflate-raw')`, `description.xml` is parsed, and each `DMXMode` becomes the same record shape as an OFL fixture. That means the patch grid, driver inference, 3D view and timeline writer all work on it unchanged.

It reads `Offset` pairs as coarse/fine, takes the attribute from whichever of `LogicalChannel` or `ChannelFunction` is recognised (real files disagree between the two), and pulls Kelvin and degree ranges from `PhysicalFrom` / `PhysicalTo`.

## Known limits

- GDTF `GeometryReference` and `DMXBreak > 1` are not handled.
- Some OFL matrix inserts are flagged unsupported rather than guessed at.
- The aim arrow shows the fixture's rig orientation, not live pan/tilt.
- A bare `DmxDriver`'s channel layout can only be inferred from its colour mode.
- Colours with no disguise slot are flagged, not mapped.

## Gotchas worth knowing

Hard-won behaviour of Designer's embedded Python API, if you fork this:

<details>
<summary><b>Expand</b></summary>

- **A script that raises reverts the whole project to its last save** — not just its own changes, but everything the API created earlier in the session. Generated Python must return `{"error": ...}` instead of raising.
- **Binding errors escape `try`/`except`.** An `AttributeError` from the C++ layer aborts the script regardless of a surrounding handler, so check a driver's type by name before touching type-specific fields.
- **`returnValue` is a JSON *string*** — parse it twice. Success is `status.code == 0`.
- **`disableSequencing` defaults to `True`**, so a field stores keys but evaluates as a constant and draws no track until it is cleared.
- **`resourceManager.load()` returns an untyped `Resource`.** Use `loadOrCreate`, `allResources`, or `state.stage.dmxLights`.
- **`loadOrCreate(path, GenericLampDriver)` returns `None`** if that path already holds a bare `DmxDriver` — it does not upgrade the type. Trash the old resource first.
- **`FixtureGroup.instances` is empty immediately after creation** — instances are built on the next frame.
- **nginx's 502 page carries no CORS headers**, so from a browser "Designer is not running" is indistinguishable from an unreachable host. An opaque `mode: 'no-cors'` probe tells them apart.
- `nLayers` on `Track` is a method, not a property. `field.name` is too.

</details>

## Contributing

It is one file with no build step — edit `index.html` and reload. The fixture index lives in `<script id="fixtureIndex" type="application/json">` on a single line, so avoid reformatting the file wholesale.

---

<div align="center">
<sub>Not an official disguise product.</sub>
</div>
