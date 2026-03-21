# Composite Building System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace monolithic building generators with a modular composite assembly system where buildings grow floor-by-floor from a parts catalog, velocity directly controls height, inter-building connectors create city fabric, and the camera is fixed overhead.

**Architecture:** Single-file `index.html` refactor. Remove 12 individual generator functions, replace with one `assembleBuilding()` that stacks parts from a shared catalog. Add `spawnConnectors()` that reads the previous chord to place trees/cars/bridges between buildings. Fix camera position at init. All connector assets use InstancedMesh pools.

**Tech Stack:** Three.js 0.162.0 (CDN), vanilla JS, HTML Canvas textures

**Spec:** `docs/superpowers/specs/2026-03-21-composite-buildings-design.md`

---

### Task 1: Fix camera and remove tracking

**Files:**
- Modify: `index.html` (camera setup ~L152-158, updateCamera ~L634-650, animate ~L816-828)

Remove all camera lerping and dynamic repositioning. Set camera once at init to frame the full grid.

- [ ] **Step 1: Replace camera setup**

Replace the camera position, desiredCamPos, desiredCamTarget, currentCamTarget with a single fixed setup:

```javascript
const camera = new THREE.PerspectiveCamera(40, W / H, 0.1, 1200);
camera.position.set(0, 280, 200);
camera.lookAt(0, 0, 0);
```

- [ ] **Step 2: Remove camera tracking from animate()**

Remove these lines from `animate()`:
```javascript
camera.position.lerp(desiredCamPos, 0.018);
currentCamTarget.lerp(desiredCamTarget, 0.018);
camera.lookAt(currentCamTarget);
```
Replace with just:
```javascript
camera.lookAt(0, 0, 0);
```

- [ ] **Step 3: Remove updateCamera() function and all calls to it**

Delete `updateCamera()` function entirely. Remove the call in `addBuilding()`. Remove `desiredCamPos`, `desiredCamTarget`, `currentCamTarget` variables. Remove camera updates from `resetCity()`.

- [ ] **Step 4: Update resize handler to maintain fixed lookAt**

```javascript
window.addEventListener('resize', () => {
  W = window.innerWidth; H = window.innerHeight;
  camera.aspect = W / H;
  camera.updateProjectionMatrix();
  renderer.setSize(W, H);
});
```

- [ ] **Step 5: Push fog far plane out to cover full grid**

Change fog to accommodate the wider view:
```javascript
scene.fog = new THREE.Fog(0x0a0a28, 150, 600);
```

- [ ] **Step 6: Verify and commit**

Open in browser. Camera should be fixed, showing the full grid area. Buildings grow underneath without camera moving.

```bash
git add index.html && git commit -m "feat: fix camera at static overhead position"
git push
```

---

### Task 2: Parts catalog — shared geometries and materials

**Files:**
- Modify: `index.html` (new section after TEXTURES & MATERIALS, ~L465)

Create the reusable parts that all buildings will be assembled from. These are shared geometry instances — not per-building allocations.

- [ ] **Step 1: Define base geometries**

Add after the TEXTURES & MATERIALS section:

