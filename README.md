
<!DOCTYPE html>
<html lang="bg">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>VARNA CITY 3D</title>

<style>
*{box-sizing:border-box}
body{margin:0;overflow:hidden;background:#07111b;color:white;font-family:Arial}
#hud{
 position:absolute;top:12px;left:12px;z-index:5;
 background:#07111be8;padding:14px;border:1px solid #36a9d8;
 border-radius:12px;min-width:210px
}
h2{margin:0 0 8px;color:#49caff}
#status{font-size:12px;color:#9edcf4}
button{
 background:#087da8;color:white;border:0;padding:10px;
 border-radius:7px;margin:4px;cursor:pointer
}
#controls{position:absolute;bottom:15px;right:12px;z-index:5}
#loading{
 position:absolute;inset:0;display:grid;place-items:center;
 background:#07111b;color:#49caff;z-index:10;font-size:20px
}
#help{font-size:11px;color:#b8c9d3;margin-top:8px}
</style>
</head>

<body>

<div id="loading">ЗАРЕЖДАНЕ НА ВАРНА...</div>

<div id="hud">
 <h2>VARNA CITY 3D</h2>
 <div id="status">Свързване с OpenStreetMap...</div>
 <div id="help">
  WASD – движение<br>
  Мишка – завъртане на камерата<br>
  Колелце – приближаване
 </div>
 <button onclick="resetCamera()">НАЧАЛО</button>
 <button onclick="toggleBuildings()">СГРАДИ</button>
 <button onclick="toggleRoads()">УЛИЦИ</button>
</div>

<div id="controls">
 <button onclick="moveCamera(0,-30)">↑</button><br>
 <button onclick="moveCamera(-30,0)">←</button>
 <button onclick="moveCamera(30,0)">→</button>
 <button onclick="moveCamera(0,30)">↓</button>
</div>

<script type="importmap">
{
 "imports":{
  "three":"https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.module.js",
  "three/addons/":"https://cdn.jsdelivr.net/npm/three@0.170.0/examples/jsm/"
 }
}
</script>

<script type="module">
import * as THREE from "three";
import {OrbitControls} from "three/addons/controls/OrbitControls.js";

const scene=new THREE.Scene();
scene.background=new THREE.Color(0x8bc6df);
scene.fog=new THREE.Fog(0x8bc6df,1500,6500);

const camera=new THREE.PerspectiveCamera(
 65,innerWidth/innerHeight,0.1,12000
);
camera.position.set(0,500,650);

const renderer=new THREE.WebGLRenderer({antialias:true});
renderer.setSize(innerWidth,innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,2));
renderer.shadowMap.enabled=true;
renderer.shadowMap.type=THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

const controls=new OrbitControls(camera,renderer.domElement);
controls.enableDamping=true;
controls.maxPolarAngle=Math.PI/2.05;
controls.target.set(0,0,0);

scene.add(new THREE.HemisphereLight(0xffffff,0x52616b,2));

const sun=new THREE.DirectionalLight(0xffffff,2.2);
sun.position.set(500,1000,400);
sun.castShadow=true;
scene.add(sun);

const world=new THREE.Group();
scene.add(world);

const buildingGroup=new THREE.Group();
const roadGroup=new THREE.Group();
const parkGroup=new THREE.Group();
world.add(buildingGroup,roadGroup,parkGroup);

// Централна Варна – първоначален район
const BBOX={
 south:43.195,
 west:27.880,
 north:43.225,
 east:27.940
};

const centerLat=(BBOX.south+BBOX.north)/2;
const centerLon=(BBOX.west+BBOX.east)/2;

function project(lat,lon){
 const x=(lon-centerLon)*111320*Math.cos(centerLat*Math.PI/180);
 const z=-(lat-centerLat)*110540;
 return new THREE.Vector3(x,0,z);
}

function polygonShape(points){
 const shape=new THREE.Shape();
 points.forEach((p,i)=>{
  const v=project(p.lat,p.lon);
  if(i===0)shape.moveTo(v.x,v.z);
  else shape.lineTo(v.x,v.z);
 });
 shape.closePath();
 return shape;
}

function makeBuilding(el){
 const pts=el.geometry;
 if(!pts||pts.length<4)return;

 const shape=polygonShape(pts);
 const tags=el.tags||{};
 const levels=Number(tags["building:levels"])||2;
 const height=Number(tags.height)||levels*3.2;

 const geometry=new THREE.ExtrudeGeometry(shape,{
  depth:height,
  bevelEnabled:false
 });

 geometry.rotateX(-Math.PI/2);

 const colors=[
  0xb9a994,0xc9c2b5,0xaaaeb1,
  0xd2c5ad,0x9daeb6,0xe0d9ca
 ];
 const color=colors[Math.abs(el.id)%colors.length];

 const material=new THREE.MeshStandardMaterial({
  color,
  roughness:.85
 });

 const mesh=new THREE.Mesh(geometry,material);
 mesh.position.y=height;
 mesh.castShadow=true;
 mesh.receiveShadow=true;
 buildingGroup.add(mesh);
}

