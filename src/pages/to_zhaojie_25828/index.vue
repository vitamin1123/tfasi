<template>
    <div ref="container" class="three-container">
      <div class="controls">
        <button @click="startWave('center')">中心波浪</button>
        <button @click="startWave('diagonal')">对角线波浪</button>
        <button @click="startWave('spiral')">螺旋波浪</button>
        <button @click="startWave('random')">随机波动</button>
        <button @click="resetBlocks">重置</button>
      </div>
    </div>
  </template>
  
  <script setup>
  import { ref, onMounted, onBeforeUnmount } from 'vue';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
  
  const hzk16Loaded = ref(false);
  let hzk16Buffer = null;

  const container = ref(null);
  
  // 配置
  const config = {
    rows: 23,
    cols: 39,
    spacing: 1.1,
    blockWidth: 1,
    blockDepth: 1,
    blockHeight: 5,
    animationSpeed: 0.03,
    colors: {
      base: 0x3498db,
      highlight: 0xe74c3c
    },
    transitions: {
      duration: 2000,
      easing: t => t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2
    },
    character: {
      char: '赵', // 要显示的汉字
      size: 16   // HZK16字库是16x16点阵
    }
  };

  async function loadHZK16() {
  try {
    const response = await fetch('/assets/hzk16s');
    const arrayBuffer = await response.arrayBuffer();
    hzk16Buffer = new Uint8Array(arrayBuffer);
    hzk16Loaded.value = true;
    console.log('HZK16字库加载成功');
  } catch (error) {
    console.error('加载HZK16字库失败:', error);
  }
}
  