```javascript
// ===================== PARTS CATALOG =====================
const FLOOR_H = 2.5;
const CELL_FIT = CELL_SIZE * 0.8;

// Base shapes (shared geometries, never modified)
const PARTS = {
  // Floor sections — unit-sized, scaled per building
  floorBox:      new THREE.BoxGeometry(1, FLOOR_H, 1),
  floorChamfer:  new THREE.CylinderGeometry(0.48, 0.5, FLOOR_H, 8),
  floorRound:    new THREE.CylinderGeometry(0.5, 0.5, FLOOR_H, 16),

  // Mutations
  balcony:       new THREE.BoxGeometry(1.3, 0.3, 0.4),
  terrace:       new THREE.BoxGeometry(1.1, FLOOR_H * 0.5, 1.1),

  // Details
  windowBay:     new THREE.BoxGeometry(0.15, 0.8, 0.5),
  acUnit:        new THREE.BoxGeometry(0.4, 0.3, 0.3),
  antenna:       new THREE.CylinderGeometry(0.03, 0.05, 2.5, 4),
  waterTank:     new THREE.CylinderGeometry(0.3, 0.3, 0.6, 8),
  vine:          new THREE.BoxGeometry(0.1, 1.5, 0.6),
  awning:        new THREE.BoxGeometry(0.8, 0.05, 0.4),
  dish:          new THREE.SphereGeometry(0.25, 6, 4, 0, Math.PI),

  // Caps
  capFlat:       new THREE.BoxGeometry(1, 0.15, 1),
  capPitched:    new THREE.ConeGeometry(0.55, 1.5, 4),
  capDome:       new THREE.SphereGeometry(0.5, 12, 8, 0, Math.PI * 2, 0, Math.PI / 2),
  capSpire:      new THREE.ConeGeometry(0.12, 3, 4),
  capGarden:     new THREE.BoxGeometry(1, 0.3, 1),
};
```

- [ ] **Step 2: Define growth bias per chord type**

```javascript
// Growth personality per chord type
const GROWTH = {
  maj:   { taper: 0.02, mutateBase: 0.10, organic: 0.1, floorGeo: 'floorBox',    cap: 'capFlat' },
  min:   { taper: 0.04, mutateBase: 0.22, organic: 0.5, floorGeo: 'floorBox',    cap: 'capPitched' },
  dim:   { taper: 0.01, mutateBase: 0.18, organic: 0.3, floorGeo: 'floorBox',    cap: 'capFlat' },
  aug:   { taper: 0.03, mutateBase: 0.15, organic: 0.2, floorGeo: 'floorRound',  cap: 'capDome' },
  dom7:  { taper: 0.03, mutateBase: 0.12, organic: 0.15,floorGeo: 'floorBox',    cap: 'capFlat' },
  min7:  { taper: 0.05, mutateBase: 0.25, organic: 0.6, floorGeo: 'floorBox',    cap: 'capSpire' },
  maj7:  { taper: 0.02, mutateBase: 0.08, organic: 0.1, floorGeo: 'floorChamfer',cap: 'capDome' },
  dim7:  { taper: 0.01, mutateBase: 0.20, organic: 0.4, floorGeo: 'floorBox',    cap: 'capFlat' },
  sus2:  { taper: 0.03, mutateBase: 0.14, organic: 0.2, floorGeo: 'floorRound',  cap: 'capFlat' },
  sus4:  { taper: 0.02, mutateBase: 0.12, organic: 0.15,floorGeo: 'floorBox',    cap: 'capPitched' },
  power: { taper: 0.00, mutateBase: 0.06, organic: 0.05,floorGeo: 'floorBox',    cap: 'capFlat' },
  chord: { taper: 0.02, mutateBase: 0.15, organic: 0.3, floorGeo: 'floorBox',    cap: 'capFlat' },
  minMaj7:{taper:0.04, mutateBase: 0.22, organic: 0.5, floorGeo: 'floorBox',    cap: 'capSpire' },
  minadd9:{taper:0.04, mutateBase: 0.20, organic: 0.45,floorGeo: 'floorBox',    cap: 'capPitched' },
  add9:  { taper: 0.03, mutateBase: 0.15, organic: 0.2, floorGeo: 'floorRound',  cap: 'capDome' },
  min9:  { taper: 0.05, mutateBase: 0.24, organic: 0.55,floorGeo: 'floorBox',    cap: 'capSpire' },
  dom9:  { taper: 0.03, mutateBase: 0.12, organic: 0.15,floorGeo: 'floorBox',    cap: 'capFlat' },
  maj9:  { taper: 0.02, mutateBase: 0.08, organic: 0.1, floorGeo: 'floorChamfer',cap: 'capDome' },
};
```

- [ ] **Step 3: Define detail mapping from intervals**

