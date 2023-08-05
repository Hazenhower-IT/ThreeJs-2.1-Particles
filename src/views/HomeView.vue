<template>
  <canvas ref="canvasRef"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import * as THREE from "three"
import gsap from "gsap"
import * as dat from "dat.gui"

//GUI
//tra le librerie piu popolari ci sono dat.GUI, control-panel, ControlKit, Guify, Oui

//in questo esempio useremo dat.GUI

// Inizializziamo la GUI (Puoi nascondere il pannello premendo H sulla tastiera)
const gui = new dat.GUI()

//const gui = new dat.GUI({ closed:true }) //Usa questo per iniziare con il pannello GUI "chiuso" ma apribile cliccando sul pulsante che appare sullo schermo
//const gui = new dat.GUI({width:400}) //Usa questo per impostare la grandezza iniziale del pannello (puo comunque essere modificata live con drag and drop)


let canvasRef = ref();

var controls

//Utilizziamo i valori del viewport per impostare la grandezza del canvas
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

//Aggiungiamo l'event listner per il resize del canvas
window.addEventListener("resize", ()=>{
  //update sizes for the canvas
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  //update the camera aspect ratio
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  //update renderer
  renderer.setSize(sizes.width, sizes.height)
  //HANDLE PIXEL RATIO
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})


//HANDLE FULL SCREEN (Il codice aggiuntivo è per permettere il funzionamento corretto anche con browser Safari)
window.addEventListener("dblclick",()=>{
  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

let scene = new THREE.Scene()

let renderer


// CAMERA
let camera = new THREE.PerspectiveCamera(
  75, 
  sizes.width / sizes.height, 
  0.1,
  100  
);
camera.position.set(0,0,3)
scene.add(camera);

//TEXTURE
const textureLoader = new THREE.TextureLoader()
const particleTexture = textureLoader.load("textures/particles/2.png")


//PARTICLES
//Particles are used to create a variety of effects: stars, snow, rain, teleportation effect,  fire, smoke, dust and many more
//you can have thousand of then with a good framerate. 
//each particles is composed of a plane (two triangles) always facing the camera

//to create particles in threejs is like creating a mesh, 
//we need a geometry, a material, and finally instead of use the Mesh, we use the Point class

const particleGeometry = new THREE.BufferGeometry()
const count = 500000


const positions = new Float32Array(count * 3) //we use a float 32 array because that's better in performance (count * 3 because we store a position array, and the position has 3 value)
const colors = new Float32Array(count*3)

//fill the position Array, and then create a new buffer Attribute for the THREE.BufferGeometry, and assign it the filled array

for(let i =0 ; i< count *3; i++){
  positions[i] = (Math.random() - 0.5) * 10
  colors[i] = Math.random()
}
particleGeometry.setAttribute("position", new THREE.BufferAttribute(positions, 3))
particleGeometry.setAttribute("color", new THREE.BufferAttribute(colors, 3))

const particleMaterial = new THREE.PointsMaterial({
  size:0.02, 
  sizeAttenuation: true, //size attenuation: if the camera is far from the particles, it looks small if it's true, otherwise has the same size
  transparent:true, 
  alphaMap: particleTexture, 
  //color: 0xff0000, 
  //alphaTest: 0.001
}) 

//we can also change this property after the instantiation, for example:
//particleMaterial.size = 0.02

//We can also change the color of the particles (THIS DOESN'T WORK ANYMORE)
particleMaterial.vertexColors = true

//Points
const particles = new THREE.Points(particleGeometry, particleMaterial)
scene.add(particles)

//but the render isn't perfect, the particles background hide the other particles even if it's alpha, and that's because all the particles are rendered  in the same order as they are created, and WebGL don't know which one is in front of the other
//there are multiple ways of fix this, and it really depend on your case and on your projects need

//USING ALPHATEST
//the alpha test is a value between 0 and 1 that tell the WebGL when not to render the pixel according to that pixel transparency
//by default the value is 0, so the pixel will be rendered anyway
//so we use 0.001
//by default he renderer render the pixel even if it's "alpha = 0 " , or invisible, in this way, the renderer know what part to render, and what part not render at all




//The next solution is using DEPTH TEST
//when drawing, the webgl test is what is being draw is closer than what's already drawn
//this is called depth testing, and we can deactivate this

//particleMaterial.depthTest = false

//but deactivating the depth testing can create bugs if you have other objects (or particles) with different colors





//another solutions is using DEPTH WRITE
//the depth of what s being drawn is stored in what we call a depth buffer 
//instead of not testing if the particle is closer than what's in this depth buffer we can tell the webGL not to write particles in that depth buffer with depthTest
particleMaterial.deepWrite = false




//The last solution is BLENDING
//blending has biggest impact on the performance than the other solutions, but it can be used for create intresting thing
//with the blending property we can tell the webGL to add the color of the pixel to the color of the pixel already drawn

particleMaterial.blending = THREE.AdditiveBlending





///////////ANIMATING PARTICLES /////////////
//There are multiple ways of aniimate particles
//the Point objects in threejs, innerhit from Object3D, so we can use the same property as whatever object to animate this particle



const clock = new THREE.Clock()

//Animate Loop
const animate = () =>{
  const elapsedTime = clock.getElapsedTime()

  //So we can update the particles all in one time
  //Update the paprticle
  //particles.rotation.y = elapsedTime * 0.02

  // or we can animate it one by one

  //the particle position are stored in the position attribute, so we can access this property
  //so we can update this, but we need to go on 3 by 3

  for(let i=0; i< count; i++){
    const i3 = i*3
    
    //next for look like a wave we are going to offset this scene animation, depending on the x position of the particles
 
    const x = particleGeometry.attributes.position.array[i3]

    particleGeometry.attributes.position.array[i3+1] = Math.sin(elapsedTime + x)  //in this way we only affect the y position, use i3+0 (o i3) for the x axes , or i3+2 for the z axes
  
  }

  //but still nothings move, and that's because threejs need to be notified when a geometry attributes changes
  //so set the needsUpdate to true on the position attribute
  particleGeometry.attributes.position.needsUpdate = true

  //BUT REMEMBER THAT ANIMATE IN THIS WAY THE PARTICLES WILL AFFECT BAD THE PERFORMANCE
  //this actually render and update thounsands of particles everty frame and is not a good idea

  //but there is a better solution
  //Using a custom shader instead of using pointMaterial of threejs but it's and advanced and very hard topic
 

  controls.update()
  
  renderer.render(scene, camera)
}

onMounted(() => {
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value
  });
 
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) // in questo modo il max valore di pixel ratio utilizzaro è 2 (altrimenti devi renderizzare troppi pixel, dispiego enorme di potenza gpu)
  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true; // permette uno effetto smooth una volta che rilasciamo l'elemento, rallenta fino a fermarsi
});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}
</style>