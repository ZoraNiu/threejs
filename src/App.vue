<template>
  <div></div>
</template>

<script setup>
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'  
import dat from 'dat.gui'



//build scene
const scene = new THREE.Scene()
//build camera
const camera = new THREE.PerspectiveCamera(75,innerWidth/innerHeight,0.1,1000)
camera.position.z = 30

//build texture
const earth = new THREE.TextureLoader().load('img/earth.jpg');
const orbiterTexture = new THREE.TextureLoader().load('img/transparent.png');
//build object
const sphere = new THREE.Mesh(
  new THREE.SphereGeometry(5,50,50), 
  new THREE.MeshBasicMaterial({ 
    map: earth
  })
  )
//billboarding orbiter
const orbiter = new THREE.Mesh(
  new THREE.PlaneGeometry(1, 1),
  new THREE.MeshBasicMaterial({ 
    map: orbiterTexture, 
    side: THREE.DoubleSide,
    transparent: true,
    opacity: 1,
  })
) 
//add to scene
scene.add(sphere)
scene.add(orbiter)

//renderer
const renderer = new THREE.WebGLRenderer(
  {
    antialias: true
  })
renderer.setSize(window.innerWidth, window.innerHeight)
document.body.appendChild(renderer.domElement)

//orbitController
const controls = new OrbitControls(camera, renderer.domElement)

// //gird ground
// const gridHelper = new THREE.GridHelper(50,50)
// scene.add(gridHelper)

//magenitic field line
var ovalPoints = [];
var arcPoints = [];
var segments = 127; // Number of points to approximate the oval shape
var angle = Math.PI / 9
  //var angle = Math.PI / controlData.Field_Intensity
  // Create points for the oval shape
  for (var j = 1; j <= (Math.PI * 2)/ angle; j++){
    for (var i = 0; i <= segments; i++) {
      var ovalAngle = ((i / segments) * Math.PI * 2)+ Math.PI;
      var x = (Math.cos(ovalAngle)*6 + 5) * (Math.cos(angle*j)); // x radius
      var y = Math.sin(ovalAngle)*7; // y radius
      var z = (Math.cos(ovalAngle)*6 + 5) * (Math.sin(angle*j));
      ovalPoints.push(new THREE.Vector3(x, y, z));
    }
    for (var c = segments; c >= 0; c--) {
      var arcAngle = ((c / segments) * Math.PI)+Math.PI/2;
      var u = (Math.cos(arcAngle)*6 +5) * (Math.cos(angle*j));
      var v = Math.sin(arcAngle)*9; // y radius
      var w = (Math.cos(arcAngle)*6 +5) * (Math.sin(angle*j));
      arcPoints.push(new THREE.Vector3(u, v, w));
    }
  }

  var geometry = new THREE.BufferGeometry().setFromPoints(ovalPoints);
  var arcgeometry = new THREE.BufferGeometry().setFromPoints(arcPoints);
  
  var material = new THREE.LineBasicMaterial({ color: 0xffffff });
  
  var line = new THREE.LineLoop(geometry, material);
  var arcline = new THREE.LineSegments(arcgeometry,material);
  
  scene.add(line);
  scene.add(arcline);

//太阳风粒子系统
const particleCount = 5000; // 粒子数量
const particles = new THREE.BufferGeometry();
const positions = new Float32Array(particleCount * 3);
const sizes = new Float32Array(particleCount);
const velocities = [];
const lifetimes = [];

// 初始化粒子位置、大小、速度和生命周期
for (let i = 0; i < particleCount; i++) {
  // 从左侧区域随机生成粒子
  positions[i * 3] = -50 + (Math.random() * 10); // x (从左侧开始)
  positions[i * 3 + 1] = (Math.random() - 0.5) * 40; // y
  positions[i * 3 + 2] = (Math.random() - 0.5) * 40; // z
  
  // 随机大小，创造疏密效果
  sizes[i] = Math.random() * 0.2 + 0.1;
  
  // 随机速度，但主要向右移动
  velocities.push({
    x: Math.random() * 0.3 + 0.2, // 主要向右的速度
    y: (Math.random() - 0.5) * 0.2, // 上下随机移动
    z: (Math.random() - 0.5) * 0.2  // 前后随机移动
  });
  
  // 随机生命周期
  lifetimes.push({
    current: 0,
    total: Math.random() * 100 + 50
  });
}

particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));
particles.setAttribute('size', new THREE.BufferAttribute(sizes, 1));