```javascript
// Which chord intervals trigger which details
const INTERVAL_DETAILS = {
  3:  'balcony',    // minor 3rd
  4:  'balcony',    // major 3rd
  5:  'windowBay',  // perfect 4th
  7:  'windowBay',  // perfect 5th
  10: 'antenna',    // minor 7th
  11: 'antenna',    // major 7th
  14: 'waterTank',  // 9th
};
```

- [ ] **Step 4: Commit**

```bash
git add index.html && git commit -m "feat: add parts catalog with shared geometries and growth config"
git push
```

---

### Task 3: assembleBuilding() — the composite growth engine

**Files:**
- Modify: `index.html` (replace all 12 generator functions + GENERATORS map with one assembleBuilding)

- [ ] **Step 1: Write assembleBuilding()**

Replace all individual build* functions and the GENERATORS map with:

```javascript
// ===================== COMPOSITE BUILDING ASSEMBLY =====================
function assembleBuilding(p) {
  const g = new THREE.Group();
  const growth = GROWTH[p.chordType] || GROWTH.chord;
  const style = p.style;
  const rng = p.rng;

  // Floor count from velocity (3-18)
  const floorCount = Math.max(3, Math.min(18, Math.round(3 + (p.velocity / 127) * 15)));

  // Base footprint
  let footW = Math.min(CELL_FIT, 4 + p.noteCount * 0.8 + rng() * 2);
  let footD = Math.min(CELL_FIT, 4 + rng() * 2);

  // Base slab
  const baseMat = solidMat(p.color.clone().multiplyScalar(0.6), 0.85, 0.15);
  const base = new THREE.Mesh(new THREE.BoxGeometry(footW + 0.5, 0.4, footD + 0.5), baseMat);
  base.position.y = 0.2;
  g.add(base);

  // Grow floors
  let currentW = footW;
  let currentD = footD;
  const floorGeo = PARTS[growth.floorGeo];
  const intervals = p.intervals || [];

  for (let f = 0; f < floorCount; f++) {
    const floorY = 0.4 + f * FLOOR_H;
    const heightRatio = f / floorCount;

    // Mutation chance increases with height
    const mutateChance = growth.mutateBase + heightRatio * 0.15;

    // Taper
    currentW *= (1 - growth.taper);
    currentD *= (1 - growth.taper);

    // Floor texture
    const floorCols = Math.max(2, Math.round(currentW / 2));
    const floorRows = 1;
    const tex = makeWindowTex(floorRows, floorCols,
      colorToHex(p.color.clone().multiplyScalar(0.4 + rng() * 0.15)),
      style.litP, rng);
    const mat = buildingMat(tex, style);

    // Floor mesh
    const floor = new THREE.Mesh(floorGeo, mat);
    floor.scale.set(currentW, 1, currentD);
    floor.position.y = floorY + FLOOR_H / 2;
    g.add(floor);

    // Organic offset (minor/dim chords drift more)
    if (rng() < growth.organic * 0.3) {
      floor.position.x += (rng() - 0.5) * 0.4;
      floor.position.z += (rng() - 0.5) * 0.4;
    }

    // Mutations
    if (rng() < mutateChance) {
      const roll = rng();
      if (roll < 0.4) {
        // Balcony
        const balcony = new THREE.Mesh(PARTS.balcony, baseMat);
        const side = rng() > 0.5 ? 1 : -1;
        const face = rng() > 0.5;
        if (face) {
          balcony.position.set(0, floorY + 0.15, side * (currentD / 2 + 0.15));
          balcony.scale.set(currentW * 0.6, 1, 1);
        } else {
          balcony.position.set(side * (currentW / 2 + 0.15), floorY + 0.15, 0);
          balcony.scale.set(1, 1, currentD * 0.6);
          balcony.rotation.y = Math.PI / 2;
        }
        g.add(balcony);
      } else if (roll < 0.7) {
        // Setback — shrink for remaining floors
        currentW *= 0.85;
        currentD *= 0.85;
      } else {
        // Cantilever — widen slightly for one floor
        floor.scale.set(currentW * 1.1, 1, currentD * 1.1);
      }
    }

    // Detail pass — driven by intervals in the chord
    for (const iv of intervals) {
      const detailKey = INTERVAL_DETAILS[iv];
      if (!detailKey || !PARTS[detailKey]) continue;
      if (rng() > 0.35) continue; // not every floor gets every detail

      const detail = new THREE.Mesh(PARTS[detailKey],
        detailKey === 'vine'
          ? solidMat(new THREE.Color().setHSL(0.3, 0.5, 0.2 + rng() * 0.1), 0.9, 0.05)
          : solidMat(p.color.clone().multiplyScalar(0.5 + rng() * 0.2), 0.7, 0.3)
      );

      // Place on a random face
      const faceRoll = rng();
      if (faceRoll < 0.25) {
        detail.position.set(currentW / 2 + 0.08, floorY + FLOOR_H * 0.5, (rng() - 0.5) * currentD * 0.6);
      } else if (faceRoll < 0.5) {
        detail.position.set(-currentW / 2 - 0.08, floorY + FLOOR_H * 0.5, (rng() - 0.5) * currentD * 0.6);
      } else if (faceRoll < 0.75) {
        detail.position.set((rng() - 0.5) * currentW * 0.6, floorY + FLOOR_H * 0.5, currentD / 2 + 0.08);
      } else {
        detail.position.set((rng() - 0.5) * currentW * 0.6, floorY + FLOOR_H * 0.5, -currentD / 2 - 0.08);
      }
      g.add(detail);
    }
  }

  // Cap
  const capH = 0.4 + floorCount * FLOOR_H;
  const capGeo = PARTS[growth.cap];
  const capMesh = new THREE.Mesh(capGeo,
    growth.cap === 'capGarden'
      ? solidMat(new THREE.Color().setHSL(0.3, 0.4, 0.25), 0.9, 0.05)
      : solidMat(p.color.clone().multiplyScalar(0.7), 0.5, 0.3)
  );
  capMesh.scale.set(currentW, 1, currentD);
  capMesh.position.y = capH;
  if (growth.cap === 'capPitched') capMesh.rotation.y = Math.PI / 4;
  g.add(capMesh);

  // Rooftop details (antenna, water tank, dish) — if chord has 7th or 9th
  if (intervals.includes(10) || intervals.includes(11)) {
    const ant = new THREE.Mesh(PARTS.antenna, solidMat(0x666677, 0.5, 0.6));
    ant.position.set((rng() - 0.5) * currentW * 0.4, capH + 1.25, (rng() - 0.5) * currentD * 0.4);
    g.add(ant);
  }
  if (intervals.includes(14)) {
    const tank = new THREE.Mesh(PARTS.waterTank, solidMat(0x556666, 0.6, 0.4));
    tank.position.set((rng() - 0.5) * currentW * 0.3, capH + 0.3, (rng() - 0.5) * currentD * 0.3);
    g.add(tank);
  }

  return g;
}
```

