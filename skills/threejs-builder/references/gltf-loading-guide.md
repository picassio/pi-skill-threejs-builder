# GLTF Loading Guide for Three.js

Modern patterns for loading, managing, and displaying 3D models in Three.js applications.

---

## Quick Start: The Minimal Pattern

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GLTF Loader</title>
    <style>
        * { margin: 0; padding: 0; }
        body { overflow: hidden; background: #000; }
        canvas { display: block; }
    </style>
</head>
<body>
    <script type="importmap">
    {
        "imports": {
            "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
            "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
        }
    }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

        // Scene setup
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        const renderer = new THREE.WebGLRenderer({ antialias: true });

        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        // Lighting
        const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
        scene.add(ambientLight);

        const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
        directionalLight.position.set(5, 10, 7);
        scene.add(directionalLight);

        // Load model
        const loader = new GLTFLoader();
        loader.load(
            'path/to/model.gltf',
            (gltf) => {
                console.log('Model loaded:', gltf);
                scene.add(gltf.scene);
                camera.position.z = 5;
            },
            (progress) => {
                console.log((progress.loaded / progress.total * 100).toFixed(0) + '%');
            },
            (error) => {
                console.error('Failed to load model:', error);
            }
        );

        // Animation loop
        renderer.setAnimationLoop(() => {
            renderer.render(scene, camera);
        });

        // Handle resize
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
```

---

## Core Concepts

### Import Maps (Essential for ES Modules)

Always use import maps to resolve Three.js module paths correctly:

```html
<script type="importmap">
{
    "imports": {
        "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
        "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
    }
}
</script>
```

This allows clean imports:
```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
```

---

## Pattern 1: Basic Loading

Simplest approach - load a single model and display it.

```javascript
const loader = new GLTFLoader();

loader.load(
    'models/character.gltf',
    (gltf) => {
        const model = gltf.scene;

        model.traverse((child) => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });

        scene.add(model);
    },
    (progress) => {
        const percentComplete = (progress.loaded / progress.total * 100);
        console.log(percentComplete + '% loaded');
    },
    (error) => {
        console.error('Failed to load model:', error);
    }
);
```

---

## Pattern 2: Promise-Based Loading

For cleaner async/await syntax and easier error handling:

```javascript
const loader = new GLTFLoader();

function loadModel(path) {
    return new Promise((resolve, reject) => {
        loader.load(
            path,
            (gltf) => {
                gltf.scene.traverse((child) => {
                    if (child.isMesh) {
                        child.castShadow = true;
                        child.receiveShadow = true;
                    }
                });
                resolve(gltf.scene);
            },
            (progress) => {
                const pct = (progress.loaded / progress.total * 100).toFixed(0);
                console.log(`Loading: ${pct}%`);
            },
            (error) => {
                console.error('Load error:', error);
                reject(error);
            }
        );
    });
}

// Usage
async function init() {
    try {
        const model = await loadModel('models/character.gltf');
        scene.add(model);
    } catch (error) {
        console.error('Failed to initialize:', error);
    }
}

init();
```

---

## Pattern 3: Loading with Fallbacks

Production-ready pattern that gracefully falls back to procedural geometry if GLTF fails:

```javascript
const loader = new GLTFLoader();

function loadModel(path, fallbackGeometry, fallbackMaterial) {
    return new Promise((resolve) => {
        loader.load(
            path,
            (gltf) => {
                gltf.scene.traverse((child) => {
                    if (child.isMesh) {
                        child.castShadow = true;
                        child.receiveShadow = true;
                    }
                });
                resolve(gltf.scene);
            },
            undefined,
            (error) => {
                console.warn(`Failed to load ${path}, using fallback:`, error);
                const mesh = new THREE.Mesh(fallbackGeometry, fallbackMaterial);
                mesh.castShadow = true;
                resolve(mesh);
            }
        );
    });
}

// Usage
async function init() {
    const playerFallback = new THREE.BoxGeometry(0.4, 0.6, 0.3);
    const playerMat = new THREE.MeshStandardMaterial({ color: 0xE9F2FF });

    const player = await loadModel(
        'assets/Character_Male_1.gltf',
        playerFallback,
        playerMat
    );

    scene.add(player);
}

init();
```

---

## Pattern 4: Batch Loading Multiple Models

Load several models sequentially with status updates:

```javascript
const loader = new GLTFLoader();

async function loadAssets(assetList) {
    const loaded = {};

    for (const asset of assetList) {
        try {
            console.log(`Loading ${asset.name}...`);

            const gltf = await new Promise((resolve, reject) => {
                loader.load(asset.path, resolve, undefined, reject);
            });

            gltf.scene.traverse((child) => {
                if (child.isMesh) {
                    child.castShadow = true;
                    child.receiveShadow = true;
                }
            });

            loaded[asset.name] = gltf.scene;
            console.log(`✓ Loaded: ${asset.name}`);

        } catch (error) {
            console.error(`✗ Failed: ${asset.name}`, error);
        }
    }

    return loaded;
}

// Usage
const assets = [
    { name: 'player', path: 'models/character.gltf' },
    { name: 'enemy', path: 'models/skeleton.gltf' },
    { name: 'ground', path: 'models/tile.gltf' }
];

loadAssets(assets).then((models) => {
    scene.add(models.player);
    scene.add(models.enemy);
});
```

---

## Pattern 5: Caching & Reuse (with Animation Support)

Load once, clone many times for performance. **CRITICAL:** Use `SkeletonUtils.clone()` for animated/skinned models!

```javascript
import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import * as SkeletonUtils from 'three/addons/utils/SkeletonUtils.js';

class ModelCache {
    constructor() {
        this.loader = new GLTFLoader();
        this.cache = new Map();
    }

