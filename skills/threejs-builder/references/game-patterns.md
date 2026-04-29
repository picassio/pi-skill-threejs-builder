# Three.js Game Patterns

Patterns for building games with Three.js, beyond simple showcase scenes.

---

## Animation State Management

For characters that switch between idle, run, jump, death, etc.

### Finding and Playing Animations

```javascript
const mixer = new THREE.AnimationMixer(model);
const animations = gltf.animations;

function findAnimation(name) {
    return animations.find(clip =>
        clip.name.toLowerCase().includes(name.toLowerCase())
    );
}

function playAnimation(name, { loop = true, timeScale = 1 } = {}) {
    const clip = findAnimation(name);
    if (!clip) return null;

    const action = mixer.clipAction(clip);
    action.reset();
    action.setLoop(loop ? THREE.LoopRepeat : THREE.LoopOnce);
    action.clampWhenFinished = !loop;
    action.timeScale = timeScale;
    action.play();

    return action;
}
```

### Crossfading Between Animations

```javascript
let currentAction = null;

function switchAnimation(name, { fadeTime = 0.1, ...options } = {}) {
    const clip = findAnimation(name);
    if (!clip) return;

    const newAction = mixer.clipAction(clip);

    // CRITICAL: Check if this action is already playing
    // Calling reset() on an already-playing action causes frame freezing!
    if (currentAction === newAction) {
        if (!newAction.isRunning()) {
            newAction.play();
        }
        return;
    }

    newAction.reset();
    newAction.setLoop(options.loop !== false ? THREE.LoopRepeat : THREE.LoopOnce);
    newAction.clampWhenFinished = !options.loop;
    newAction.timeScale = options.timeScale || 1;
    newAction.enabled = true;
    newAction.paused = false;

    if (currentAction) {
        currentAction.fadeOut(fadeTime);
    }

    newAction.fadeIn(fadeTime).play();
    currentAction = newAction;
}
```

---

## Animation Selection Pitfalls (CRITICAL)

GLTF models may have multiple animations. **Incorrect selection causes:**
- Sheep playing death animations instead of idle
- Wolves frozen (no animation match found)
- Characters stuck in T-pose

### Safe Animation Selection

```javascript
// ❌ WRONG - First animation might be death!
const action = mixer.clipAction(animations[0]);
action.play();

// ✓ CORRECT - Explicit filtering with priority order
function selectSafeAnimation(animations, preferredTypes = ['idle', 'eat', 'graze']) {
    const safeAnims = animations.filter(a => {
        const name = a.name.toLowerCase();
        return !name.includes('death') &&
               !name.includes('die') &&
               !name.includes('dead');
    });

    for (const type of preferredTypes) {
        const match = safeAnims.find(a =>
            a.name.toLowerCase().includes(type)
        );
        if (match) return match;
    }

    if (safeAnims.length > 0) return safeAnims[0];

    console.warn('No safe animation found, using:', animations[0]?.name);
    return animations[0];
}
```

---

## Facing Direction for Side-Scrollers

GLTF models typically face -Z (into the screen). For side-scrollers:

```javascript
function normalizeModel(model, targetHeight, faceDirection = 'right') {
    // ... scaling logic ...

    if (faceDirection === 'right') {
        model.rotation.y = Math.PI / 2;  // Face +X
    } else if (faceDirection === 'left') {
        model.rotation.y = -Math.PI / 2; // Face -X
    }

    return model;
}
```

---

## Game Loop with State Machine

```javascript
const GameState = {
    LOADING: 'loading',
    MENU: 'menu',
    PLAYING: 'playing',
    PAUSED: 'paused',
    GAME_OVER: 'gameover'
};

const state = {
    current: GameState.LOADING,
    timeScale: 1.0,
    score: 0
};

const clock = new THREE.Clock();
const mixers = [];

function gameLoop() {
    const dt = Math.min(clock.getDelta(), 0.1);
    const scaledDt = dt * state.timeScale;

    for (const mixer of mixers) {
        mixer.update(scaledDt);
    }

    switch (state.current) {
        case GameState.PLAYING:
            updatePlayer(scaledDt);
            updateObstacles(scaledDt);
            updateBackground(scaledDt);
            checkCollisions();
            updateScore(dt);
            break;

        case GameState.PAUSED:
            break;

        case GameState.MENU:
            updateBackground(dt * 0.3);
            break;
    }

    updateScreenEffects(dt);
    renderer.render(scene, camera);
}

renderer.setAnimationLoop(gameLoop);
```