const ranges = [
    [0xA1, 0xA9,  0xA1, 0xFE],
    [0xB0, 0xF7,  0xA1, 0xFE],
    [0x81, 0xA0,  0x40, 0xFE],
    [0xAA, 0xFE,  0x40, 0xA0],
    [0xA8, 0xA9,  0x40, 0xA0],
    [0xAA, 0xAF,  0xA1, 0xFE],
    [0xF8, 0xFE,  0xA1, 0xFE],
    [0xA1, 0xA7,  0x40, 0xA0],
  ];
  const codes = new Uint16Array(23940);
  let i = 0;
  for (const [b1Begin, b1End, b2Begin, b2End] of ranges) {
    for (let b2 = b2Begin; b2 <= b2End; b2++) {
      if (b2 !== 0x7F) {
        for (let b1 = b1Begin; b1 <= b1End; b1++) {
          codes[i++] = b2 << 8 | b1;
        }
      }
    }
  }
  const str = new TextDecoder('gbk').decode(codes);
  const table = new Uint16Array(65536);
  for (let i = 0; i < str.length; i++) {
    table[str.charCodeAt(i)] = codes[i];
  }

  // 将字符串转换为 GBK 编码
  function stringToGbk(str) {
    const buf = new Uint8Array(str.length * 2);
    let n = 0;
    for (let i = 0; i < str.length; i++) {
      const code = str.charCodeAt(i);
      if (code < 0x80) {
        buf[n++] = code;
      } else {
        const gbk = table[code];
        buf[n++] = gbk & 0xFF;
        buf[n++] = gbk >> 8;
      }
    }
    return buf.subarray(0, n);
  }

  // 计算汉字在 HZK16 中的偏移量
  function hzkOffset(hz) {
    const buffer1 = stringToGbk(hz);
    const q = buffer1[0];
    const w = buffer1[1];
    return ((q - 161) * 94 + w - 161) * 32;
  }

  // 获取 HZK16 汉字点阵数据
  function getHZK16Character(char) {
    if (!hzk16Buffer) return new Array(32).fill(0);
    const offset = hzkOffset(char);
    return hzk16Buffer.slice(offset, offset + 32);
  }
  
  // 获取汉字的点阵数据
  function getCharacterMatrix(char) {
    const buffer = getHZK16Character(char);
    const matrix = [];
    for (let i = 0; i < 16; i++) {
      matrix[i] = [];
      for (let j = 0; j < 2; j++) {
        const byte = buffer[i * 2 + j];
        for (let k = 0; k < 8; k++) {
          matrix[i][j * 8 + k] = (byte & (0x80 >> k)) !== 0;
        }
      }
    }
    console.log(`${char} hzk16Buffer： `, matrix);
    return matrix;
  }
  
  let scene, camera, renderer, controls, blocks = [];
  let currentAnimation = null;
  let transitionStart = 0;
  let isTransitioning = false;
  let nextAnimationType = null;
  let characterMatrix = []; // 存储汉字点阵数据
  
  // 初始化场景
  function init() {
    // 创建场景
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0xf0f0f0);
  
    // 创建相机
    camera = new THREE.PerspectiveCamera(
      60,
      container.value.clientWidth / container.value.clientHeight, 
      0.1, 
      1000
    );
    camera.position.set(0, 60, 100);
  
    // 创建渲染器
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(container.value.clientWidth, container.value.clientHeight);
    renderer.shadowMap.enabled = true;
    container.value.appendChild(renderer.domElement);
  
    // 添加轨道控制器
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.minDistance = 40;
    controls.maxDistance = 200;
  
    // 添加灯光
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
    scene.add(ambientLight);
  
    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(15, 40, 30);
    directionalLight.castShadow = true;
    scene.add(directionalLight);
  
    // 创建897个长方形方块
    createBlocks();
  
    // 获取汉字点阵数据
    characterMatrix = getCharacterMatrix(config.character.char);
  
    // 开始动画循环
    animate();
  }
  
  // 创建长方形方块网格
  function createBlocks() {
    const geometry = new THREE.BoxGeometry(
      config.blockWidth, 
      config.blockHeight,
      config.blockDepth
    );
    
    const totalRows = config.rows;
    const totalCols = config.cols;
    
    // 计算起始位置使网格居中
    const startX = -((totalCols - 1) * config.spacing) / 2;
    const startZ = -((totalRows - 1) * config.spacing) / 2;
    
    for (let row = 0; row < totalRows; row++) {
      for (let col = 0; col < totalCols; col++) {
        const material = new THREE.MeshPhongMaterial({ 
          color: config.colors.base,
          shininess: 40,
          specular: 0x111111
        });
        
        const block = new THREE.Mesh(geometry, material);
        
        block.position.x = startX + col * config.spacing;
        block.position.z = startZ + row * config.spacing;
        block.position.y = config.blockHeight / 2;
        
        block.userData = {
          originalY: config.blockHeight / 2,
          targetY: config.blockHeight / 2,
          row,
          col,
          baseColor: config.colors.base,
          isCharacterBlock: false // 标记是否为汉字方块
        };
        
        block.castShadow = true;
        block.receiveShadow = true;
        
        scene.add(block);
        blocks.push(block);
      }
    }
  }
  
  // 检查是否是汉字方块
  function isCharacterBlock(row, col) {
    const charSize = config.character.size;
    const totalRows = config.rows;
    const totalCols = config.cols;
    
    // 计算汉字显示区域的起始位置(居中)
    const startRow = Math.floor((totalRows - charSize) / 2);
    const startCol = Math.floor((totalCols - charSize) / 2);
    
    const i = row - startRow;
    const j = col - startCol;
    
    return i >= 0 && i < charSize && 
           j >= 0 && j < charSize && 
           characterMatrix[i] && 
           characterMatrix[i][j];
  }
  
  // 动画循环
  function animate() {
    requestAnimationFrame(animate);
    
    const now = Date.now();
    
    // 处理过渡动画
    if (isTransitioning) {
      const elapsed = now - transitionStart;
      const progress = Math.min(elapsed / config.transitions.duration, 1);
      const easedProgress = config.transitions.easing(progress);
      
      if (progress >= 1) {
        isTransitioning = false;
        if (nextAnimationType) {
          currentAnimation = nextAnimationType;
          nextAnimationType = null;
        }
      }
    }
    
    // 更新方块位置和颜色
    blocks.forEach((block, index) => {
      // 平滑过渡到目标高度
      block.position.y += (block.userData.targetY - block.position.y) * config.animationSpeed;
      
      // 平滑过渡颜色
      if (block.material.color.getHex() !== block.userData.targetColor) {
        const currentColor = new THREE.Color(block.material.color.getHex());
        const targetColor = new THREE.Color(block.userData.targetColor || block.userData.baseColor);
        block.material.color.lerp(targetColor, 0.1);
      }
    });
    
    // 应用当前动画
    if (currentAnimation === 'center') {
      centerWaveAnimation();
    } else if (currentAnimation === 'diagonal') {
      diagonalWaveAnimation();
    } else if (currentAnimation === 'spiral') {
      spiralWaveAnimation();
    } else if (currentAnimation === 'random') {
      randomWaveAnimation();
    }
    
    controls.update();
    renderer.render(scene, camera);
  }
  
  // 中心波浪动画 (修改版，汉字方块在波浪高峰时固定)
  function centerWaveAnimation() {
    const centerRow = config.rows / 2 - 0.5;
    const centerCol = config.cols / 2 - 0.5;
    const time = Date.now() * 0.002;
    
    blocks.forEach(block => {
      const dx = block.userData.col - centerCol;
      const dz = block.userData.row - centerRow;
      const distance = Math.sqrt(dx * dx + dz * dz);
      
      let wave = Math.sin(distance * 0.3 - time) * 1.5;
      
      // 如果是汉字方块且在波浪高峰附近，固定到最高点
      if (isCharacterBlock(block.userData.row, block.userData.col)) {
        const wavePos = distance * 0.3 - time;
        if (Math.abs(wavePos % (Math.PI * 2) - Math.PI/2 < 0.5)) {
          wave = 1.5; // 固定到最高点
        }
      }
      
      block.userData.targetY = wave + (config.blockHeight / 2);
      
      const colorFactor = (wave + 1.5) / 3;
      block.userData.targetColor = new THREE.Color().lerpColors(
        new THREE.Color(config.colors.base),
        new THREE.Color(config.colors.highlight),
        colorFactor
      ).getHex();
    });
  }
  
  // 对角线波浪动画 (修改版，汉字方块在波浪高峰时固定)
  function diagonalWaveAnimation() {
    const time = Date.now() * 0.002;
    
    blocks.forEach(block => {
      const diagonalPos = (block.userData.col / config.cols) + (block.userData.row / config.rows);
      let wave = Math.sin(diagonalPos * 8 - time) * 2;
      
      // 如果是汉字方块且在波浪高峰附近，固定到最高点
      if (isCharacterBlock(block.userData.row, block.userData.col)) {
        const wavePos = diagonalPos * 8 - time;
        if (Math.abs(wavePos % (Math.PI * 2) - Math.PI/2 < 0.5)) {
          wave = 2; // 固定到最高点
        }
      }
      
      block.userData.targetY = wave + (config.blockHeight / 2);
      
      const gradient = (block.userData.col + block.userData.row) / (config.cols + config.rows);
      block.userData.targetColor = new THREE.Color().lerpColors(
        new THREE.Color(config.colors.base),
        new THREE.Color(config.colors.highlight),
        gradient
      ).getHex();
    });
  }
  
  // 螺旋波浪动画 (修改版，汉字方块在波浪高峰时固定)
  function spiralWaveAnimation() {
    const centerRow = config.rows / 2 - 0.5;
    const centerCol = config.cols / 2 - 0.5;
    const time = Date.now() * 0.0015;
    
    blocks.forEach(block => {
      const dx = block.userData.col - centerCol;
      const dz = block.userData.row - centerRow;
      const distance = Math.sqrt(dx * dx + dz * dz);
      const angle = Math.atan2(dz, dx);
      
      let wave = Math.sin(distance * 0.2 + angle * 2 - time) * 1.8;
      
      // 如果是汉字方块且在波浪高峰附近，固定到最高点
      if (isCharacterBlock(block.userData.row, block.userData.col)) {
        const wavePos = distance * 0.2 + angle * 2 - time;
        if (Math.abs(wavePos % (Math.PI * 2) - Math.PI/2 < 0.5)) {
          wave = 1.8; // 固定到最高点
        }
      }
      
      block.userData.targetY = wave + (config.blockHeight / 2);
      
      const spiralFactor = ((angle + Math.PI) / (Math.PI * 2) + distance * 0.05) % 1;
      block.userData.targetColor = new THREE.Color().lerpColors(
        new THREE.Color(config.colors.base),
        new THREE.Color(config.colors.highlight),
        spiralFactor
      ).getHex();
    });
  }
  
  // 随机波动动画 (修改版，汉字方块在波浪高峰时固定)
  function randomWaveAnimation() {
    const time = Date.now() * 0.002;
    
    blocks.forEach(block => {
      const seed = block.userData.row * config.cols + block.userData.col;
      let wave = Math.sin(seed + time) * 
               Math.sin(seed * 0.3 + time * 0.7) * 
               Math.sin(seed * 0.1 + time * 0.3) * 1.5;
      
      // 如果是汉字方块且在波浪高峰附近，固定到最高点
      if (isCharacterBlock(block.userData.row, block.userData.col)) {
        const wavePos = seed + time;
        if (Math.abs(wavePos % (Math.PI * 2) - Math.PI/2 < 0.5)) {
          wave = 1.5; // 固定到最高点
        }
      }
      
      block.userData.targetY = wave + (config.blockHeight / 2);
      
      const colorWave = (Math.sin(seed * 0.5 + time * 0.4) + 1) / 2;
      block.userData.targetColor = new THREE.Color().lerpColors(
        new THREE.Color(config.colors.base),
        new THREE.Color(config.colors.highlight),
        colorWave
      ).getHex();
    });
  }
  
  // 开始特定类型的波浪动画
  function startWave(type) {
    if (isTransitioning) {
      nextAnimationType = type;
      return;
    }
    
    transitionToFlat(() => {
      isTransitioning = true;
      transitionStart = Date.now();
      nextAnimationType = type;
    });
  }
  
  // 过渡到平整状态
  function transitionToFlat(callback) {
    isTransitioning = true;
    transitionStart = Date.now();
    
    blocks.forEach(block => {
      block.userData.transitionStartY = block.userData.targetY;
      block.userData.targetY = config.blockHeight / 2;
      block.userData.targetColor = block.userData.baseColor;
    });
    
    setTimeout(callback, config.transitions.duration);
  }
  
  // 重置所有方块
  function resetBlocks() {
    startWave(null);
    currentAnimation = null;
  }
  
  onMounted(async() => {
    await loadHZK16();
    init();
    window.addEventListener('resize', onWindowResize);
    
    setTimeout(() => {
      startWave('center');
    }, 1000);
  });
  
  onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
    if (renderer) {
      renderer.dispose();
    }
  });
  
  function onWindowResize() {
    camera.aspect = container.value.clientWidth / container.value.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  }
  </script>
  
  <style>
  .three-container {
    width: 100%;
    height: 100vh;
    position: absolute;
    top: 0;
    left: 0;
    overflow: hidden;
  }
  
  .controls {
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    display: flex;
    gap: 10px;
    z-index: 100;
  }
  
  .controls button {
    padding: 8px 16px;
    background: rgba(255, 255, 255, 0.8);
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
    font-family: Arial, sans-serif;
    font-size: 14px;
    transition: all 0.3s;
  }
  
  .controls button:hover {
    background: rgba(255, 255, 255, 1);
    box-shadow: 0 0 10px rgba(255, 255, 255, 0.5);
  }
  </style>