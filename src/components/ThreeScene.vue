<!-- src/components/ThreeScene.vue -->
<template>
  <div ref="threeContainer" class="fixed top-0 left-0 w-full h-full -z-10"></div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';
import { RGBELoader } from 'three/examples/jsm/loaders/RGBELoader.js';
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader.js';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

const threeContainer = ref<HTMLDivElement>();
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let controls: OrbitControls;
let ambientLight: THREE.AmbientLight;
let pointLight: THREE.PointLight;

const initThree = () => {
  if (!threeContainer.value) return;


  // Scene
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x111111);

  // Camera
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 1.5, 4); // 稍微高一点，远一点

new EXRLoader()
  .setDataType(THREE.FloatType)
  .load('/indoor_pool_4k.exr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping;

    // 1. 环境光照 & 反射
    scene.environment = texture;

    // 2. 真正显示在“天空盒”里
    scene.background = texture;
    // scene.backgroundBlurriness = 0.3; // 可微调模糊度
  });

  

  // Lighting
  ambientLight = new THREE.AmbientLight(0x404040, 0.5); // 微弱的环境光
  scene.add(ambientLight);

  const hemiLight = new THREE.HemisphereLight(0xebf7ff, 0x080820, 1.2);
  scene.add(hemiLight);

  /* === 3. 落地灯暖点光源（保持氛围）=== */
  const pointLight = new THREE.PointLight(0xffaa55, 3.5, 25, 1.8);
  pointLight.position.set(1.5, 1.5, 0.5);
  pointLight.castShadow = true;
  pointLight.shadow.mapSize.set(1024, 1024);
  scene.add(pointLight);

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.4;
  threeContainer.value.appendChild(renderer.domElement);

  // Controls (可选，用于调试视角)
  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.minDistance = 2;
  controls.maxDistance = 10;
  controls.target.set(0, 1, 0); // 看向场景中心偏上

  /* === 1. 环境 HDR 贴图（全局光照） === */
  

  // 模拟一个地板
  // const floorGeometry = new THREE.PlaneGeometry(20, 20);
  // const floorMaterial = new THREE.MeshStandardMaterial({ color: 0x222222, roughness: 0.8 });
  // const floor = new THREE.Mesh(floorGeometry, floorMaterial);
  // floor.rotation.x = -Math.PI / 2;
  // floor.receiveShadow = true;
  // scene.add(floor);

  // 加载模型 (这里用简单几何体代替，实际项目中用GLTFLoader加载精细模型)
  // 例如：书柜 (BoxGeometry)
  const bookshelfGeo = new THREE.BoxGeometry(1.5, 2.5, 0.5);
  const bookshelfMat = new THREE.MeshStandardMaterial({ color: 0x5c4033, roughness: 0.7 }); // 木色
  const bookshelf = new THREE.Mesh(bookshelfGeo, bookshelfMat);
  bookshelf.position.set(-2, 1.25, -1);
  bookshelf.castShadow = true;
  bookshelf.receiveShadow = true;
  scene.add(bookshelf);

  // 沙发 (BoxGeometry)
  const sofaGeo = new THREE.BoxGeometry(1.8, 0.8, 0.8);
  const sofaMat = new THREE.MeshStandardMaterial({ color: 0x333344, roughness: 0.6 }); // 深色皮质
  const sofa = new THREE.Mesh(sofaGeo, sofaMat);
  sofa.position.set(-0.5, 0.4, -1.5);
  sofa.castShadow = true;
  sofa.receiveShadow = true;
  scene.add(sofa);

  // 落地灯杆
  const lampPoleGeo = new THREE.CylinderGeometry(0.05, 0.05, 1.5, 16);
  const lampPoleMat = new THREE.MeshStandardMaterial({ color: 0xaaaaaa, metalness: 0.8 });
  const lampPole = new THREE.Mesh(lampPoleGeo, lampPoleMat);
  lampPole.position.set(pointLight.position.x, 0.75, pointLight.position.z);
  lampPole.castShadow = true;
  scene.add(lampPole);

  // 落地灯罩 (SphereGeometry)
  const lampShadeGeo = new THREE.SphereGeometry(0.3, 16, 8, 0, Math.PI * 2, 0, Math.PI / 2);
  const lampShadeMat = new THREE.MeshStandardMaterial({ 
    color: 0xffcc88, 
    emissive: 0xffaa55, 
    emissiveIntensity: 1, 
    transparent: true, 
    opacity: 0.7 
  });
  const lampShade = new THREE.Mesh(lampShadeGeo, lampShadeMat);
  lampShade.position.set(pointLight.position.x, pointLight.position.y - 0.1, pointLight.position.z);
  scene.add(lampShade);

  // Animation loop
  const animate = () => {
    requestAnimationFrame(animate);
    controls.update();
    renderer.render(scene, camera);
  };
  animate();

  // Resize handler
  const handleResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  };
  window.addEventListener('resize', handleResize);

  // Cleanup
  onUnmounted(() => {
    window.removeEventListener('resize', handleResize);
    if (threeContainer.value) {
      threeContainer.value.removeChild(renderer.domElement);
    }
    renderer.dispose();
  });
};

onMounted(initThree);
</script>