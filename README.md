# SU to Blender Live Link

Minimal real-time SketchUp to Blender link prototype.

## What works now

- SketchUp writes a full mesh snapshot to `C:\tmp\su_to_blender_live_link\snapshot.json`.
- Blender watches that snapshot file and creates or updates objects in a `SketchUp Live Link` collection.
- Faces are sent as SketchUp-triangulated mesh polygons, which handles offsets, holes, and complex faces more reliably than raw face outlines.
- Object names, rough hierarchy IDs, UV coordinates, camera data, material colors, alpha values, and exported SketchUp texture files are transferred.
- SketchUp exports material textures to `C:\tmp\su_to_blender_live_link\textures`.
- Blender can send selected mesh objects back to SketchUp over `127.0.0.1:37732`.
- SketchUp checks for changed models every `2` seconds and skips duplicate snapshots.
- SketchUp includes `Plugins > SU to Blender Live Link > Show Status` for connection diagnostics.
- Blender shows the latest receive status in `View3D > Sidebar > SU Link`.

Live polling is SketchUp to Blender. Blender to SketchUp is currently an explicit `Send Selected to SketchUp` action.

## Install in Blender

1. Open Blender.
2. Go to `Edit > Preferences > Add-ons > Install...`.
3. Select `packages/su_to_blender_blender_addon.zip`.
4. Enable `SU to Blender Live Link`.
5. Use `View3D > Sidebar > SU Link > Start Receiver`.
6. Confirm the panel shows `UI processor: running`.
7. To send Blender-created meshes back, select mesh objects and click `Send Selected to SketchUp`.

## Install in SketchUp

1. Open SketchUp.
2. Go to `Window > Extension Manager > Install Extension...`.
3. Select `packages/su_to_blender_live_link.rbz`.
4. Restart SketchUp.
5. Use `Plugins > SU to Blender Live Link > Start Live Sync`.

`Start Live Sync` also starts the SketchUp receiver on `127.0.0.1:37732`, which Blender uses for `Send Selected to SketchUp`. You can also start it separately with `Plugins > SU to Blender Live Link > Start SketchUp Receiver`.

If SketchUp feels slow on a large model, stop live sync and use `Sync Once` after hiding unnecessary geometry. The current prototype protects SketchUp by refusing live snapshots above `80,000` faces.

For quick development without reinstalling the `.rbz`, copy `sketchup/su_to_blender.rb` and the `sketchup/su_to_blender/` folder into SketchUp's `Plugins` folder.

Typical SketchUp Plugins folders:

- Windows: `%AppData%\SketchUp\SketchUp 2024\SketchUp\Plugins`
- macOS: `~/Library/Application Support/SketchUp 2024/SketchUp/Plugins`

Adjust the SketchUp version folder to match your installed version.

## Development Notes

Protocol is newline-delimited JSON over TCP:

```json
{
  "type": "snapshot",
  "protocol": 1,
  "source": "sketchup",
  "unit": "meter",
  "objects": []
}
```

The SketchUp side currently sends transformed world-space vertices in meters. The Blender side replaces object meshes in place and removes objects that disappear from the latest snapshot.

## Troubleshooting Link Issues

Use this order when SketchUp says it sent data but Blender stays empty:

1. In Blender, click `SU Link > Self Test`.
   - If this fails, the Blender receiver is not accepting packets correctly.
   - If this succeeds, `Raw packets` should increase.
2. In SketchUp, click `Test Blender Connection`.
   - This sends a ping and waits for Blender's ACK receipt.
   - SketchUp now only reports success after Blender confirms receipt.
3. In Blender, check:
   - `Connected: yes`
   - `Raw packets` greater than `0`
   - `Queue` greater than `0`
4. If `Raw packets` stays `0`, SketchUp is not reaching the receiver shown in this Blender window. Restart both apps and make sure only one Blender instance is running the add-on on port `37731`.
5. If `Raw packets` increases but no objects appear, the problem is in snapshot parsing or mesh creation; the Blender panel will show the Python error.
6. SketchUp now waits for Blender to finish applying a snapshot before accepting it as synced. If Blender reports an error, SketchUp will retry on the next change or manual `Sync Once` instead of silently treating the failed snapshot as complete.

If Blender shows `Writing to ID classes in this context is not allowed`, update to the latest add-on package and restart Blender. Version `0.3.0` applies incoming snapshots from a modal UI event processor instead of a timer/drawing context. The panel must show `UI processor: running`.

SketchUp writes snapshots in a background thread. From Blender add-on `0.4.0`, SU to Blender no longer depends on TCP packets; `Raw packets` is only for self-test/legacy diagnostics. For normal sync, check `Applied` and the snapshot file status line.

For texture sync, check Blender's `UV layers` and `Textures loaded/missing` lines. If `UV layers` is `0`, the SketchUp material probably has no front-face UV mapping in the exported faces. If missing is greater than `0`, inspect `C:\tmp\su_to_blender_live_link\textures` to confirm SketchUp exported the texture image.

For large scenes, the SketchUp face limit is now `1,000,000`. Use `Sync Selection Once` or `Force Resend Selection` for faster iteration on a selected group/component/area instead of exporting the whole model every time.

If sync feels slow, use SketchUp `Show Status` to inspect `Perf build/json/write` and `Snapshot size`, then compare it with Blender's `Last apply time`. For view-only updates use `Sync Camera Only`, which avoids rebuilding geometry.

## Next Useful Milestones

1. Add incremental updates using SketchUp observers instead of polling full snapshots.
2. Preserve component nesting as Blender parent-child transforms instead of flattened world-space meshes.
3. Transfer UVs and texture image files.
4. Add Blender to SketchUp transform sync for selected objects.
5. Package SketchUp as `.rbz` and Blender as a versioned `.zip`.
