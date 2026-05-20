# Babylon.js Procedural + Parametric Modeling

How to build high-quality Babylon.js 8 models directly in code, especially when the model is **parametric**, **repeatable**, **animation-ready**, or too structured to be a one-off pile of primitives.

Read this when asked to create a non-trivial Babylon.js model in code: machines, robots, vehicles, warehouse equipment, architectural systems, furniture, procedural buildings, repeated parts, animated mechanisms, or any model that should scale cleanly with options.

This file combines two patterns that should be used together:

- **Procedural modeling**: choosing the right Babylon builders, CSG2, custom `VertexData`, materials, tessellation, edge treatment, and animation hierarchy.
- **Parametric mesh factories**: turning those modeling choices into reusable TypeScript factories with `Options`, `PROPORTIONS`, `Dims`, shared materials, cached templates, thin instances, animation handles, and safe merging.

---

## Table of Contents

- [Quick Reference](#quick-reference)
- [When to Use This Resource](#when-to-use-this-resource)
- [Agent Response Contract](#agent-response-contract)
- [Modeling Workflow](#modeling-workflow)
- [Primitive Builder Catalog](#primitive-builder-catalog)
- [Factory Architecture](#factory-architecture)
- [Options, Defaults, PROPORTIONS, and Dims](#options-defaults-proportions-and-dims)
- [Local Frame Convention](#local-frame-convention)
- [Shared Material Factory](#shared-material-factory)
- [Non-Box Cross-Sections: ExtrudePolygon + earcut](#non-box-cross-sections-extrudepolygon--earcut)
- [CSG2 Boolean Operations](#csg2-boolean-operations)
- [Custom Geometry with VertexData](#custom-geometry-with-vertexdata)
- [Animation-Ready Hierarchy](#animation-ready-hierarchy)
- [Template Caching](#template-caching)
- [Thin-Instance Templates](#thin-instance-templates)
- [Cloning Hierarchical Templates](#cloning-hierarchical-templates)
- [Merging and Consolidation](#merging-and-consolidation)
- [Quality Techniques](#quality-techniques)
- [Worked Example: Composite Factory Skeleton](#worked-example-composite-factory-skeleton)
- [Modeling Checklist](#modeling-checklist)
- [Common Gotchas](#common-gotchas)
- [Related Babylon.js References](#related-babylonjs-references)

---

## Quick Reference

**Core rule:** build models as semantic assemblies, not scattered primitives.

```text
root TransformNode
  ├── static merged geometry
  ├── movableHandleA TransformNode
  │     └── geometry that moves with A
  └── movableHandleB TransformNode
        └── nestedHandleC TransformNode
              └── geometry that composes B + C motion
```

**Use a factory when:**

- Dimensions, proportions, counts, or hole patterns vary.
- The shape repeats dozens to millions of times.
- The model has named moving parts.
- The implementation needs to be inspected, modified, or reused by another agent.

**Factory shape:**

```typescript
export interface PartOptions { index?: number; width?: number; height?: number; depth?: number; }
export interface PartHandles { root: TransformNode; /* animation handles here */ }

export class PartFactoryService {
  private static readonly DEFAULT_WIDTH = 1000;
  private static readonly PROPORTIONS = { /* grouped by sub-part */ } as const;

  public create(options: PartOptions = {}): PartHandles {
    const dims = this.deriveDims(options);
    const root = new TransformNode("partRoot", this.scene);
    // create handles first, then geometry, then merge safely
    return { root };
  }
}
```

**Decision matrix:**

| Need | Use | Avoid |
|---|---|---|
| Parametric model | Factory class with `Options`, `PROPORTIONS`, `Dims` | Magic numbers spread through helpers |
| Static repeated copies | Cached template mesh + thin instances | Per-copy geometry |
| Repeated copies with moving sub-parts | `instantiateHierarchy` clones | Thin instances |
| One-off authored asset | glTF/GLB import | Rebuilding sculpted art in code |
| Cable, pipe, rail, handle | `CreateTube`, `ExtrudeShape`, `ExtrudeShapeCustom` | Many short cylinders |
| Wheel, hub, knob, bottle, column | `CreateLathe` | Stacked cylinders/spheres |
| Custom plate, C/U/T channel, letter, floor plan | `ExtrudePolygon` + `earcut` | Box-only approximations |
| Real holes, windows, slots, boolean cuts | CSG2 | Fake overlap when the cut is visible |
| Formula-driven or unusual mesh | `VertexData` | Fighting built-in builders |
| Animated joint, hinge, wheel, lift | `TransformNode` handle per degree of freedom | Animating merged sub-meshes |

---

## When to Use This Resource

Use this guide for models that must be generated in Babylon.js code and still look intentional.

Prefer this approach when:

- The user asks for a **procedural** or **code-built** model.
- The model needs **parameters**: size, count, proportions, variants, hole patterns.
- The model needs **motion**: wheels, hinges, turrets, robot joints, lift carriages, extending forks, doors.
- The model needs **many repeated copies**: racks, posts, rails, crates, vehicles, fixtures, warehouse elements.
- The model should be **easy for another agent to maintain**.

Prefer **glTF/GLB** when the asset is sculpted, organic, heavily textured, or already authored in Blender/Maya/3ds Max and does not need parametric variation.

Prefer **Solid Particle System** or **Node Geometry** for highly procedural simulation-like geometry where individual authored proportions matter less than particle behavior, formula generation, or large-scale procedural surfaces.

---

## Agent Response Contract

When asked to build a Babylon.js model, answer in this order unless the user asks for a different format:

1. **Model intent** — what the model should communicate visually.
2. **Local frame** — origin, axes, units, and option-to-axis mapping.
3. **Hierarchy tree** — root, handles, mesh leaves, and movement boundaries.
4. **Material palette** — 3–6 shared materials, preferably PBR.
5. **Primitive strategy** — which builders will be used and why.
6. **Factory implementation** — TypeScript code with options, defaults, `PROPORTIONS`, `Dims`, helpers, handles, and safe merging.
7. **Validation checklist** — dimensions, pivots, material reuse, merge boundaries, clone/thin-instance strategy.

Do not return only isolated `CreateBox` calls for a non-trivial model. Even a simple model should have semantic names, shared materials, a documented local frame, and a clean root node.

---

## Modeling Workflow

### 1. Reference first

Before creating geometry, identify:

- Overall bounding box.
- Main sub-assemblies.
- Repeated elements.
- Curved, swept, lathed, or custom-profile parts.
- Which parts move, on which axis, around which pivot.
- Which details are cosmetic and can be hidden, simplified, or added after consolidation.

When dimensions are missing, choose sensible defaults and expose them in `Options`. Do not bury assumptions inside helper methods.

### 2. Sketch the hierarchy before geometry

```text
vehicleRoot
  ├── chassisStaticGeo
  ├── wheelPivot_FL        ← rotates around axle
  │     ├── tireGeo
  │     └── hubGeo
  ├── wheelPivot_FR
  └── liftCarrier          ← translates in Y
        └── forkExtender   ← translates relative to carrier
              └── forkGeo
```

Rules:

- One `TransformNode` per independent degree of freedom.
- Name handles by function: `elbowPivot`, `wheelPivot_FL`, `liftCarrier`, `forkExtender`.
- Geometry lives under the handle that moves it.
- Do not merge across a movement boundary.
- Return typed handles from factories instead of relying on string lookups.

### 3. Choose the right primitive

Use primitives for shape language, not just convenience.

```text
box-like structural mass     → CreateBox, or ExtrudePolygon for chamfered/custom plates
round mechanical parts       → CreateCylinder, CreateSphere, CreateCapsule, CreateTorus
rotational profiles          → CreateLathe
path-following geometry      → CreateTube, ExtrudeShape, ExtrudeShapeCustom
custom flat silhouette       → ExtrudePolygon + earcut
cloth/grid/freeform surface  → CreateRibbon or updatable ground
boolean holes/cuts           → CSG2
fully custom mesh            → VertexData
```

### 4. Build top-down

Create the root and motion handles first; then attach geometry under the correct parent.

Bad:

```typescript
const panel = CreateBox("panel", options, scene);
panel.position = worldPosition;
// Later, trying to retrofit a hinge pivot becomes confusing.
```

Good:

```typescript
const hingePivot = new TransformNode("hingePivot", scene);
hingePivot.setParent(root);
hingePivot.position = hingeWorldPosition;

const panel = CreateBox("doorPanelGeo", options, scene);
panel.setParent(hingePivot);
panel.position = new Vector3(panelWidth / 2, 0, 0); // local offset from hinge
```

### 5. Improve quality before adding more pieces

A simple, well-proportioned model with softened edges, purposeful materials, and correct pivots is better than a noisy model made from many cubes.

Apply at least one quality technique to hero geometry:

- Chamfer or round hard 90° edges.
- Use cylinders, tubes, lathes, or extrusions for mechanical detail.
- Use `ExtrudePolygon` for custom plates instead of unmodified boxes.
- Use edge rendering for crisp technical readability.
- Add decals and small accent details after consolidation.
- Allocate tessellation based on screen importance.

---

## Primitive Builder Catalog

Use deep imports for tree-shaking.

### Box family

```typescript
import { CreateBox } from "@babylonjs/core/Meshes/Builders/boxBuilder";
import { CreateTiledBox } from "@babylonjs/core/Meshes/Builders/tiledBoxBuilder";

CreateBox("box", { width, height, depth, faceUV, faceColors, wrap: true }, scene);
CreateTiledBox("crate", { size: 2, tileSize: 0.5, tileWidth: 0.5 }, scene);
```

Use boxes for structural masses, not for every detail. For hero plates, prefer a chamfered `ExtrudePolygon` over a plain box.

### Sphere, ico sphere, hemisphere

```typescript
import { CreateSphere } from "@babylonjs/core/Meshes/Builders/sphereBuilder";
import { CreateIcoSphere } from "@babylonjs/core/Meshes/Builders/icoSphereBuilder";
import { CreateHemisphere } from "@babylonjs/core/Meshes/Builders/hemisphereBuilder";

CreateSphere("sphere", { diameter: 2, segments: 32, diameterX, diameterY, diameterZ }, scene);
CreateIcoSphere("ico", { radius: 1, subdivisions: 4, flat: false }, scene);
CreateHemisphere("dome", { diameter: 2, segments: 24 }, scene);
```

Use `IcoSphere` for more uniform triangles, deformation, organic surfaces, and faceted forms. Use UV spheres when longitude/latitude UVs matter.

### Cylinder, cone, capsule

```typescript
import { CreateCylinder } from "@babylonjs/core/Meshes/Builders/cylinderBuilder";
import { CreateCapsule } from "@babylonjs/core/Meshes/Builders/capsuleBuilder";
import { Mesh } from "@babylonjs/core/Meshes/mesh";

CreateCylinder("cylinder", {
  height: 3,
  diameterTop: 1,      // 0 for cone
  diameterBottom: 2,
  tessellation: 32,
  subdivisions: 1,
  cap: Mesh.CAP_ALL,
}, scene);

CreateCapsule("capsule", { height: 2, radius: 0.5, tessellation: 24, capSubdivisions: 8 }, scene);
```

Use capsules for rounded rods, dampers, handles, bumpers, and anything that would otherwise be a cylinder plus two hemispheres.

### Torus, disc, ground, tiled planes

```typescript
import { CreateTorus } from "@babylonjs/core/Meshes/Builders/torusBuilder";
import { CreateTorusKnot } from "@babylonjs/core/Meshes/Builders/torusKnotBuilder";
import { CreateDisc } from "@babylonjs/core/Meshes/Builders/discBuilder";
import { CreateTiledGround } from "@babylonjs/core/Meshes/Builders/groundBuilder";
import { CreateTiledPlane } from "@babylonjs/core/Meshes/Builders/tiledPlaneBuilder";

CreateTorus("tire", { diameter: 2, thickness: 0.5, tessellation: 48 }, scene);
CreateTorusKnot("knot", { radius: 1, tube: 0.3, radialSegments: 128, tubularSegments: 64, p: 2, q: 3 }, scene);
CreateDisc("disc", { radius: 1, tessellation: 48 }, scene);
CreateTiledGround("floor", { xmin: -10, zmin: -10, xmax: 10, zmax: 10, subdivisions: { w: 8, h: 8 } }, scene);
CreateTiledPlane("panel", { size: 4, tileSize: 1 }, scene);
```

Use tiled variants when texture repetition should be controlled by geometry/UV generation rather than manual UV scaling.

### Polyhedra and geodesics

```typescript
import { CreatePolyhedron } from "@babylonjs/core/Meshes/Builders/polyhedronBuilder";
import { CreateGeodesic } from "@babylonjs/core/Meshes/Builders/geodesicBuilder";
import { CreateGoldberg } from "@babylonjs/core/Meshes/Builders/goldbergBuilder";

CreatePolyhedron("poly", { type: 2, size: 1, flat: true }, scene);
CreateGeodesic("geo", { m: 1, n: 1, size: 1 }, scene);
CreateGoldberg("gold", { m: 2, n: 0, size: 1 }, scene);
```

Use these for stylized rocks, crystals, low-poly objects, domes, honeycomb surfaces, and sci-fi panels.

### Paths, sweeps, lathes, ribbons

```typescript
import { CreateTube } from "@babylonjs/core/Meshes/Builders/tubeBuilder";
import { CreateLathe } from "@babylonjs/core/Meshes/Builders/latheBuilder";
import { CreateRibbon } from "@babylonjs/core/Meshes/Builders/ribbonBuilder";
import { ExtrudeShape, ExtrudeShapeCustom } from "@babylonjs/core/Meshes/Builders/shapeBuilder";

CreateTube("cable", { path: points, radius: 0.05, tessellation: 16, cap: Mesh.CAP_ALL }, scene);

const profile = [
  new Vector3(0.0, 0.0, 0),
  new Vector3(0.6, 0.0, 0),
  new Vector3(0.5, 0.3, 0),
  new Vector3(0.5, 1.0, 0),
  new Vector3(0.3, 1.1, 0),
  new Vector3(0.0, 1.1, 0),
];
CreateLathe("vaseOrHub", { shape: profile, tessellation: 48, closed: true }, scene);

ExtrudeShape("rail", { shape: crossSection, path, cap: Mesh.CAP_ALL }, scene);
ExtrudeShapeCustom("twistedRail", {
  shape: crossSection,
  path,
  scaleFunction: (i, distance) => 1 + 0.2 * Math.sin(distance),
  rotationFunction: (i, distance) => distance * 0.1,
}, scene);

CreateRibbon("surface", { pathArray: gridOfPaths, sideOrientation: Mesh.DOUBLESIDE }, scene);
```

`Lathe`, `Tube`, and `ExtrudeShapeCustom` are the workhorses for bolts, glasses, antennae, handles, pipes, cables, wheels, rails, and columns. Prefer them over stacking many primitives.

### Polygons, extruded polygons, text, and lines

```typescript
import { CreatePolygon, ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import { CreateText } from "@babylonjs/core/Meshes/Builders/textBuilder";
import { CreateGreasedLine } from "@babylonjs/core/Meshes/Builders/greasedLineBuilder";
import earcut from "earcut";

const shape = [
  new Vector3(-1, 0, -1),
  new Vector3( 1, 0, -1),
  new Vector3( 1, 0,  1),
  new Vector3(-1, 0,  1),
];
const holes = [[
  new Vector3(-0.3, 0, -0.3),
  new Vector3( 0.3, 0, -0.3),
  new Vector3( 0.3, 0,  0.3),
  new Vector3(-0.3, 0,  0.3),
]];

CreatePolygon("floor", { shape, holes }, scene, earcut);
ExtrudePolygon("wall", { shape, holes, depth: 3 }, scene, earcut);
CreateText("label", "A1", fontData, { size: 1, depth: 0.1 }, scene, earcut);
CreateGreasedLine("wire", { points: [[v1, v2, v3]] }, { width: 0.05, color: new Color3(1, 0.5, 0) }, scene);
```

Use `GreasedLine` instead of `CreateLines` when visual width must stay readable at distance.

---

## Factory Architecture

A factory is a small, mostly stateless service that turns options into a Babylon hierarchy.

```typescript
import type { Scene } from "@babylonjs/core/scene";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";

export interface MachineOptions {
  index?: number;
  width?: number;   // local X
  height?: number;  // local Y
  depth?: number;   // local Z
}

export interface MachineHandles {
  root: TransformNode;
  liftCarrier?: TransformNode;
  forkExtender?: TransformNode;
}

/**
 * Builds one parametric machine assembly.
 *
 * Local frame: Y is vertical. Origin is centered on the bottom footprint.
 * `width` maps to local X, `height` maps to Y, and `depth` maps to local Z.
 */
export class MachineFactoryService {
  private static readonly DEFAULT_WIDTH = 1000;
  private static readonly DEFAULT_HEIGHT = 1200;
  private static readonly DEFAULT_DEPTH = 800;

  private static readonly PROPORTIONS = {
    body: { heightFrac: 0.75, insetX: 40, insetZ: 40 },
    deck: { heightFrac: 0.05, chamfer: 12 },
  } as const;

  constructor(
    private readonly scene: Scene,
    private readonly systemId: string,
    private readonly materialFactory: BabylonMaterialFactory,
  ) {}

  public create(options: MachineOptions = {}): MachineHandles {
    const index = options.index ?? 0;
    const dims = this.deriveDims(options);
    const root = new TransformNode(`machine_${index}.${this.systemId}`, this.scene);

    this.createBody(root, dims);
    this.createDeck(root, dims);

    this.consolidateByMaterial(root);
    return { root };
  }

  private deriveDims(options: MachineOptions): MachineDims {
    const sizeX = options.width ?? MachineFactoryService.DEFAULT_WIDTH;
    const sizeY = options.height ?? MachineFactoryService.DEFAULT_HEIGHT;
    const sizeZ = options.depth ?? MachineFactoryService.DEFAULT_DEPTH;
    const P = MachineFactoryService.PROPORTIONS;
    const bodyHeight = sizeY * P.body.heightFrac;
    const deckHeight = sizeY * P.deck.heightFrac;

    return { sizeX, sizeY, sizeZ, bodyHeight, bodyTop: bodyHeight, deckHeight };
  }

  private createBody(parent: TransformNode, dims: MachineDims): void { /* ... */ }
  private createDeck(parent: TransformNode, dims: MachineDims): void { /* ... */ }
  private consolidateByMaterial(root: TransformNode, skip: TransformNode[] = []): void { /* ... */ }
}

interface MachineDims {
  sizeX: number;
  sizeY: number;
  sizeZ: number;
  bodyHeight: number;
  bodyTop: number;
  deckHeight: number;
}
```

Factory invariants:

- `create()` returns a root or typed handles object.
- Helpers take `Dims`; helpers do not reread raw options.
- Shared material references come from a material factory.
- Motion boundaries are represented by named `TransformNode`s.
- Merging happens after parenting and never across animation handles.
- Repeated fully-static geometry uses cached templates and thin instances.

---

## Options, Defaults, PROPORTIONS, and Dims

### Options + defaults

Make all options optional unless the model cannot function without one.

```typescript
export interface ChassisOptions {
  /** Stable index for generated node names. */
  index?: number;
  /** Long body dimension in scene millimetres. Maps to local Z. */
  width?: number;
  /** Body height in scene millimetres. Maps to local Y. */
  height?: number;
  /** Short body dimension in scene millimetres. Maps to local X. */
  length?: number;
}
```

Use `??` defaults inside `deriveDims`, not throughout helper methods.

### PROPORTIONS table

The most common parametric modeling failure is leaking magic numbers everywhere. Use one frozen table.

```typescript
private static readonly PROPORTIONS = {
  body: {
    baselineYFrac: 0.25,  // bodyBaselineY = sizeY * 0.25
    heightFrac: 0.60,     // chosen so body + deck + bands fit total height
    insetX: 25,           // bodyWidth = sizeX - 25
    insetZ: 25,           // bodyDepth = sizeZ - 25
  },
  deck: {
    heightFrac: 0.08,
    insetX: 70,
    insetZ: 140,
    chamfer: 10,
  },
  cornerPost: {
    diameter: 80,
    height: 50,
    tessellation: 28,
  },
} as const;
```

Conventions:

- Fractions that multiply a body dimension use a `Frac` suffix.
- Raw numbers are absolute scene units.
- Group values by part.
- Comment the reason for non-obvious values next to the value.

### Dims

`Dims` is the precomputed geometry contract for helpers.

```typescript
interface Dims {
  sizeX: number;
  sizeY: number;
  sizeZ: number;
  bodyBaselineY: number;
  bodyHeight: number;
  bodyTop: number;
  deckHeight: number;
}

private deriveDims(options: ChassisOptions): Dims {
  const sizeX = options.length ?? ChassisFactoryService.DEFAULT_LENGTH;
  const sizeY = options.height ?? ChassisFactoryService.DEFAULT_HEIGHT;
  const sizeZ = options.width  ?? ChassisFactoryService.DEFAULT_WIDTH;
  const P = ChassisFactoryService.PROPORTIONS;

  const bodyBaselineY = sizeY * P.body.baselineYFrac;
  const bodyHeight = sizeY * P.body.heightFrac;

  return {
    sizeX,
    sizeY,
    sizeZ,
    bodyBaselineY,
    bodyHeight,
    bodyTop: bodyBaselineY + bodyHeight,
    deckHeight: sizeY * P.deck.heightFrac,
  };
}
```

With `Dims`, helpers cannot silently disagree about where one sub-part ends and the next begins.

---

## Local Frame Convention

Document the local frame near the top of the factory file or class JSDoc.

```typescript
/**
 * Builds a parametric chassis from a body shell, chamfered top deck, and posts.
 *
 * Local frame: Y is vertical with the bottom face at Y = 0.
 * Origin is centered on the bottom footprint.
 * User-facing `width` maps to local Z; `length` maps to local X.
 */
export class ChassisFactoryService { /* ... */ }
```

State:

- Origin location.
- Up axis.
- Unit convention if relevant, for example millimetres.
- Which user-facing option maps to which local axis.
- Whether the object is centered or anchored at a corner/base.

---

## Shared Material Factory

Use a shared material factory so identical logical materials are the same Babylon material instance.

```typescript
import type { Scene } from "@babylonjs/core/scene";
import { Color3 } from "@babylonjs/core/Maths/math.color";
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import { Texture } from "@babylonjs/core/Materials/Textures/texture";
import type { Material } from "@babylonjs/core/Materials/material";

export type MaterialKey = "aluminium" | "steel" | "rubber" | "plastic" | "accent" | "glass";

interface PbrSpec {
  color?: string;
  textureUrl?: string;
  metallic: number;
  roughness: number;
  alpha?: number;
}

const PALETTE: Record<MaterialKey, PbrSpec> = {
  aluminium: { color: "#DFE4F4", metallic: 0.95, roughness: 0.40 },
  steel:     { color: "#6E7378", metallic: 0.75, roughness: 0.45 },
  rubber:    { color: "#111112", metallic: 0.00, roughness: 0.85 },
  plastic:   { color: "#1E88E5", metallic: 0.00, roughness: 0.50 },
  accent:    { color: "#F9AB01", metallic: 0.10, roughness: 0.55 },
  glass:     { color: "#CDEBFF", metallic: 0.00, roughness: 0.08, alpha: 0.35 },
};

export class BabylonMaterialFactory {
  private readonly materials = new Map<MaterialKey, PBRMaterial>();

  constructor(private readonly scene: Scene) {}

  public getMaterial(key: MaterialKey): Material {
    const cached = this.materials.get(key);
    if (cached) return cached;

    const name = `babylon-shared-${key}`;
    const existing = this.scene.getMaterialByName(name);
    if (existing instanceof PBRMaterial) {
      this.materials.set(key, existing);
      return existing;
    }

    const spec = PALETTE[key];
    const material = new PBRMaterial(name, this.scene);
    if (spec.textureUrl) {
      material.albedoTexture = new Texture(spec.textureUrl, this.scene);
    } else if (spec.color) {
      material.albedoColor = Color3.FromHexString(spec.color);
    }
    material.metallic = spec.metallic;
    material.roughness = spec.roughness;
    if (spec.alpha !== undefined) {
      material.alpha = spec.alpha;
      material.transparencyMode = PBRMaterial.PBRMATERIAL_ALPHABLEND;
    }

    this.materials.set(key, material);
    return material;
  }
}
```

Why this matters:

- Babylon groups draw calls by material identity, not just identical material settings.
- `consolidateByMaterial` only works if mesh materials are the exact same object.
- A controlled palette makes the model look designed rather than random.

---

## Non-Box Cross-Sections: ExtrudePolygon + earcut

Use `ExtrudePolygon` for lipped channels, chamfered plates, gear-like outlines, signs, T-slots, U-channels, floor plans, and non-rectangular structural shapes.

```typescript
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import { ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import type { Mesh } from "@babylonjs/core/Meshes/mesh";
import type { Scene } from "@babylonjs/core/scene";
import earcut from "earcut";

function createChamferedPlate(
  name: string,
  width: number,
  depth: number,
  thickness: number,
  chamfer: number,
  scene: Scene,
): Mesh {
  const halfX = width / 2;
  const halfZ = depth / 2;
  const c = Math.max(0, Math.min(chamfer, Math.min(halfX, halfZ) * 0.49));

  const shape = [
    new Vector3(-halfX + c, 0, -halfZ),
    new Vector3( halfX - c, 0, -halfZ),
    new Vector3( halfX,     0, -halfZ + c),
    new Vector3( halfX,     0,  halfZ - c),
    new Vector3( halfX - c, 0,  halfZ),
    new Vector3(-halfX + c, 0,  halfZ),
    new Vector3(-halfX,     0,  halfZ - c),
    new Vector3(-halfX,     0, -halfZ + c),
  ];

  const mesh = ExtrudePolygon(name, { shape, depth: thickness }, scene, earcut);

  // ExtrudePolygon geometry is easier to reuse when its bottom face is anchored at Y = 0.
  mesh.position.y = thickness;
  mesh.bakeCurrentTransformIntoVertices();
  return mesh;
}
```

Gotchas:

- `earcut` is required and must be passed to polygon builders.
- Keep winding consistent. Use counter-clockwise outer rings and clockwise holes.
- If the extruded geometry appears inverted or offset, anchor it by lifting and baking.
- There is no tessellation parameter; add more vertices to approximate curves.

### Lipped C-channel example

```typescript
function createLippedCChannelShape(width: number, depth: number, thickness: number, lipLength: number): Vector3[] {
  const halfW = width / 2;
  const halfD = depth / 2;
  const lip = Math.max(thickness + 0.01, Math.min(lipLength, halfW - thickness));

  const innerLeft = -halfW + thickness;
  const innerRight = halfW - thickness;
  const innerBack = -halfD + thickness;
  const innerFront = halfD - thickness;
  const leftLipInnerEdge = -halfW + lip;
  const rightLipInnerEdge = halfW - lip;

  return [
    new Vector3( halfW,             0, -halfD),
    new Vector3( halfW,             0,  halfD),
    new Vector3( rightLipInnerEdge, 0,  halfD),
    new Vector3( rightLipInnerEdge, 0,  innerFront),
    new Vector3( innerRight,        0,  innerFront),
    new Vector3( innerRight,        0,  innerBack),
    new Vector3( innerLeft,         0,  innerBack),
    new Vector3( innerLeft,         0,  innerFront),
    new Vector3( leftLipInnerEdge,  0,  innerFront),
    new Vector3( leftLipInnerEdge,  0,  halfD),
    new Vector3(-halfW,             0,  halfD),
    new Vector3(-halfW,             0, -halfD),
  ];
}
```

---

## CSG2 Boolean Operations

Use CSG2 when the cut itself is visible: holes, slots, windows, negative volumes, chamfered intersections, and engineering-like cutouts.

```typescript
import { CSG2, InitializeCSG2Async } from "@babylonjs/core/Meshes/csg2";
import { CreateBox } from "@babylonjs/core/Meshes/Builders/boxBuilder";
import { CreateCylinder } from "@babylonjs/core/Meshes/Builders/cylinderBuilder";

await InitializeCSG2Async(); // once per app/session

const plate = CreateBox("plateSource", { width: 4, height: 0.2, depth: 2 }, scene);
const cutter = CreateCylinder("holeCutter", { diameter: 0.5, height: 3, tessellation: 24 }, scene);
cutter.rotation.x = Math.PI / 2;

const a = CSG2.FromMesh(plate);
const b = CSG2.FromMesh(cutter);
const result = a.subtract(b);

const drilled = result.toMesh("plateWithHole", scene, { centerMesh: false, rebuildNormals: true });
plate.dispose();
cutter.dispose();
a.dispose();
b.dispose();
result.dispose();
```

Rules:

- Initialize CSG2 once before use.
- CSG operates in world space; position and rotate operands before `CSG2.FromMesh`.
- Keep operand tessellation modest.
- Dispose source meshes and intermediate CSG2 objects.
- Prefer composition over CSG when overlapping internal geometry is not visible.

---

## Custom Geometry with VertexData

Use `VertexData` when no builder fits: formula surfaces, custom UVs, unusual symmetries, voxel surfaces, procedural terrain, or highly controlled topology.

```typescript
import { VertexData } from "@babylonjs/core/Meshes/mesh.vertexData";
import { Mesh } from "@babylonjs/core/Meshes/mesh";

const mesh = new Mesh("customPyramid", scene);
const data = new VertexData();

data.positions = [
   0, 1, 0,
  -1, 0, 1,
   1, 0, 1,
   1, 0,-1,
  -1, 0,-1,
];
data.indices = [
  0, 1, 2,
  0, 2, 3,
  0, 3, 4,
  0, 4, 1,
  1, 4, 3,
  1, 3, 2,
];
data.uvs = [0.5, 1, 0, 0, 1, 0, 1, 0, 0, 0];
VertexData.ComputeNormals(data.positions, data.indices, data.normals = []);
data.applyToMesh(mesh, false);
```

Patterns:

- Duplicate vertices per face for flat shading.
- Share vertices for smooth shading.
- Use `data.colors` for per-vertex color.
- Pass `updatable: true` only when you need to mutate geometry later.
- For heightmaps and formula surfaces, starting from `CreateGround` or `CreateRibbon` with updatable vertices is often simpler than writing indices manually.

---

## Animation-Ready Hierarchy

A model that will animate must be built with the hierarchy before geometry is finalized. Retrofitting pivots after merging is painful.

### Pivot-group pattern

Each joint, hinge, wheel, door, turret, lift carriage, or gripper gets a `TransformNode` located at the motion origin. Geometry is parented under it with local offsets.

```typescript
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";
import { Vector3 } from "@babylonjs/core/Maths/math.vector";

const robotRoot = new TransformNode("robotRoot", scene);

const shoulderPivot = new TransformNode("shoulderPivot", scene);
shoulderPivot.parent = robotRoot;
shoulderPivot.position = new Vector3(0, 0.4, 0);

const upperArm = CreateCapsule("upperArmGeo", { height: 1.2, radius: 0.16, tessellation: 24 }, scene);
upperArm.parent = shoulderPivot;
upperArm.position.y = 0.6; // local offset; geometry extends out from the joint

// Animation later:
shoulderPivot.rotation.z = Math.PI / 6;
```

Rules:

- One pivot `TransformNode` per independent degree of freedom.
- Use nested pivots for multi-axis joints.
- Geometry local position describes how far the part sticks out from the joint.
- Pivot names are an API for animation code.
- Keep one root above the topmost moving part so the whole assembly can be moved/scaled.

### Stacked handles

```text
root
  └── liftCarrier        (translates Y)
        └── forkExtender (translates X/Z relative to carrier)
              └── forkGeo
```

```typescript
export interface LiftHandles {
  root: TransformNode;
  liftCarrier: TransformNode;
  forkExtender: TransformNode;
}

const liftCarrier = new TransformNode("liftCarrier", scene);
liftCarrier.setParent(root);

const forkExtender = new TransformNode("forkExtender", scene);
forkExtender.setParent(liftCarrier);

// Motions compose automatically.
liftCarrier.position.y = liftY;
forkExtender.position.x = extensionX;
```

### Naming conventions

- `<part>Root` for a complete assembly: `robotRoot`, `craneRoot`.
- `<joint>Pivot` for transform-only pivot nodes: `elbowPivot`, `axlePivot`.
- `<part>Geo` for visible mesh leaves: `wheelGeo`, `forearmGeo`.
- `<part>Cosmetic` for decorative child groups.
- `_L`, `_R`, `_FL`, `_FR`, `_BL`, `_BR` suffixes for mirrored or repeated sides.

---

## Template Caching

If identical numeric inputs create identical geometry, cache the source template mesh.

```typescript
import type { Mesh } from "@babylonjs/core/Meshes/mesh";

type TemplateSlot = "pole" | "rail" | "plate";

interface CachedTemplate {
  mesh: Mesh;
  cacheKey: string;
}

private readonly templates = new Map<TemplateSlot, CachedTemplate>();

private ensureTemplate(slot: TemplateSlot, cacheKey: string, build: () => Mesh): Mesh {
  const cached = this.templates.get(slot);
  if (cached && cached.cacheKey === cacheKey) return cached.mesh;

  cached?.mesh.dispose();
  const mesh = build();
  this.templates.set(slot, { mesh, cacheKey });
  return mesh;
}
```

The cache key must include every geometry-affecting parameter.

```typescript
private ensurePole(options: PoleOptions): Mesh {
  const width = options.width ?? DEFAULT_WIDTH;
  const depth = options.depth ?? DEFAULT_DEPTH;
  const thickness = options.thickness ?? DEFAULT_THICKNESS;
  const lipLength = options.lipLength ?? DEFAULT_LIP_LENGTH;
  const cacheKey = `${width}|${depth}|${thickness}|${lipLength}`;

  return this.ensureTemplate("pole", cacheKey, () =>
    this.buildPole({ width, depth, thickness, lipLength }),
  );
}
```

---

## Thin-Instance Templates

Use thin instances for fully-static repeated templates where all copies share one source geometry.

Conventions:

1. Build the template along its main axis at `UNIT_TEMPLATE_LENGTH = 1000`.
2. Keep the template disabled: `mesh.setEnabled(false)`.
3. Scale each instance by `realLength / 1000` on the main axis.
4. Do not use thin instances for templates with movable sub-parts.

```typescript
import "@babylonjs/core/Meshes/thinInstanceMesh";
import { Matrix } from "@babylonjs/core/Maths/math.vector";
import type { Mesh } from "@babylonjs/core/Meshes/mesh";
import type { Material } from "@babylonjs/core/Materials/material";

private static readonly UNIT_TEMPLATE_LENGTH = 1000;

private finalizeTemplate(mesh: Mesh, material: Material): Mesh {
  mesh.material = material;
  mesh.isPickable = false;
  mesh.receiveShadows = true;
  mesh.setEnabled(false);
  return mesh;
}

function applyPoleInstances(template: Mesh, poles: Array<{ x: number; z: number; heightMm: number }>): void {
  const matrices = new Float32Array(16 * poles.length);

  for (let i = 0; i < poles.length; i++) {
    const scaleY = poles[i].heightMm / 1000;
    Matrix.Scaling(1, scaleY, 1)
      .multiply(Matrix.Translation(poles[i].x, 0, poles[i].z))
      .copyToArray(matrices, i * 16);
  }

  template.thinInstanceSetBuffer("matrix", matrices, 16, false);
  template.thinInstanceBufferUpdated("matrix");
}
```

Thin-instance limitations:

- One material for all instances of the source mesh.
- No per-instance child hierarchy.
- All-or-nothing visibility unless you manage buffers yourself.
- One source bounding strategy; individual picking/control is limited.

---

## Cloning Hierarchical Templates

Use `instantiateHierarchy` or cloning for repeated assemblies with independent moving parts.

```typescript
const template = liftFactory.create({ index: 0, height: 3200, marginBottom: 200 });
const cloneRoot = template.root.instantiateHierarchy(null) as TransformNode | null;

const cloneCarrier = cloneRoot?.getChildTransformNodes(true)
  .find((node) => node.name.startsWith("liftCarrier"));

if (cloneCarrier) {
  cloneCarrier.position.y = 750;
}
```

Guidance:

- Use thin instances for static repeated geometry.
- Use hierarchy clones for repeated animated assemblies.
- Return handles from `create()` for source assemblies.
- For cloned assemblies, either build a handle-rebinding helper or use stable names.

Mirroring caveat: negative scaling flips triangle winding. For mirrored hierarchies, use double-sided materials, rebake/fix winding, or build mirrored geometry explicitly.

---

## Merging and Consolidation

Merge only static geometry and only after the hierarchy is correct.

### `mergeIntoStatic`

Use inside helpers when the helper knows the listed meshes will never animate independently.

```typescript
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";
import type { Material } from "@babylonjs/core/Materials/material";

function mergeIntoStatic(parent: TransformNode, name: string, meshes: Mesh[], material: Material): Mesh | null {
  if (meshes.length === 0) return null;
  if (meshes.length === 1) {
    const only = meshes[0];
    only.name = name;
    only.material = material;
    only.setParent(parent);
    return only;
  }

  const merged = Mesh.MergeMeshes(meshes, true, true);
  if (!merged) return null;
  merged.name = name;
  merged.setParent(parent);
  merged.material = material;
  merged.isPickable = false;
  merged.receiveShadows = true;
  return merged;
}
```

### `consolidateByMaterial`

Use at the end of `create()` to collapse incidental static meshes by material. Pass animation handles in `skip` so the consolidator does not pull children across a movement boundary.

```typescript
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import { StandardMaterial } from "@babylonjs/core/Materials/standardMaterial";
import type { Material } from "@babylonjs/core/Materials/material";
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";

function consolidateByMaterial(root: TransformNode, skip: TransformNode[] = []): void {
  const byMaterial = new Map<Material, Mesh[]>();
  const skipSet = new Set<TransformNode>(skip);

  const isMergeable = (material: Material): boolean => {
    if (material instanceof StandardMaterial) return false; // common for decals/labels
    if (material instanceof PBRMaterial && material.transparencyMode === PBRMaterial.PBRMATERIAL_ALPHABLEND) return false;
    return true;
  };

  const visit = (node: TransformNode): void => {
    for (const child of node.getChildren()) {
      if (child instanceof TransformNode && skipSet.has(child)) continue;

      if (child instanceof Mesh && child.material && isMergeable(child.material)) {
        const list = byMaterial.get(child.material) ?? [];
        list.push(child);
        byMaterial.set(child.material, list);
        continue;
      }

      if (child instanceof TransformNode) {
        visit(child);
      }
    }
  };

  visit(root);

  for (const [material, meshes] of byMaterial) {
    if (meshes.length < 2) continue;
    const merged = Mesh.MergeMeshes(meshes, true, true);
    if (!merged) continue;
    merged.name = `${root.name}__${material.name}`;
    merged.setParent(root);
    merged.material = material;
    merged.isPickable = false;
    merged.receiveShadows = true;
  }
}
```

### Two-layer merging pattern

```typescript
public create(options: LiftOptions): LiftHandles {
  const root = new TransformNode("liftRoot", this.scene);

  this.createFoundation(root, options);      // may use mergeIntoStatic internally
  this.createMast(root, options);            // static
  const carrier = this.createCarrier(root, options); // animation handle

  consolidateByMaterial(root, [carrier]);    // static root only
  consolidateByMaterial(carrier);            // carrier's internal static geometry

  this.createDecals(root, options);          // after consolidation
  return { root, liftCarrier: carrier };
}
```

---

## Quality Techniques

### Tessellation budgets

- Large hero spheres: `segments: 32` minimum; `48–64` for close-ups.
- Cylinders/tubes in silhouette: `tessellation: 32` minimum for hero use.
- Curved smooth surfaces should not span more than roughly 30° per polygon.
- Tube, extrude, and lathe paths should be sampled densely enough that turns do not visibly kink.
- Use LOD for distance.

```typescript
mesh.addLODLevel(50, lowPolyMesh);
mesh.addLODLevel(200, null); // cull beyond 200 units
```

### Rounded and chamfered edges

Sharp 90° boxes are the strongest “made by code” tell.

Use one of these:

- Chamfered `ExtrudePolygon` plates.
- Capsules instead of cylinders where rounded ends make sense.
- Tubes and lathes for rods, handles, wheels, and hubs.
- CSG2 for visible bevel/cut geometry.
- Composed rounded boxes: smaller box + spheres/cylinders at corners/edges, then merge.
- Edge rendering for a crisp technical/drafted look.

```typescript
mesh.enableEdgesRendering(0.95, true);
mesh.edgesWidth = 1.0;
```

### Materials

A polished model usually has 3–6 shared materials, not one material per mesh. Good default palette:

```text
aluminium: light metal surfaces
steel: darker structural metal
rubber: tires, bumpers, belts
plastic: guards, covers, housings
accent: warning details, handles, active parts
glass: covers, lights, sensors
```

### Bake static, keep dynamic separate

Use `bakeCurrentTransformIntoVertices()` only for geometry whose pivot no longer matters. Do not bake animated parts after setting up pivots.

### Details and decals

Add small labels, decals, transparent plates, and UI-like indicators after consolidation so they are not accidentally merged into opaque static geometry.

---

## Worked Example: Composite Factory Skeleton

This skeleton demonstrates the preferred structure without overfitting to a specific domain.

```typescript
import type { Scene } from "@babylonjs/core/scene";
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { CreateBox } from "@babylonjs/core/Meshes/Builders/boxBuilder";
import { CreateCylinder } from "@babylonjs/core/Meshes/Builders/cylinderBuilder";
import { ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import type { MaterialKey, BabylonMaterialFactory } from "./BabylonMaterialFactory";
import earcut from "earcut";

export interface ParametricCarrierOptions {
  index?: number;
  width?: number;       // local X
  height?: number;      // local Y
  depth?: number;       // local Z
  hasFork?: boolean;
}

export interface ParametricCarrierHandles {
  root: TransformNode;
  liftCarrier: TransformNode;
  forkExtender?: TransformNode;
}

interface CarrierDims {
  sizeX: number;
  sizeY: number;
  sizeZ: number;
  bodyHeight: number;
  deckHeight: number;
  bodyTop: number;
  forkLength: number;
}

/**
 * Parametric carrier assembly.
 *
 * Local frame: Y is up. Origin is centered on the bottom footprint.
 * `width` maps to X, `height` maps to Y, `depth` maps to Z.
 * The lift carrier translates in local Y. The optional fork extender translates in local X.
 */
export class ParametricCarrierFactoryService {
  private static readonly DEFAULT_WIDTH = 1200;
  private static readonly DEFAULT_HEIGHT = 1800;
  private static readonly DEFAULT_DEPTH = 900;

  private static readonly PROPORTIONS = {
    body: { heightFrac: 0.55, insetX: 80, insetZ: 60 },
    deck: { heightFrac: 0.06, insetX: 40, insetZ: 40, chamfer: 18 },
    mast: { width: 120, depth: 120, count: 2 },
    carrier: { widthFrac: 0.65, height: 180, depthFrac: 0.80, startYFrac: 0.35 },
    fork: { lengthFrac: 0.65, width: 70, height: 35, gapFrac: 0.35 },
  } as const;

  constructor(
    private readonly scene: Scene,
    private readonly systemId: string,
    private readonly materialFactory: BabylonMaterialFactory,
  ) {}

  public create(options: ParametricCarrierOptions = {}): ParametricCarrierHandles {
    const index = options.index ?? 0;
    const dims = this.deriveDims(options);
    const root = new TransformNode(`carrier_${index}.${this.systemId}`, this.scene);

    this.createBody(root, dims);
    this.createDeck(root, dims);
    this.createMasts(root, dims);
    const liftCarrier = this.createLiftCarrier(root, dims, options.hasFork ?? true);

    consolidateByMaterial(root, [liftCarrier]);
    consolidateByMaterial(liftCarrier);

    return {
      root,
      liftCarrier,
      forkExtender: liftCarrier.getChildTransformNodes(true).find((n) => n.name.startsWith("forkExtender")),
    };
  }

  private deriveDims(options: ParametricCarrierOptions): CarrierDims {
    const sizeX = options.width ?? ParametricCarrierFactoryService.DEFAULT_WIDTH;
    const sizeY = options.height ?? ParametricCarrierFactoryService.DEFAULT_HEIGHT;
    const sizeZ = options.depth ?? ParametricCarrierFactoryService.DEFAULT_DEPTH;
    const P = ParametricCarrierFactoryService.PROPORTIONS;
    const bodyHeight = sizeY * P.body.heightFrac;
    const deckHeight = sizeY * P.deck.heightFrac;

    return {
      sizeX,
      sizeY,
      sizeZ,
      bodyHeight,
      deckHeight,
      bodyTop: bodyHeight,
      forkLength: sizeX * P.fork.lengthFrac,
    };
  }

  private createBody(parent: TransformNode, dims: CarrierDims): void {
    const P = ParametricCarrierFactoryService.PROPORTIONS.body;
    const body = CreateBox("bodyGeo", {
      width: dims.sizeX - P.insetX,
      height: dims.bodyHeight,
      depth: dims.sizeZ - P.insetZ,
    }, this.scene);
    body.setParent(parent);
    body.position.y = dims.bodyHeight / 2;
    body.material = this.materialFactory.getMaterial("aluminium");
    body.isPickable = false;
    body.receiveShadows = true;
  }

  private createDeck(parent: TransformNode, dims: CarrierDims): void {
    const P = ParametricCarrierFactoryService.PROPORTIONS.deck;
    const deck = createChamferedPlate(
      "deckGeo",
      dims.sizeX - P.insetX,
      dims.sizeZ - P.insetZ,
      dims.deckHeight,
      P.chamfer,
      this.scene,
    );
    deck.setParent(parent);
    deck.position.y = dims.bodyTop;
    deck.material = this.materialFactory.getMaterial("steel");
    deck.isPickable = false;
    deck.receiveShadows = true;
  }

  private createMasts(parent: TransformNode, dims: CarrierDims): void {
    const P = ParametricCarrierFactoryService.PROPORTIONS.mast;
    const x = dims.sizeX / 2 - P.width / 2;
    const z = dims.sizeZ / 2 - P.depth / 2;

    for (const zSign of [-1, 1] as const) {
      const mast = CreateBox("mastGeo", { width: P.width, height: dims.sizeY, depth: P.depth }, this.scene);
      mast.setParent(parent);
      mast.position = new Vector3(x, dims.sizeY / 2, zSign * z);
      mast.material = this.materialFactory.getMaterial("steel");
      mast.isPickable = false;
      mast.receiveShadows = true;
    }
  }

  private createLiftCarrier(parent: TransformNode, dims: CarrierDims, hasFork: boolean): TransformNode {
    const P = ParametricCarrierFactoryService.PROPORTIONS.carrier;
    const carrier = new TransformNode(`liftCarrier.${this.systemId}`, this.scene);
    carrier.setParent(parent);
    carrier.position.y = dims.sizeY * P.startYFrac;

    const carriage = CreateBox("carriageGeo", {
      width: dims.sizeX * P.widthFrac,
      height: P.height,
      depth: dims.sizeZ * P.depthFrac,
    }, this.scene);
    carriage.setParent(carrier);
    carriage.material = this.materialFactory.getMaterial("accent");
    carriage.isPickable = false;
    carriage.receiveShadows = true;

    if (hasFork) {
      this.createForkExtender(carrier, dims);
    }

    return carrier;
  }

  private createForkExtender(parent: TransformNode, dims: CarrierDims): TransformNode {
    const P = ParametricCarrierFactoryService.PROPORTIONS.fork;
    const forkExtender = new TransformNode(`forkExtender.${this.systemId}`, this.scene);
    forkExtender.setParent(parent);
    forkExtender.position.x = -dims.sizeX * 0.25;

    const gap = dims.sizeZ * P.gapFrac;
    for (const zSign of [-1, 1] as const) {
      const fork = CreateBox("forkGeo", { width: dims.forkLength, height: P.height, depth: P.width }, this.scene);
      fork.setParent(forkExtender);
      fork.position = new Vector3(-dims.forkLength / 2, -P.height, zSign * gap / 2);
      fork.material = this.materialFactory.getMaterial("steel");
      fork.isPickable = false;
      fork.receiveShadows = true;
    }

    return forkExtender;
  }
}
```

This example intentionally uses a simple visual language. For a hero model, replace plain boxes with chamfered plates, tubes, lathes, or CSG2 cut parts where the silhouette demands it.

---

## Modeling Checklist

Before finalizing generated code:

1. The model has a single root `TransformNode`.
2. The local frame is documented.
3. Options are JSDoc’d and mapped to axes.
4. Defaults live in static constants.
5. Magic numbers are grouped in `PROPORTIONS`.
6. `deriveDims()` precomputes geometry dimensions.
7. Helpers accept `Dims`; they do not reread raw options.
8. Materials come from a shared material factory.
9. Curved parts use proper builders: tube, lathe, torus, capsule, sphere, cylinder.
10. Custom plates/channels use `ExtrudePolygon` instead of box approximations.
11. Real visible cuts use CSG2; invisible overlaps use composition.
12. Each moving part has a named handle.
13. Multi-axis motion uses nested handles.
14. Static sub-parts are merged only after parenting is correct.
15. Consolidation skips animation handles.
16. Decals/transparent labels are added after consolidation.
17. Static repeated templates use thin instances.
18. Animated repeated templates use hierarchy clones.
19. A quick pivot test rotates/translates each handle and confirms only expected children move.
20. Node names are semantic enough for future animation code.

---

## Common Gotchas

1. **Thin instances flatten hierarchy.** They are not for templates with movable children.
2. **Merging across a handle breaks animation.** Always pass movement handles to the consolidation skip list.
3. **Material identity matters.** Two identical-looking materials are still two material instances.
4. **ExtrudePolygon needs `earcut`.** Pass it explicitly.
5. **Polygon winding matters.** Incorrect winding can flip faces or holes.
6. **CSG2 works in world space.** Transform operands before conversion.
7. **Dispose CSG2 intermediates.** They can hold WASM-side memory.
8. **Do not bake animated pivots.** Baking transforms destroys the pivot behavior you probably need.
9. **Negative scaling flips winding.** Mirrored geometry may need double-sided materials or winding fixes.
10. **Transparent meshes should not be blindly merged.** Preserve depth sorting and render order.
11. **Edge rendering is visual, not geometry.** It improves readability but does not bevel a model.
12. **Too many materials hurt both performance and coherence.** Start with a small palette.
13. **World-positioned children become confusing.** Parent first, then use local offsets.
14. **Cache keys must include every geometry-affecting option.** Missing one creates stale templates.
15. **A pile of primitives is not a model.** A model has structure, naming, material discipline, and motion boundaries.

---

## Related Babylon.js References

- [meshes.md](meshes.md) — mesh builders, transforms, instances, thin instances, cloning, merging, picking.
- [materials.md](materials.md) — PBR, Standard materials, textures, environment lighting.
- [animation-loading.md](animation-loading.md) — Animation API, animation groups, asset containers.
- [performance.md](performance.md) — profiling, draw calls, instancing, memory management.
- Live docs: `/features/featuresDeepDive/mesh/creation/set`, `/features/featuresDeepDive/mesh/creation/param`, `/features/featuresDeepDive/mesh/creation/custom`, `/features/featuresDeepDive/mesh/transforms/parent_pivot`.
- CSG2 live docs: `/features/featuresDeepDive/mesh/copies/csgPart1`.
