# Babylon.js Animation & Asset Loading

## Table of Contents
- [Animation Basics](#animation-basics)
- [Animation Groups](#animation-groups)
- [Easing Functions](#easing-functions)
- [Skeletal Animation](#skeletal-animation)
- [Asset Loading](#asset-loading)
- [AssetContainer](#assetcontainer)
- [Scene Serialization](#scene-serialization)

## Animation Basics

```typescript
import { Animation } from "@babylonjs/core/Animations/animation";

// Create animation
const anim = new Animation(
  "moveX",                              // name
  "position.x",                         // target property (dot notation)
  30,                                   // frames per second
  Animation.ANIMATIONTYPE_FLOAT,        // value type
  Animation.ANIMATIONLOOPMODE_CYCLE     // loop mode
);

// Animation types
Animation.ANIMATIONTYPE_FLOAT;       // single number
Animation.ANIMATIONTYPE_VECTOR3;     // Vector3
Animation.ANIMATIONTYPE_VECTOR2;     // Vector2
Animation.ANIMATIONTYPE_COLOR3;      // Color3
Animation.ANIMATIONTYPE_COLOR4;      // Color4
Animation.ANIMATIONTYPE_QUATERNION;  // Quaternion
Animation.ANIMATIONTYPE_MATRIX;      // Matrix
Animation.ANIMATIONTYPE_SIZE;        // Size

// Loop modes
Animation.ANIMATIONLOOPMODE_RELATIVE; // incremental
Animation.ANIMATIONLOOPMODE_CYCLE;    // restart from beginning
Animation.ANIMATIONLOOPMODE_CONSTANT; // stop at last frame

// Define keyframes
const keys = [
  { frame: 0, value: 0 },
  { frame: 30, value: 5 },
  { frame: 60, value: 0 },
];
anim.setKeys(keys);

// Attach and play
mesh.animations.push(anim);
const animatable = scene.beginAnimation(
  mesh,    // target
  0,       // from frame
  60,      // to frame
  true,    // loop
  1.0,     // speed ratio
  () => {} // onAnimationEnd callback
);

// Control playback
animatable.pause();
animatable.restart();
animatable.stop();
animatable.speedRatio = 2.0;
animatable.goToFrame(30);
```

### Animatable Properties (dot notation)
- `position.x`, `position.y`, `position.z`
- `rotation.x`, `rotation.y`, `rotation.z`
- `scaling.x`, `scaling.y`, `scaling.z`
- `material.alpha`
- `material.diffuseColor`
- Any numeric/vector/color property

## Animation Groups

Group multiple animations for synchronized control:

```typescript
import { AnimationGroup } from "@babylonjs/core/Animations/animationGroup";

const group = new AnimationGroup("group", scene);
group.addTargetedAnimation(anim1, mesh1);
group.addTargetedAnimation(anim2, mesh2);

group.play(true);  // true = loop
group.pause();
group.stop();
group.reset();
group.speedRatio = 1.5;
group.goToFrame(30);

// Events
group.onAnimationEndObservable.add(() => {});
group.onAnimationGroupPlayObservable.add(() => {});

// Normalize frame ranges
group.normalize(0, 100);

// Weight blending (0-1)
group.setWeightForAllAnimatables(0.5);
```

## Easing Functions

```typescript
import { CubicEase, EasingFunction } from "@babylonjs/core/Animations/easing";

const ease = new CubicEase();
ease.setEasingMode(EasingFunction.EASINGMODE_EASEINOUT);
anim.setEasingFunction(ease);

// Available easing functions:
// CircleEase, BackEase, BounceEase, CubicEase, ElasticEase,
// ExponentialEase, PowerEase, QuadraticEase, QuarticEase,
// QuinticEase, SineEase, BezierCurveEase

// Easing modes:
// EASINGMODE_EASEIN, EASINGMODE_EASEOUT, EASINGMODE_EASEINOUT
```

## Skeletal Animation

glTF models often include skeletal animations loaded as AnimationGroups:

```typescript
const result = await ImportMeshAsync("model.glb", scene);
const animGroups = result.animationGroups;

// Play specific animation
animGroups[0].play(true);  // e.g., "idle"
animGroups[1].play(true);  // e.g., "walk"

// Stop all
animGroups.forEach(g => g.stop());

// Blend between animations using weights
animGroups[0].setWeightForAllAnimatables(0.5);
animGroups[1].setWeightForAllAnimatables(0.5);
```

## Asset Loading

### Modern API (Babylon.js 6+)

```typescript
// Load and add to scene
await BABYLON.AppendSceneAsync("model.glb", scene);

// Load into container (inspect before adding)
const container = await BABYLON.LoadAssetContainerAsync("model.glb", scene);
container.addAllToScene();

// Load meshes only
const result = await BABYLON.ImportMeshAsync("model.glb", scene);
// result.meshes, result.skeletons, result.animationGroups, etc.

// Load from URL with root
const container = await BABYLON.LoadAssetContainerAsync(
  "https://example.com/models/model.glb",
  scene
);

// Load from base64
await BABYLON.AppendSceneAsync("data:;base64,ENCODED_DATA", scene);
```

### Loader Registration (for tree-shaking)

```typescript
// Dynamic loading (recommended for bundle size)
import { registerBuiltInLoaders } from "@babylonjs/loaders/dynamic";
registerBuiltInLoaders();

// Or import specific loaders
import "@babylonjs/loaders/glTF/2.0/glTFLoader";
import "@babylonjs/loaders/glTF/2.0/Extensions/KHR_draco_mesh_compression";
import "@babylonjs/loaders/glTF/2.0/Extensions/KHR_texture_transform";
```

### Supported Formats
- `.gltf` / `.glb` - glTF 2.0 (recommended)
- `.babylon` - native format
- `.obj` - Wavefront OBJ
- `.stl` - STL (3D printing)
- `.splat` / `.ply` - Gaussian Splatting

### Loading Events

```typescript
import { SceneLoader } from "@babylonjs/core/Loading/sceneLoader";

SceneLoader.OnPluginActivatedObservable.add((plugin) => {
  // Configure loader plugin
});

// Progress callback
const container = await BABYLON.LoadAssetContainerAsync("model.glb", scene, {
  onProgress: (event) => {
    const pct = event.lengthComputable ? (event.loaded / event.total * 100) : 0;
  }
});
```

## AssetContainer

```typescript
const container = await BABYLON.LoadAssetContainerAsync("model.glb", scene);

// Inspect contents
container.meshes;          // AbstractMesh[]
container.materials;       // Material[]
container.textures;        // BaseTexture[]
container.animationGroups; // AnimationGroup[]
container.skeletons;       // Skeleton[]
container.lights;          // Light[]
container.transformNodes;  // TransformNode[]

// Add/remove from scene
container.addAllToScene();
container.removeAllFromScene();

// Instantiate multiple copies
const instances = container.instantiateModelsToScene(
  (name) => name + "_copy",  // name function
  false                       // clone materials
);
instances.rootNodes;          // root transform nodes
instances.animationGroups;    // cloned animation groups

// Dispose
container.dispose();
```

## Scene Serialization

```typescript
import { SceneSerializer } from "@babylonjs/core/Misc/sceneSerializer";

// Serialize entire scene
const json = SceneSerializer.Serialize(scene);

// Serialize single mesh
const meshJson = SceneSerializer.SerializeMesh(mesh);

// Load from serialized data
BABYLON.SceneLoader.Load("", "data:" + JSON.stringify(json), engine);
```
