# Composite Building System — Design Spec

## Summary

Replace monolithic building generators with a modular assembly system. Buildings are composed from small reusable parts (base, floors, mutations, details, caps) grown upward using biological rules. Velocity directly controls height. Inter-building connectors (trees, cars, bridges) create city fabric between buildings based on chord relationships.

## Building Assembly

### Parts Catalog

- **Base**: slab, cylinder, L-shape, courtyard
- **Floor**: standard box, tapered, rounded, chamfered
- **Mutation** (15% chance per floor, increases with height): balcony, setback, cantilever, terrace
- **Detail** (driven by chord intervals): window bay, AC unit, water tank, antenna, vine, awning
- **Cap**: flat, pitched, dome, spire, garden

### Chord-to-Building Mapping

| Input | Controls |
|---|---|
| Velocity (0-127) | Floor count (3-18), direct proportion |
| Root note (C-B) | Base shape + color hue |
| Chord type | Growth bias (major=clean, minor=organic, dim=industrial, aug=rounded) |
| Note count | Detail density per floor |
| Intervals | Which details spawn (3rd=balconies, 5th=window bays, 7th=rooftop items) |

### Growth Process

1. Place base at grid cell (shape from root note)
2. For each floor (count from velocity):
   - Inherit previous floor's footprint
   - Roll for mutation (15% base chance, +2% per floor)
   - Apply chord-type growth bias
3. Detail pass: iterate chord intervals, add protrusions
4. Place cap (from chord type)

## Inter-building Connections

| Chord Relationship | Connector |
|---|---|
| Same root note | Shared courtyard (trees, bench) |
| Same chord type | Walkway/bridge |
| No relationship | Street elements (car, lamppost) |
| Adjacent cells | Physical bridge at shared floor height |
| Far cells | Ground-level trees/path only |

### Connector Assets (low-poly, stylized)

- Tree: cone + cylinder, 2-3 sizes
- Car: small rounded box
- Bench: slab on two legs
- Bridge: flat box spanning between buildings
- Path: lighter ground strip

## Grid & Camera

- 80x80 grid, CELL_SIZE=9, DLA/Eden/Sprawl placement (unchanged)
- Camera: fixed position (0, 220, 160), looking at origin, FOV 40
- No camera movement — city grows underneath
- Height = velocity: 3-18 floors, each ~2.5 units

## Performance

- Reuse geometries from parts catalog (shared BufferGeometry instances)
- Share materials per chord style (reduce material count)
- Keep InstancedMesh for roads/lamps
- Connectors use InstancedMesh pools (trees, cars, benches)
