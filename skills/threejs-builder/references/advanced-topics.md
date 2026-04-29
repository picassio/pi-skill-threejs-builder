# Advanced Three.js Topics

Progressive disclosure reference for topics beyond simple scenes.

**Related guides:**
- [`gltf-loading-guide.md`](gltf-loading-guide.md) - Loading, caching, and cloning 3D models
- [`game-patterns.md`](game-patterns.md) - Game loops, screen effects, animation states, parallax
- [`capacitor-ios.md`](capacitor-ios.md) - Deploying to iOS via Capacitor

---

## Loading 3D Models (GLTF/GLB)

**→ See dedicated guide: [`gltf-loading-guide.md`](gltf-loading-guide.md)**

For comprehensive GLTF loading patterns including basic loading, promise-based approaches, fallbacks, batch loading, caching, and troubleshooting, refer to the dedicated GLTF loading guide.

Quick example using import maps:

```html
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

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);

    const loader = new GLTFLoader();
    loader.load('path/to/model.glb', (gltf) => {
        gltf.scene.traverse((child) => {
            if (child.isMesh) {
                child.castShadow = true;
                child.receiveShadow = true;
            }
        });
        scene.add(gltf.scene);
        camera.position.z = 5;
    });

    renderer.setAnimationLoop(() => {
        renderer.render(scene, camera);
    });
</script>
```

---

## Post-Processing (Bloom, Depth of Field)

**Key improvement: Import maps** resolve Three.js module paths correctly, avoiding long unpkg URLs. See the GLTF loading section above for setup.

```javascript
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'https://unpkg.com/three@0.160.0/examples/jsm/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'https://unpkg.com/three@0.160.0/examples/jsm/postprocessing/UnrealBloomPass.js';

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.toneMapping = THREE.ReinhardToneMapping;

const renderScene = new RenderPass(scene, camera);

const bloomPass = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    1.5,  // strength
    0.4,  // radius
    0.85  // threshold
);

const composer = new EffectComposer(renderer);
composer.addPass(renderScene);
composer.addPass(bloomPass);

renderer.setAnimationLoop(() => {
    composer.render();
});
```

---

## Custom Shaders (ShaderMaterial)

```javascript
const vertexShader = `
    varying vec2 vUv;
    void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
`;

const fragmentShader = `
    uniform float time;
    varying vec2 vUv;
    void main() {
        vec3 color = 0.5 + 0.5 * cos(time + vUv.xyx + vec3(0, 2, 4));
        gl_FragColor = vec4(color, 1.0);
    }
`;

const material = new THREE.ShaderMaterial({
    vertexShader,
    fragmentShader,
    uniforms: {
        time: { value: 0 }
    }
});

renderer.setAnimationLoop((time) => {
    material.uniforms.time.value = time * 0.001;
    renderer.render(scene, camera);
});
```

---

## Text and Sprites

```javascript
function createTextSprite(message, scale = 1) {
    const canvas = document.createElement('canvas');
    const context = canvas.getContext('2d');
    canvas.width = 256;
    canvas.height = 64;

    context.fillStyle = 'rgba(0, 0, 0, 0)';
    context.fillRect(0, 0, canvas.width, canvas.height);
    context.font = 'Bold 24px Arial';
    context.fillStyle = 'white';
    context.textAlign = 'center';
    context.fillText(message, canvas.width / 2, canvas.height / 2);

    const texture = new THREE.CanvasTexture(canvas);
    const material = new THREE.SpriteMaterial({ map: texture });
    const sprite = new THREE.Sprite(material);
    sprite.scale.set(scale * 4, scale, 1);
    return sprite;
}
```

---

## Raycasting (Mouse Picking)

```javascript
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();

window.addEventListener('click', (event) => {
    mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
    mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(mouse, camera);
    const intersects = raycaster.intersectObjects(scene.children);

    if (intersects.length > 0) {
        intersects[0].object.material.color.setHex(Math.random() * 0xffffff);
    }
});
```