- [ ] **Step 2: Update addBuilding() to use assembleBuilding()**

Replace the generator lookup and params in addBuilding():

```javascript
function addBuilding(chord, velocity) {
  const seed = chord.notes.reduce((a, b) => a * 31 + b, 7) + Date.now();
  const rng = seededRandom(seed);
  const style = STYLES[chord.type] || STYLES.chord;
  const color = colorFromStyle(chord.root, style);
  const placement = findPlacement(rng);
  if (!placement) return;
  const { gx, gz, algo } = placement;
  const world = gridToWorld(gx, gz);

  const group = assembleBuilding({
    velocity,
    color,
    style,
    rng,
    chordType: chord.type,
    noteCount: chord.notes.length,
    intervals: chord.intervals || [],
    root: chord.root,
  });

  group.position.set(world.x, 0, world.z);
  group.rotation.y = rng() * Math.PI * 2;
  group.scale.y = 0.01;
  scene.add(group);
  spawning.push({ group, progress: 0 });
  buildings.push({ group, gx, gz, chord, velocity });

  generateRoadsForBuilding(gx, gz, rng);
  startRoadGrowth();

  countEl.textContent = buildings.length;
  const algoClass = algo === 'DLA' ? 'algo-dla' : algo === 'Eden' ? 'algo-eden' : algo === 'Sprawl' ? 'algo-sprawl' : 'dim';
  algoEl.className = algoClass;
  algoEl.textContent = algo;
}
```

