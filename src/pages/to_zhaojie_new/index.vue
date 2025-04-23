<template>
  <div ref="container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { CSS3DRenderer, CSS3DObject } from 'three/addons/renderers/CSS3DRenderer.js';

const container = ref(null);
let camera, scene, renderer, controls, css3dRenderer;
let points;
const COLS = 80; // 屏幕列数
const ROWS = 63; // 屏幕行数 (80x63=5040接近5000)
const POINT_COUNT = COLS * ROWS;
const INITIAL_DURATION = 8;  // 初始扩散动画(只播放一次)
const TO_SCREEN_DURATION = 6; // 飞向屏幕
const SCREEN_DURATION = 4;     // 屏幕停留
const TEXT_DURATION = 10;      // 文字动画
const FIREWORK_DURATION = 12;  // 烟花动画

// 创建视频播放器元素

const createVideoElement = () => {
  const videoDiv = document.createElement('div');
  videoDiv.style.width = '500px'; // 增大尺寸
  videoDiv.style.height = '400px';
  videoDiv.style.backgroundColor = '#000';
  videoDiv.style.position = 'absolute';
  videoDiv.style.zIndex = '1000';

  const iframe = document.createElement('iframe');
  iframe.style.width = '100%';
  iframe.style.height = '100%';
  iframe.style.border = '0';
  iframe.src = '//player.bilibili.com/player.html?isOutside=true&aid=886441191&bvid=BV1jK4y1p7Hg&cid=290464814&p=1&autoplay=true&muted=0';
  videoDiv.appendChild(iframe);

  // 创建3D平面对象来承载视频
  const object = new CSS3DObject(videoDiv);
  object.position.set(0, 0, -30); // 放在更远的位置
  object.scale.set(0.08, 0.08, 0.08); // 适当缩放
  
  // 创建一个组来包含视频对象，方便调整
  const group = new THREE.Group();
  group.add(object);
  group.position.set(0, 0, -50); // 整体位置
  
  return group;
};

// 创建屏幕像素点阵 (精确整数网格)
const createScreenPoints = () => {
  const points = [];
  const width = 8;  // 屏幕物理宽度
  const height = width * (ROWS / COLS); // 保持比例
  
  for(let row = 0; row < ROWS; row++) {
    for(let col = 0; col < COLS; col++) {
      const index = row * COLS + col;
      points.push({
        x: (col / COLS - 0.5) * width,
        y: (0.5 - row / ROWS) * height,
        z: 0,
        screenX: (col / COLS - 0.5) * width,
        screenY: (0.5 - row / ROWS) * height,
        screenZ: 0
      });
    }
  }
  return points;
};

// 烟花形状定义 (放大版)
const createFireworkShape = () => {
  const points = [];
  const layers = 5;
  for(let l = 0; l < layers; l++) {
    const radius = 0.8 + l * 0.5; // 增大半径
    const count = 100 + l * 50;
    for(let i = 0; i < count; i++) {
      const angle = (i * 2 * Math.PI) / count;
      const variance = 0.15 * Math.random();
      points.push({
        x: radius * Math.cos(angle) + variance,
        y: radius * Math.sin(angle) + variance,
        z: (l - layers/2) * 0.3
      });
    }
  }
  return points;
};

// 加载汉字图片并离散为点阵 (映射到屏幕点阵)
const loadChineseCharacter = async () => {
  return new Promise((resolve) => {
    const img = new Image();
    img.src = './assets/imgs/赵洁.png';
    img.onload = () => {
      const canvas = document.createElement('canvas');
      const ctx = canvas.getContext('2d');
      canvas.width = COLS;
      canvas.height = ROWS; // 使用屏幕点阵分辨率
      ctx.drawImage(img, 0, 0, COLS, ROWS);
      
      const imageData = ctx.getImageData(0, 0, COLS, ROWS);
      const data = imageData.data;
      const points = [];
      
      for(let row = 0; row < ROWS; row++) {
        for(let col = 0; col < COLS; col++) {
          const index = (row * COLS + col) * 4;
          if(data[index + 3] > 128) { // 半透明以上视为有效点
            points.push({ 
              row, 
              col,
              active: true
            });
          }
        }
      }
      resolve(points);
    };
  });
};

