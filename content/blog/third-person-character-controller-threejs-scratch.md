+++
title = "Third-Person Character Controller in Three.js from Scratch"
date = "2026-01-20"
tags = [
  "javascript",
  "threejs",
]
+++

Third person controller games are arguably more immersive than their first person counterparts. They allow you to see your character, make strategic decisions and interact with the environment in unique ways. It’s no wonder some of the greatest games of all time, think Assassin's Creed, The Last of Us and God of War are played solely in third person. Due to its popularity, most game engines have 3rd person controllers built in, so adding one to your game is simply a matter of clicking a couple buttons. On one hand, this allows developers to focus on other aspects of their games; however, it also detracts from teaching game devs how to build controllers from scratch and understand the mathematics behind controllers.

Admittedly, I am someone who typically uses prebuilt controller systems so I thought it would be a fun challenge to build my own 3rd person controller with animations and object collisions in Three.js: the defacto library (powered by WebGL) for creating 3D computer graphics with JavaScript. Let’s get started shall we? [Here is a link to the code if you’d like to follow along.](https://github.com/myl117/js-canvas-experiments/tree/master/third-person-controller)

![The final project scene including player model and collision box](/images/third-person-character-controller-threejs-scratch/screenshot.jpg)
_The final project scene including player model and collision box_

## Getting Started

Firstly, let’s create an index.html file. Since the project requires importing ES modules, a live server will be needed to run it. Our index.html file will simply be our application shell to import our entrypoint index.js script as well as initialise an import map so we can control how the browser resolves modules.

```json
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
    "three/examples/jsm/loaders/GLTFLoader.js": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/GLTFLoader.js",
    "three/examples/jsm/loaders/FBXLoader.js": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/loaders/FBXLoader.js",
    "lil-gui": "https://cdn.jsdelivr.net/npm/lil-gui@0.18.0/dist/lil-gui.esm.min.js"
  }
}
```

We’ll import a GLTF loader for our model/animations and the three.js core module before adding some basic boilerplate to our index.js to set up our environment.

```js
import * as THREE from "three";
import { initLights } from "./helpers/lights.js";
import { initHelpers } from "./helpers/helpers.js";
import { loadObjects } from "./objects.js";

const scene = new THREE.Scene();

// All this code is redundant as the new camera system follows the player
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000,
);

camera.position.z = 5;
const radius = camera.position.z;
const angle = THREE.MathUtils.degToRad(-30);
camera.position.x = radius * Math.sin(angle);
camera.position.z = radius * Math.cos(angle);
camera.position.y = 4;
camera.lookAt(0, 0, 0);

initLights(scene, THREE);
const { renderer, controls } = initHelpers(camera, scene, THREE);
const { gravityCube, character } = await loadObjects(scene, THREE);
```

I’ve created some basic modules to initialise lights, helpers to set up an orbit camera and a file which will house the logic for objects and our character controller. initHelpers sets up our main renderer and grid helpers:

```js
import { OrbitControls } from "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/OrbitControls.js";

const initHelpers = (camera, scene, THREE) => {
  const gridHelper = new THREE.GridHelper(10, 10);
  scene.add(gridHelper);

  const axesHelper = new THREE.AxesHelper(5);
  scene.add(axesHelper);

  // converts 3d data to pixels
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true; // smooth movement
  controls.dampingFactor = 0.05;

  return { renderer, controls };
};

export { initHelpers };
```

initLights sets a hemispheric light as well as some directional lights and corresponding helpers to light up our scene.

```js
const initLights = (scene, THREE) => {
  const lights = [
    [-3, 3, -3],
    [3, 3, 3],
    [-3, -3, -3],
  ];

  for (const l of lights) {
    const light = new THREE.DirectionalLight(0xffffff, 1);
    light.position.set(l[0], l[1], l[2]);
    scene.add(light);

    const lightHelper = new THREE.DirectionalLightHelper(light, 1);
    scene.add(lightHelper);
  }

  const hemLight = new THREE.HemisphereLight(0xffffbb, 0x080820, 1);
  scene.add(hemLight);
};

export { initLights };
```

Our loadObjects method returns the two objects that we will have in our scene. A character and a cube to test collisions. The cube is initially suspended in the air and affected by gravity but this is not a defining feature of this project and for brevity, can be removed if you’d like.

```js
import { loadCharacter } from "./character.js";

const loadGravityCube = (scene, THREE) => {
  // render cube with coloured mesh
  const geometry = new THREE.BoxGeometry(1, 1, 1);
  const material = new THREE.MeshStandardMaterial({ color: 0x89cff0 });
  const cube = new THREE.Mesh(geometry, material);
  scene.add(cube);

  cube.position.y = 4;
  cube.position.x = 3;

  let velocityY = 0;
  const gravity = -0.5 * 0.01;
  const floorY = 0.5;

  const applyGravity = () => {
    velocityY += gravity;

    // Apply velocity to cube
    cube.position.y += velocityY;

    // Stop on floor
    if (cube.position.y <= floorY) {
      cube.position.y = floorY;
      velocityY = 0;
    }
  };

  return { cube, applyGravity };
};

let gravityCube;

const loadObjects = async (scene, THREE) => {
  gravityCube = loadGravityCube(scene, THREE);
  const character = await loadCharacter(scene, THREE);

  return { gravityCube, character };
};

export { loadObjects, gravityCube };
```

But our character.js is where the bulk of our code will reside for our character controller. To get started, let’s import our GLTF loader and create two helper methods called _loadAnimatedModel_ and _loadAnimationOnly_ respectively. The former loads a model with an animation and the latter allows us to load animations and affix them to our loaded model skeleton. We’ll also create an object to define our animation paths. The GLTF loader will expect .glb files but if you're using something like Mixamo which exports models as FBX, you can use the three.js FBX loader. I decided to convert the .fbx files to .glb using FBX2glTF as FBX files can be quite large and require more computation to load.

```js
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { gravityCube } from "./objects.js";

const loadAnimatedModel = (scene, THREE, url) => {
  return new Promise((resolve, reject) => {
    const loader = new GLTFLoader();

    loader.load(
      url,
      (gltf) => {
        const model = gltf.scene;

        if (model.children.length > 0) {
          model.children[0].rotation.y = Math.PI; // 180 deg turn
        }

        model.scale.set(1, 1, 1);
        model.position.set(0, 0, 0);
        scene.add(model);

        const mixer = new THREE.AnimationMixer(model);
        const actions = {};

        gltf.animations.forEach((clip) => {
          actions[clip.name] = mixer.clipAction(clip);
        });

        const tPoseAction = actions[gltf.animations[0].name];
        const walkAction = actions[gltf.animations[1].name];

        // walkAction.play();
        // tPoseAction.play();

        resolve({ model, mixer, actions, walkAction, tPoseAction });
      },
      undefined,
      (error) => reject(error),
    );
  });
};

const loadAnimationOnly = (THREE, url) => {
  return new Promise((resolve, reject) => {
    const loader = new GLTFLoader();
    loader.load(
      url,
      (gltf) => {
        const clip = gltf.animations[0];
        resolve(clip);
      },
      undefined,
      (error) => reject(error),
    );
  });
};

const anims = {
  walking: "../assets/models/walking.glb",
  running: "../assets/animations/running.glb",
  idle: "../assets/animations/idle.glb",
};
```

Now that we’ve set up our helper methods, let’s define our main character controller function called _loadCharacter_. This is where our initial model is loaded (I’ve chosen the walking animation but idle would be the more obvious choice), additional animations are set, constants (speed, animation fade duration, etc.) and key press tracking which we’ll need in our update loop. Since we don’t want our character phasing through walls, let’s also set a bounding box around our character.

```js
const { model, mixer, actions, walkAction } = await loadAnimatedModel(
  scene,
  THREE,
  anims.walking,
);

const runningClip = await loadAnimationOnly(THREE, anims.running);
const runningAction = mixer.clipAction(runningClip);

const idleClip = await loadAnimationOnly(THREE, anims.idle);
const idleAction = mixer.clipAction(idleClip);

idleAction.play();

const rotationSpeed = 0.04;
const speed = 0.025; // movement speed
const runSpeed = 0.05;
const fadeDuration = 0.3;
const keys = {};

// Track key presses
window.addEventListener("keydown", (e) => {
  keys[e.key.toLowerCase()] = true;
});
window.addEventListener("keyup", (e) => {
  keys[e.key.toLowerCase()] = false;
});

let activeAction = idleAction;

let playerBox = new THREE.Box3();
const playerHelper = new THREE.Box3Helper(playerBox, 0x00ff00);
scene.add(playerHelper);

const playerBoxVector = new THREE.Vector3(0.9, 1.8, 0.9);

const updatePlayerBox = () => {
  playerBox.setFromCenterAndSize(
    model.position.clone().setY(1),
    playerBoxVector,
  );
  playerHelper.updateMatrixWorld(true);
};
```

The update loop is synced with delta time from our main animation loop and is where our player’s position and animations are updated depending on which keys are pressed. The logic behind the controller itself is fairly straightforward: A/D keys rotate the player around the Y axis and when the user presses W, we create a forward vector which we rotate around Y. We then scale this by the player’s speed and add the vector to the player’s position but only if the player is not colliding with the gravity cube. It would look pretty absurd if our character animation remains idle while it moves through our scene so let’s also update our animation to the appropriate one: idle if standing, walking if W is pressed and running if shift is pressed. Using the _fadeIn_ and _fadeOut_ methods, the animations seamlessly blend together.

```js
const update = (delta) => {
  let moving = false;

  const running = keys["shift"] || keys["shiftleft"] || keys["shiftright"];
  const currentSpeed = running ? runSpeed : speed;

  // --- Rotate character left/right ---
  if (keys["a"] || keys["arrowleft"]) {
    model.rotation.y += rotationSpeed; // positive = turn left
  }
  if (keys["d"] || keys["arrowright"]) {
    model.rotation.y -= rotationSpeed; // negative = turn right
  }

  // --- Move forward only ---
  if (keys["w"] || keys["arrowup"]) {
    const forward = new THREE.Vector3(0, 0, -1)
      .applyEuler(new THREE.Euler(0, model.rotation.y, 0))
      .multiplyScalar(currentSpeed);

    const cubeBox = new THREE.Box3().setFromObject(gravityCube.cube);

    const cubeHelper = new THREE.Box3Helper(
      new THREE.Box3().setFromObject(gravityCube.cube),
      0xff0000,
    );
    scene.add(cubeHelper);

    let canMoveX = true;
    let canMoveZ = true;

    // Check X axis
    const testPosX = model.position
      .clone()
      .add(new THREE.Vector3(forward.x, 0, 0));
    const boxX = new THREE.Box3().setFromCenterAndSize(
      testPosX.clone().setY(1),
      playerBoxVector,
    );
    if (boxX.intersectsBox(cubeBox)) canMoveX = false;

    // Check Z axis
    const testPosZ = model.position
      .clone()
      .add(new THREE.Vector3(0, 0, forward.z));
    const boxZ = new THREE.Box3().setFromCenterAndSize(
      testPosZ.clone().setY(1),
      playerBoxVector,
    );
    if (boxZ.intersectsBox(cubeBox)) canMoveZ = false;

    // Apply allowed movement
    const newPos = model.position.clone();
    if (canMoveX) newPos.x += forward.x;
    if (canMoveZ) newPos.z += forward.z;

    model.position.copy(newPos);
    moving = canMoveX || canMoveZ;
  }

  updatePlayerBox();

  let targetAction;
  if (!moving) targetAction = idleAction;
  else targetAction = running ? runningAction : walkAction;

  if (activeAction !== targetAction) {
    activeAction.fadeOut(fadeDuration);

    targetAction.reset();
    targetAction.enabled = true;
    targetAction.fadeIn(fadeDuration);
    targetAction.play();

    activeAction = targetAction;
  }

  // --- Update mixer ---
  mixer.update(delta);
};
```

The core character controller logic is now implemented! But our camera stays in a fixed position and does not follow our character so let’s fix this. We need to spin our camera around the Y axis using Euler angles so that we stay behind the player. Then we clone the player position and add the rotated offset resulting in a position where the camera should be. Using linear interpolation, we can smoothly move the camera without snapping. This logic as well as calling our _character.update_ method and passing delta time will sit in our main game loop.

```js
const clock = new THREE.Clock();

function animate() {
  const delta = clock.getDelta();

  character.update(delta);
  gravityCube.applyGravity();

  // follow our player
  const cameraOffset = new THREE.Vector3(0, 3, 4);
  const rotatedOffset = cameraOffset
    .clone()
    .applyEuler(new THREE.Euler(0, character.model.rotation.y, 0));
  const desiredPos = character.model.position.clone().add(rotatedOffset);
  camera.position.lerp(desiredPos, 0.1);
  camera.lookAt(character.model.position);

  requestAnimationFrame(animate);
  renderer.render(scene, camera);
  controls.update();
}

animate();
```

And that brings this mini experiment to a close. We haven’t built Witcher, not by a long chalk but we do have a character controller, a rudimentary collision system and a dynamically animated rig.

Thanks for reading!

## References

[Three.js – JavaScript 3D Library](http://three.js)

[Quaternion and euler rotations in Unity](https://docs.unity3d.com/6000.3/Documentation/Manual/QuaternionAndEulerRotationsInUnity.html)

[godotengine/FBX2glTF](https://github.com/godotengine/FBX2glTF)

[3D collision detection - Game development](https://developer.mozilla.org/en-US/docs/Games/Techniques/3D_collision_detection)