- [ ] **Step 3: Delete all old generator functions**

Remove: `buildSkyscraper`, `buildGothic`, `buildIndustrial`, `buildDome`, `buildArtDeco`, `buildCathedral`, `buildCrystal`, `buildFactory`, `buildWaterTower`, `buildArch`, `buildBrutalist`, `buildGeneric`, and the `GENERATORS` map.

- [ ] **Step 4: Update buildings array to store metadata**

Change `buildings` from storing bare groups to storing objects with chord context (needed for connectors in Task 4). Update resetCity() accordingly:

```javascript
function resetCity() {
  buildings.forEach(b => scene.remove(b.group));
  // ... rest unchanged
}
```

Also update spawning references if they use `buildings[i]` directly.

- [ ] **Step 5: Verify and commit**

Open in browser. Play chords — buildings should grow as stacked composite floors with varying shapes, balconies, details. Different chord types should produce visually distinct growth patterns.

```bash
git add index.html && git commit -m "feat: replace generators with composite building assembly system"
git push
```

---

### Task 4: Inter-building connectors

**Files:**
- Modify: `index.html` (new section + additions to addBuilding)

- [ ] **Step 1: Create connector InstancedMesh pools**

Add after the streetlamp section:

```javascript
// ===================== CONNECTORS (InstancedMesh pools) =====================
const CONN_MAX = 600;

// Tree: cone on cylinder
const treePoolGeo = new THREE.ConeGeometry(0.8, 2.2, 6);
const treePoolMat = new THREE.MeshStandardMaterial({
  color: 0x2a5a2a, roughness: 0.9, metalness: 0.05,
  emissive: 0x0a1a0a, emissiveIntensity: 0.2,
});
const treeMesh = new THREE.InstancedMesh(treePoolGeo, treePoolMat, CONN_MAX);
treeMesh.count = 0;
scene.add(treeMesh);

const trunkGeo = new THREE.CylinderGeometry(0.1, 0.15, 1.2, 5);
const trunkMat = new THREE.MeshStandardMaterial({ color: 0x4a3a2a, roughness: 0.9 });
const trunkMesh = new THREE.InstancedMesh(trunkGeo, trunkMat, CONN_MAX);
trunkMesh.count = 0;
scene.add(trunkMesh);

// Car: small box
const carGeo = new THREE.BoxGeometry(1.2, 0.6, 2.2);
const carMat = new THREE.MeshStandardMaterial({ color: 0x334455, roughness: 0.5, metalness: 0.4 });
const carMesh = new THREE.InstancedMesh(carGeo, carMat, CONN_MAX);
carMesh.count = 0;
scene.add(carMesh);

// Bench
const benchGeo = new THREE.BoxGeometry(1.5, 0.25, 0.4);
const benchMat = new THREE.MeshStandardMaterial({ color: 0x5a4a3a, roughness: 0.8 });
const benchMesh = new THREE.InstancedMesh(benchGeo, benchMat, CONN_MAX);
benchMesh.count = 0;
scene.add(benchMesh);

let treeCount = 0, carCount = 0, benchCount = 0;

function addTree(wx, wz) {
  if (treeCount >= CONN_MAX) return;
  const s = 0.7 + Math.random() * 0.6;
  // Trunk
  dummy.position.set(wx, 0.6, wz);
  dummy.scale.set(s, s, s);
  dummy.updateMatrix();
  trunkMesh.setMatrixAt(treeCount, dummy.matrix);
  // Canopy
  dummy.position.y = 1.8 * s;
  dummy.updateMatrix();
  treeMesh.setMatrixAt(treeCount, dummy.matrix);
  treeCount++;
  treeMesh.count = treeCount;
  trunkMesh.count = treeCount;
  treeMesh.instanceMatrix.needsUpdate = true;
  trunkMesh.instanceMatrix.needsUpdate = true;
  dummy.scale.set(1, 1, 1);
}

function addCar(wx, wz, angle) {
  if (carCount >= CONN_MAX) return;
  dummy.position.set(wx, 0.3, wz);
  dummy.rotation.set(0, angle, 0);
  // Random car color via instance color
  dummy.updateMatrix();
  carMesh.setMatrixAt(carCount, dummy.matrix);
  carCount++;
  carMesh.count = carCount;
  carMesh.instanceMatrix.needsUpdate = true;
  dummy.rotation.set(0, 0, 0);
}

function addBench(wx, wz, angle) {
  if (benchCount >= CONN_MAX) return;
  dummy.position.set(wx, 0.13, wz);
  dummy.rotation.set(0, angle, 0);
  dummy.updateMatrix();
  benchMesh.setMatrixAt(benchCount, dummy.matrix);
  benchCount++;
  benchMesh.count = benchCount;
  benchMesh.instanceMatrix.needsUpdate = true;
  dummy.rotation.set(0, 0, 0);
}
```