---

## Time Scaling (Slow Motion)

```javascript
function triggerSlowMoSmooth(factor, holdTime, rampTime) {
    state.timeScale = factor;

    setTimeout(() => {
        const startTime = performance.now();
        const rampMs = rampTime * 1000;

        function ramp() {
            const elapsed = performance.now() - startTime;
            const t = Math.min(elapsed / rampMs, 1);
            state.timeScale = factor + (1 - factor) * t;

            if (t < 1) requestAnimationFrame(ramp);
        }
        ramp();
    }, holdTime * 1000);
}

// Usage: 0.15x for 0.2s, then ramp to 1x over 0.12s
triggerSlowMoSmooth(0.15, 0.2, 0.12);
```

---

## Screen Effects

### Camera Shake

```javascript
const cameraBasePos = { x: 2, y: 5, z: 16 };
let shakeIntensity = 0;
let shakeDuration = 0;

function shakeScreen(intensity, duration) {
    shakeIntensity = intensity;
    shakeDuration = duration;
}

function updateShake(dt) {
    if (shakeDuration > 0) {
        shakeDuration -= dt;
        const decay = shakeDuration / 0.3;
        const offset = shakeIntensity * decay;

        camera.position.x = cameraBasePos.x + (Math.random() - 0.5) * offset;
        camera.position.y = cameraBasePos.y + (Math.random() - 0.5) * offset;
    } else {
        camera.position.x = cameraBasePos.x;
        camera.position.y = cameraBasePos.y;
    }
}
```

### Screen Flash

```javascript
function flashScreen(color, duration) {
    const overlay = document.getElementById('flash-overlay');
    overlay.style.backgroundColor = color;
    overlay.style.opacity = 0.3;

    setTimeout(() => {
        overlay.style.opacity = 0;
    }, duration * 1000);
}
```

### Zoom Pulse

```javascript
let zoomTarget = 1.0;
let zoomCurrent = 1.0;

function zoomPulse(scale, duration) {
    zoomTarget = scale;
    setTimeout(() => { zoomTarget = 1.0; }, duration * 500);
}

function updateZoom(dt) {
    zoomCurrent += (zoomTarget - zoomCurrent) * dt * 10;
    camera.zoom = zoomCurrent;
    camera.updateProjectionMatrix();
}
```

---

## Squash & Stretch

```javascript
function jumpAnticipation(model) {
    model.scale.set(1.15, 0.8, 1.15);
    setTimeout(() => { model.scale.set(1, 1, 1); }, 80);
}

function landingSquash(model) {
    model.scale.set(1.2, 0.75, 1.2);
    setTimeout(() => { model.scale.set(0.95, 1.1, 0.95); }, 60);
    setTimeout(() => { model.scale.set(1, 1, 1); }, 150);
}
```

---

## Parallax Background Layers

```javascript
const PARALLAX = {
    clouds: 0.1,
    farTrees: 0.3,
    nearTrees: 0.5,
    crowd: 0.7,
    ground: 1.0
};

function updateParallax(dt, baseSpeed) {
    for (const [layerName, objects] of Object.entries(layers)) {
        const speed = baseSpeed * PARALLAX[layerName] * dt;

        for (const obj of objects) {
            obj.position.x -= speed;
            if (obj.position.x < -30) {
                obj.position.x += 60;
                obj.position.z = -5 - Math.random() * 10;
            }
        }
    }
}
```

---

## Object Pooling

```javascript
class ObjectPool {
    constructor(createFn, initialSize = 10) {
        this.createFn = createFn;
        this.pool = [];
        this.active = [];

        for (let i = 0; i < initialSize; i++) {
            const obj = createFn();
            obj.visible = false;
            this.pool.push(obj);
        }
    }

    spawn(x, y, z) {
        let obj = this.pool.pop();
        if (!obj) obj = this.createFn();

        obj.position.set(x, y, z);
        obj.visible = true;
        this.active.push(obj);
        return obj;
    }

    despawn(obj) {
        obj.visible = false;
        const idx = this.active.indexOf(obj);
        if (idx !== -1) this.active.splice(idx, 1);
        this.pool.push(obj);
    }

    updateAll(callback) {
        for (let i = this.active.length - 1; i >= 0; i--) {
            const shouldDespawn = callback(this.active[i]);
            if (shouldDespawn) this.despawn(this.active[i]);
        }
    }
}
```

