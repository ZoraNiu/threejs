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
    opacity: 0.8,
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

//gird ground
const gridHelper = new THREE.GridHelper(50,50)
scene.add(gridHelper)

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

//orbitor control dat.GUI
const controlData = {
  semi_majorAxis: 10,//半长轴 a
  eccentricity: 0,//偏心率 e
  inclinationAngle: 0,//轨道倾角 i
  longitudeOfAscendingNode: 0, // 升交点经度 Ω
  argumentOfPeriapsis: 0,// 近地点幅角 ω
  meanAnomaly: 0,// 平近点角 M
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
//f.width(500); 
f.add(controlData,"semi_majorAxis").min(10).max(25).step(1).name('Semi-major Axis (a)').onChange(orbiterMovement);
f.add(controlData,"eccentricity").min(0).max(1).step(0.01).name('Eccentricity (e)').onChange(orbiterMovement);
f.add(controlData,"inclinationAngle").min(0).max(90).step(1).name('Inclination Angle (i)').onChange(orbiterMovement);
f.add(controlData,"longitudeOfAscendingNode").min(0).max(360).name('Longitude of Ascending Node (Ω)').onChange(orbiterMovement);
f.add(controlData,"argumentOfPeriapsis").min(0).max(360).name('Argument of Periapsis (ω)').onChange(orbiterMovement);
//f.add(controlData,"Field_Intensity",{Low:9,High:18})

f.open


//move
//let clock = new THREE.Clock()
function animate(){
  //orbiter
  // const elapsed =  clock.getElapsedTime();
  // let a = controlData.semi_majorAxis
  // let b = Math.sqrt((1-Math.pow(controlData.eccentricity, 2))*a*a)
  // let i = Math.toRadians(controlData.inclinationAngle)
  // orbiter.position.x = Math.sin(elapsed)*(0.5+5*a)
  // orbiter.position.z = Math.cos(elapsed)*(0.5+5*b)
  // orbiter.position.y = 0


  controlData.meanAnomaly += 0.2 //* elapsed; // 随时间增加平近点角
  if (controlData.meanAnomaly > 360) controlData.meanAnomaly -= 360;

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
</style>
