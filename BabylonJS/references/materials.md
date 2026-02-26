# Babylon.js Materials

## Table of Contents
- [PBRMaterial](#pbrmaterial)
- [StandardMaterial](#standardmaterial)
- [Material Common Properties](#material-common-properties)
- [Textures](#textures)
- [Environment & HDR](#environment--hdr)
- [Node Material](#node-material)
- [Shader Material](#shader-material)

## PBRMaterial

The recommended material for physically-based rendering. Two workflows:

### Metallic-Roughness Workflow (preferred)
```typescript
import { PBRMaterial } from "@babylonjs/core/Materials/PBR/pbrMaterial";
import { Color3 } from "@babylonjs/core/Maths/math.color";

const pbr = new PBRMaterial("pbr", scene);
pbr.albedoColor = new Color3(1.0, 0.766, 0.336);  // base color
pbr.metallic = 1.0;      // 0 = dielectric, 1 = metal
pbr.roughness = 0.4;     // 0 = mirror, 1 = matte

// Textures
pbr.albedoTexture = new Texture("albedo.png", scene);
pbr.metallicTexture = new Texture("metallic-roughness.png", scene);
// metallicTexture: blue channel = metallic, green channel = roughness
pbr.bumpTexture = new Texture("normal.png", scene);
pbr.ambientTexture = new Texture("ao.png", scene);

// Environment reflection (required for realistic PBR)
pbr.reflectionTexture = hdrTexture;
```

### Specular-Glossiness Workflow
```typescript
const pbr = new PBRMaterial("pbr", scene);
pbr.albedoColor = new Color3(1, 1, 1);
pbr.reflectivityColor = new Color3(0.9, 0.9, 0.9);  // specular color
pbr.microSurface = 0.8;  // glossiness (inverse of roughness)
```

### PBR Sub-features
```typescript
// Emissive
pbr.emissiveColor = new Color3(0, 0, 0);
pbr.emissiveTexture = new Texture("emissive.png", scene);
pbr.emissiveIntensity = 1.0;

// Clear coat
pbr.clearCoat.isEnabled = true;
pbr.clearCoat.intensity = 0.5;
pbr.clearCoat.roughness = 0.1;

// Sub-surface (refraction, translucency)
pbr.subSurface.isRefractionEnabled = true;
pbr.subSurface.indexOfRefraction = 1.5;
pbr.subSurface.tintColor = Color3.Teal();

// Anisotropy
pbr.anisotropy.isEnabled = true;
pbr.anisotropy.intensity = 1.0;
pbr.anisotropy.direction = new Vector2(1, 0);

// Sheen (fabric-like)
pbr.sheen.isEnabled = true;
pbr.sheen.intensity = 0.5;
pbr.sheen.color = new Color3(0.8, 0.1, 0.1);
```

## StandardMaterial

Simpler, non-physically-based. Uses diffuse/specular/emissive/ambient model.

```typescript
import { StandardMaterial } from "@babylonjs/core/Materials/standardMaterial";

const mat = new StandardMaterial("mat", scene);
mat.diffuseColor = new Color3(1, 0, 0);     // base color
mat.specularColor = new Color3(0.5, 0.5, 0.5); // highlight color
mat.emissiveColor = new Color3(0, 0, 0);    // self-illumination
mat.ambientColor = new Color3(0.1, 0.1, 0.1);

// Textures
mat.diffuseTexture = new Texture("texture.png", scene);
mat.specularTexture = new Texture("spec.png", scene);
mat.emissiveTexture = new Texture("emissive.png", scene);
mat.bumpTexture = new Texture("normal.png", scene);
mat.opacityTexture = new Texture("opacity.png", scene);

// Specular power (shininess)
mat.specularPower = 64;  // higher = tighter highlights

mesh.material = mat;
```

## Material Common Properties

```typescript
// Transparency
mat.alpha = 0.5;  // 0 = invisible, 1 = opaque
mat.transparencyMode = Material.MATERIAL_ALPHABLEND;
// MATERIAL_OPAQUE, MATERIAL_ALPHATEST, MATERIAL_ALPHABLEND, MATERIAL_ALPHATESTANDBLEND

// Texture alpha
mat.diffuseTexture.hasAlpha = true;
mat.useAlphaFromDiffuseTexture = true;

// Backface culling
mat.backFaceCulling = true;  // default: true
mat.sideOrientation = Material.ClockWiseSideOrientation;

// Wireframe
mat.wireframe = true;

// Z-offset (prevent z-fighting)
mat.zOffset = -1;

// Disable lighting
mat.disableLighting = true;

// Freeze for performance (no shader recompilation)
mat.freeze();
mat.unfreeze();
```

## Textures

```typescript
import { Texture } from "@babylonjs/core/Materials/Textures/texture";
import { CubeTexture } from "@babylonjs/core/Materials/Textures/cubeTexture";
import { DynamicTexture } from "@babylonjs/core/Materials/Textures/dynamicTexture";

// Standard texture
const tex = new Texture("path.png", scene);
tex.uScale = 2;  // tile horizontally
tex.vScale = 2;  // tile vertically
tex.uOffset = 0.5;
tex.vOffset = 0.5;
tex.wrapU = Texture.WRAP_ADDRESSMODE;  // WRAP, CLAMP, MIRROR
tex.wrapV = Texture.WRAP_ADDRESSMODE;

// Dynamic texture (canvas-based, draw with 2D context)
const dynTex = new DynamicTexture("dyn", { width: 512, height: 256 }, scene);
const ctx = dynTex.getContext();
ctx.fillStyle = "red";
ctx.fillRect(0, 0, 512, 256);
dynTex.update();

// Draw text on dynamic texture
dynTex.drawText("Hello", null, null, "bold 48px Arial", "white", "transparent", true);

// Cube texture (skybox/reflection)
const cubeTex = CubeTexture.CreateFromPrefilteredData("env.env", scene);
scene.environmentTexture = cubeTex;
```

## Environment & HDR

```typescript
import { CubeTexture } from "@babylonjs/core/Materials/Textures/cubeTexture";
import { HDRCubeTexture } from "@babylonjs/core/Materials/Textures/hdrCubeTexture";

// .env file (recommended - prefiltered, compact)
const envTex = CubeTexture.CreateFromPrefilteredData("environment.env", scene);
scene.environmentTexture = envTex;

// .hdr file (runtime processing, needs WebGL2)
const hdrTex = new HDRCubeTexture("environment.hdr", scene, 128);
scene.environmentTexture = hdrTex;

// Quick default environment
scene.createDefaultEnvironment();

// Skybox from environment texture
scene.createDefaultSkybox(envTex, true, 1000);
```

## Node Material

Visual shader editor - create materials without writing GLSL/WGSL:

```typescript
import { NodeMaterial } from "@babylonjs/core/Materials/Node/nodeMaterial";

// Load from snippet server
const nodeMat = await NodeMaterial.ParseFromSnippetAsync("snippetId", scene);

// Load from file
const nodeMat = await NodeMaterial.ParseFromFileAsync("name", "url.json", scene);

// Create programmatically
const nodeMat = new NodeMaterial("nodeMat", scene);
// ... add blocks programmatically
nodeMat.build();

mesh.material = nodeMat;
```

Node Material Editor (NME): https://nme.babylonjs.com/

## Shader Material

Custom GLSL/WGSL shaders:

```typescript
import { ShaderMaterial } from "@babylonjs/core/Materials/shaderMaterial";
import { Effect } from "@babylonjs/core/Materials/effect";

// Store shader code
Effect.ShadersStore["customVertexShader"] = `
  precision highp float;
  attribute vec3 position;
  uniform mat4 worldViewProjection;
  void main() {
    gl_Position = worldViewProjection * vec4(position, 1.0);
  }
`;
Effect.ShadersStore["customFragmentShader"] = `
  precision highp float;
  uniform vec3 color;
  void main() {
    gl_FragColor = vec4(color, 1.0);
  }
`;

const shaderMat = new ShaderMaterial("shader", scene, "custom", {
  attributes: ["position"],
  uniforms: ["worldViewProjection", "color"],
});
shaderMat.setColor3("color", new Color3(1, 0, 0));
```
