<template>
  <div></div>
</template>

<script setup>
import * as THREE from 'three';
import { onMounted, onBeforeUnmount } from 'vue';

// 场景、相机、渲染器
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(90, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 6;

const renderer = new THREE.WebGLRenderer({
  antialias: true,
});
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.outputColorSpace = THREE.SRGBColorSpace;
document.body.appendChild(renderer.domElement);

// 加载纹理
const textureLoader = new THREE.TextureLoader();
const texture = textureLoader.load('/assets/zhaojie.png');
const depthTexture = textureLoader.load('/assets/zhaojie1.png');
texture.colorSpace = THREE.SRGBColorSpace;

// 陀螺仪参数
const deviceOrientation = new THREE.Vector2();
let isMobile = /iPhone|iPad|iPod|Android/i.test(navigator.userAgent);

// 创建平面
const geometry = new THREE.PlaneGeometry(6.8, 10.24);
const material = new THREE.ShaderMaterial({
  uniforms: {
    uTexture: { value: texture },
    uDepthTexture: { value: depthTexture },
    uMouse: { value: deviceOrientation }, // 使用设备方向代替鼠标
  },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform sampler2D uTexture;
    uniform sampler2D uDepthTexture;
    uniform vec2 uMouse;
    varying vec2 vUv;
    void main() {
      vec4 depth = texture2D(uDepthTexture, vUv);
      float depthValue = depth.r;
      
      // 调整陀螺仪灵敏度
      float x = vUv.x + uMouse.x * 0.03 * depthValue;
      float y = vUv.y + uMouse.y * 0.03 * depthValue;
      
      vec4 newColor = texture2D(uTexture, vec2(x, y));
      newColor.rgb = pow(newColor.rgb, vec3(1.0/2.2));
      gl_FragColor = newColor;
    }
  `,
});

const plane = new THREE.Mesh(geometry, material);
scene.add(plane);

// 处理设备方向
const handleOrientation = (event) => {
  if (!event.beta || !event.gamma) return;
  
  // 将陀螺仪数据映射到 [-1, 1] 范围
  deviceOrientation.x = THREE.MathUtils.clamp(event.gamma / 45, -1, 1);
  deviceOrientation.y = THREE.MathUtils.clamp(event.beta / 45, -1, 1);
};

// 请求设备方向权限
const requestPermission = () => {
  if (typeof DeviceOrientationEvent !== 'undefined' && 
      typeof DeviceOrientationEvent.requestPermission === 'function') {
    DeviceOrientationEvent.requestPermission()
      .then(permissionState => {
        if (permissionState === 'granted') {
          window.addEventListener('deviceorientation', handleOrientation);
        }
      })
      .catch(console.error);
  } else {
    window.addEventListener('deviceorientation', handleOrientation);
  }
};

// 动画循环
const animate = () => {
  requestAnimationFrame(animate);
  renderer.render(scene, camera);
};
animate();

// 生命周期
onMounted(() => {
  if (isMobile) {
    requestPermission();
  } else {
    // 保留桌面端鼠标支持
    window.addEventListener('mousemove', (event) => {
      deviceOrientation.x = (event.clientX / window.innerWidth) * 2 - 1;
      deviceOrientation.y = -(event.clientY / window.innerHeight) * 2 + 1;
    });
  }
});

onBeforeUnmount(() => {
  window.removeEventListener('deviceorientation', handleOrientation);
});
</script>

<style>
/* 原有样式保持不变 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

canvas {
  width: 100vw;
  height: 100vh;
  display: block;
  position: fixed;
  top: 0;
  left: 0;
}
</style>