// 创建着色器材质，使粒子看起来更像太阳风
const particleMaterial = new THREE.ShaderMaterial({
  uniforms: {
    color: { value: new THREE.Color(0xff8c00) }, // 橙色
  },
  vertexShader: `
    attribute float size;
    varying vec3 vColor;
    void main() {
      vColor = vec3(1.0, 0.55, 0.0); // 橙色
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      gl_PointSize = size * (300.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: `
    varying vec3 vColor;
    void main() {
      // 创建圆形粒子
      float r = 0.5;
      vec2 uv = gl_PointCoord - vec2(0.5);
      float d = length(uv);
      float c = smoothstep(r, r-0.1, d);
      
      if (d > r) discard;
      
      // 添加发光效果
      gl_FragColor = vec4(vColor, c * 0.8);
    }
  `,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending
});

const particleSystem = new THREE.Points(particles, particleMaterial);
scene.add(particleSystem);

// 碰撞检测和效果
const impactParticles = new THREE.BufferGeometry();
const impactPositions = new Float32Array(500 * 3); // 碰撞粒子
const impactSizes = new Float32Array(500);
const impacts = [];

// 初始化碰撞粒子（初始不可见）
for (let i = 0; i < 500; i++) {
  impactPositions[i * 3] = 0;
  impactPositions[i * 3 + 1] = 0;
  impactPositions[i * 3 + 2] = 0;
  impactSizes[i] = 0;
  impacts.push({
    active: false,
    life: 0
  });
}

impactParticles.setAttribute('position', new THREE.BufferAttribute(impactPositions, 3));
impactParticles.setAttribute('size', new THREE.BufferAttribute(impactSizes, 1));

const impactMaterial = new THREE.ShaderMaterial({
  uniforms: {
    color: { value: new THREE.Color(0xffaa00) }, // 更亮的橙色
  },
  vertexShader: `
    attribute float size;
    varying vec3 vColor;
    void main() {
      vColor = vec3(1.0, 0.7, 0.3); // 亮橙色
      vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
      gl_PointSize = size * (300.0 / -mvPosition.z);
      gl_Position = projectionMatrix * mvPosition;
    }
  `,
  fragmentShader: `
    varying vec3 vColor;
    void main() {
      float r = 0.5;
      vec2 uv = gl_PointCoord - vec2(0.5);
      float d = length(uv);
      float c = smoothstep(r, r-0.1, d);
      
      if (d > r) discard;
      
      gl_FragColor = vec4(vColor, c * 0.9);
    }
  `,
  transparent: true,
  depthWrite: false,
  blending: THREE.AdditiveBlending
});

const impactSystem = new THREE.Points(impactParticles, impactMaterial);
scene.add(impactSystem);

//orbitor control dat.GUI
const controlData = {
  semi_majorAxis: 10,//半长轴 a
  eccentricity: 0,//偏心率 e
  inclinationAngle: 0,//轨道倾角 i
  longitudeOfAscendingNode: 0, // 升交点经度 Ω
  argumentOfPeriapsis: 0,// 近地点幅角 ω
  meanAnomaly: 0,// 平近点角 M
  isMove: false,//卫星是否移动
  solarWindIntensity: 1.0 // 太阳风强度
}

function orbiterMovement(){
  const {semi_majorAxis:a, eccentricity:e, inclinationAngle:i, longitudeOfAscendingNode: omega, argumentOfPeriapsis:w, meanAnomaly:m } = controlData

  //角度转弧度
  const rad = Math.PI / 180;
  const inclinationRad = i * rad;
  const omegaRad = omega * rad;
  const wRad = w * rad;
  const mRad = m * rad;

  //计算真近点角
  let E = mRad; // 初始值
            for (let i = 0; i < 10; i++) { // 迭代求解开普勒方程
                E = mRad + e * Math.sin(E);
            }
  const nu = 2 * Math.atan2(Math.sqrt(1 + e) * Math.sin(E / 2), Math.sqrt(1 - e) * Math.cos(E / 2));

  // 计算轨道平面内的坐标
  const r = a * (1 - e * e) / (1 + e * Math.cos(nu)); // 距离
  const xOrbit = r * Math.cos(nu);
  const yOrbit = r * Math.sin(nu);
  
  const x = xOrbit * (Math.cos(wRad) * Math.cos(omegaRad) - Math.sin(wRad) * Math.cos(inclinationRad) * Math.sin(omegaRad)) -
                yOrbit * (Math.sin(wRad) * Math.cos(omegaRad) + Math.cos(wRad) * Math.cos(inclinationRad) * Math.sin(omegaRad));
  const z = xOrbit * (Math.cos(wRad) * Math.sin(omegaRad) + Math.sin(wRad) * Math.cos(inclinationRad) * Math.cos(omegaRad)) +
                yOrbit * (Math.cos(wRad) * Math.cos(inclinationRad) * Math.cos(omegaRad) - Math.sin(wRad) * Math.sin(omegaRad));
  const y = xOrbit * (Math.sin(wRad) * Math.sin(inclinationRad)) + yOrbit * (Math.cos(wRad) * Math.sin(inclinationRad));
  orbiter.position.set(x, y, z);
}


const gui = new dat.GUI()
const f = gui.addFolder('Control Data')
f.add(controlData,"semi_majorAxis").min(10).max(25).step(1).name('Semi-major Axis (a)').onChange(orbiterMovement);
f.add(controlData,"eccentricity").min(0).max(1).step(0.01).name('Eccentricity (e)').onChange(orbiterMovement);
f.add(controlData,"inclinationAngle").min(0).max(90).step(1).name('Inclination Angle (i)').onChange(orbiterMovement);
f.add(controlData,"longitudeOfAscendingNode").min(0).max(360).name('Longitude of Ascending Node (Ω)').onChange(orbiterMovement);
f.add(controlData,"argumentOfPeriapsis").min(0).max(360).name('Argument of Periapsis (ω)').onChange(orbiterMovement);
f.add(controlData,"isMove").name('Move Satellite');
f.add(controlData,"solarWindIntensity").min(0.1).max(3).step(0.1).name('Solar Wind Intensity');
f.domElement.id = "gui"
f.open()

// 检测粒子与地球的碰撞并创建碰撞效果
function checkCollisions() {
  const earthRadius = 5; // 地球半径
  const positions = particles.attributes.position.array;
  const impactPositions = impactParticles.attributes.position.array;
  const impactSizes = impactParticles.attributes.size.array;

  for (let i = 0; i < particleCount; i++) {
    const x = positions[i * 3];
    const y = positions[i * 3 + 1];
    const z = positions[i * 3 + 2];

    // 计算粒子到地球中心的距离
    const distance = Math.sqrt(x * x + y * y + z * z);

    // 如果粒子接触到地球表面
    if (distance < earthRadius + 0.5) {
      // 重置粒子位置到左侧
      positions[i * 3] = -50 + (Math.random() * 10);
      positions[i * 3 + 1] = (Math.random() - 0.5) * 40;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 40;

      // 创建碰撞效果
      for (let j = 0; j < impacts.length; j++) {
        if (!impacts[j].active) {
          impacts[j].active = true;
          impacts[j].life = 1.0;
          
          // 设置碰撞粒子的位置
          impactPositions[j * 3] = x;
          impactPositions[j * 3 + 1] = y;
          impactPositions[j * 3 + 2] = z;
          impactSizes[j] = Math.random() * 1.5 + 0.5;
          break;
        }
      }
    }
  }

  // 更新碰撞效果
  for (let i = 0; i < impacts.length; i++) {
    if (impacts[i].active) {
      impacts[i].life -= 0.05;
      if (impacts[i].life <= 0) {
        impacts[i].active = false;
        impactSizes[i] = 0;
      }
    }
  }

  particles.attributes.position.needsUpdate = true;
  impactParticles.attributes.position.needsUpdate = true;
  impactParticles.attributes.size.needsUpdate = true;
}

//move
function animate(){
  if (controlData.isMove){
    controlData.meanAnomaly += 0.2;
    if (controlData.meanAnomaly > 360) controlData.meanAnomaly -= 360;
  }

  // 更新太阳风粒子
  const positions = particles.attributes.position.array;
  for (let i = 0; i < particleCount; i++) {
    // 应用速度和强度
    positions[i * 3] += velocities[i].x * controlData.solarWindIntensity;
    positions[i * 3 + 1] += velocities[i].y * controlData.solarWindIntensity;
    positions[i * 3 + 2] += velocities[i].z * controlData.solarWindIntensity;

    // 更新生命周期
    lifetimes[i].current++;
    if (lifetimes[i].current >= lifetimes[i].total || positions[i * 3] > 50) {
      // 重置粒子到左侧
      positions[i * 3] = -50 + (Math.random() * 10);
      positions[i * 3 + 1] = (Math.random() - 0.5) * 40;
      positions[i * 3 + 2] = (Math.random() - 0.5) * 40;
      lifetimes[i].current = 0;
      
      // 随机更新速度
      velocities[i].x = Math.random() * 0.3 + 0.2;
      velocities[i].y = (Math.random() - 0.5) * 0.2;
      velocities[i].z = (Math.random() - 0.5) * 0.2;
    }
  }
  particles.attributes.position.needsUpdate = true;

  // 检测碰撞
  checkCollisions();

  orbiterMovement()
  orbiter.lookAt(camera.position);

  //earth
  requestAnimationFrame(animate)
  sphere.rotation.y -= 0.001

  //orbitControllerUpdate
  controls.update();
  //render
  renderer.render(scene,camera)
}

animate()
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

#gui {
  position: absolute;
  right: 0;
  width: 500px;
}
</style>