function makeRoad(el){
 const pts=el.geometry;
 if(!pts||pts.length<2)return;

 const type=el.tags?.highway||"road";
 const widths={
  motorway:13,trunk:12,primary:10,
  secondary:8,tertiary:6,
  residential:4.5,service:3.2,
  footway:1.5,path:1.3
 };
 const width=widths[type]||3;

 const vertices=[];
 const indices=[];

 for(let i=0;i<pts.length-1;i++){
  const a=project(pts[i].lat,pts[i].lon);
  const b=project(pts[i+1].lat,pts[i+1].lon);

  const dx=b.x-a.x;
  const dz=b.z-a.z;
  const length=Math.hypot(dx,dz);
  if(length===0)continue;

  const nx=-dz/length*width/2;
  const nz=dx/length*width/2;

  const base=vertices.length/3;

  vertices.push(
   a.x+nx,.4,a.z+nz,
   a.x-nx,.4,a.z-nz,
   b.x+nx,.4,b.z+nz,
   b.x-nx,.4,b.z-nz
  );

  indices.push(
   base,base+1,base+2,
   base+1,base+3,base+2
  );
 }

 if(!vertices.length)return;

 const geometry=new THREE.BufferGeometry();
 geometry.setAttribute(
  "position",new THREE.Float32BufferAttribute(vertices,3)
 );
 geometry.setIndex(indices);
 geometry.computeVertexNormals();

 const road=new THREE.Mesh(
  geometry,
  new THREE.MeshStandardMaterial({color:0x343b42})
 );
 roadGroup.add(road);
}

function makePark(el){
 const pts=el.geometry;
 if(!pts||pts.length<4)return;

 const shape=polygonShape(pts);
 const geometry=new THREE.ShapeGeometry(shape);
 geometry.rotateX(-Math.PI/2);

 const mesh=new THREE.Mesh(
  geometry,
  new THREE.MeshStandardMaterial({
   color:0x4b9856,
   side:THREE.DoubleSide
  })
 );
 mesh.position.y=.15;
 parkGroup.add(mesh);
}

async function loadCity(){
 const box=`${BBOX.south},${BBOX.west},${BBOX.north},${BBOX.east}`;

 const query=`
 [out:json][timeout:60];
 (
  way["building"](${box});
  way["highway"](${box});
  way["leisure"="park"](${box});
  way["landuse"~"grass|forest|recreation_ground"](${box});
 );
 out body geom;
 `;

 const endpoints=[
  "https://overpass-api.de/api/interpreter",
  "https://overpass.kumi.systems/api/interpreter"
 ];

 let data=null;

 for(const endpoint of endpoints){
  try{
   const response=await fetch(endpoint,{
    method:"POST",
    body:query
   });
   if(!response.ok)continue;
   data=await response.json();
   break;
  }catch(e){
   console.warn("Overpass endpoint unavailable",e);
  }
 }

 if(!data)throw new Error("Неуспешно зареждане на картата");

 let buildings=0,roads=0,parks=0;

 for(const el of data.elements){
  if(el.tags?.building){
   makeBuilding(el);buildings++;
  }else if(el.tags?.highway){
   makeRoad(el);roads++;
  }else if(
   el.tags?.leisure==="park"||
   el.tags?.landuse==="grass"||
   el.tags?.landuse==="forest"||
   el.tags?.landuse==="recreation_ground"
  ){
   makePark(el);parks++;
  }
 }

 document.getElementById("status").innerHTML=
  `Сгради: ${buildings}<br>`+
  `Улици: ${roads}<br>`+
  `Зелени площи: ${parks}<br>`+
  `Район: Централна Варна`;

 document.getElementById("loading").style.display="none";
}

const ground=new THREE.Mesh(
 new THREE.PlaneGeometry(9000,9000),
 new THREE.MeshStandardMaterial({color:0x777d78})
);
ground.rotation.x=-Math.PI/2;
ground.position.y=-1;
ground.receiveShadow=true;
scene.add(ground);

window.resetCamera=()=>{
 camera.position.set(0,500,650);
 controls.target.set(0,0,0);
 controls.update();
};

window.toggleBuildings=()=>{
 buildingGroup.visible=!buildingGroup.visible;
};

window.toggleRoads=()=>{
 roadGroup.visible=!roadGroup.visible;
};

window.moveCamera=(x,z)=>{
 camera.position.x+=x;
 camera.position.z+=z;
 controls.target.x+=x;
 controls.target.z+=z;
};

const keys={};
window.addEventListener("keydown",e=>keys[e.key.toLowerCase()]=true);
window.addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);

function animate(){
 requestAnimationFrame(animate);

 const speed=keys.shift?5:2;

 if(keys.w||keys.arrowup)moveCamera(0,-speed);
 if(keys.s||keys.arrowdown)moveCamera(0,speed);
 if(keys.a||keys.arrowleft)moveCamera(-speed,0);
 if(keys.d||keys.arrowright)moveCamera(speed,0);

 controls.update();
 renderer.render(scene,camera);
}

window.addEventListener("resize",()=>{
 camera.aspect=innerWidth/innerHeight;
 camera.updateProjectionMatrix();
 renderer.setSize(innerWidth,innerHeight);
});

loadCity().catch(error=>{
 document.getElementById("loading").innerText=
 "Грешка при зареждане: "+error.message;
 document.getElementById("status").innerText=
 "Провери интернет връзката и опитай отново.";
});

animate();
</script>
</body>
</html>

