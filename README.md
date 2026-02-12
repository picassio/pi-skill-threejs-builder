# pi-skill-threejs-builder

A [pi](https://github.com/badlogic/pi) skill package for creating Three.js web applications.

## What it does

When you ask pi to create a Three.js scene, app, 3D showcase, or any browser-based 3D graphics project, this skill provides comprehensive guidance covering:

- **Scene setup** — minimal HTML template, ES modules, responsive rendering
- **Geometries** — primitives (Box, Sphere, Torus, Icosahedron, etc.)
- **Materials** — Basic, Standard, Physical, Phong with PBR properties
- **Lighting** — Ambient, Directional, Point, Spot with shadow setup
- **Animation** — rotation, wave motion, mouse interaction, `setAnimationLoop`
- **Camera controls** — OrbitControls, fixed game cameras
- **Coordinate system** — right-handed axes, GLTF orientation, camera-relative movement

### Reference guides (loaded on demand)

| Topic | Description |
|-------|-------------|
| **GLTF Loading** | Loading, caching, cloning 3D models, SkeletonUtils, model normalization |
| **Reference Frames** | Calibration, anchoring, axis correctness, forward direction debugging |
| **Game Patterns** | State machines, animation switching, parallax, object pooling, screen effects |
| **Advanced Topics** | Post-processing (bloom), custom shaders, physics (Cannon.js), instancing, raycasting |

### Calibration helper

Includes a reusable ES module (`gltf-calibration-helpers.mjs`) that visualizes axes, bounding boxes, and forward direction arrows on GLTF models for quick reference-frame debugging.

## Install

```bash
# From git (global)
pi install https://github.com/picassio/pi-skill-threejs-builder

# Project-local install (shared with team via .pi/settings.json)
pi install -l https://github.com/picassio/pi-skill-threejs-builder

# From a local clone
pi install /path/to/pi-skill-threejs-builder
```

## Usage

Once installed, the skill is auto-discovered. Just ask pi to build something with Three.js:

```
> Create a Three.js scene with a rotating torus knot and bloom post-processing
```

Or invoke it explicitly:

```
/skill:threejs-builder Create a particle field with mouse interaction
```

## Structure

```
pi-skill-threejs-builder/
├── package.json                          # pi package manifest
├── README.md
├── LICENSE
└── skills/
    └── threejs-builder/
        ├── SKILL.md                      # Main skill instructions
        ├── references/
        │   ├── gltf-loading-guide.md     # GLTF loading patterns
        │   ├── reference-frame-contract.md # Calibration & anchoring
        │   ├── game-patterns.md          # Game dev patterns
        │   └── advanced-topics.md        # Shaders, physics, post-processing
        └── scripts/
            ├── README.md                 # Script usage docs
            ├── install-gltf-calibration-helpers.py
            └── gltf-calibration-helpers.mjs
```

## Credits

Skill content adapted from [chongdashu/threejs-toonshooter](https://github.com/chongdashu/threejs-toonshooter) Claude Code skill, converted to the [Agent Skills](https://agentskills.io) standard for pi.

## License

MIT
