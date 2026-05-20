---
name: babylonjs
description: "Babylon.js 8 3D engine development expertise with curated API patterns, code examples, and on-demand documentation access. Use when working with Babylon.js scenes, meshes, materials (PBR/Standard), cameras, lights, shadows, GUI (2D/3D), animations, physics (Havok), thin instances, glTF loading, post-processing, WebXR, procedural/code-built 3D models, or any 3D rendering task. Covers: (1) Scene setup and engine initialization (WebGL/WebGPU), (2) Mesh creation, transforms, instancing, and merging, (3) PBR and Standard materials with textures, (4) Camera types (ArcRotate, Universal, Follow), (5) Lighting and shadows, (6) 2D/3D GUI with AdvancedDynamicTexture, (7) Animation system and animation groups, (8) Asset loading (glTF, OBJ, STL), (9) Performance optimization and profiling, (10) ASRS/warehouse digital twin visualization patterns, (11) Procedural modeling: building high-quality 3D models from primitives with animation-ready parent/pivot hierarchies, CSG2 booleans, lathes, extrudes, and custom VertexData."
---

# Babylon.js 8

## Quick Reference

Babylon.js is a powerful open-source 3D engine for the web. Version 8 supports WebGL2 and WebGPU.

**Coordinate system:** Left-handed (X=right, Y=up, Z=forward). Rotations in radians.

**NPM packages:**
- `@babylonjs/core` - Engine, scene, meshes, materials, cameras, lights
- `@babylonjs/gui` - 2D/3D GUI controls
- `@babylonjs/loaders` - glTF, OBJ, STL loaders
- `@babylonjs/materials` - Extra material types
- `@babylonjs/inspector` - Debug inspector

**Tree-shaking:** Import from deep paths for minimal bundles:
```typescript
import { Scene } from "@babylonjs/core/scene";
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import { Color3, Color4 } from "@babylonjs/core/Maths/math.color";
import { Mesh } from "@babylonjs/core/Meshes/mesh";
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import { ArcRotateCamera } from "@babylonjs/core/Cameras/arcRotateCamera";
```

**Side-effect imports** (enable features without referencing exports):
```typescript
import "@babylonjs/core/Meshes/thinInstanceMesh";
import "@babylonjs/core/Rendering/edgesRenderer";
import "@babylonjs/core/Collisions/collisionCoordinator";
import "@babylonjs/loaders/glTF/2.0/glTFLoader";
```

## Minimal Scene Setup

```typescript
import { Engine } from "@babylonjs/core/Engines/engine";
import { Scene } from "@babylonjs/core/scene";
import { ArcRotateCamera } from "@babylonjs/core/Cameras/arcRotateCamera";
import { HemisphericLight } from "@babylonjs/core/Lights/hemisphericLight";
import { Vector3 } from "@babylonjs/core/Maths/math.vector";
import { CreateSphere } from "@babylonjs/core/Meshes/Builders/sphereBuilder";
import { CreateGround } from "@babylonjs/core/Meshes/Builders/groundBuilder";

const engine = new Engine(canvas, true);
const scene = new Scene(engine);
const camera = new ArcRotateCamera("cam", 0, Math.PI/4, 10, Vector3.Zero(), scene);
camera.attachControl(canvas, true);
const light = new HemisphericLight("light", new Vector3(0, 1, 0), scene);
const sphere = CreateSphere("sphere", { diameter: 2 }, scene);
const ground = CreateGround("ground", { width: 6, height: 6 }, scene);
sphere.position.y = 1;

engine.runRenderLoop(() => scene.render());
window.addEventListener("resize", () => engine.resize());
```

## Key Patterns

### PBR Material (most common for realistic rendering)
```typescript
const pbr = new PBRMaterial("pbr", scene);
pbr.albedoColor = new Color3(1.0, 0.766, 0.336);
pbr.metallic = 0.3;
pbr.roughness = 0.7;
// Optional: connect to environment for reflections
pbr.reflectionTexture = scene.environmentTexture;
mesh.material = pbr;
```

### Thin Instances (high-performance batching)
```typescript
import "@babylonjs/core/Meshes/thinInstanceMesh";
const buffer = new Float32Array(16 * count);
for (let i = 0; i < count; i++) {
  Matrix.Translation(x, y, z).copyToArray(buffer, i * 16);
}
mesh.thinInstanceSetBuffer("matrix", buffer, 16, false);
mesh.thinInstanceBufferUpdated("matrix");
```

### Asset Loading (glTF)
```typescript
import "@babylonjs/loaders/glTF/2.0/glTFLoader";
const container = await BABYLON.LoadAssetContainerAsync("model.glb", scene);
container.addAllToScene();
```

### Observable Pattern (Babylon's event system)
```typescript
scene.onBeforeRenderObservable.add(() => { /* per frame */ });
const observer = scene.onPointerObservable.add((info) => { /* pointer events */ });
scene.onPointerObservable.remove(observer); // unsubscribe
```

