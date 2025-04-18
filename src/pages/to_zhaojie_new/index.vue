<template>
  <div ref="container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { GUI } from 'three/addons/libs/lil-gui.module.min.js';

const container = ref(null);
let camera, scene, renderer, controls, gui, sceneRT;

onMounted(async () => {
  // Initialize Three.js scene
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.25, 20);
  camera.position.set(-1.8, 0.6, 2.7);

  scene = new THREE.Scene();

  // Use WebGLRenderer instead of WebGPURenderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  container.value.appendChild(renderer.domElement);

  // Controls
  controls = new OrbitControls(camera, renderer.domElement);
  controls.addEventListener('change', render);
  controls.minDistance = 2;
  controls.maxDistance = 10;
  controls.update();

  // Load background
  const loader = new THREE.CubeTextureLoader().setPath('./assets/city/');
  scene.background = await loader.loadAsync(['px.jpg', 'nx.jpg', 'py.jpg', 'ny.jpg', 'pz.jpg', 'nz.jpg']);

  // Create colored spheres
  const spherePositions = [
    { color: 0x0000ff, position: { z: -1 } },
    { color: 0xff0000, position: { z: 1 } },
    { color: 0xff00ff, position: { x: 1 } },
    { color: 0x00ffff, position: { x: -1 } },
    { color: 0xffff00, position: { y: -1 } },
    { color: 0x00ff00, position: { y: 1 } }
  ];

  spherePositions.forEach(({ color, position }) => {
    const sphere = new THREE.Mesh(
      new THREE.SphereGeometry(0.2, 64, 64),
      new THREE.MeshBasicMaterial({ color })
    );
    Object.entries(position).forEach(([axis, value]) => {
      sphere.position[axis] = value;
    });
    scene.add(sphere);
  });

  // Create PMREM scene
  const pmremGenerator = new THREE.PMREMGenerator(renderer);
  sceneRT = pmremGenerator.fromScene(scene);

  // Create main sphere with environment map
  const sphereMaterial = new THREE.MeshStandardMaterial({
    envMap: sceneRT.texture,
    roughness: 0,
    metalness: 1.0
  });
  
  scene.add(
    new THREE.Mesh(
      new THREE.SphereGeometry(0.5, 64, 64),
      sphereMaterial
    )
  );

  // GUI
  gui = new GUI();
  gui.add(sphereMaterial, 'roughness', 0, 1, 0.001).name('roughness').onChange(() => render());

  // Event listeners
  window.addEventListener('resize', onWindowResize);

  // Initial render
  render();
});

onBeforeUnmount(() => {
  // Cleanup
  if (gui) gui.destroy();
  if (controls) controls.dispose();
  if (renderer) renderer.dispose();
  window.removeEventListener('resize', onWindowResize);
});

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  render();
}

function render() {
  if (renderer && scene && camera) {
    renderer.render(scene, camera);
  }
}
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
}
</style>