- [ ] **Step 2: Write spawnConnectors()**

```javascript
function spawnConnectors(currentBuilding, rng) {
  if (buildings.length < 2) return;
  const prev = buildings[buildings.length - 2];
  const curr = currentBuilding;

  const prevWorld = gridToWorld(prev.gx, prev.gz);
  const currWorld = gridToWorld(curr.gx, curr.gz);

  // Midpoint between the two buildings
  const mx = (prevWorld.x + currWorld.x) / 2;
  const mz = (prevWorld.z + currWorld.z) / 2;

  const sameRoot = prev.chord.root === curr.chord.root;
  const sameType = prev.chord.type === curr.chord.type;
  const dist = Math.abs(prev.gx - curr.gx) + Math.abs(prev.gz - curr.gz);
  const adjacent = dist <= 2;

  if (sameRoot) {
    // Shared courtyard — trees and bench
    const count = 2 + Math.floor(rng() * 3);
    for (let i = 0; i < count; i++) {
      addTree(
        mx + (rng() - 0.5) * CELL_SIZE * 0.8,
        mz + (rng() - 0.5) * CELL_SIZE * 0.8
      );
    }
    addBench(mx + (rng() - 0.5) * 2, mz + (rng() - 0.5) * 2, rng() * Math.PI);
  } else if (sameType) {
    // Walkway feel — trees in a line
    const steps = Math.max(2, Math.floor(dist));
    for (let i = 0; i < steps; i++) {
      const t = (i + 0.5) / steps;
      addTree(
        prevWorld.x + (currWorld.x - prevWorld.x) * t + (rng() - 0.5) * 2,
        prevWorld.z + (currWorld.z - prevWorld.z) * t + (rng() - 0.5) * 2
      );
    }
  } else {
    // Street elements — cars and lamppost
    const carAngle = Math.atan2(currWorld.z - prevWorld.z, currWorld.x - prevWorld.x);
    addCar(mx + (rng() - 0.5) * 3, mz + (rng() - 0.5) * 3, carAngle + (rng() - 0.5) * 0.3);
    if (rng() < 0.5) {
      addCar(mx + (rng() - 0.5) * 5, mz + (rng() - 0.5) * 5, carAngle + Math.PI + (rng() - 0.5) * 0.3);
    }
  }

  // Adjacent buildings get a bridge
  if (adjacent && prev.velocity > 60 && curr.velocity > 60) {
    const bridgeH = Math.min(
      0.4 + Math.round(3 + (prev.velocity / 127) * 15) * FLOOR_H * 0.4,
      0.4 + Math.round(3 + (curr.velocity / 127) * 15) * FLOOR_H * 0.4
    );
    const dx = currWorld.x - prevWorld.x;
    const dz = currWorld.z - prevWorld.z;
    const length = Math.sqrt(dx * dx + dz * dz);
    const angle = Math.atan2(dz, dx);

    const bridgeGeo = new THREE.BoxGeometry(length, 0.3, 0.8);
    const bridgeMat = solidMat(0x3a3a4a, 0.7, 0.3);
    const bridge = new THREE.Mesh(bridgeGeo, bridgeMat);
    bridge.position.set(mx, bridgeH, mz);
    bridge.rotation.y = angle;
    scene.add(bridge);
  }
}
```