---

## Environment Maps (Reflections)

```javascript
import { RGBELoader } from 'https://unpkg.com/three@0.160.0/examples/jsm/loaders/RGBELoader.js';

const rgbeLoader = new RGBELoader();
rgbeLoader.load('path/to/environment.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping;
    scene.environment = texture;
    scene.background = texture;
});

const material = new THREE.MeshStandardMaterial({
    color: 0x444444,
    metalness: 1,
    roughness: 0.1
});
```

---

## InstancedMesh (Many Similar Objects)

```javascript
const count = 1000;
const geometry = new THREE.BoxGeometry(0.2, 0.2, 0.2);
const material = new THREE.MeshStandardMaterial({ color: 0x44aa88 });
const mesh = new THREE.InstancedMesh(geometry, material, count);

const dummy = new THREE.Object3D();
for (let i = 0; i < count; i++) {
    dummy.position.set(
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20,
        (Math.random() - 0.5) * 20
    );
    dummy.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, 0);
    dummy.updateMatrix();
    mesh.setMatrixAt(i, dummy.matrix);
}

scene.add(mesh);
```

---

## Physics Integration (Cannon.js)

```javascript
import * as CANNON from 'https://unpkg.com/cannon-es@0.20.0/dist/cannon-es.js';

const world = new CANNON.World();
world.gravity.set(0, -9.82, 0);

// Create physics body synced with Three.js mesh
const body = new CANNON.Body({
    mass: 1,
    shape: new CANNON.Sphere(0.5),
    position: new CANNON.Vec3(0, 5, 0)
});
world.addBody(body);

const groundBody = new CANNON.Body({
    type: CANNON.Body.STATIC,
    shape: new CANNON.Plane()
});
groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0);
world.addBody(groundBody);

const timeStep = 1 / 60;
renderer.setAnimationLoop(() => {
    world.step(timeStep);
    mesh.position.copy(body.position);
    mesh.quaternion.copy(body.quaternion);
    renderer.render(scene, camera);
});
```

---

## Installation with npm

For production apps, install Three.js via npm:

```bash
npm install three
```

```javascript
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

// Same API as CDN version
```

---

## TypeScript Support

Three.js includes TypeScript definitions:

```typescript
import * as THREE from 'three';

const scene: THREE.Scene = new THREE.Scene();
const geometry: THREE.BoxGeometry = new THREE.BoxGeometry(1, 1, 1);
const material: THREE.MeshStandardMaterial = new THREE.MeshStandardMaterial({
    color: 0x44aa88
});
const cube: THREE.Mesh = new THREE.Mesh(geometry, material);
scene.add(cube);
```

---

## Key Module Import Paths (r160+)

```javascript
// Core
import * as THREE from 'three';

// Addons (three/addons/ in npm, examples/jsm/ in CDN)
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
```

---

## Performance Tips

1. **Reuse geometries and materials**: Create once, use many times
2. **Use InstancedMesh**: For 100+ identical objects
3. **Limit shadow map resolution**: 1024-2048 is usually sufficient
4. **Disable antialiasing**: For pixel art or performance-critical apps
5. **Use frustum culling**: Objects outside view are skipped (automatic)
6. **Merge geometries**: Combine static objects into one mesh
7. **Use LOD (Level of Detail)**: Switch to simpler geometries at distance

---

## Debug Helpers

```javascript
scene.add(new THREE.GridHelper(10, 10));
scene.add(new THREE.AxesHelper(5));

// Stats.js for performance monitoring
import Stats from 'https://unpkg.com/three@0.160.0/examples/jsm/libs/stats.module.js';
const stats = new Stats();
document.body.appendChild(stats.dom);

renderer.setAnimationLoop(() => {
    stats.begin();
    renderer.render(scene, camera);
    stats.end();
});
```
