<template>
  <div style="position: relative; width: 100%; height: 80vh;">
    <div ref="canvasContainer" class="fill-height w-100" style="height: 100%"></div>

    <div class="control-panel">
      <h3>🤖 Robot Control</h3>

      <div class="slider-group" v-for="(val, key, index) in jointValues" :key="key">
        <label>{{ key.toUpperCase() }} (Axis {{ jointSetting[index].axis }})</label>
        <div class="slider-row">
          <input type="range" v-model.number="jointValues[key]" :min="jointSetting[index].min"
            :max="jointSetting[index].max" step="0.01">
          <span>{{ jointValues[key].toFixed(2) }}</span>
        </div>
      </div>

      <hr />

      <div class="grip-control">
        <label>
          <input type="checkbox" v-model="isGripped">
          🧲 Grip Object
        </label>
      </div>
    </div>
  </div>
</template>

<script setup>
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { nextTick, onMounted, onUnmounted, ref, reactive, watch } from 'vue';

const baseUrl = import.meta.env.BASE_URL

const canvasContainer = ref(null)

const jointSetting = [
  { axis: 'Z', min: -3.14, max: 3.14 },
  { axis: 'X', min: -1.55, max: 1.55 },
  { axis: 'Z', min: -3.14, max: 3.14 },
  { axis: 'X', min: -1.55, max: 1.55 },
  { axis: 'Z', min: -3.14, max: 3.14 },
  { axis: 'X', min: -2.14, max: 2.14 }
];

const jointValues = reactive({
  j1: 0,
  j2: 0,
  j3: 0,
  j4: 0,
  j5: 0,
  j6: 0
})

const isGripped = ref(false) // 그립 상태

let resizeObserver
let scene, camera, renderer, controls, animationId
let robotArmGroup
let handAnchor = null
let grippedObject = null
let robot = { j1: null, j2: null, j3: null, j4: null, j5: null, j6: null }


const initThreeJS = () => {
  if (!canvasContainer.value) return

  // 1. Scene
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0xc8cccf)

  // 2. Camera
  const width = canvasContainer.value.clientWidth
  const height = canvasContainer.value.clientHeight
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 100)
  camera.position.set(2, 2, 2)

  // 3. Renderer
  renderer = new THREE.WebGLRenderer({
    antialias: window.devicePixelRatio < 2,
    alpha: true,
    powerPreference: 'high-performance'
  });
  renderer.setSize(width, height)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  canvasContainer.value.appendChild(renderer.domElement)

  // 4. Controls
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.target.set(0, 0.5, 0)
  controls.maxPolarAngle = Math.PI / 2.2

  // 5. Lights & Environment
  const pmremGenerator = new THREE.PMREMGenerator(renderer);
  pmremGenerator.compileEquirectangularShader();
  const roomEnvironment = new RoomEnvironment();
  scene.environment = pmremGenerator.fromScene(roomEnvironment).texture;

  const cameraLight = new THREE.DirectionalLight(0xb9e4fa, 1.0);
  cameraLight.position.set(10, 20, 10);
  cameraLight.castShadow = true;

  cameraLight.shadow.mapSize.width = 2048;
  cameraLight.shadow.mapSize.height = 2048;

  const d = 5;
  cameraLight.shadow.camera.left = -d;
  cameraLight.shadow.camera.right = d;
  cameraLight.shadow.camera.top = d;
  cameraLight.shadow.camera.bottom = -d;

  cameraLight.shadow.bias = -0.0001;

  cameraLight.target.position.set(0, 0, 0);
  scene.add(cameraLight);
  scene.add(cameraLight.target);

  const gridHelper = new THREE.GridHelper(20, 20, 0xa4a7ab, 0xa4a7ab)
  scene.add(gridHelper)

  const planeGeometry = new THREE.PlaneGeometry(20, 20)
  const planeMaterial = new THREE.ShadowMaterial({ opacity: 0.5 })
  const plane = new THREE.Mesh(planeGeometry, planeMaterial)
  plane.rotation.x = -Math.PI / 2
  plane.receiveShadow = true
  scene.add(plane)

  // 6. Loaders
  const ktx2Loader = new KTX2Loader();
  ktx2Loader.setTranscoderPath(`${baseUrl}basis/`);
  ktx2Loader.detectSupport(renderer);

  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath(`${baseUrl}draco/`);

  const loader = new GLTFLoader();
  loader.setKTX2Loader(ktx2Loader);
  loader.setDRACOLoader(dracoLoader);

  // 7. Model Load
  loader.load(`${baseUrl}robot-arm.glb`, (gltf) => {
    const model = gltf.scene;
    scene.add(model);

    model.traverse((child) => {
      if (child.isMesh) {
        child.castShadow = true;
        child.receiveShadow = true;
      }
    });

    robotArmGroup = model.getObjectByName('main');
    if (!robotArmGroup) console.error("Main group not found");

    handAnchor = model.getObjectByName('holdobjecthand');

    // 관절 매핑
    robot.j1 = model.getObjectByName('0');
    robot.j2 = model.getObjectByName('1');
    robot.j3 = model.getObjectByName('2');
    robot.j4 = model.getObjectByName('3');
    robot.j5 = model.getObjectByName('4');
    robot.j6 = model.getObjectByName('5');

    animate();

  }, undefined, (error) => {
    console.error('An error happened loading the GLTF:', error);
  });
}