---

## Fixed Game Camera (Not OrbitControls)

For side-scrollers and fixed-view games:

```javascript
// Simple side-view camera
function setupGameCamera() {
    const camera = new THREE.PerspectiveCamera(45, 960/540, 0.1, 100);
    camera.position.set(2, 5, 16);
    camera.lookAt(2, 1, 0);
    return camera;
}

// Cinematic variant with slight tilt
function setupCinematicCamera() {
    const camera = new THREE.PerspectiveCamera(50, 960/540, 0.1, 100);
    camera.position.set(0, 8, 14);
    camera.lookAt(2, 1, 0);
    camera.rotation.z = 0.03; // Slight Dutch angle
    return camera;
}

// Toggle between camera modes
let cinematicMode = false;
const cameraPositions = {
    simple: { x: 2, y: 5, z: 16, fov: 45, tilt: 0 },
    cinematic: { x: 0, y: 8, z: 14, fov: 50, tilt: 0.03 }
};

function toggleCameraMode() {
    cinematicMode = !cinematicMode;
    const pos = cinematicMode ? cameraPositions.cinematic : cameraPositions.simple;

    camera.position.set(pos.x, pos.y, pos.z);
    camera.fov = pos.fov;
    camera.rotation.z = pos.tilt;
    camera.updateProjectionMatrix();
    camera.lookAt(2, 1, 0);
}
```

---

## Near-Miss Detection

```javascript
function checkNearMiss(player, obstacle, threshold = 0.8) {
    if (obstacle.position.x > player.position.x) return false;
    if (obstacle.passed) return false;

    obstacle.passed = true;
    const verticalGap = player.position.y - obstacle.height;

    if (verticalGap > 0 && verticalGap < threshold) {
        triggerNearMissReward();
        return true;
    }
    return false;
}
```

---

## Floating Text Popup

```css
.floating-text {
    position: absolute;
    font-weight: bold;
    pointer-events: none;
    animation: floatUp 0.6s ease-out forwards;
}
@keyframes floatUp {
    0% { opacity: 1; transform: translateY(0) scale(1); }
    100% { opacity: 0; transform: translateY(-40px) scale(1.2); }
}
```

```javascript
function showFloatingText(text, color, x = '50%', y = '35%') {
    const popup = document.createElement('div');
    popup.className = 'floating-text';
    popup.textContent = text;
    popup.style.color = color;
    popup.style.left = x;
    popup.style.top = y;
    popup.style.transform = 'translateX(-50%)';
    popup.style.fontSize = '1.4rem';
    popup.style.textShadow = `0 0 10px ${color}`;

    document.getElementById('ui').appendChild(popup);
    setTimeout(() => popup.remove(), 600);
}
```

---

## Best Practices Summary

| Pattern | When to Use |
|---------|-------------|
| Animation state management | Characters with multiple animations |
| Animation selection filtering | Avoid death/error animations as default |
| Facing direction rotation | Side-scrollers with GLTF models |
| Game state machine | Any game with menu/play/pause/gameover |
| Time scaling | Slow-mo for impact moments |
| Screen shake | Death, heavy impacts |
| Screen flash | Near-miss, milestones, damage |
| Squash & stretch | Jump, land, any snappy motion |
| Parallax layers | Scrolling games with depth |
| Object pooling | Spawning many objects (obstacles, particles) |
| Fixed camera | Games (not model viewers) |
| Near-miss detection | Rewarding close calls |

---

## Anti-Patterns

❌ **Creating objects in the game loop** — memory leak, GC stalls

❌ **Mixing real time and game time inconsistently** — score affected by slow-mo

❌ **Forgetting to clean up animation mixers** — remove from update list when entity is removed

---

## Deploying Games to iOS

For deploying Three.js games to iOS via Capacitor (touch controls, SPM workflow, native sync), see [`capacitor-ios.md`](capacitor-ios.md).
