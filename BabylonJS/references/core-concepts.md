# Babylon.js Core Concepts

## Table of Contents
- [Engine & Scene Setup](#engine--scene-setup)
- [Cameras](#cameras)
- [Lights](#lights)
- [Observable Pattern](#observable-pattern)
- [Scene Lifecycle](#scene-lifecycle)
- [Coordinate System](#coordinate-system)

## Engine & Scene Setup

```typescript
import { Engine } from "@babylonjs/core/Engines/engine";
import { Scene } from "@babylonjs/core/scene";

// Standard WebGL engine
const engine = new Engine(canvas, true /* antialias */);
const scene = new Scene(engine);

// Optimized scene constructor for large scenes
const scene = new Scene(engine, {
  useGeometryUniqueIdsMap: true,
  useMaterialMeshMap: true,
  useClonedMeshMap: true,
});

// Render loop
engine.runRenderLoop(() => scene.render());
window.addEventListener("resize", () => engine.resize());

// WebGPU engine (Babylon.js 6+)
import { WebGPUEngine } from "@babylonjs/core/Engines/webgpuEngine";
const engine = new WebGPUEngine(canvas);
await engine.initAsync();
```

## Cameras

### ArcRotateCamera (most common for 3D viewers)
```typescript
import { ArcRotateCamera } from "@babylonjs/core/Cameras/arcRotateCamera";
import { Vector3 } from "@babylonjs/core/Maths/math.vector";

// alpha=longitude, beta=latitude, radius=distance, target
const camera = new ArcRotateCamera("cam", 0, Math.PI/4, 10, Vector3.Zero(), scene);
camera.attachControl(canvas, true);

// Limits
camera.lowerBetaLimit = 0.1;
camera.upperBetaLimit = Math.PI / 2;
camera.lowerRadiusLimit = 5;
camera.upperRadiusLimit = 50;
camera.wheelDeltaPercentage = 0.01;
camera.zoomToMouseLocation = true;

// Set position directly (overrides alpha/beta/radius)
camera.setPosition(new Vector3(10, 10, 10));
```

### UniversalCamera (FPS-style)
```typescript
import { UniversalCamera } from "@babylonjs/core/Cameras/universalCamera";
const camera = new UniversalCamera("cam", new Vector3(0, 5, -10), scene);
camera.setTarget(Vector3.Zero());
camera.attachControl(canvas, true);
```

### FollowCamera
```typescript
import { FollowCamera } from "@babylonjs/core/Cameras/followCamera";
const camera = new FollowCamera("cam", new Vector3(0, 10, -10), scene);
camera.lockedTarget = targetMesh;
camera.radius = 30;
camera.heightOffset = 10;
camera.rotationOffset = 0;
camera.cameraAcceleration = 0.005;
camera.maxCameraSpeed = 10;
```

## Lights

### Light Types
```typescript
import { HemisphericLight } from "@babylonjs/core/Lights/hemisphericLight";
import { PointLight } from "@babylonjs/core/Lights/pointLight";
import { DirectionalLight } from "@babylonjs/core/Lights/directionalLight";
import { SpotLight } from "@babylonjs/core/Lights/spotLight";

// Ambient-like light
const hemi = new HemisphericLight("hemi", new Vector3(0, 1, 0), scene);
hemi.intensity = 0.7;
hemi.groundColor = new Color3(0.2, 0.2, 0.2);

// Point light (omni-directional)
const point = new PointLight("point", new Vector3(0, 10, 0), scene);
point.intensity = 0.8;
point.range = 100;

// Directional light (sun-like)
const dir = new DirectionalLight("dir", new Vector3(-1, -2, -1), scene);

// Spot light (cone)
const spot = new SpotLight("spot", new Vector3(0, 30, -10),
  new Vector3(0, -1, 0), Math.PI / 3, 2, scene);
```

### Shadow Setup
```typescript
import { ShadowGenerator } from "@babylonjs/core/Lights/Shadows/shadowGenerator";

const shadowGen = new ShadowGenerator(1024, directionalLight);
shadowGen.addShadowCaster(mesh, true);
shadowGen.useBlurExponentialShadowMap = true;
shadowGen.blurScale = 2;

ground.receiveShadows = true;
```

### Light Limits
```typescript
// Default max 4 lights per material
material.maxSimultaneousLights = 6; // increase if needed
```

## Observable Pattern

Babylon.js uses Observables (not DOM events) for its event system:

```typescript
import { Observable } from "@babylonjs/core/Misc/observable";

// Subscribe
const observer = observable.add((eventData) => { /* handler */ });

// Unsubscribe
observable.remove(observer);
// or
observable.removeCallback(callbackFn);

// Common scene observables
scene.onBeforeRenderObservable.add(() => { /* each frame before render */ });
scene.onAfterRenderObservable.add(() => { /* each frame after render */ });
scene.onPointerObservable.add((pointerInfo) => { /* pointer events */ });
scene.onKeyboardObservable.add((kbInfo) => { /* keyboard events */ });

// Pointer event types
import { PointerEventTypes } from "@babylonjs/core/Events/pointerEvents";
scene.onPointerObservable.add((pointerInfo) => {
  switch (pointerInfo.type) {
    case PointerEventTypes.POINTERDOWN: break;
    case PointerEventTypes.POINTERUP: break;
    case PointerEventTypes.POINTERMOVE: break;
    case PointerEventTypes.POINTERPICK: break;
    case PointerEventTypes.POINTERTAP: break;
  }
});
```

## Scene Lifecycle

```
Engine created → Scene created → Assets loaded → Render loop starts
                                                       ↓
                                              onBeforeRender
                                                       ↓
                                              Active meshes evaluated
                                                       ↓
                                              onAfterRender
                                                       ↓
                                              Frame displayed
```

Key methods:
```typescript
scene.render();                    // Render one frame
scene.dispose();                   // Clean up everything
scene.getEngine();                 // Get engine reference
scene.getMeshByName("name");       // Find mesh
scene.getMaterialByName("name");   // Find material
scene.getMeshById("id");           // Find by unique ID
```

## Coordinate System

Babylon.js uses a **left-handed** coordinate system:
- **X** = right
- **Y** = up
- **Z** = forward (into screen)

Rotation is in **radians**. Use `BABYLON.Tools.ToRadians(degrees)` for conversion.

Units are arbitrary - choose a convention (mm, cm, m) and stay consistent. glTF uses meters.
