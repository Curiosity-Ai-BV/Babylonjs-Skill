<p align="center">
  <img src="https://cdn.babylonjs.com/babylonIdentity/babylonjs_identity_color_dark.svg" alt="Babylon.js Logo" width="400"/>
</p>

<h1 align="center">Babylon.js AI Coding Skill</h1>

<p align="center">
  <strong>A curated AI skill for Babylon.js 8 — giving your AI coding assistant deep expertise in 3D web development.</strong>
</p>

<p align="center">
  <a href="https://doc.babylonjs.com"><img src="https://img.shields.io/badge/Babylon.js-8.x-E8590C?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIyNCIgaGVpZ2h0PSIyNCI+PHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0xMiAyTDIgMTJsMTAgMTAgMTAtMTBMMTIgMnoiLz48L3N2Zz4=" alt="Babylon.js 8"></a>
  <a href="#"><img src="https://img.shields.io/badge/TypeScript-Ready-3178C6?style=flat-square&logo=typescript&logoColor=white" alt="TypeScript"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-green?style=flat-square" alt="License: MIT"></a>
</p>

---

## What is this?

This is a **skill** (also called a **knowledge pack**) designed for AI coding assistants. It gives your AI agent structured, production-grade knowledge about [Babylon.js 8](https://www.babylonjs.com/) — the powerful open-source 3D engine for the web.

Instead of relying on an AI's general training data (which can be outdated or hallucinated), this skill provides **curated, accurate API patterns and code examples** that the AI reads on-demand whenever you're working on a Babylon.js project.

### Why use a skill?

| Without skill | With skill |
|---|---|
| AI guesses at API names | AI references correct, versioned APIs |
| Outdated patterns (v4/v5 era) | Up-to-date Babylon.js 8 patterns |
| Missing tree-shaking imports | Proper deep-path imports included |
| No awareness of gotchas | Documents 8 critical gotchas |
| Generic boilerplate | Production-ready code patterns |

---

## 📦 What's Included

The skill is organized into a main reference file (`SKILL.md`) and six deep-dive reference documents:

| Reference File | Topics Covered |
|---|---|
| **`SKILL.md`** | Quick reference, minimal scene setup, key patterns, gotchas |
| **`references/core-concepts.md`** | Engine/Scene setup, cameras (ArcRotate, Universal, Follow), lights, shadows, observables, coordinate system |
| **`references/meshes.md`** | Mesh builders, transforms, TransformNode, instances, thin instances, clones, merging, picking & raycasting |
| **`references/materials.md`** | PBRMaterial, StandardMaterial, textures, environment/HDR, Node Material, Shader Material |
| **`references/gui.md`** | AdvancedDynamicTexture, controls (text, buttons, sliders, inputs), containers, layout, events |
| **`references/animation-loading.md`** | Animation API, groups, easing, skeletal animation, glTF/OBJ/STL loading, AssetContainer |
| **`references/performance.md`** | Scene/mesh/material optimization, instancing strategy comparison, monitoring, memory management |
| **`references/doc-urls.md`** | Complete URL map to the official Babylon.js documentation for on-demand fetching |

### Coverage Highlights

- **Scene setup** — WebGL2 and WebGPU engine initialization
- **Mesh creation** — All built-in builders, parametric shapes, and custom vertex data
- **PBR Materials** — Metallic-roughness & specular-glossiness workflows, sub-features (clear coat, sub-surface, sheen, anisotropy)
- **Cameras** — ArcRotateCamera, UniversalCamera, FollowCamera with limits and behaviors
- **Lighting & Shadows** — Hemispheric, point, directional, spot lights with shadow generators
- **2D/3D GUI** — Full AdvancedDynamicTexture control reference with layout system
- **Animation** — Keyframe animation, animation groups, easing functions, skeletal animation from glTF
- **Asset Loading** — Modern async API for glTF, OBJ, STL, Gaussian Splatting
- **Performance** — Instancing strategy decision matrix, scene performance priority modes, memory management
- **On-demand docs** — URL map for topics not in the reference files (physics, WebXR, post-processing, etc.)

---

## 🚀 Installation

### Antigravity / Gemini CLI

Place the `BabylonJS` folder in your skills directory:

```
~/.gemini/antigravity/skills/babylonjs/
├── SKILL.md
└── references/
    ├── core-concepts.md
    ├── meshes.md
    ├── materials.md
    ├── gui.md
    ├── animation-loading.md
    ├── performance.md
    └── doc-urls.md
```

The AI will automatically pick up the skill when you work on Babylon.js-related code.

### Other AI Coding Assistants

The `.skill` bundle file (`babylonjs.skill`) is also included for tools that support the packaged skill format. Refer to your AI tool's documentation for installation instructions.

### Manual / Generic Setup

If your AI tool supports custom instructions or knowledge files, you can point it to the `SKILL.md` file and the `references/` directory. The `SKILL.md` file is self-contained with instructions on how to use the reference files.

---

## 💡 Usage Examples

Once installed, simply ask your AI assistant to help with Babylon.js tasks. The skill kicks in automatically:

```
"Create a 3D scene with a PBR gold sphere on a ground plane"
"Add shadow casting from a directional light"
"Render 10,000 instances of a box using thin instances"
"Set up a glTF model loader with progress tracking"
"Build a GUI overlay with sliders to control material properties"
"Optimize my scene for better frame rates"
```

The AI will pull from the skill's curated reference files to generate correct, production-ready code.

---

## 🧠 How It Works

```
You ask a Babylon.js question
        │
        ▼
AI detects Babylon.js context
        │
        ▼
SKILL.md is loaded (quick reference + gotchas)
        │
        ▼
Relevant reference file(s) are consulted
        │
        ▼
If topic isn't in references → doc-urls.md provides
the URL to fetch live documentation on-demand
        │
        ▼
AI generates correct, tree-shakeable,
production-ready Babylon.js 8 code
```

---

## 📚 Topics Not in Reference Files

The following topics are available via on-demand documentation fetching (URLs in `doc-urls.md`):

- Physics V2 (Havok)
- WebXR / VR / AR
- Post-processing pipelines
- Solid Particle System
- Crowd Navigation
- Node Geometry (procedural)
- Frame Graph
- Flow Graph
- Smart Filters
- Gaussian Splatting

---

## 🤝 Contributing

Contributions are welcome! If you find outdated patterns, missing APIs, or have suggestions for additional reference material:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/add-physics-reference`)
3. Add or update reference files in `BabylonJS/references/`
4. Submit a pull request

### Guidelines

- Keep code examples concise and production-ready
- Always include proper tree-shaking imports
- Document gotchas and common pitfalls
- Target Babylon.js 8.x APIs

---

## 📄 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

<p align="center">
  Made by <a href="https://github.com/adspiceprospice"><strong>adspiceprospice</strong></a> — <a href="https://curiosity.ai"><strong>Curiosity Ai B.V.</strong></a>
</p>
