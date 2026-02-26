# Babylon.js Performance Optimization

## Table of Contents
- [Scene-Level Optimizations](#scene-level-optimizations)
- [Mesh Optimizations](#mesh-optimizations)
- [Material Optimizations](#material-optimizations)
- [Rendering Optimizations](#rendering-optimizations)
- [Instancing Strategy](#instancing-strategy)
- [Performance Monitoring](#performance-monitoring)
- [Performance Priority Modes](#performance-priority-modes)
- [Memory Management](#memory-management)

## Scene-Level Optimizations

```typescript
// Skip pointer movement picking (big CPU win if not needed)
scene.skipPointerMovePicking = true;

// Disable auto-clear when viewport is fully covered
scene.autoClear = false;
scene.autoClearDepthAndStencil = false;

// Freeze active meshes list (static scenes)
scene.freezeActiveMeshes();
scene.unfreezeActiveMeshes(); // re-enable

// Block dirty mechanism during batch operations
scene.blockMaterialDirtyMechanism = true;
// ... batch changes ...
scene.blockMaterialDirtyMechanism = false;

// Block free active meshes during batch dispose
scene.blockfreeActiveMeshesAndRenderingGroups = true;
// ... dispose meshes ...
scene.blockfreeActiveMeshesAndRenderingGroups = false;

// Optimized scene constructor
const scene = new Scene(engine, {
  useGeometryUniqueIdsMap: true,
  useMaterialMeshMap: true,
  useClonedMeshMap: true,
});
```

## Mesh Optimizations

```typescript
// Use TransformNode for non-renderable containers (not empty Mesh!)
const group = new TransformNode("group", scene);

// Freeze world matrix for static meshes
mesh.freezeWorldMatrix();

// Disable bounding info sync for static meshes
mesh.doNotSyncBoundingInfo = true;
mesh.alwaysSelectAsActiveMesh = true;  // pair with above

// Convert to unindexed mesh (when vertex reuse is low)
mesh.convertToUnIndexedMesh();

// Culling strategy
mesh.cullingStrategy = AbstractMesh.CULLINGSTRATEGY_BOUNDINGSPHERE_ONLY;
// Options: STANDARD, BOUNDINGSPHERE_ONLY,
//          OPTIMISTIC_INCLUSION, OPTIMISTIC_INCLUSION_THEN_BSPHERE_ONLY

// Bake transform to avoid matrix recalculation
mesh.bakeCurrentTransformIntoVertices();
```

## Material Optimizations

```typescript
// Freeze static materials
material.freeze();

// Depth pre-pass for complex shaders
material.needDepthPrePass = true;

// Limit simultaneous lights
material.maxSimultaneousLights = 4; // default is 4

// Use texture LOD
texture.lodGenerationScale = 0.5;

// Prefer PBRMaterial over StandardMaterial for quality/perf ratio
```

## Rendering Optimizations

```typescript
// Rendering groups (control draw order)
mesh.renderingGroupId = 0; // 0-3, rendered in order

// Disable auto-clear for specific rendering groups
scene.setRenderingAutoClearDepthStencil(renderingGroupIdx, false, false, false);

// Edge rendering (expensive - use sparingly)
mesh.enableEdgesRendering();
mesh.edgesWidth = 4.0;
mesh.edgesColor = new Color4(0, 0, 0, 1);

// Rendering order optimization
scene.setRenderingOrder(renderingGroupId, opaqueSortFn, alphaTestSortFn, transparentSortFn);
```

## Instancing Strategy

Choose the right approach based on needs:

| Approach | Draw Calls | JS Objects | Per-Instance Material | Individual Culling | Best For |
|---|---|---|---|---|---|
| **Thin Instances** | 1 per source | None | No (custom attrs only) | No | Thousands of identical meshes |
| **Instances** | 1 per source | 1 per instance | No (share source) | Yes | Hundreds with individual control |
| **Clones** | 1 per clone | 1 per clone | Yes | Yes | Few copies needing different materials |
| **Mesh.MergeMeshes** | 1 total | 1 total | Multi-material option | No | Static geometry batching |

### Thin Instances Checklist
```typescript
import "@babylonjs/core/Meshes/thinInstanceMesh";

// Batch creation (fastest)
const buffer = new Float32Array(16 * count);
for (let i = 0; i < count; i++) {
  Matrix.Translation(x, y, z).copyToArray(buffer, i * 16);
}
mesh.thinInstanceSetBuffer("matrix", buffer, 16, false); // false = updateable

// Signal update after modifying buffer
mesh.thinInstanceBufferUpdated("matrix");

// Limit visible count
mesh.thinInstanceCount = visibleCount;
```

## Performance Monitoring

```typescript
// Engine instrumentation
import { EngineInstrumentation } from "@babylonjs/core/Instrumentation/engineInstrumentation";
const engineInstr = new EngineInstrumentation(engine);
engineInstr.captureGPUFrameTime = true;
engineInstr.captureShaderCompilationTime = true;

// Scene instrumentation
import { SceneInstrumentation } from "@babylonjs/core/Instrumentation/sceneInstrumentation";
const sceneInstr = new SceneInstrumentation(scene);
sceneInstr.captureFrameTime = true;
sceneInstr.captureActiveMeshesEvaluationTime = true;
sceneInstr.captureRenderTime = true;

// Read metrics
const fps = engine.getFps();
const drawCalls = sceneInstr.drawCallsCounter.current;
const activeMeshes = scene.getActiveMeshes().length;
const totalVertices = scene.getTotalVertices();

// Inspector (debug tool)
import "@babylonjs/inspector";
scene.debugLayer.show();
scene.debugLayer.hide();
```

## Performance Priority Modes

```typescript
// Automatic optimization levels (Babylon.js 5.22+)
scene.performancePriority = ScenePerformancePriority.BackwardCompatible; // default, no changes
scene.performancePriority = ScenePerformancePriority.Intermediate;      // auto-freeze, skip picking
scene.performancePriority = ScenePerformancePriority.Aggressive;        // skip frustum, disable bounds

// Intermediate mode auto-enables:
// - scene.skipPointerMovePicking = true
// - material.freeze() on all materials
// - mesh.doNotSyncBoundingInfo = true
// - mesh.isSkeletonAnimationSheet = true (when applicable)

// Aggressive mode additionally:
// - Skips frustum clipping
// - Disables bounding info updates
```

## Memory Management

```typescript
// Dispose individual mesh
mesh.dispose();

// Dispose material (checks if shared)
material.dispose(true);  // true = force even if shared

// Dispose texture
texture.dispose();

// Dispose entire scene
scene.dispose();

// Engine disposal
engine.dispose();

// Check if disposed
if (!mesh.isDisposed()) { /* safe to use */ }

// Handle WebGL context loss
engine.onContextLostObservable.add(() => { /* pause */ });
engine.onContextRestoredObservable.add(() => { /* rebuild */ });
```