    async load(path) {
        if (this.cache.has(path)) {
            return this.cache.get(path);
        }

        return new Promise((resolve, reject) => {
            this.loader.load(
                path,
                (gltf) => {
                    gltf.scene.traverse((child) => {
                        if (child.isMesh) {
                            child.castShadow = true;
                            child.receiveShadow = true;
                        }
                    });
                    this.cache.set(path, {
                        scene: gltf.scene,
                        animations: gltf.animations
                    });
                    resolve(this.cache.get(path));
                },
                undefined,
                reject
            );
        });
    }

    clone(path) {
        const cached = this.cache.get(path);
        if (!cached) {
            throw new Error(`Model ${path} not in cache. Load it first.`);
        }

        // CRITICAL: Use SkeletonUtils.clone for animated models!
        // Regular .clone() breaks skeleton bone references
        const hasAnimations = cached.animations && cached.animations.length > 0;
        return hasAnimations
            ? SkeletonUtils.clone(cached.scene)
            : cached.scene.clone();
    }

    getAnimations(path) {
        return this.cache.get(path)?.animations || [];
    }
}

// Usage
const cache = new ModelCache();
const mixers = [];

async function init() {
    await cache.load('models/enemy.gltf');

    for (let i = 0; i < 5; i++) {
        const enemy = cache.clone('models/enemy.gltf');
        enemy.position.x = i * 3;
        scene.add(enemy);

        const animations = cache.getAnimations('models/enemy.gltf');
        if (animations.length > 0) {
            const mixer = new THREE.AnimationMixer(enemy);
            mixer.clipAction(animations[0]).play();
            mixers.push(mixer);
        }
    }
}

// In animation loop
function animate() {
    const delta = clock.getDelta();
    mixers.forEach(mixer => mixer.update(delta));
    renderer.render(scene, camera);
}
```

**Why SkeletonUtils.clone() is required:**
- Regular `.clone()` doesn't properly duplicate skeleton/bone hierarchies
- Cloned skinned meshes reference the original skeleton's bones
- This causes cloned models to stay at origin or move with the original
- `SkeletonUtils.clone()` creates independent bone hierarchies for each clone

---

## Pattern 6: Model Normalization

Scale and position GLTF models consistently.

**CRITICAL:** Do NOT use `box.setFromObject(model)` for animated GLTF models! It includes invisible armature bones, helpers, and skeleton rigs which extend far beyond the visible mesh.

```javascript
// ✓ CORRECT - only visible mesh geometry
function normalizeModel(model, targetSize = 1.5) {
    model.position.set(0, 0, 0);
    model.rotation.set(0, 0, 0);

    // Compute bounding box ONLY from visible mesh geometry
    const box = new THREE.Box3();
    model.traverse((child) => {
        if (child.isMesh && child.geometry) {
            child.geometry.computeBoundingBox();
            const meshBox = child.geometry.boundingBox.clone();
            meshBox.applyMatrix4(child.matrixWorld);
            box.union(meshBox);
        }
    });

    if (box.isEmpty()) {
        box.setFromObject(model);
    }

    const size = box.getSize(new THREE.Vector3());
    const maxDim = Math.max(size.x, size.y, size.z);

    const scale = targetSize / maxDim;
    model.scale.setScalar(scale);
    model.updateMatrixWorld(true);

    // Recompute bounds after scale
    const scaledBox = new THREE.Box3();
    model.traverse((child) => {
        if (child.isMesh && child.geometry) {
            const meshBox = child.geometry.boundingBox.clone();
            meshBox.applyMatrix4(child.matrixWorld);
            scaledBox.union(meshBox);
        }
    });

    if (scaledBox.isEmpty()) {
        scaledBox.setFromObject(model);
    }

    // Position so bottom of visible mesh sits at y=0
    model.position.y = -scaledBox.min.y;

    return model;
}
```

---

## Common Pitfalls & Solutions

### ❌ GLTF Won't Load - File Not Found
- Verify file path is correct (relative to HTML file)
- Use a local web server (`python3 -m http.server 8000`)
- Check browser console for exact error

### ❌ Animated Model Floats Above Ground
**Cause**: `Box3.setFromObject()` includes invisible skeleton bones.
**Solution**: Compute bounds only from visible mesh geometry (see Pattern 6).

### ❌ Cloned Animated Model Stays at Origin
**Cause**: Regular `.clone()` doesn't duplicate skeleton/bone hierarchies.
**Solution**: Use `SkeletonUtils.clone()` for any animated/skinned model.

### ❌ No Shadows on GLTF Models
**Solution**: Traverse and enable shadows on all meshes:
```javascript
gltf.scene.traverse((child) => {
    if (child.isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
    }
});
```

---

## Advanced: Draco Compression

```javascript
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const loader = new GLTFLoader();
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('https://www.gstatic.com/draco/v1/decoders/');
loader.setDRACOLoader(dracoLoader);

loader.load('model.glb', (gltf) => {
    scene.add(gltf.scene);
});
```

---

## Best Practices Summary

| Practice | Benefit |
|----------|---------|
| **Use import maps** | Cleaner imports, works with CDN modules |
| **Wrap in promises** | Better error handling, easier async/await |
| **Add fallbacks** | Graceful degradation if models fail |
| **Cache & clone** | Better performance when spawning many instances |
| **SkeletonUtils.clone()** | **Required** for animated/skinned models |
| **Enable shadows** | Traverse & set castShadow/receiveShadow |
| **Normalize scale** | Consistent sizing across different models |
| **Mesh-only bounds** | Use mesh geometry, not setFromObject() for animated models |
| **Show progress** | Better UX for large models |
| **Use local server** | Avoid CORS, proper relative paths |