onMounted(async () => {
  // 初始化场景
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.25, 100);
  camera.position.set(0, 0, 15);

  scene = new THREE.Scene();
  // 初始化WebGL渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  container.value.appendChild(renderer.domElement);
  // 初始化CSS3D渲染器
  css3dRenderer = new CSS3DRenderer();
  css3dRenderer.setSize(window.innerWidth, window.innerHeight);
  css3dRenderer.domElement.style.position = 'absolute';
  css3dRenderer.domElement.style.top = '0';
  css3dRenderer.domElement.style.pointerEvents = 'none'; // 防止遮挡交互
  container.value.appendChild(css3dRenderer.domElement);
  // 关键修改：确保CSS3D渲染器在WebGL渲染器之后渲染
  const originalRender = css3dRenderer.render.bind(css3dRenderer);
  css3dRenderer.render = (scene, camera) => {
    renderer.render(scene, camera); // 先渲染WebGL内容
    originalRender(scene, camera); // 再渲染CSS3D内容
  };
  // 添加视频播放器
  const videoElement = createVideoElement();
  scene.add(videoElement);

  // 加载全景背景
  const loader = new THREE.CubeTextureLoader().setPath('./assets/city/');
  scene.background = await loader.loadAsync(['px.jpg', 'nx.jpg', 'py.jpg', 'ny.jpg', 'pz.jpg', 'nz.jpg']);



  controls = new OrbitControls(camera, renderer.domElement);
  controls.minDistance = 5;
  controls.maxDistance = 50;

  // 创建环境光
  scene.add(new THREE.AmbientLight(0x404040));

  // 加载汉字点阵和烟花形状
  const chinesePoints = await loadChineseCharacter();
  const fireworkPoints = createFireworkShape();
  const screenPoints = createScreenPoints();
  
  // 创建点阵系统
  const geometry = new THREE.BufferGeometry();
  const positions = new Float32Array(POINT_COUNT * 3);
  const colors = new Float32Array(POINT_COUNT * 3);
  const sizes = new Float32Array(POINT_COUNT);
  const activePoints = new Array(POINT_COUNT).fill(false);
  const initialYPositions = new Float32Array(POINT_COUNT);

  // 标记汉字对应的活跃点
  chinesePoints.forEach(point => {
    const index = point.row * COLS + point.col;
    if(index < POINT_COUNT) activePoints[index] = true;
  });

  // 初始化位置 - 集中在中心但保留Y轴初始位置
  for(let i = 0; i < POINT_COUNT; i++) {
    positions[i*3] = (Math.random() - 0.5) * 2; // X轴小范围随机
    positions[i*3+1] = (Math.random() - 0.5) * 0.5; // Y轴更集中
    positions[i*3+2] = (Math.random() - 0.5) * 2; // Z轴小范围随机
    
    // 存储初始Y位置用于单轴扩散
    initialYPositions[i] = positions[i*3+1];
    
    // 彩虹色
    const hue = i / POINT_COUNT;
    const color = new THREE.Color().setHSL(hue, 1.0, 0.7);
    colors[i*3] = color.r;
    colors[i*3+1] = color.g;
    colors[i*3+2] = color.b;
    
    sizes[i] = 0.1;
  }

  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
  geometry.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

  const material = new THREE.PointsMaterial({
    size: 0.2,
    vertexColors: true,
    sizeAttenuation: true,
    transparent: true,
    opacity: 1.0,
    blending: THREE.AdditiveBlending
  });

  points = new THREE.Points(geometry, material);
  scene.add(points);

  // 动画控制
  let animationStartTime = Date.now();
  let currentStage = 'initial'; // 初始扩散阶段
  let initialSpreadComplete = false;

  const animate = () => {
    requestAnimationFrame(animate);
    
    const time = Date.now() * 0.001;
    const elapsed = Date.now() - animationStartTime;
    let progress = 0;
    let stageDuration = INITIAL_DURATION;
    
    // 设置各阶段参数
    switch(currentStage) {
      case 'initial': stageDuration = INITIAL_DURATION; break;
      case 'toScreen': stageDuration = TO_SCREEN_DURATION; break;
      case 'screen': stageDuration = SCREEN_DURATION; break;
      case 'text': stageDuration = TEXT_DURATION; break;
      case 'firework': stageDuration = FIREWORK_DURATION; break;
    }
    
    progress = Math.min(elapsed / (stageDuration * 1000), 1);
    
    // 更新点属性
    const positions = points.geometry.attributes.position.array;
    const colors = points.geometry.attributes.color.array;

    // 颜色动画
    for(let i = 0; i < POINT_COUNT; i++) {
      const hue = (i / POINT_COUNT + time * 0.02) % 1;
      let brightness = 0.7;
      if(currentStage === 'screen' || currentStage === 'text') brightness = 0.9;
      
      const pulse = Math.sin(time * 3 + i * 0.001) * 0.1 + 0.9;
      const color = new THREE.Color().setHSL(hue, 1.0, brightness * pulse);
      colors[i*3] = color.r;
      colors[i*3+1] = color.g;
      colors[i*3+2] = color.b;
    }

    // 阶段转换逻辑
    if(progress >= 1) {
      if(elapsed > stageDuration * 1000 + 1000) { // 1秒延迟
        animationStartTime = Date.now();
        
        switch(currentStage) {
          case 'initial': 
            currentStage = 'toScreen'; 
            initialSpreadComplete = true;
            break;
          case 'toScreen': currentStage = 'screen'; break;
          case 'screen': currentStage = 'text'; break;
          case 'text': currentStage = 'firework'; break;
          case 'firework': currentStage = 'screen'; break;
        }
      }
    }

    // 应用当前阶段动画
    for(let i = 0; i < POINT_COUNT; i++) {
      const screenPoint = screenPoints[i];
      let targetX, targetY, targetZ;
      
      if(currentStage === 'initial') {
        // 单轴(Y轴)扩散动画
        const spreadFactor = Math.sin(time * 0.2 + i * 0.01) * 10 * progress; // Y轴扩散因子
        targetX = positions[i*3] + (Math.random() - 0.5) * 0.1;
        targetY = initialYPositions[i] + spreadFactor; // 主要在Y轴上扩散
        targetZ = positions[i*3+2] + (Math.random() - 0.5) * 0.1;
      }
      else if(currentStage === 'toScreen') {
        // 飞向屏幕位置
        const flyProgress = easeOutCubic(progress);
        targetX = screenPoint.screenX * flyProgress;
        targetY = screenPoint.screenY * flyProgress;
        targetZ = screenPoint.screenZ * flyProgress;
      }
      else if(currentStage === 'screen') {
        // 精确屏幕位置
        targetX = screenPoint.screenX;
        targetY = screenPoint.screenY;
        targetZ = screenPoint.screenZ;
      }
      else if(currentStage === 'text') {
        // 文字形状 - 只激活汉字对应的点
        if(activePoints[i]) {
          targetX = screenPoint.screenX;
          targetY = screenPoint.screenY;
          targetZ = screenPoint.screenZ;
          sizes[i] = 0.3; // 放大活跃点
        } else {
          targetX = targetY = targetZ = 0; // 非活跃点移动到中心
          sizes[i] = 0.05; // 缩小非活跃点
        }
      }
      else if(currentStage === 'firework') {
        // 烟花形状 - 均匀分布
        const point = fireworkPoints[i % fireworkPoints.length] || {x:0,y:0,z:0};
        targetX = point.x;
        targetY = point.y;
        targetZ = point.z;
        sizes[i] = 0.25; // 统一大小
      }
      
      // 平滑移动
      positions[i*3] += (targetX - positions[i*3]) * 0.1;
      positions[i*3+1] += (targetY - positions[i*3+1]) * 0.1;
      positions[i*3+2] += (targetZ - positions[i*3+2]) * 0.1;
    }

    // 动态调整点大小
    if(currentStage === 'text' || currentStage === 'firework') {
      points.material.size = 0.25 + Math.sin(time * 2) * 0.05;
    } else {
      points.material.size = 0.2;
    }

    points.geometry.attributes.position.needsUpdate = true;
    points.geometry.attributes.color.needsUpdate = true;
    points.geometry.attributes.size.needsUpdate = true;
    
    // 渲染两种内容
    renderer.render(scene, camera);
    css3dRenderer.render(scene, camera);
  };

  // 缓动函数
  function easeOutCubic(t) {
    return 1 - Math.pow(1 - t, 3);
  }

  animate();
  window.addEventListener('resize', onWindowResize);
});

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  css3dRenderer.setSize(window.innerWidth, window.innerHeight);
}

onBeforeUnmount(() => {
  if (controls) controls.dispose();
  if (renderer) renderer.dispose();
  if (css3dRenderer) {
    container.value.removeChild(css3dRenderer.domElement);
  }
  window.removeEventListener('resize', onWindowResize);
});
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
  background: #000;
}

/* 确保视频播放器在最上层 */
#container {
  position: relative;
}
</style>