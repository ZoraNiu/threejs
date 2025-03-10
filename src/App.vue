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

function updateMagneticFieldLine() {
  // 移除旧的线条
  scene.remove(line);
  scene.remove(arcline);

  var segmentsWind = 127;
  var angleWind = Math.PI / 9;
  var ovalPointsWind = [];
  var arcPointsWind = [];
  var arcPointsWind1 = [];

  // 生成多条磁感线
  for (var d = 6 ; d <= (Math.PI * 1.4)/ angleWind; d++) {
    for (var h = 0; h <= segmentsWind; h++) {
      var ovalAngleWind = ((h / segmentsWind) * Math.PI * 2) + Math.PI;
      var x1 = (Math.cos(ovalAngleWind) * 5 + 5) * (Math.cos(angleWind * d));
      var y1 = Math.sin(ovalAngleWind) * 9;
      var z1 = (Math.cos(ovalAngleWind) * 5 + 5) * (Math.sin(angleWind * d));
      ovalPointsWind.push(new THREE.Vector3(x1, y1, z1));
    }

    var geometryWind = new THREE.BufferGeometry().setFromPoints(ovalPointsWind);
    var material = new THREE.LineBasicMaterial({ color: 0xffffff });
    var lineWind = new THREE.LineLoop(geometryWind, material);
    scene.add(lineWind);

  }

  for (var k = 16 ; k <= (Math.PI * 2)/ angleWind; k++) {

    for (var g = segmentsWind; g >= 0; g--) {
      var arcAngleWind = ((g / segmentsWind) * Math.PI) + Math.PI / 2;
      var u1 = (Math.cos(arcAngleWind) * 23 + 19) * (Math.cos(angleWind * k));
      var v1 = Math.sin(arcAngleWind) * 9;
      var w1 = (Math.cos(arcAngleWind) * 23 + 19) * (Math.sin(angleWind * k));
      arcPointsWind.push(new THREE.Vector3(u1, v1, w1));
    }
    var arcgeometryWind = new THREE.BufferGeometry().setFromPoints(arcPointsWind);
    var arclineWind = new THREE.LineSegments(arcgeometryWind, material);
    scene.add(arclineWind);
  }
  
  for (var l = 1 ; l <= (Math.PI * 0.3)/ angleWind; l++) {

    for (var m = segmentsWind; m >= 0; m--) {
      var arcAngleWind1 = ((m / segmentsWind) * Math.PI) + Math.PI / 2;
      var u2 = (Math.cos(arcAngleWind1) * 23 + 19) * (Math.cos(angleWind * l));
      var v2 = Math.sin(arcAngleWind1) * 9;
      var w2 = (Math.cos(arcAngleWind1) * 23 + 19) * (Math.sin(angleWind * l));
      arcPointsWind1.push(new THREE.Vector3(u2, v2, w2));
    }
    var arcgeometryWind1 = new THREE.BufferGeometry().setFromPoints(arcPointsWind1);
    var arclineWind1 = new THREE.LineSegments(arcgeometryWind1, material);
    scene.add(arclineWind1);
  }
}



//orbitor control dat.GUI
const controlData = {
  semi_majorAxis: 10,//半长轴 a
  eccentricity: 0,//偏心率 e
  inclinationAngle: 0,//轨道倾角 i
  longitudeOfAscendingNode: 0, // 升交点经度 Ω
  argumentOfPeriapsis: 0,// 近地点幅角 ω
  meanAnomaly: 0,// 平近点角 M
  isMove: false,//卫星是否移动
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
f.add(controlData,"isMove")
f.domElement.id = "gui"
f.open()


//move
function animate(){
  requestAnimationFrame(animate)

  if (controlData.isMove){
    controlData.meanAnomaly += 0.2 //* elapsed; // 随时间增加平近点角
    if (controlData.meanAnomaly > 360) controlData.meanAnomaly -= 360; 
  }
  
  if(controlData.longitudeOfAscendingNode !== 0){
    updateMagneticFieldLine();
  }

  orbiterMovement()

  orbiter.lookAt(camera.position);

  //earth
  
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
