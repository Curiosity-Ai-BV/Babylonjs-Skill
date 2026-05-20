# Babylon.js Parametric Mesh Factories

When a mesh needs to **scale parametrically** (size, proportions, hole count) AND **repeat hundreds to millions of times**, neither glTF imports nor ad-hoc `CreateBox` calls hold up. A *parametric factory* — a small, stateless class that composes Babylon primitives into a single reusable template mesh — is the durable shape.

This page documents the pattern. Examples are domain-neutral; the technique applies equally to warehouses, furniture, vehicles, architectural elements, robots, and procedural buildings.

## Table of Contents
- [When to Use This Pattern](#when-to-use-this-pattern)
- [Anatomy of a Factory](#anatomy-of-a-factory)
- [Options + Defaults](#options--defaults)
- [PROPORTIONS Table](#proportions-table)
- [Dims: Pre-Computed Derived Dimensions](#dims-pre-computed-derived-dimensions)
- [Local Frame Convention](#local-frame-convention)
- [Shared Material Factory](#shared-material-factory)
- [Non-Box Cross-Sections (ExtrudePolygon + earcut)](#non-box-cross-sections-extrudepolygon--earcut)
- [Template Caching](#template-caching)
- [Thin-Instance Templates](#thin-instance-templates)
- [Worked Example: Lipped C-Channel Post](#worked-example-lipped-c-channel-post)
- [Worked Example: Composite Assembly](#worked-example-composite-assembly)
- [Animatable Children: Hierarchy for Movable Sub-Assemblies](#animatable-children-hierarchy-for-movable-sub-assemblies)
- [Worked Example: Lift with Fork (Movable Sub-Assembly)](#worked-example-lift-with-fork-movable-sub-assembly)
- [Worked Example: Transfer Car with Forks (Stacked Handles)](#worked-example-transfer-car-with-forks-stacked-handles)
- [Two-Layer Merging: mergeIntoStatic vs. consolidateByMaterial](#two-layer-merging-mergeintostatic-vs-consolidatebymaterial)
- [Composition over CSG](#composition-over-csg)

## When to Use This Pattern

Use a factory when:
- The mesh is **parametric** — height, width, leg count, hole pattern vary per call.
- The mesh **repeats** dozens to millions of times. Thin instances pay off only if every copy shares one source geometry.
- The mesh is **non-trivial** enough that scattering `CreateBox` calls through rendering code becomes a maintenance hazard.

Prefer **glTF/GLB import** when:
- The model is authored in DCC software (Blender, Maya, 3ds Max).
- It does not need to vary parametrically.
- It does not need thousands of copies.

Prefer **Solid Particle System** or **Node Geometry** when:
- Geometry is fully procedural with no human-authored proportion choices.
- Per-particle behaviour (physics, lifecycle, simulation) matters.

## Anatomy of a Factory

```typescript
import { Scene } from "@babylonjs/core/scene";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";

export interface MyPartOptions {
  index?: number;
  width?: number;
  height?: number;
  length?: number;
}

export class MyPartFactoryService {
  private static readonly DEFAULT_WIDTH = 1345;
  private static readonly DEFAULT_HEIGHT = 202;
  private static readonly DEFAULT_LENGTH = 980;

  private static readonly PROPORTIONS = { /* see below */ } as const;

  constructor(
    private readonly scene: Scene,
    private readonly systemId: string,
    private readonly materialFactory: MyMaterialFactoryService,
  ) {}

  public create(options: MyPartOptions = {}): TransformNode {
    const index = options.index ?? 0;
    const dims = this.deriveDims(options);

    const root = new TransformNode(`my_part_template_${index}.${this.systemId}`, this.scene);
    this.createBody(root, dims);
    this.createDeck(root, dims);
    this.createCornerPosts(root, dims);

    return root;
  }

  private deriveDims(options: MyPartOptions): Dims { /* ... */ }
  private createBody(parent: TransformNode, dims: Dims): void { /* ... */ }
  private createDeck(parent: TransformNode, dims: Dims): void { /* ... */ }
  private createCornerPosts(parent: TransformNode, dims: Dims): void { /* ... */ }
}
```

Invariants:
- **Stateless** except for an optional template cache.
- `create()` returns a `TransformNode` root. The caller decides where it sits in the world.
- Every helper takes a `Dims` object. No helper re-reads `options`.
- Per-instance variability lives in *clones* or *thin-instance buffers*, not in the factory.

## Options + Defaults

All-optional fields with `??` defaults inside `deriveDims`. Cheap to evolve, no breaking changes when adding new dimensions.

```typescript
export interface MyPartOptions {
  /** Stable index used in node names so multiple templates do not collide. */
  index?: number;
  /** Long body dimension in scene millimetres. Maps to local Z. */
  width?: number;
  /** Body height in scene millimetres. Maps to local Y. */
  height?: number;
  /** Short body dimension in scene millimetres. Maps to local X. */
  length?: number;
}
```

**JSDoc every option with the scene axis it maps to.** Readers of the interface should not have to dig into `deriveDims` to discover that `width` controls Z.

## PROPORTIONS Table

The single biggest mistake in parametric meshes is letting magic numbers leak into every helper. The fix is one frozen `PROPORTIONS` constant the whole class reads from:

```typescript
private static readonly PROPORTIONS = {
  body: {
    baselineYFrac: 0.25,     // bodyBaselineY = sizeY * 0.25
    heightFrac:    0.60,     // bodyHeight    = sizeY * 0.60 — chosen so baseline + body + deck + bands = 1.0
    insetX:        25,       // bodyWidth     = sizeX - 25
    insetZ:        25,       // bodyDepth     = sizeZ - 25
  },
  deck: {
    heightFrac: 0.08,        // of sizeY
    insetX:     70,          // raw mm
    chamfer:    10,
  },
  cornerPost: {
    diameter:     80,
    height:       50,
    tessellation: 28,
  },
} as const;
```

Conventions that scale:
- **Fractions multiply a body dimension.** Name them with a `Frac` suffix. Raw numbers are absolute scene units.
- **Group keys by part.** Helpers grab the whole sub-object once at the top: `const B = MyFactory.PROPORTIONS.body;`.
- **Comment every value with the *why*.** If `0.60` is "chosen so baseline + body + deck + bands sum to 1.0", that reasoning lives next to the number — or the next edit will break it.

## Dims: Pre-Computed Derived Dimensions

Helpers must not call `options.height ?? DEFAULT` repeatedly. `deriveDims` does that once, and every helper takes the resulting `Dims`:

```typescript
interface Dims {
  sizeX: number;
  sizeY: number;
  sizeZ: number;
  /** Y at which the body shell's bottom edge sits. */
  bodyBaselineY: number;
  bodyHeight: number;
  /** Y at which the body shell ends. */
  bodyTop: number;
  deckHeight: number;
}

private deriveDims(options: MyPartOptions): Dims {
  const sizeX = options.length ?? MyPartFactoryService.DEFAULT_LENGTH;
  const sizeY = options.height ?? MyPartFactoryService.DEFAULT_HEIGHT;
  const sizeZ = options.width  ?? MyPartFactoryService.DEFAULT_WIDTH;
  const P = MyPartFactoryService.PROPORTIONS;

  const bodyBaselineY = sizeY * P.body.baselineYFrac;
  const bodyHeight    = sizeY * P.body.heightFrac;

  return {
    sizeX, sizeY, sizeZ,
    bodyBaselineY,
    bodyHeight,
    bodyTop: bodyBaselineY + bodyHeight,
    deckHeight: sizeY * P.deck.heightFrac,
  };
}
```

Why this matters: with `Dims`, two helpers cannot silently disagree on where the body ends and the deck begins. Without it, you ship a part with a 0.5 mm gap and don't notice until QA.

## Local Frame Convention

Document the local frame at the **top of the file**, in a JSDoc block — not buried in a method. Readers (you, in three months) need to know:

- where the **origin** sits (centre of base, bottom-left corner, ...)
- which axis is **up** (Y is Babylon's default — say it anyway)
- which user-facing option maps to which local axis (`width → Z`, `length → X`)

```typescript
/**
 * Builds the visual Babylon hierarchy for one parametric assembly.
 *
 * Local frame: Y is vertical with the bottom face at Y = 0. The user-facing
 * `width` parameter (long body dimension) maps to local Z, and `length` (short
 * body dimension) maps to local X.
 */
export class MyPartFactoryService { /* ... */ }
```

## Shared Material Factory

Every factory pulls materials from a shared `MaterialFactory` keyed by a small palette. This matters for two reasons:

1. **GPU material identity.** Two identical `PBRMaterial` objects do not collapse into one draw call — Babylon groups by *material identity*, not by spec. A scene-wide named cache forces sharing.
2. **`consolidateByMaterial` works.** Mesh consolidation only collapses meshes that share the *exact same material instance*.

```typescript
import { Color3 } from "@babylonjs/core/Maths/math.color";
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import type { Material } from "@babylonjs/core/Materials/material";
import { Texture } from "@babylonjs/core/Materials/Textures/texture";
import type { Scene } from "@babylonjs/core/scene";

export type MaterialKey = "aluminium" | "steel" | "rubber" | "accent";

interface PbrSpec {
  color?: string;
  textureUrl?: string;
  metallic: number;
  roughness: number;
}

const PALETTE: Record<MaterialKey, PbrSpec> = {
  aluminium: { color: "#DFE4F4", metallic: 0.95, roughness: 0.40 },
  steel:     { color: "#6E7378", metallic: 0.75, roughness: 0.45 },
  rubber:    { color: "#111112", metallic: 0.45, roughness: 0.65 },
  accent:    { color: "#F9AB01", metallic: 0.90, roughness: 0.90 },
};

export class MyMaterialFactoryService {
  private readonly materials = new Map<MaterialKey, PBRMaterial>();

  constructor(private readonly scene: Scene) {}

  public getMaterial(key: MaterialKey): Material {
    const cached = this.materials.get(key);
    if (cached) return cached;

    // Scene-wide fallback so two factories asking for "aluminium" share one GPU material.
    const name = `my-material-${key}`;
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
    this.materials.set(key, material);
    return material;
  }
}
```

**Naming caveat:** key shared materials *scene-globally*, not per system. Two subsystems asking for the same logical material (`"aluminium"`) must resolve to the same `PBRMaterial` instance, or `consolidateByMaterial` will treat them as different and produce extra draw calls.

## Non-Box Cross-Sections (ExtrudePolygon + earcut)

`CreateBox` handles rectangular prisms. For everything else — lipped C-channels, chamfered plates, octagonal beams, gear teeth, T-slots — use `ExtrudePolygon`:

```typescript
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import { ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import earcut from "earcut";

const halfW = width / 2;
const halfL = length / 2;
const shape: Vector3[] = [
  new Vector3( halfW, 0, -halfL),
  new Vector3( halfW, 0,  halfL),
  new Vector3(-halfW, 0,  halfL),
  new Vector3(-halfW, 0, -halfL),
];

const mesh = ExtrudePolygon(
  "my_template",
  { shape, depth: height },
  scene,
  earcut,
);
```

Gotchas:
- **earcut is required.** Pass it as the fourth argument. It is an npm package (`earcut`), not bundled with `@babylonjs/core`.
- **Pivot sits on the top face.** `ExtrudePolygon` hangs geometry below `Y = 0`. To put the bottom at `Y = 0`, lift and bake:
  ```typescript
  mesh.position = new Vector3(0, height, 0);
  mesh.bakeCurrentTransformIntoVertices();
  ```
- **Winding order matters.** CCW outer ring; CW for holes.
- **No tessellation parameter.** The polygon is triangulated by earcut from your vertex count. Add more vertices to approximate curves.

## Template Caching

If `create()` is called multiple times with identical numeric inputs, return the same `Mesh`. This avoids dispose churn and surprise re-tessellation. The pattern is a **content-hashed cache key**:

```typescript
import type { Mesh } from "@babylonjs/core/Meshes/mesh";

interface CachedTemplate {
  mesh: Mesh;
  cacheKey: string;
}

private readonly templates = new Map<TemplateSlot, CachedTemplate>();

private ensureTemplate(
  slot: TemplateSlot,
  cacheKey: string,
  build: () => Mesh,
): Mesh {
  const cached = this.templates.get(slot);
  if (cached && cached.cacheKey === cacheKey) return cached.mesh;
  cached?.mesh.dispose();
  const mesh = build();
  this.templates.set(slot, { mesh, cacheKey });
  return mesh;
}
```

The cache key is a `|`-joined string of every parameter that affects geometry:

```typescript
private ensurePole(options: PoleOptions): Mesh {
  const width     = options.width     ?? DEFAULT_WIDTH;
  const length    = options.length    ?? DEFAULT_LENGTH;
  const thickness = options.thickness ?? DEFAULT_THICKNESS;
  const cacheKey = `${width}|${length}|${thickness}`;
  return this.ensureTemplate("pole", cacheKey, () =>
    this.buildPole({ width, length, thickness }),
  );
}
```

**Gotcha:** if a geometry-affecting field is missing from the cache key, callers will get stale meshes when they change it. Tedious bug to track down — keep the key in sync with `buildX`'s inputs.

## Thin-Instance Templates

Templates exist to be thin-instanced thousands of times. Two conventions make the call site trivial:

1. **`UNIT_TEMPLATE_LENGTH = 1000`.** Every template's main axis is built at exactly this length. Callers set `thinInstance.scaling.<axis> = realLengthMm / 1000` and the world extent equals `realLengthMm`.
2. **`setEnabled(false)` on the template.** The template itself never renders; only its thin instances do.

```typescript
private static readonly UNIT_TEMPLATE_LENGTH = 1000;

private finalize(mesh: Mesh, material: Material): Mesh {
  mesh.material = material;
  mesh.isPickable = false;
  mesh.receiveShadows = true;
  mesh.setEnabled(false);   // template stays invisible
  return mesh;
}
```

Call site:

```typescript
import "@babylonjs/core/Meshes/thinInstanceMesh";
import { Matrix } from "@babylonjs/core/Maths/math.vector";

const template = rackingFactory.refreshTemplates(config).pole;
const matrices = new Float32Array(16 * poleCount);
for (let i = 0; i < poleCount; i++) {
  const heightScale = poles[i].heightMm / 1000;   // template is 1000 mm tall
  Matrix.Scaling(1, heightScale, 1)
    .multiply(Matrix.Translation(poles[i].x, 0, poles[i].z))
    .copyToArray(matrices, i * 16);
}
template.thinInstanceSetBuffer("matrix", matrices, 16, false);
```

## Worked Example: Lipped C-Channel Post

A single-primitive template — a vertical post with a C-channel cross-section, lips on the opening side. Demonstrates `ExtrudePolygon` + earcut, `UNIT_TEMPLATE_LENGTH`, baking, and `finalize`.

```typescript
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import type { Material } from "@babylonjs/core/Materials/material";
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import type { Scene } from "@babylonjs/core/scene";
import earcut from "earcut";

export interface PoleOptions {
  width?: number;       // cross-section width (X)
  length?: number;      // cross-section depth (Z)
  thickness?: number;   // wall thickness
  lipLength?: number;   // lip length on opening side
}

export class PoleFactoryService {
  private static readonly UNIT_TEMPLATE_LENGTH = 1000;
  private static readonly EPSILON_MM = 0.01;
  private static readonly DEFAULT_WIDTH = 100;
  private static readonly DEFAULT_LENGTH = 80;
  private static readonly DEFAULT_THICKNESS = 10;
  private static readonly DEFAULT_LIP_LENGTH = 35;

  constructor(
    private readonly scene: Scene,
    private readonly materialFactory: MyMaterialFactoryService,
  ) {}

  public build(options: PoleOptions = {}): Mesh {
    const width     = options.width     ?? PoleFactoryService.DEFAULT_WIDTH;
    const length    = options.length    ?? PoleFactoryService.DEFAULT_LENGTH;
    const thickness = options.thickness ?? PoleFactoryService.DEFAULT_THICKNESS;
    const height    = PoleFactoryService.UNIT_TEMPLATE_LENGTH;

    const halfW = width / 2;
    const halfL = length / 2;

    // Keep the lip wider than the wall and never crossing the centreline.
    const lipLength = Math.max(
      thickness + PoleFactoryService.EPSILON_MM,
      Math.min(options.lipLength ?? PoleFactoryService.DEFAULT_LIP_LENGTH, halfW - thickness),
    );

    // Lipped C-channel footprint in XZ. Web at Z = -halfL, opening at Z = +halfL. CCW outer ring.
    const innerLeft  = -halfW + thickness;
    const innerRight =  halfW - thickness;
    const innerBack  = -halfL + thickness;
    const innerFront =  halfL - thickness;
    const back       = -halfL;
    const opening    =  halfL;
    const leftLipInnerEdge  = -halfW + lipLength;
    const rightLipInnerEdge =  halfW - lipLength;

    const shape: Vector3[] = [
      new Vector3( halfW,              0, back),
      new Vector3( halfW,              0, opening),
      new Vector3( rightLipInnerEdge,  0, opening),
      new Vector3( rightLipInnerEdge,  0, innerFront),
      new Vector3( innerRight,         0, innerFront),
      new Vector3( innerRight,         0, innerBack),
      new Vector3( innerLeft,          0, innerBack),
      new Vector3( innerLeft,          0, innerFront),
      new Vector3( leftLipInnerEdge,   0, innerFront),
      new Vector3( leftLipInnerEdge,   0, opening),
      new Vector3(-halfW,              0, opening),
      new Vector3(-halfW,              0, back),
    ];

    const mesh = ExtrudePolygon("pole_template", { shape, depth: height }, this.scene, earcut);

    // ExtrudePolygon hangs geometry below Y = 0; lift + bake to put bottom at Y = 0.
    mesh.position = new Vector3(0, height, 0);
    mesh.bakeCurrentTransformIntoVertices();

    return this.finalize(mesh, this.materialFactory.getMaterial("steel"));
  }

  private finalize(mesh: Mesh, material: Material): Mesh {
    mesh.material = material;
    mesh.isPickable = false;
    mesh.receiveShadows = true;
    mesh.setEnabled(false);
    return mesh;
  }
}
```

## Worked Example: Composite Assembly

A multi-primitive parametric assembly — a body shell, a chamfered top deck, and four corner posts. Demonstrates `PROPORTIONS`, `Dims`, helper composition under a `TransformNode`, and `consolidateByMaterial`.

```typescript
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import type { Material } from "@babylonjs/core/Materials/material";
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import { StandardMaterial } from "@babylonjs/core/Materials/standardMaterial";
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";
import { CreateBox } from "@babylonjs/core/Meshes/Builders/boxBuilder";
import { CreateCylinder } from "@babylonjs/core/Meshes/Builders/cylinderBuilder";
import { ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import type { Scene } from "@babylonjs/core/scene";
import earcut from "earcut";

export interface ChassisOptions {
  index?: number;
  width?: number;    // long body dimension, maps to local Z
  height?: number;   // body height, maps to local Y
  length?: number;   // short body dimension, maps to local X
}

interface Dims {
  sizeX: number;
  sizeY: number;
  sizeZ: number;
  bodyBaselineY: number;
  bodyHeight: number;
  bodyTop: number;
  deckHeight: number;
}

/**
 * Builds a parametric chassis from a body shell, chamfered top deck, and four
 * corner posts.
 *
 * Local frame: Y is vertical with the bottom face at Y = 0. The user-facing
 * `width` parameter (long body dimension) maps to local Z, and `length` (short
 * body dimension) maps to local X.
 */
export class ChassisFactoryService {
  private static readonly DEFAULT_WIDTH = 1345;
  private static readonly DEFAULT_HEIGHT = 202;
  private static readonly DEFAULT_LENGTH = 980;

  private static readonly PROPORTIONS = {
    body: {
      baselineYFrac: 0.25,    // bodyBaselineY = sizeY * 0.25
      heightFrac:    0.60,    // bodyHeight    = sizeY * 0.60
      insetX:        25,      // bodyWidth     = sizeX - 25
      insetZ:        25,      // bodyDepth     = sizeZ - 25
    },
    deck: {
      heightFrac: 0.08,
      insetX:     70,
      insetZ:    140,
      chamfer:    10,
    },
    cornerPost: {
      diameter:     80,
      height:       50,
      yCentreDivisor: 1.8,    // postYCentre = bodyTop - postHeight / 1.8
      tessellation: 28,
    },
  } as const;

  constructor(
    private readonly scene: Scene,
    private readonly systemId: string,
    private readonly materialFactory: MyMaterialFactoryService,
  ) {}

  public create(options: ChassisOptions = {}): TransformNode {
    const index = options.index ?? 0;
    const dims = this.deriveDims(options);

    const root = new TransformNode(`chassis_template_${index}.${this.systemId}`, this.scene);

    this.createBody(root, dims);
    this.createDeck(root, dims);
    this.createCornerPosts(root, dims);

    this.consolidateByMaterial(root);
    return root;
  }

  private deriveDims(options: ChassisOptions): Dims {
    const sizeX = options.length ?? ChassisFactoryService.DEFAULT_LENGTH;
    const sizeY = options.height ?? ChassisFactoryService.DEFAULT_HEIGHT;
    const sizeZ = options.width  ?? ChassisFactoryService.DEFAULT_WIDTH;
    const P = ChassisFactoryService.PROPORTIONS;

    const bodyBaselineY = sizeY * P.body.baselineYFrac;
    const bodyHeight    = sizeY * P.body.heightFrac;
    return {
      sizeX, sizeY, sizeZ,
      bodyBaselineY,
      bodyHeight,
      bodyTop: bodyBaselineY + bodyHeight,
      deckHeight: sizeY * P.deck.heightFrac,
    };
  }

  private createBody(parent: TransformNode, dims: Dims): void {
    const B = ChassisFactoryService.PROPORTIONS.body;
    const body = CreateBox("body", {
      width:  dims.sizeX - B.insetX,
      height: dims.bodyHeight,
      depth:  dims.sizeZ - B.insetZ,
    }, this.scene);
    body.setParent(parent);
    body.position = new Vector3(0, dims.bodyBaselineY + dims.bodyHeight / 2, 0);
    body.material = this.materialFactory.getMaterial("aluminium");
    body.isPickable = false;
    body.receiveShadows = true;
  }

  private createDeck(parent: TransformNode, dims: Dims): void {
    const D = ChassisFactoryService.PROPORTIONS.deck;
    this.createChamferedPlate(
      parent,
      "deck",
      dims.sizeX - D.insetX,
      dims.sizeZ - D.insetZ,
      dims.deckHeight,
      D.chamfer,
      new Vector3(0, dims.bodyTop, 0),
      "steel",
    );
  }

  private createCornerPosts(parent: TransformNode, dims: Dims): void {
    const P = ChassisFactoryService.PROPORTIONS.cornerPost;
    const B = ChassisFactoryService.PROPORTIONS.body;
    const halfX = (dims.sizeX - B.insetX) / 2;
    const halfZ = (dims.sizeZ - B.insetZ) / 2;
    const yCentre = dims.bodyTop - P.height / P.yCentreDivisor;

    for (const [xSign, zSign] of [[-1,-1], [+1,-1], [-1,+1], [+1,+1]] as const) {
      const post = CreateCylinder("corner_post", {
        diameter:     P.diameter,
        height:       P.height,
        tessellation: P.tessellation,
      }, this.scene);
      post.setParent(parent);
      post.position = new Vector3(xSign * halfX, yCentre, zSign * halfZ);
      post.material = this.materialFactory.getMaterial("steel");
      post.isPickable = false;
      post.receiveShadows = true;
    }
  }

  /**
   * Rectangular plate with 45° chamfered corners. Extrudes an 8-sided polygon.
   * `position` is the plate's centre on its bottom face.
   */
  private createChamferedPlate(
    parent: TransformNode,
    name: string,
    width: number,
    depth: number,
    thickness: number,
    chamfer: number,
    position: Vector3,
    material: MaterialKey,
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
    const mesh = ExtrudePolygon(name, { shape, depth: thickness }, this.scene, earcut);
    mesh.setParent(parent);
    mesh.position = new Vector3(position.x, position.y + thickness / 2, position.z);
    mesh.material = this.materialFactory.getMaterial(material);
    mesh.isPickable = false;
    mesh.receiveShadows = true;
    return mesh;
  }

  /**
   * Merges meshes that share a material into one mesh per material to cut
   * draw calls. `skip` lists animation handles whose subtree must NOT be
   * pulled across the movement boundary — call once per movement scope:
   * `consolidateByMaterial(root, [carrier])` then `consolidateByMaterial(carrier)`.
   *
   * Run AFTER all primitives are parented; run BEFORE adding decals or any
   * textured planes you do NOT want merged.
   */
  private consolidateByMaterial(root: TransformNode, skip: TransformNode[] = []): void {
    const byMaterial = new Map<Material, Mesh[]>();
    const skipSet = new Set<TransformNode>(skip);

    // Decals (StandardMaterial) and PBR alpha-blend keep their own depth sort.
    const isMergeable = (m: Material): boolean => {
      if (m instanceof StandardMaterial) return false;
      if (m instanceof PBRMaterial && m.transparencyMode === PBRMaterial.PBRMATERIAL_ALPHABLEND) return false;
      return true;
    };

    const visit = (node: TransformNode): void => {
      for (const child of node.getChildren()) {
        if (child instanceof TransformNode && skipSet.has(child)) continue;
        if (child instanceof Mesh && child.material && isMergeable(child.material)) {
          const arr = byMaterial.get(child.material);
          if (arr) arr.push(child);
          else byMaterial.set(child.material, [child]);
        } else if (child instanceof TransformNode) {
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
}
```

## Animatable Children: Hierarchy for Movable Sub-Assemblies

Most useful templates have moving parts — a lift carriage that travels up the mast, forks that extend off a transfer car, a conveyor band that scrolls, a turret that rotates. These cannot be merged into a single static mesh: the merged geometry has no per-part transform left to animate.

The pattern is **TransformNode-as-animation-handle**: every movable sub-assembly gets its own named `TransformNode` that the system layer translates/rotates/scales at runtime. Static children parented under it ride along automatically.

### The rule

1. Wrap every movable sub-assembly in its own named `TransformNode` *before* its primitives are added.
2. Helpers parent primitives (and any internal `mergeIntoStatic` results) under that handle.
3. The factory either **returns the handle** as part of its result, or builds with a stable name so the system layer can look it up.
4. The merging passes scope explicitly to one movement layer at a time — the handle is passed in a `skip` list when consolidating its parent, then consolidated *as its own root* in a second pass.

```typescript
private createLiftCarrier(parent: TransformNode, dims: Dims): TransformNode {
  // The handle. The system layer translates this node in Y to move the
  // carriage up and down the mast. Everything parented under it rides along.
  const carrier = new TransformNode(`lift_carrier.${this.systemId}`, this.scene);
  carrier.setParent(parent);
  carrier.position = new Vector3(0, dims.startY, 0);

  this.createForkCarriage(carrier, dims);
  this.createForkPlate(carrier, dims);
  this.createChainConveyor(carrier, dims);   // a whole sub-assembly that rides with the fork

  return carrier;
}
```

### Pivot points

Animation often rotates around a non-default point — a hinge, a wheel axle, a turret base. Set the pivot on the handle, *before* parenting children:

```typescript
const wheel = new TransformNode(`wheel_front_left.${this.systemId}`, this.scene);
wheel.setParent(parent);
wheel.position = wheelCentre;             // world position of the axle
wheel.setPivotPoint(Vector3.Zero());      // rotate around its own origin

// Tire + hub primitives parented under `wheel`; both spin when `wheel.rotation.x` changes.
```

### Stacking handles for multi-axis motion

When a sub-assembly itself has movable sub-parts (a fork carriage that extends *and* rides a lift), nest handles. Each layer owns one axis of motion:

```
root (static foundation, mast, head)
  └── lift_carrier        (translates Y — moves up the mast)
        └── fork_extender (translates X — extends/retracts off the carriage)
              └── fork_plate, chain_conveyor, ...
```

The system layer animates each node independently:

```typescript
lift.handles.liftCarrier.position.y = liftCurve(t);
lift.handles.forkExtender.position.x = forkCurve(t);
```

Because `fork_extender` is a child of `lift_carrier`, retracting the fork while the lift descends composes the two motions automatically — no math at the call site.

### Stable lookup via a handles object

The factory returns a typed `Handles` object so the system layer doesn't depend on string lookups:

```typescript
export interface LiftHandles {
  root: TransformNode;
  liftCarrier: TransformNode;
}

public create(options: LiftOptions): LiftHandles {
  const root = new TransformNode(`lift.${this.systemId}`, this.scene);
  this.createFoundation(root, options);
  this.createMast(root, options);
  this.createHead(root, options);
  const liftCarrier = this.createLiftCarrier(root, options);

  // Two consolidation passes: static structure at root (skip the carrier),
  // then the carrier's own subtree (so internal parts collapse but stay movable).
  this.consolidateByMaterial(root, [liftCarrier]);
  this.consolidateByMaterial(liftCarrier);
  return { root, liftCarrier };
}
```

### Cloning animatable templates

`TransformNode.instantiateHierarchy` clones the whole subtree, preserving every handle. Each clone gets its own independent animation handles — this is how one lift template spawns 50 individually-moving lifts on screen.

```typescript
const clone = lift.root.instantiateHierarchy(null);
const cloneCarrier = clone?.getChildTransformNodes(true)
  .find(n => n.name.startsWith("lift_carrier."));
// `cloneCarrier` is the per-instance handle, independent of the source template.
```

Caveat: thin instances **do not** preserve sub-node hierarchy — they're a flat matrix buffer. Templates with movable sub-parts need `instantiateHierarchy` clones, not thin instances. Use thin instances only for fully-static templates (poles, rails, pallets).

### Anti-patterns to avoid

- **Bare animatable mesh under root.** A single `Mesh` (not wrapped in a TransformNode) can be animated directly, but the moment you want sub-parts on it you cannot retro-fit a parent without re-baking. Wrap from day one.
- **Merging *across* a movement boundary.** If the consolidator pulls a mesh out from under an animation handle and re-parents it to root, it stops moving. The skip-list rule prevents this.
- **Forgetting to skip in other passes.** Edge rendering setup, shadow caster registration, bounding-box recompute — anything that walks the hierarchy must also respect the skip list, or it can flatten the structure you depend on.
- **Animating a merged mesh's sub-mesh indices.** Doesn't work. Once meshes are merged, sub-meshes share one transform.
- **Thin-instancing a hierarchical template.** Thin instances flatten to one matrix per copy. Movable parts disappear. Use `instantiateHierarchy` for per-copy animation.

## Worked Example: Lift with Fork (Movable Sub-Assembly)

A parametric vertical lift: U-shape foundation, a mast column, a head plate at the top, and — the whole point — a **lift carrier** that travels up and down the mast carrying a fork plate and chain conveyor.

```typescript
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import type { Material } from "@babylonjs/core/Materials/material";
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import { StandardMaterial } from "@babylonjs/core/Materials/standardMaterial";
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";
import { CreateBox } from "@babylonjs/core/Meshes/Builders/boxBuilder";
import { ExtrudePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";
import type { Scene } from "@babylonjs/core/scene";
import earcut from "earcut";

export interface LiftOptions {
  index?: number;
  height: number;          // total lift height (mm), foundation bottom to head top
  marginBottom: number;    // initial carrier Y position above the foundation
}

export interface LiftHandles {
  root: TransformNode;
  /** Translates in Y. Carries the fork plate and chain conveyor. */
  liftCarrier: TransformNode;
}

/**
 * Vertical lift with a movable carrier.
 *
 * Local frame: Y up. Foundation bottom at Y = 0. Mast on +X side.
 *
 * Hierarchy:
 *   root
 *     ├── foundation_u          (static, ExtrudePolygon)
 *     ├── mast_column           (static, box)
 *     ├── head_plate            (static, box)
 *     └── lift_carrier          ← animation handle (translates in Y)
 *           ├── fork_carriage   (rides inside the mast)
 *           ├── fork_u_plate    (U-shape that holds the load)
 *           └── chain_conveyor  (a sub-assembly — see chain conveyor pattern)
 */
export class LiftFactoryService {
  static readonly LIFT_LENGTH = 2000;
  static readonly LIFT_WIDTH = 1730;

  private static readonly PROPORTIONS = {
    foundation: { height: 100, thickness: 200, interiorChamfer: { x: 250, z: 250 } },
    mast:       { width: 200, depth: 300 },
    head:       { width: 200, height: 200, length: 600 },
    fork: {
      carriageWidth:  300,
      carriageHeight: 400,
      carriageDepth:  1200,
      backThickness:  50,
      length:        1200,
      plateHeight:    40,
      distanceBetweenForks: 800,
    },
  } as const;

  constructor(
    private readonly scene: Scene,
    private readonly systemId: string,
    private readonly materialFactory: MyMaterialFactoryService,
  ) {}

  public create(options: LiftOptions): LiftHandles {
    const index = options.index ?? 0;
    const root = new TransformNode(`lift_${index}.${this.systemId}`, this.scene);

    this.createFoundation(root);
    this.createMast(root, options);
    this.createHead(root, options);
    const liftCarrier = this.createLiftCarrier(root, options);

    // Two consolidation passes. Static structure under root: merge everything
    // EXCEPT the carrier (skip-list). Then the carrier's own subtree: merge
    // its internal parts so the carriage + fork + conveyor become fewer
    // draw calls, while still riding the carrier when it animates.
    this.consolidateByMaterial(root, [liftCarrier]);
    this.consolidateByMaterial(liftCarrier);

    return { root, liftCarrier };
  }

  // ─────────────────────────────────────────────────────────────────────
  // Static structure
  // ─────────────────────────────────────────────────────────────────────

  private createFoundation(root: TransformNode): void {
    // U-shape foundation, opening toward -X with the back wall at +X. Arms
    // are half the lift length. Both interior corners are chamfered.
    const P = LiftFactoryService.PROPORTIONS.foundation;
    const length = LiftFactoryService.LIFT_LENGTH;
    const width  = LiftFactoryService.LIFT_WIDTH;
    const t = P.thickness;
    const armLength = (length - t) / 2;
    const openingX = length / 2 - t - armLength;
    const cx = Math.min(P.interiorChamfer.x, armLength);
    const cz = Math.min(P.interiorChamfer.z, width / 2 - t);

    const shape: Vector3[] = [
      new Vector3( length / 2,           0,  width / 2),
      new Vector3( openingX,             0,  width / 2),
      new Vector3( openingX,             0,  width / 2 - t),
      new Vector3( length / 2 - t - cx,  0,  width / 2 - t),
      new Vector3( length / 2 - t,       0,  width / 2 - t - cz),
      new Vector3( length / 2 - t,       0, -width / 2 + t + cz),
      new Vector3( length / 2 - t - cx,  0, -width / 2 + t),
      new Vector3( openingX,             0, -width / 2 + t),
      new Vector3( openingX,             0, -width / 2),
      new Vector3( length / 2,           0, -width / 2),
    ];

    const mesh = ExtrudePolygon("foundation_u", { shape, depth: P.height }, this.scene, earcut);
    mesh.setParent(root);
    mesh.position = new Vector3(0, P.height, 0);
    mesh.material = this.materialFactory.getMaterial("aluminium");
    mesh.isPickable = false;
    mesh.receiveShadows = true;
  }

  private createMast(root: TransformNode, options: LiftOptions): void {
    const P = LiftFactoryService.PROPORTIONS.mast;
    const H = LiftFactoryService.PROPORTIONS.head;
    const mastHeight = Math.max(0, options.height - H.height);
    const mastCenterX = LiftFactoryService.LIFT_LENGTH / 2 - P.width / 2;

    const mast = CreateBox("mast_column", {
      width: P.width, height: mastHeight, depth: P.depth,
    }, this.scene);
    mast.setParent(root);
    mast.position = new Vector3(mastCenterX, mastHeight / 2, 0);
    mast.material = this.materialFactory.getMaterial("steel");
    mast.isPickable = false;
    mast.receiveShadows = true;
  }

  private createHead(root: TransformNode, options: LiftOptions): void {
    const H = LiftFactoryService.PROPORTIONS.head;
    const head = CreateBox("head_plate", {
      width: H.length, height: H.height, depth: H.width,
    }, this.scene);
    head.setParent(root);
    head.position = new Vector3(
      LiftFactoryService.LIFT_LENGTH / 2 - H.length / 2,
      options.height - H.height / 2,
      0,
    );
    head.material = this.materialFactory.getMaterial("steel");
    head.isPickable = false;
    head.receiveShadows = true;
  }

  // ─────────────────────────────────────────────────────────────────────
  // Movable carrier
  // ─────────────────────────────────────────────────────────────────────

  private createLiftCarrier(root: TransformNode, options: LiftOptions): TransformNode {
    const F = LiftFactoryService.PROPORTIONS.fork;
    const H = LiftFactoryService.PROPORTIONS.head;
    const mastHeight = Math.max(0, options.height - H.height);
    const carrierY = Math.max(0, Math.min(options.marginBottom, mastHeight - F.carriageHeight));

    // ── The animation handle. System layer translates this in Y. ──
    const carrier = new TransformNode(`lift_carrier.${this.systemId}`, this.scene);
    carrier.setParent(root);
    carrier.position = new Vector3(0, carrierY, 0);

    // Carriage box: rides inside the mast's -X face. Only `backThickness`
    // worth of width pokes out in -X; the rest hides in the mast volume.
    const mastFrontX = LiftFactoryService.LIFT_LENGTH / 2 - LiftFactoryService.PROPORTIONS.mast.width;
    const insertion = F.carriageWidth - F.backThickness;
    const carriageEndX = mastFrontX + insertion;
    const carriageCenterX = carriageEndX - F.carriageWidth / 2;

    const carriage = CreateBox("fork_carriage", {
      width: F.carriageWidth, height: F.carriageHeight, depth: F.carriageDepth,
    }, this.scene);
    carriage.setParent(carrier);
    carriage.position = new Vector3(carriageCenterX, F.carriageHeight / 2, 0);
    carriage.material = this.materialFactory.getMaterial("accent");
    carriage.isPickable = false;
    carriage.receiveShadows = true;

    // Fork U-plate. CCW outline so ExtrudePolygon's normals point outward
    // after the +Y extrude. Outer Z extent = distanceBetweenForks + 2*backThickness.
    const halfOpening = F.distanceBetweenForks / 2;
    const halfDepth   = halfOpening + F.backThickness;
    const forkBackX   = mastFrontX;
    const innerBackX  = forkBackX - F.backThickness;
    const forkTipX    = forkBackX - F.length;

    const shape: Vector3[] = [
      new Vector3(forkBackX,   0,  halfDepth),
      new Vector3(forkTipX,    0,  halfDepth),
      new Vector3(forkTipX,    0,  halfOpening),
      new Vector3(innerBackX,  0,  halfOpening),
      new Vector3(innerBackX,  0, -halfOpening),
      new Vector3(forkTipX,    0, -halfOpening),
      new Vector3(forkTipX,    0, -halfDepth),
      new Vector3(forkBackX,   0, -halfDepth),
    ];

    const forkPlate = ExtrudePolygon("fork_u_plate", { shape, depth: F.plateHeight }, this.scene, earcut);
    forkPlate.setParent(carrier);
    // ExtrudePolygon's pivot sits on the extrusion's top face; add plateHeight
    // to land the U flat on the carriage's bottom edge.
    forkPlate.position = new Vector3(0, F.plateHeight, 0);
    forkPlate.material = this.materialFactory.getMaterial("accent");
    forkPlate.isPickable = false;
    forkPlate.receiveShadows = true;

    // The chain conveyor sub-assembly would go here, also parented under
    // `carrier` so the whole thing — carriage + fork + conveyor — rides as
    // one when the system animates `carrier.position.y`.

    return carrier;
  }

  // consolidateByMaterial — same as the composite worked example,
  // with the skip-list parameter shown above.
}
```

System-layer animation:

```typescript
const lift = liftFactory.create({ index: 0, height: 8000, marginBottom: 200 });
scene.onBeforeRenderObservable.add(() => {
  const t = performance.now() / 1000;
  lift.liftCarrier.position.y = 200 + 3000 * (0.5 + 0.5 * Math.sin(t));
  // Foundation, mast, head do not move. Carriage, fork plate, and conveyor
  // all ride with `liftCarrier` automatically.
});
```

## Worked Example: Transfer Car with Forks (Stacked Handles)

A transfer car runs along a rail and carries a fork sub-assembly that can extend perpendicularly to the rail direction. Two stacked animation handles — one per axis of motion.

```
root (parent: rail station / aisle)
  └── chassis             ← translates along X (or follows a rail path)
        ├── (chassis body, wheels, cabin — static under chassis)
        └── fork_extender ← translates along Z (extends off the chassis)
              ├── carriage box
              └── fork U-plate
```

```typescript
export interface TransferCarHandles {
  root: TransformNode;
  /** Translates along the rail direction (X). */
  chassis: TransformNode;
  /** Translates perpendicular to the rail (Z). Child of `chassis`. */
  forkExtender: TransformNode;
}

public create(options: TransferCarOptions): TransferCarHandles {
  const root = new TransformNode(`transfer_car.${this.systemId}`, this.scene);

  // Chassis handle: rides the rail.
  const chassis = new TransformNode(`chassis.${this.systemId}`, this.scene);
  chassis.setParent(root);
  this.createChassisBody(chassis, options);
  this.createWheels(chassis, options);

  // Fork extender: child of chassis, so retraction composes with chassis motion.
  const forkExtender = new TransformNode(`fork_extender.${this.systemId}`, this.scene);
  forkExtender.setParent(chassis);
  this.createForkCarriage(forkExtender, options);
  this.createForkPlate(forkExtender, options);

  // Three consolidation scopes: static-under-root (none in this case),
  // chassis-but-not-forkExtender, and forkExtender on its own.
  this.consolidateByMaterial(root, [chassis]);
  this.consolidateByMaterial(chassis, [forkExtender]);
  this.consolidateByMaterial(forkExtender);

  return { root, chassis, forkExtender };
}
```

System-layer animation composes naturally:

```typescript
// Drive the car along the rail and extend the fork at the destination.
car.chassis.position.x = railPosition(t);              // car moves
car.forkExtender.position.z = extension(t);            // fork extends — relative to the car
```

Because `forkExtender` is a child of `chassis`, extending the fork while the car drives composes the two motions correctly — no need to recompute the fork's world position.

### Forks-Only Template

A "forks only" module (a fork sub-assembly destined to be mounted on whatever moves it — a shuttle satellite, a transfer car, an AGV) is just the `forkExtender` carved out as its own factory. It returns the `TransformNode` so the caller's parent assembly can mount it wherever:

```typescript
export class ForksFactoryService {
  public create(options: ForkOptions): TransformNode {
    const forks = new TransformNode(`forks.${this.systemId}`, this.scene);
    this.createForkCarriage(forks, options);
    this.createForkPlate(forks, options);
    this.consolidateByMaterial(forks);
    return forks;
  }
}
```

Caller mounts it under their own animation handle:

```typescript
const forks = forksFactory.create({ length: 1200 });
forks.setParent(myShuttle.satelliteExtender);  // forks now ride the satellite extender
forks.position = mountPoint;
```

The pattern composes: forks-only is the inner layer; lifts and transfer cars wrap it in their own movement handles.

## Two-Layer Merging: mergeIntoStatic vs. consolidateByMaterial

Parametric factories benefit from two distinct merging passes, each operating at a different layer.

### Layer 1 — `mergeIntoStatic` inside a helper

Use this when a single helper builds a known-static cluster of primitives that share one material: four bolt heads, eight wheel-bank tires, twelve corner posts, every panel of a sealed body shell.

```typescript
/**
 * Merges a list of known-static meshes that share one material into a single
 * mesh under `parent`. Source meshes are disposed. `MergeMeshes` bakes each
 * source's world transform into the merged vertices and returns a mesh with
 * no parent — `setParent` re-attaches under the group node while compensating
 * local transform so world position is unchanged.
 */
private mergeIntoStatic(
  parent: TransformNode,
  name: string,
  meshes: Mesh[],
  material: MaterialKey,
): void {
  if (meshes.length === 0) return;
  const merged = Mesh.MergeMeshes(meshes, true, true);
  if (!merged) return;
  merged.name = name;
  merged.setParent(parent);
  merged.material = this.materialFactory.getMaterial(material);
  merged.isPickable = false;
  merged.receiveShadows = true;
}
```

Used like:

```typescript
private createCornerPosts(parent: TransformNode, dims: Dims): void {
  const postMeshes: Mesh[] = [];
  for (const [xSign, zSign] of [[-1,-1],[+1,-1],[-1,+1],[+1,+1]] as const) {
    postMeshes.push(this.createCylinder(parent, "corner_post", { /* ... */ }));
  }
  this.mergeIntoStatic(parent, `corner_posts.${this.systemId}`, postMeshes, "steel");
}
```

Why do this here, not in the final pass? Two reasons:

1. **Locality of intent.** The helper *knows* these four cylinders are corner posts and will never animate independently. The final pass can only guess from material grouping.
2. **Cross-material merging when intent is known.** `MergeMeshes` supports `multiMultiMaterials = true` so you can pre-merge tires + hubs of one wheel even though they use different materials, then let the final pass consolidate further by material across the rest of the template.

### Layer 2 — `consolidateByMaterial` final pass

Run once at the end of `create()`, after every helper has placed and pre-merged its parts. This pass walks the surviving children, groups by **material reference** (not name), and merges per material — collapsing parts that happen to share materials across helpers (body + bands both aluminium, deck + underframe both dark steel).

The rules:

- **Group by material identity, not name.** Two materials with the same name but different instances are different draw calls — that's exactly the bug a shared `MaterialFactory` prevents.
- **Skip `StandardMaterial` and PBR alpha-blend.** Decals and transparent planes need their own depth sort; merging them breaks transparency ordering.
- **Skip `TransformNode`s flagged `metadata.animatable = true`** and do not recurse into them. Their meshes stay where they are.
- **Skip single-mesh groups** (`meshes.length < 2`) — merging one mesh with itself wastes a buffer copy with no benefit.

(The implementation lives in the composite worked example above.)

### When to run which

| Pass | Where | Inputs | Why |
|---|---|---|---|
| `mergeIntoStatic` | Inside a helper | Specific list of known-static meshes | Helper knows these will never animate independently |
| `consolidateByMaterial` | End of `create()` | Whole hierarchy except animation handles | Catches incidental cross-helper material overlap |

### Order matters

```typescript
public create(options: ChassisOptions = {}): ChassisHandles {
  const dims = this.deriveDims(options);
  const root = new TransformNode(/* ... */);

  this.createBody(root, dims);          // calls mergeIntoStatic internally
  this.createDeck(root, dims);          // calls mergeIntoStatic internally
  const liftDeck = this.createLiftDeck(root, dims);   // animatable handle

  this.consolidateByMaterial(root);     // final pass; skips liftDeck

  // Decals AFTER consolidation. StandardMaterial would be skipped anyway,
  // but running them after is safer — they cannot be accidentally pulled
  // into an earlier merge by a future code change.
  this.createBodyDecals(root, dims);

  return { root, liftDeck, /* ... */ };
}
```

## Composition over CSG

For overlapping geometry, **parent overlapping primitives under a `TransformNode`**. Do not reach for CSG (`Mesh.CreateFromCSG` or similar boolean operations).

Why:
- CSG output is one merged mesh that cannot be thin-instanced cleanly.
- CSG is expensive at build time (n² edge intersections).
- Visual seams between two intersecting primitives are invisible to the camera as long as one is fully inside the other.

When CSG **is** the right call:
- A hole is genuinely cut through a part (slot, pin hole) and the camera sees through it.
- The user-facing silhouette demands clean boolean geometry (engineering drawing, exact bounding box).

For everything else: overlap primitives, share materials, let `consolidateByMaterial` collapse the draw calls.

## Pattern Summary

| Concept | Pattern |
|---|---|
| Class shape | Stateless service. `constructor(scene, systemId, materialFactory)`. |
| Entry point | `create(options): TransformNode` (composite) or `build(options): Mesh` (single template). |
| Defaults | `static readonly DEFAULT_*` constants. `options.x ?? DEFAULT_X` inside `deriveDims`. |
| Proportions | One frozen `PROPORTIONS` table. Grouped by part. Fractions named `xFrac`. |
| Derived dims | `deriveDims(options): Dims`, threaded through every helper. |
| Frame | Documented in class JSDoc. Y up. User axes mapped to local axes. |
| Materials | Shared `MaterialFactory`. Scene-wide named cache (`my-material-${key}`). |
| Cross-sections | `ExtrudePolygon` + `earcut` for non-box shapes. Lift + bake to anchor at Y = 0. |
| Caching | `ensureTemplate(slot, cacheKey, build)`. `cacheKey` includes every geometry-affecting input. |
| Thin instances | `UNIT_TEMPLATE_LENGTH = 1000`. Template is `setEnabled(false)`. Caller scales by `mm / 1000`. |
| Composition | Overlap primitives under a `TransformNode`. Avoid CSG. |
| Animatable children | Each movable sub-assembly is its own named `TransformNode`. Children parented under it ride along when the node is animated. |
| Stacked handles | Nest handles for multi-axis motion (`lift_carrier → fork_extender → ...`). Each layer owns one axis; child motion composes with parent motion automatically. |
| Pivot points | `node.setPivotPoint(local)` on the animation handle, *before* parenting children. |
| Returning handles | Factory returns a typed `Handles` object (`{ root, liftCarrier, forkExtender, ... }`) so callers don't depend on string lookups. |
| Cloning movables | `instantiateHierarchy` preserves handles per clone. Thin instances **flatten** the hierarchy — use only for fully-static templates. |
| Intra-group merging | `mergeIntoStatic(parent, name, meshes, material)` inside helpers — for known-static clusters that share one material. |
| Final consolidation | `consolidateByMaterial(root, skip)` once *per movement scope*. Skip list lists animation handles. Run again on each handle as its own root. |
| Decals | Add AFTER consolidation. `StandardMaterial` keeps its own depth sort. |
