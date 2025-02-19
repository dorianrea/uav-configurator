<script setup>
import { onMounted, ref } from 'vue';


defineProps({
  msg: String,
})

const count = ref(0)
const showUI = ref(false);
const showTitle = ref(false);

import * as THREE from 'three';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { UltraHDRLoader } from 'three/addons/loaders/UltraHDRLoader.js';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import CameraControls from 'camera-controls';
import { BloomEffect, DepthOfFieldEffect, KawaseBlurPass, EffectComposer, EffectPass, RenderPass, BoxBlurMaterial, SSAOEffect } from "postprocessing";
import { HalfFloatType } from "three";
// import * as POSTPROCESSING from "postprocessing"
// import { MotionBlurEffect, VelocityDepthNormalPass  } from "realism-effects"

CameraControls.install( { THREE: THREE } );

let canvas;

const clock = new THREE.Clock();

const params = {
				autoRotate: true,
				metalness: 1.0,
				roughness: 0.0,
				exposure: 1.0,
				resolution: '2k',
				type: 'HalfFloatType'
			};

  let camera, renderer, scene, controls, cameraControls,
  composer,
  prop, model, shadowPlane ;
  let drift = true;

  let bloomParams = {
    intensity: 1.5,
  };

async function buildLighting(){
  const directionalLight = new THREE.DirectionalLight( 0xffffff, 20 );
  directionalLight.position.set(0,10,0);
  // scene.add( directionalLight );
  
  const loader = new UltraHDRLoader();
  loader.setDataType( THREE.FloatType );

  const loadEnvironment = function ( resolution = '2k', type = 'HalfFloatType' ) {

    loader.setDataType( THREE[ type ] );

    loader.load( `./3d/textures/hdri/spruit_sunrise_${resolution}.hdr.jpg`, function ( texture ) {

      texture.mapping = THREE.EquirectangularReflectionMapping;
      texture.needsUpdate = true;

      // scene.background = texture;
      scene.background = new THREE.Color(0x333333);
      scene.environment = texture;

      console.log("Finished Building Lighting")
      addModels();
      return;

    } );

  };

  await loadEnvironment( params.resolution, params.type );
}

async function buildRenderer(){
  renderer = new THREE.WebGLRenderer({
    powerPreference: "high-performance",
    antialias: false,
    stencil: false,
    depth: false,
  });
  renderer.setSize( window.innerWidth, window.innerHeight );
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = params.exposure;
  // Enable shadow mapping
  renderer.shadowMap.enabled = true;

  composer = new EffectComposer(renderer, {
    frameBufferType: HalfFloatType
  });
  // composer = new POSTPROCESSING.EffectComposer(renderer)
  composer.addPass(new RenderPass(scene, camera));
  composer.addPass(new EffectPass(camera,

    // new DepthOfFieldEffect(camera, {
    //   bokehScale: 2,
    //   focusDistance: 0,
    //   focalLength: 0.05
    // }),
    // new SSAOEffect({

    // }),
    new BloomEffect({
      intensity: bloomParams.intensity,
    }), 


  ));
//   const velocityDepthNormalPass = new VelocityDepthNormalPass(scene, camera)
//   composer.addPass(velocityDepthNormalPass);
//   // Motion Blur
// const motionBlurEffect = new MotionBlurEffect(velocityDepthNormalPass);
//   const effectPass = new POSTPROCESSING.EffectPass(camera, motionBlurEffect);
//   composer.addPass(effectPass)

  onWindowResize();
  // renderer.setAnimationLoop( animate );

  canvas.appendChild( renderer.domElement );

  await buildLighting();

  console.log("Finished Building Renderer");
  return;
}

async function buildScene(){
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera( 45, window.innerWidth / window.innerHeight, 0.1, 1000 );

  await buildRenderer();
  

  cameraControls = new CameraControls( camera, renderer.domElement );
  cameraControls.addEventListener( 'controlend', ()=>{
    console.log("Camera Position: ", cameraControls.getPosition(), "Target: ", cameraControls.getTarget());
  });
  cameraControls.minDistance = 1;
  cameraControls.maxDistance = 100;
  cameraControls.setPosition(15,10,20, false);
  // cameraControls.disconnect();
  // controls = new OrbitControls( camera, renderer.domElement );
  

  console.log("Finished Building Scene");
  return;
}

function addModels(){
  // Optional: Provide a DRACOLoader instance to decode compressed mesh data
  const dracoLoader = new DRACOLoader();
      dracoLoader.setDecoderPath( '/examples/jsm/libs/draco/' );
      dracoLoader.setDecoderConfig({ type: 'js' });

      // Instantiate a loader
      const loader = new GLTFLoader();
      loader.setDRACOLoader( dracoLoader );

      // Load a glTF resource
      loader.load(
        // resource URL
        './3d/models/MQ_9_Reaper_final.glb',
        // called when the resource is loaded
        function ( gltf ) {
          gltf.scene.scale.setScalar(2.54);
          model = gltf.scene;
          animate();
          scene.add( gltf.scene );
          gltf.scene.traverse((model)=>{
            if(!model.isObject3D)return;
            if(!model.isMesh)return;
            // console.log(model.name);
            model.castShadow = true;
            model.receiveShadow = true;
            switch (model.name) {
              case "Prop_Cone004":
                prop = model;
                break;
            
              default:
                break;
            }
            // model.traverse((child)=>{
            //   console.log(chi);

            // })
          })
          // cameraControls.fitToBox(gltf.scene, false);

          // gltf.animations; // Array<THREE.AnimationClip>
          // gltf.scene; // THREE.Group
          // gltf.scenes; // Array<THREE.Group>
          // gltf.cameras; // Array<THREE.Camera>
          // gltf.asset; // Object
          
          //Here we can wait for the UAV to fly in
          setTimeout(() => {
            showTitle.value = showUI.value = true;
          }, 500);

        },
        // called while loading is progressing
        function ( xhr ) {

          // console.log( ( xhr.loaded / xhr.total * 100 ) + '% loaded' );

        },
        // called when loading has errors
        function ( error ) {

          console.log( 'An error happened' );

        }
    );

  console.log("Finished Adding Models");
  return;
}


