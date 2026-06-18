# nuggetfield

An infinite, randomly generated field of solid glowing orbs you can float through -
a browser homage to the VR "orb field" simulation from *The Lawnmower Man*.

Thousands of shaded orbs of varying size and colour stretch out in every direction. You
drift through them weightlessly with zero-g thrusters and look around with the mouse. 

Built with [Three.js](https://threejs.org/) `0.184.0` (loaded from a CDN via an import map).

## Run it

ES-module import maps need an `http(s)` origin, so you can't just double-click the file.
Serve the folder with any static server and open it in the browser:

```bash
# pick one, from inside this folder:
npx serve .
# or
python -m http.server 8000
```

Then open the printed URL (e.g. <http://localhost:8000>). Click **CLICK TO FLY** to lock the
mouse and start. Press **Esc** to release the mouse.

Works in any modern WebGL2 browser (Chrome / Edge / Firefox / Safari). A discrete GPU is
nicer but not required.

## Controls

| Input | Action |
| --- | --- |
| **Mouse** | look around (yaw / pitch) |
| **W / S** | thrust forward / back |
| **A / D** | strafe left / right |
| **Space / C** | thrust up / down |
| **Shift** | boost (hold) |
| **X** | brake - come to a full stop |
| **V** | toggle Google-Cardboard split-screen 3D |
| **Esc** | release the mouse |

Movement is **true zero-g**: thrust adds momentum that's preserved indefinitely - nothing
damps it, so you keep coasting until you thrust the other way or hold **X** to come to a
full stop. Bring your own downtempo. 🎧

## How the "infinite" field works

The ~2,400 orbs are kept inside a cube centred on the camera. Each frame, any orb that
drifts out one face of the cube is wrapped to the opposite face (and handed a fresh size +
colour). Exponential fog hides the wrap edge, so you can
thrust in any direction forever and never reach a boundary. It's cheap and genuinely endless
- the trade-off is the universe isn't *persistent* (fly out and back and you'll see a fresh
arrangement, not the identical orbs).

Each orb is an icosphere whose vertices are pushed around by a few octaves of Perlin noise
(`ImprovedNoise`) and flat-shaded, so they read as irregular, faceted "space rocks" rather
than smooth spheres. A handful of distinct noise-displaced shapes are shared across the
field (one `InstancedMesh` per shape), and every orb also gets a random rotation and a
non-uniform scale, so no two look quite alike. A single directional "sun" plus a soft
hemisphere fill give every orb one consistent set of highlights - the illusion of one
powerful unseen light over the whole scene.

The orbs are alive: each drifts on its own gentle random velocity, tumbles about a random
axis, and bounces off its neighbours with **gentle elastic collisions** (mass ∝ volume, so
big rocks shrug off small ones). You're solid too: fly into an orb and it bounces off you,
and the big rocks thump you back, lending the field real heft. Collisions stay cheap via a **uniform-grid spatial hash** -
each orb only tests its ~27 neighbouring cells, so the whole field is roughly O(N) rather
than O(N²) and holds 60 fps at a few thousand orbs (watch the FPS readout in the HUD).

Post-processing: speed-reactive motion-blur trails (`AfterimagePass`), soft bloom glow
(`UnrealBloomPass`), and an `OutputPass` for correct tone-mapping/colour.

### Google Cardboard 3D

Press **V** (or open the page with `?stereo`, e.g. `localhost:8000/?stereo`) for a
side-by-side stereo pair (with a thin centre line to help line things up) that you can view
in a Google-Cardboard-style phone holder - or free-view by crossing your eyes - for real
depth. A `StereoCamera` derives a left/right eye from the main camera and renders them into
the two halves of the frame - and because it's wired in as the composer's first pass, **the
bloom and motion-blur still apply to both eyes**. Post-processing runs once at full size, so
the only extra cost is drawing the (instanced) field twice; it stays smooth. Tune the depth
with `eyeSeparation` (bigger = stronger pop-out) and `stereoFocus` (the zero-parallax
distance - things nearer pop toward you, farther ones recede). No lens barrel-distortion
correction is applied, which is fine for casual viewing.

## Tuning

Every look/feel value lives in the `CONFIG` object at the top of the `<script>` in
[`index.html`](index.html). A few worth a knob-twiddle:

| Key | What it does |
| --- | --- |
| `count` | number of orbs (1500–5000) - main perf lever |
| `fieldRadius` | how far the field extends around you |
| `sizeMin` / `sizeMax` / `sizeBias` | orb size range and small-vs-large skew |
| `noiseAmp` / `noiseFreq` | how lumpy/irregular the rocks are (0 amp = smooth) |
| `geoDetail` / `shapeVariants` / `scaleJitter` | facet coarseness, shape variety, non-uniform stretch |
| `flatFacets` | `true` = faceted low-poly; `false` = smooth lumps |
| `drift` / `driftSpeed` | gentle per-orb drift on/off and max speed |
| `spin` / `spinMin` / `spinMax` / `spinBias` | random-axis tumble: on/off, rate range, and slow-vs-fast skew |
| `collisions` / `restitution` | elastic collisions on/off and bounciness |
| `collisionScale` / `collisionGrid` | collision radius vs visual size; spatial-hash resolution |
| `cameraCollisions` / `cameraRadius` / `cameraMass` / `cameraBounce` | you bump orbs too: on/off, your bubble size, mass, bounciness |
| `satMin/Max`, `litMin/Max` | colour palette saturation / lightness |
| `sunColor/Intensity/Dir`, `fill*` | the single "sun" + ambient fill |
| `accel`, `maxSpeed`, `boost`, `damping` | flight feel (`damping: 0` = pure zero-g) |
| `fogDensity` | how quickly orbs fade into the void (keep ≳ the wrap edge hidden) |
| `bloomStrength/Radius/Threshold`, `trailMax` | glow + motion-blur intensity |
| `eyeSeparation` / `stereoFocus` | Cardboard 3D depth strength and zero-parallax distance |

If you bump `count` or `sizeMax` a lot, you may also want to nudge `fieldRadius` /
`fogDensity` so the wrap edge stays buried in the fog.
