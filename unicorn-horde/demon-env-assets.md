# Demon Assault — Static Environment Asset List

All dimensions are in **world units** (same space the `imp.glb` / `hellhound.glb` models live in).
The arena floor is **3000 (X) × 2300 (Z)**; the perimeter wall is **60** tall.

**Scale reference (so props read at the right size):**
| Thing | Approx. width (world units) |
|---|---|
| Player (armored human) | ~30 (radius 15) |
| Imp | ~36 · Hellhound ~44 · Gargoyle ~48 · Bilespawn ~52 |
| Brute | ~76 · **Baron ~124** |
| Wall height | 60 |

**Delivery format (match the existing character pipeline):**
GLB, Y-up, origin at the base center (feet on ground = y0), single mesh where possible, webp textures ≤512px, run through
`npx @gltf-transform/cli optimize IN OUT --compress false --simplify false --texture-compress webp --texture-size 512`.
Keep **lights, fire, embers, and the pupil-track / lid-open motion in-engine** — model the solid geometry only, but expose the noted child nodes by name.

---

## Asset table

| # | Asset | Current identity | Count | Footprint (X × Z) | Height (Y) | Notes |
|---|-------|------------------|-------|-------------------|-----------|-------|
| 1 | **Ground floor** | Charred hellstone + lava veins | 1 | 3000 × 2300 (flat) | 0 | Texture, not a mesh. Author at **2048×1570** (or 3000×2300), aspect 1.30:1. Near-black cool stone w/ molten core, bright lava veins + pools, worn stone processional paths (center ring + N/S/E/W spokes + perimeter loop). |
| 2 | **Perimeter wall** | Dark stone rampart + ember trim | 4 pieces | 2 × (3000 × 40) + 2 × (40 × 2300) | 60 | Or deliver 1 tiling segment + a wall texture. Thin glowing ember cap runs along the top edge. |
| 3 | **Brazier** | Stone fire brazier (was "bins") | 6 pairs | pair fills 66 × 62; each ~30 dia | ~64 | Deployed as a **pair** ~34 apart. Pedestal column + wide fire bowl. Fire sphere + point light stay in-engine. |
| 4 | **Skull pile (tall)** | Bone mound (was market stall) | ~8 | 58 × 150 | ~50 | Dark mound base + heaped skulls. Solid cover the size of the footprint. |
| 5 | **Skull pile (wide)** | Bone mound | ~8 | 150 × 58 | ~50 | Same as #4, rotated footprint. |
| 6 | **Fiery watching eye** | Demon eye on a gnarled stalk (was "tree") | ~15 | ~34 dia (stalk r7, eyeball r17) | ~100 | Tapered stalk + dark socket + glowing eyeball at top. **Expose `eyeball`, `iris`, `pupil` as named nodes** — engine rotates the pupil to track the player. Glow + ground firelight pool (240×240) stay in-engine. |
| 7 | **Extraction pad** | Holy/green safe disc (world center) | 1 | ~206 dia | 6 | Low disc, flush to ground. Green stays the one cool/safe focal color. Torus rim + `$/EXTRACT` decal (bake into texture or leave to engine) + point light stay in-engine. |
| 8 | **Extraction portal** | Green ring doorway | 1 | ~48 × 48 (vertical) | ~48 | Concentric vertical rings forming a doorway/tunnel; opening faces south. Floats at north edge of the pad; engine scales it up to ~2.5× as extraction charges. |
| 9 | **Reliquary chest** | Dark loot chest, gold band + ember lock | 5 / run | 42 × 31 | ~34 | **Expose `lid` as a named node** (engine tips it open). Gold band + ember rune lock. |
| — | *Outer apron* | Flat dark backdrop plane | 1 | 9000 × 6900 | -1 | Optional. Solid near-black plane beyond the walls so the horizon isn't a hard edge; probably fine to leave procedural. |

---

## Canvas textures currently generated in-code (replace with image files if going that route)

| Texture | Current px | Used by |
|---|---|---|
| Ground | 1500 × 1150 | Floor (#1) |
| Skull | 64 × 64 | Skull piles (#4/#5) |
| Firelight pool (glow) | 64 × 64 | Eyes / braziers ground glow |
| Soft shadow blob | 64 × 64 | Under every prop/character |
| Pad `$/EXTRACT` decal | 256 × 256 | Extraction pad (#7) |
