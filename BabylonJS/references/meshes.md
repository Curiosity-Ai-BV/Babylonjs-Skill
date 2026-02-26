# Babylon.js Meshes

## Table of Contents
- [Mesh Builders (Set Shapes)](#mesh-builders-set-shapes)
- [Parametric Shapes](#parametric-shapes)
- [Mesh Transforms](#mesh-transforms)
- [TransformNode](#transformnode)
- [Instances](#instances)
- [Thin Instances](#thin-instances)
- [Clones](#clones)
- [Mesh Merging](#mesh-merging)
- [Picking & Raycasting](#picking--raycasting)
- [Bounding Info](#bounding-info)

## Mesh Builders (Set Shapes)

All builders: `MeshBuilder.Create<Shape>(name, options, scene)`.

```typescript
import { MeshBuilder } from "@babylonjs/core/Meshes/meshBuilder";
// Or import individual builders for tree-shaking:
import { CreateBox } from "@babylonjs/core/Meshes/Builders/boxBuilder";
import { CreateSphere } from "@babylonjs/core/Meshes/Builders/sphereBuilder";
import { CreateCylinder } from "@babylonjs/core/Meshes/Builders/cylinderBuilder";
import { CreatePlane } from "@babylonjs/core/Meshes/Builders/planeBuilder";
import { CreateGround } from "@babylonjs/core/Meshes/Builders/groundBuilder";
import { CreateTube } from "@babylonjs/core/Meshes/Builders/tubeBuilder";
import { CreateDisc } from "@babylonjs/core/Meshes/Builders/discBuilder";
import { CreateTorus } from "@babylonjs/core/Meshes/Builders/torusBuilder";
import { CreatePolygon } from "@babylonjs/core/Meshes/Builders/polygonBuilder";

// Box
CreateBox("box", { width: 2, height: 1, depth: 3 }, scene);

// Sphere
CreateSphere("sphere", { diameter: 2, segments: 32 }, scene);

// Cylinder / Cone
CreateCylinder("cyl", { height: 3, diameterTop: 1, diameterBottom: 2, tessellation: 24 }, scene);

// Plane
CreatePlane("plane", { width: 5, height: 3, sideOrientation: Mesh.DOUBLESIDE }, scene);

// Ground
CreateGround("ground", { width: 10, height: 10, subdivisions: 4 }, scene);

// Tube (path-based)
CreateTube("tube", { path: [v1, v2, v3], radius: 0.5, tessellation: 16 }, scene);

// Disc
CreateDisc("disc", { radius: 2, tessellation: 64 }, scene);

// Polygon (requires earcut)
CreatePolygon("polygon", { shape: [v1, v2, v3, v4], depth: 1 }, scene, earcut);

// Capsule
MeshBuilder.CreateCapsule("capsule", { height: 2, radius: 0.5 }, scene);

// Torus
CreateTorus("torus", { diameter: 2, thickness: 0.5, tessellation: 32 }, scene);
```

Common options on all builders:
- `updatable: boolean` - allow vertex data updates after creation
- `sideOrientation` - `Mesh.FRONTSIDE` (default), `Mesh.BACKSIDE`, `Mesh.DOUBLESIDE`
- `faceUV: Vector4[]` - per-face UV mapping (box: 6 faces)
- `faceColors: Color4[]` - per-face colors

## Parametric Shapes

```typescript
import { CreateLines } from "@babylonjs/core/Meshes/Builders/linesBuilder";
import { CreateRibbon } from "@babylonjs/core/Meshes/Builders/ribbonBuilder";
import { CreateLathe } from "@babylonjs/core/Meshes/Builders/latheBuilder";

// Lines
CreateLines("lines", { points: [v1, v2, v3], updatable: true }, scene);

// Dashed Lines
MeshBuilder.CreateDashedLines("dashed", { points: [v1, v2], dashSize: 3, gapSize: 1 }, scene);

// Ribbon (from path arrays)
CreateRibbon("ribbon", { pathArray: [path1, path2], closePath: false }, scene);

// Lathe (revolution of shape profile)
CreateLathe("lathe", { shape: profilePoints, radius: 1, tessellation: 24 }, scene);

// Extrusion
MeshBuilder.ExtrudeShape("extrude", { shape: shapePoints, path: extrusionPath }, scene);
```

## Mesh Transforms

```typescript
// Position
mesh.position = new Vector3(x, y, z);
mesh.position.x = 5;

// Rotation (Euler angles in radians)
mesh.rotation = new Vector3(rx, ry, rz);
mesh.rotation.y = Math.PI / 4;

// Quaternion rotation (overrides .rotation when set)
mesh.rotationQuaternion = Quaternion.FromEulerAngles(rx, ry, rz);

// Scaling
mesh.scaling = new Vector3(sx, sy, sz);

// Bake current transform into vertices (reset transform to identity)
mesh.bakeCurrentTransformIntoVertices();

// Parent-child transforms
child.parent = parentNode;
// child's position/rotation/scaling become local to parent

// LookAt
mesh.lookAt(targetPosition);

// Pivot point
mesh.setPivotPoint(new Vector3(0, -1, 0));
```

## TransformNode

Use TransformNode for grouping/hierarchy (lighter than Mesh):

```typescript
import { TransformNode } from "@babylonjs/core/Meshes/transformNode";

const group = new TransformNode("group", scene);
mesh1.parent = group;
mesh2.parent = group;
group.position.y = 5; // moves both children
group.rotation.y = Math.PI; // rotates both children
```

## Instances

Instances share geometry + material with source mesh. Separate transform per instance. Single draw call per source mesh.

```typescript
const instance = sourceMesh.createInstance("instance1");
instance.position = new Vector3(5, 0, 0);

// Instances inherit material from source
// To override: not directly supported, use clones instead
```

## Thin Instances

Maximum performance for many identical meshes. No JS objects created per instance - just matrix buffers.

```typescript
import "@babylonjs/core/Meshes/thinInstanceMesh"; // Side-effect import

// Add one at a time
const matrix = Matrix.Translation(x, y, z);
const idx = mesh.thinInstanceAdd(matrix);

// Add self (include original mesh position)
mesh.thinInstanceAddSelf();

// Batch add (much faster for many instances)
const buffer = new Float32Array(16 * count);
for (let i = 0; i < count; i++) {
  Matrix.Translation(x, y, z).copyToArray(buffer, i * 16);
}
mesh.thinInstanceSetBuffer("matrix", buffer, 16);

// Update single instance
mesh.thinInstanceSetMatrixAt(idx, newMatrix);

// Per-instance custom attributes
mesh.thinInstanceRegisterAttribute("color", 4);
mesh.thinInstanceSetAttributeAt("color", idx, [1, 0, 0, 1]);

// Batch set custom attribute buffer
const colorBuffer = new Float32Array(4 * count);
mesh.thinInstanceSetBuffer("color", colorBuffer, 4);

// Control visible count
mesh.thinInstanceCount = 100; // show first 100

// Performance: mark buffer as static or dynamic
mesh.thinInstanceSetBuffer("matrix", buffer, 16, false); // false = updateable
mesh.thinInstanceBufferUpdated("matrix"); // signal GPU update needed

// Enable picking on thin instances
mesh.thinInstanceEnablePicking = true;
```

**Thin instance limitations:**
- All-or-nothing visibility (no per-instance frustum culling)
- Single bounding box for all instances
- Mixed positive/negative determinant matrices need separate meshes
- Cloned meshes need `mesh.makeGeometryUnique()` first

## Clones

Full copy with independent material assignment:

```typescript
const clone = mesh.clone("cloneName");
clone.position.x = 10;
clone.material = differentMaterial; // can override material
```

## Mesh Merging

```typescript
import { Mesh } from "@babylonjs/core/Meshes/mesh";

// Merge multiple meshes into one (reduces draw calls)
const merged = Mesh.MergeMeshes(
  [mesh1, mesh2, mesh3],
  true,   // disposeSource
  true,   // allow32BitsIndices
  undefined, // parent
  false,  // subdivideWithSubMeshes
  true    // multiMultiMaterials
);
```

## Picking & Raycasting

```typescript
// Scene picking
scene.onPointerDown = (evt, pickResult) => {
  if (pickResult.hit) {
    const mesh = pickResult.pickedMesh;
    const point = pickResult.pickedPoint;
    const normal = pickResult.getNormal();
  }
};

// Manual ray
import { Ray } from "@babylonjs/core/Culling/ray";
const ray = new Ray(origin, direction, length);
const hit = scene.pickWithRay(ray);

// Pick with predicate
const hit = scene.pick(pointerX, pointerY, (mesh) => mesh.isPickable);
```

## Bounding Info

```typescript
const bounds = mesh.getBoundingInfo();
const min = bounds.boundingBox.minimumWorld;
const max = bounds.boundingBox.maximumWorld;
const center = bounds.boundingBox.centerWorld;
const radius = bounds.boundingSphere.radiusWorld;

// Show bounding box
mesh.showBoundingBox = true;

// Enable edges
mesh.enableEdgesRendering();
mesh.edgesWidth = 4.0;
mesh.edgesColor = new Color4(0, 0, 1, 1);
```
