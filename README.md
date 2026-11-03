# OFL Importer

A single HTML file for importing DMX lighting fixtures into disguise Designer.

Browse a fixture library, lay the rig out on a DMX universe map, and import it into the open Designer session over its HTTP Session API.

## Quick start

1. Start Designer and open a project.
2. Open `index.html` in a browser.
3. Set **IP** and **Port** in the header (`127.0.0.1` and `81` for the local machine).
4. Pick fixtures, address them on the map, and press **Import all to Designer**.

Save the project in Designer to keep what has been imported.

## Patching

Pick a manufacturer bottom-left and a model and mode bottom-right. The detail panel shows the channel count, physical size, driver and channel map for the selected mode.

Drag a fixture onto the universe map to address it. Each block is as wide as the mode's channel footprint, and names increment as you place them. Add universes as you need them.

**Import all to Designer** sends the fixtures one at a time and logs each one.

## Fixture libraries

- **Open Fixture Library** — 647 fixtures across 134 manufacturers and 2,878 modes, included in the file. **Rescan Fixtures folder** rebuilds the index from a local OFL export.
- **GDTF** — **Load GDTF** reads a `.gdtf` in the browser and patches from it the same way.

## Reading the scene

**Fetch scene** reads the DMX lights already patched in the open session and shows them on the map alongside anything you are planning.

For each one the detail panel lists what every channel controls, read from its driver. Where the driver stores explicit offsets the rows are editable, so a channel can be reassigned and written back.

A fetched light can also be re-addressed by dragging its block on the map — only its DMX address changes.

## Visualiser

A 3D view of the rig read from the session. Click a fixture to select it and control intensity, colour (RGB or CCT), and pan and tilt from the console beside it. **FLASH** puts the selected fixture at full white to identify it.

## Timeline

Record the console's current look as a keyframe at the playhead. Keyframes can be dragged along a layer, the timeline can be scrubbed, and the result plays back in the browser. Keyframes are written into Designer's timeline.