- [ ] **Step 3: Call spawnConnectors() from addBuilding()**

In addBuilding(), after pushing to buildings array, add:

```javascript
spawnConnectors(buildings[buildings.length - 1], rng);
```

- [ ] **Step 4: Update resetCity() to clear connector pools**

Add to resetCity():
```javascript
treeCount = 0; treeMesh.count = 0; trunkMesh.count = 0;
treeMesh.instanceMatrix.needsUpdate = true;
trunkMesh.instanceMatrix.needsUpdate = true;
carCount = 0; carMesh.count = 0;
carMesh.instanceMatrix.needsUpdate = true;
benchCount = 0; benchMesh.count = 0;
benchMesh.instanceMatrix.needsUpdate = true;
// Remove any bridge meshes
scene.children.filter(c => c.isMesh && c.geometry?.parameters?.depth === 0.8).forEach(c => scene.remove(c));
```

- [ ] **Step 5: Verify and commit**

Open in browser. Play multiple chords. Between buildings you should see trees (same root), cars (different), bridges (adjacent). Connectors should appear at midpoints between buildings.

```bash
git add index.html && git commit -m "feat: add inter-building connectors (trees, cars, benches, bridges)"
git push
```

---

### Task 5: Velocity → height direct mapping + cleanup

**Files:**
- Modify: `index.html` (addBuilding, detectChord)

- [ ] **Step 1: Pass velocity through chord detection**

In `detectChord()`, ensure the return includes intervals as raw numbers (not just the joined string). Currently intervals are computed but used only for the key lookup. Keep them in the return:

```javascript
function detectChord(notes) {
  if (notes.length < 3) return null;
  const sorted = [...notes].sort((a, b) => a - b);
  const root = sorted[0] % 12;
  const intervals = sorted.map(n => (n % 12 - root + 12) % 12).sort((a, b) => a - b);
  return { root, name: NOTE_NAMES[root], type: CHORD_MAP[intervals.join(',')] || 'chord', notes: sorted, intervals };
}
```

- [ ] **Step 2: Verify velocity is passed correctly to assembleBuilding**

In `addBuilding()`, confirm the velocity param is the raw MIDI velocity (0-127) passed from `tryDetectChord()`. The keyboard default of 100 should produce ~15 floor buildings. MIDI velocity of 30 should produce ~6 floor buildings.

- [ ] **Step 3: Remove old height calculation**

If any old `h = 12 + (velocity/127)*40 + rng()*12` remains, remove it. Height is now entirely determined inside `assembleBuilding()` via `floorCount`.

- [ ] **Step 4: Verify and commit**

Play with different velocities (if MIDI) or keyboard (fixed 100). Buildings should be clearly taller with harder key presses on MIDI. Keyboard buildings should be consistently ~15 floors.

```bash
git add index.html && git commit -m "feat: velocity directly controls floor count, cleanup old height calc"
git push
```

---

### Task 6: Final polish and cleanup

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Remove dead code**

Remove any remaining references to old generator functions, old GENERATORS map, or unused variables. Remove `colorToHex` if no longer used. Remove old `h, w, d` calculations from addBuilding if they're now in assembleBuilding.

- [ ] **Step 2: Verify complete flow end-to-end**

1. Open in browser
2. Camera is fixed — doesn't move
3. Play C major (A+D+G keys) — clean, boxy building with minimal mutations
4. Play A minor (H+K+D keys) — organic, drifting building with more balconies
5. Play different chords — trees/cars/benches appear between buildings
6. Play adjacent cells — bridges connect tall buildings
7. Esc resets everything cleanly
8. Roads still grow over time

- [ ] **Step 3: Final commit**

```bash
git add index.html && git commit -m "chore: remove dead code, final cleanup"
git push
```