async function init(){
  await buildScene();

  // Create cloud texture
  const cloudTexture = new THREE.TextureLoader().load('./3d/textures/Cloud-PNG-Cutout-Architecture-Photoshop_03.png');
  cloudTexture.wrapS = cloudTexture.wrapT = THREE.RepeatWrapping;

  // Create a large plane for cloud shadows
  shadowPlane = new THREE.Mesh(
    new THREE.PlaneGeometry(1000, 1000),
    new THREE.MeshBasicMaterial({ 
      map: cloudTexture,
      transparent: true,
      opacity: 0.5,

    })
  );
  shadowPlane.rotation.x = -Math.PI / 2;
  shadowPlane.position.y = 10; // Position above the model
  shadowPlane.castShadow = true;
  // scene.add(shadowPlane);

  const geometry = new THREE.BoxGeometry( 65, 12, 36 );
  const material = new THREE.MeshBasicMaterial( { color: 0x00ff00, wireframe:true } );
  const cube = new THREE.Mesh( geometry, material );
  // scene.add( cube );
}

function setCanvasDimensions(canvas, width, height) {
  const ratio = window.devicePixelRatio;
  canvas.width = width * ratio;
  canvas.height = height * ratio;
  canvas.style.width = `${width}px`;
  canvas.style.height = `${height}px`;
}

function onWindowResize() {
  const width = window.innerWidth;
  const height = window.innerHeight;

  camera.aspect = width / height;
  camera.updateProjectionMatrix();

  renderer.setSize(width, height);
  setCanvasDimensions(renderer.domElement, width, height);
}


window.addEventListener('resize', onWindowResize, false);


function animate() {
  requestAnimationFrame( animate );
	// cube.rotation.x += 0.01;
	// cube.rotation.y += 0.01;
  // snip
	const delta = clock.getDelta();
  const time = performance.now() * 0.001;
	// const hasControlsUpdated = 
  cameraControls.update( delta );
  // controls.update();

	// renderer.render( scene, camera );
  if(model != null && drift){
    model.position.y = Math.sin(time * 1.0) * 0.25 + Math.sin(time * 1.5) * 0.125 + Math.sin(time * 2.0) * 0.075;
    model.rotation.z = Math.sin(time * 0.5) * 0.01 + Math.sin(time * 0.75) * 0.025 + Math.sin(time * 1.0) * 0.0125;
  }

  if(prop != null){
    prop.rotation.z -= 0.5;
  }

  shadowPlane.material.map.offset.set(0, time * 0.01);

  composer.render();

}

onMounted(()=>{
  canvas = document.getElementById("three-canvas");
  init();




});

// window.addEventListener('keydown', function(event) {
//   if (event.code === 'Space') {
//     event.preventDefault(); // Prevent default spacebar behavior (e.g., scrolling)
//     console.log('Spacebar pressed', showUI.value);
//     showUI.value = !showUI.value;
//     // Your code here
//   }
// });


</script>

<template>

  <v-row no-gutters style="height: 100vh; width: 100vw; overflow: hidden;">
    <v-slide-y-transition><v-row v-if="showTitle" class="title" :style="{ visibility: showTitle ? 'visible' : 'hidden'}">MQ-9 Reaper</v-row></v-slide-y-transition>
    <div id="three-canvas"></div>
    <v-row class="my-4" align="center" justify="space-between">
      <!-- <h2 class="text-h4 font-weight-bold">MQ-9 Reaper</h2> -->
    </v-row>
    <v-col style="margin: 0; padding: 0; z-index: 1;" cols="1">
      <v-col style="margin: 0; padding: 0; z-index: 1;" cols="4">
        <v-slide-x-transition>
          <v-container style="margin: 0; padding: 0;" v-show="showUI"  id="ui-container">
            <!-- UI Stuff -->

          </v-container>
        </v-slide-x-transition>
      </v-col>
    </v-col>
  </v-row>
  
  <!-- <h1>{{ msg }}</h1>

  <div class="card">
    <button type="button" @click="count++">count is {{ count }}</button>
    <p>
      Edit
      <code>components/HelloWorld.vue</code> to test HMR
    </p>
  </div>

  <p>
    Check out
    <a href="https://vuejs.org/guide/quick-start.html#local" target="_blank"
      >create-vue</a
    >, the official Vue + Vite starter
  </p>
  <p>
    Learn more about IDE Support for Vue in the
    <a
      href="https://vuejs.org/guide/scaling-up/tooling.html#ide-support"
      target="_blank"
      >Vue Docs Scaling up Guide</a
    >.
  </p>
  <p class="read-the-docs">Click on the Vite and Vue logos to learn more</p> -->
</template>

<style scoped>
#three-canvas{
  width: 100vw;
  height: 100vh;
  position: fixed;
  top: 0;
  left: 0;
  z-index: 0;
}

#ui-container{
  background-color: rgba(0,0,0,0.3);
  z-index: 1;
  height: 100vh;
  overflow: hidden;
}

.title{
  z-index: 2;
  position: absolute;
  height: 10vh;
  width: 100vw;
  justify-content: center;
  margin-top: 2vh;
  font-size: 3em;
  /* font-family: ; */
}

</style>