const animate = () => {
  animationId = requestAnimationFrame(animate)

  if (robotArmGroup && robot.j1) {
    robot.j1.rotation.z = jointValues.j1
    robot.j2.rotation.x = jointValues.j2
    robot.j3.rotation.z = jointValues.j3
    robot.j4.rotation.x = jointValues.j4
    robot.j5.rotation.z = jointValues.j5
    robot.j6.rotation.x = jointValues.j6
  }

  if (controls) controls.update()
  if (renderer && scene && camera) renderer.render(scene, camera)
}

watch(isGripped, (newVal) => {
  if (newVal) {
    gripObject();
  } else {
    releaseObject();
  }
});

const gripObject = () => {
  if (!handAnchor || grippedObject) return;

  const geometry = new THREE.BoxGeometry(6, 6, 6);
  const material = new THREE.MeshStandardMaterial({ color: 0xff0000 });
  grippedObject = new THREE.Mesh(geometry, material);
  grippedObject.castShadow = true;
  grippedObject.receiveShadow = true;
  grippedObject.position.set(0, 0, 0);

  handAnchor.add(grippedObject);
};

const releaseObject = () => {
  if (!handAnchor || !grippedObject) return;

  handAnchor.remove(grippedObject);
  grippedObject.geometry.dispose();
  grippedObject.material.dispose();
  grippedObject = null;
};

const onWindowResize = () => {
  if (!canvasContainer.value || !camera || !renderer) return
  const width = canvasContainer.value.clientWidth
  const height = canvasContainer.value.clientHeight

  if (width === 0 || height === 0) return

  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
}

onMounted(async () => {
  await nextTick()
  initThreeJS()

  resizeObserver = new ResizeObserver(() => {
    onWindowResize()
  })

  if (canvasContainer.value) {
    resizeObserver.observe(canvasContainer.value)
  }
})

onUnmounted(() => {
  cancelAnimationFrame(animationId)
  if (resizeObserver) resizeObserver.disconnect()
  if (renderer) renderer.dispose()
})
</script>

<style scoped>
.control-panel {
  position: absolute;
  top: 20px;
  right: 20px;
  width: 280px;
  padding: 15px;
  background: rgba(255, 255, 255, 0.9);
  border-radius: 12px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  backdrop-filter: blur(5px);
  z-index: 10;
}

.control-panel h3 {
  margin-top: 0;
  margin-bottom: 15px;
  font-size: 1.1rem;
  color: #333;
}

.slider-group {
  margin-bottom: 8px;
}

.slider-group label {
  display: block;
  font-size: 0.8rem;
  font-weight: bold;
  color: #555;
  margin-bottom: 2px;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.slider-row input[type="range"] {
  flex: 1;
}

.slider-row span {
  font-size: 0.8rem;
  width: 35px;
  text-align: right;
  font-family: monospace;
}

.grip-control {
  margin-top: 10px;
  font-weight: bold;
}

.grip-control label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
}
</style>