### Procedural Modeling — Animation-Ready Hierarchy
Build complex models from primitives by placing a `TransformNode` at every joint *before* attaching geometry. One pivot per degree of freedom; geometry's local position is its offset from the joint.
```typescript
const robotRoot = new TransformNode("robotRoot", scene);
const shoulderPivot = new TransformNode("shoulderPivot", scene);
shoulderPivot.parent = robotRoot;                  // pivot lives at the joint axis
const upperArm = CreateCapsule("upperArm", { height: 1.2, radius: 0.16 }, scene);
upperArm.parent = shoulderPivot;
upperArm.position.y = 0.6;                          // geometry sticks out from joint
// Animation later: shoulderPivot.rotation.y = ...  // rotates whole arm cleanly
```
For booleans (drilled holes, carved slots), use **CSG2** (Babylon 8). For rotationally symmetric parts (bottles, hubs, columns) use `CreateLathe`. For path-swept parts (cables, rails) use `CreateTube` or `ExtrudeShape`. See [procedural-parametric-modeling.md](references/procedural-parametric-modeling

## Reference Files

Read these files for detailed API patterns on specific topics:

- **[core-concepts.md](references/core-concepts.md)** - Engine/Scene setup, cameras, lights, shadows, observables, coordinate system
- **[meshes.md](references/meshes.md)** - Mesh builders, transforms, TransformNode, instances, thin instances, clones, merging, picking
- **[procedural-parametric-modeling.md](references/procedural-parametric-modeling.md)** - Building high-quality 3D models in code from primitives. Full primitive catalog (lathe, tube, extrude, polyhedra, geodesics, CSG2 booleans, custom VertexData), quality techniques (tessellation, bevels, edges, material palettes), and a complete animation-ready parent/pivot hierarchy pattern with a worked robot-arm example. **Read this when asked to build any non-trivial model in code or when a model will need to animate.**
- **[materials.md](references/materials.md)** - PBR, Standard, textures, environment/HDR, Node Material, Shader Material
- **[gui.md](references/gui.md)** - AdvancedDynamicTexture, all control types, containers, layout, events
- **[animation-loading.md](references/animation-loading.md)** - Animation API, groups, easing, skeletal animation, asset loading, AssetContainer
- **[performance.md](references/performance.md)** - Scene/mesh/material optimization, instancing strategy comparison, monitoring, memory management

## On-Demand Documentation

For topics not covered in the reference files, fetch from the live docs:

- **Doc site:** `https://doc.babylonjs.com`
- **GitHub raw markdown:** `https://raw.githubusercontent.com/BabylonJS/Documentation/master/content/{path}.md`
- **[doc-urls.md](references/doc-urls.md)** - Complete URL map for all doc sections

To fetch a specific topic, read the URL map to find the path, then use WebFetch on the doc site URL or fetch raw markdown from GitHub.

### Topics covered only in live docs (not in reference files)
- Physics V2 (Havok): `/features/featuresDeepDive/physics`
- WebXR/VR/AR: `/features/featuresDeepDive/webXR`
- Post-processing pipelines: `/features/featuresDeepDive/postProcesses`
- Solid Particle System: `/features/featuresDeepDive/particles/solid_particle_system`
- Crowd navigation: `/features/featuresDeepDive/crowdNavigation`
- Node Geometry (procedural): `/features/featuresDeepDive/mesh/nodeGeometry`
- Frame Graph: `/features/featuresDeepDive/frameGraph`
- Flow Graph: `/features/featuresDeepDive/flowGraph`
- Smart Filters: `/features/featuresDeepDive/smartFilters`
- Gaussian Splatting: `/features/featuresDeepDive/importers` (splat/ply formats)

## Common Gotchas

1. **Quaternion vs Euler:** Setting `mesh.rotationQuaternion` disables `mesh.rotation`. To switch back: `mesh.rotationQuaternion = null`.
2. **Side-effect imports:** Features like thin instances, edge rendering, loaders, and collisions require importing their module even if you don't use the export directly.
3. **Dispose everything:** Babylon.js doesn't garbage-collect GPU resources. Always call `.dispose()` on meshes, materials, textures when done.
4. **Material sharing:** Modifying a shared material affects all meshes using it. Clone or create new materials for independent changes.
5. **TransformNode vs Mesh:** Use TransformNode for grouping/hierarchy. Empty Mesh objects waste CPU on frustum evaluation.
6. **Left-handed coordinates:** glTF is right-handed; Babylon auto-converts on import. Manual coordinate math may need adjustment.
7. **Alpha sorting:** Transparent meshes require proper render ordering. Use `mesh.alphaIndex` or rendering groups.
8. **Thin instance limitations:** All-or-nothing visibility, single bounding box, no per-instance material. Use regular instances when individual control is needed.
9. **Pivot before geometry:** When code-building animated models, create and parent the joint's `TransformNode` first, then attach geometry as a child with a local offset. Retro-fitting pivots after `MergeMeshes` is painful — merge only after the hierarchy works.
10. **CSG2 setup:** Babylon 8's `CSG2` (boolean ops backed by Manifold) requires `await InitializeCSG2Async()` once before use and operates in world space. Always dispose intermediate `CSG2.FromMesh` objects — they hold WASM memory.
11. **Negative scaling flips winding:** Mirroring a hierarchy via `scaling.x *= -1` inverts triangle winding and breaks backface culling. Fix with `material.sideOrientation = Mesh.DOUBLESIDE` or rebake.
