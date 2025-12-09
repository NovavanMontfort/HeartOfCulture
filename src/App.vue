<template>
  <div class="wrapper">
    <!-- 3D hart canvas -->
    <div
  ref="container"
  class="three-container"
  :style="{ opacity: showHeart ? 1 : 0, transition: 'opacity 0.1s ease' }" 
></div>
<!-- snelheid van inladen -->

<div
  class="text-background"
  :style="{ opacity: showHeart ? 1 : 0, transition: 'opacity 0.1s ease' }"
>
  JOIN THE CULT
</div>


    <!-- Kinetische tekst animatie bovenop alles -->
    <div ref="kineticType" id="kinetic-type">
      <div
        v-for="n in 20"
        :key="n"
        :class="n % 2 === 1 ? 'type-line odd' : 'type-line even'"
      >
        CULT JOIN THE CULT JOIN THE CULT JOIN THE CULT JOIN THE CULT JOIN THE CULT
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from "vue";
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { HDRLoader } from "three/examples/jsm/loaders/HDRLoader";
import gsap from "gsap";
import { CustomEase } from "gsap/CustomEase";


const showHeart = ref(false)

gsap.registerPlugin(CustomEase);

const container = ref(null);
const kineticType = ref(null);

let scene, camera, renderer, heartModel, animationFrameId;
let resizeObserver = null;

// Muisrotatie variabelen
let mouseX = 0,
  mouseY = 0;
let targetRotationX = 0,
  targetRotationY = 0;

// Scroll rotatie variabele
let scrollRotation = 0;
const SCROLL_FACTOR = 0.0025;

// Klok voor animatie
let clock = new THREE.Clock();

// --- Helpers / event handlers ---
function onMouseMove(event) {
  if (!container.value) return;
  const rect = container.value.getBoundingClientRect();
  mouseX = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  mouseY = -(((event.clientY - rect.top) / rect.height) * 2 - 1);
  targetRotationY = mouseX * 0.6;
  targetRotationX = -mouseY * 0.6;
}

let lastLoggedScrollY = -1;
function onScroll(e) {
  // documentElement.scrollTop fallback als nodig
  const sy = e.target?.scrollTop ?? window.scrollY ?? document.documentElement.scrollTop ?? 0;
  scrollRotation = sy * SCROLL_FACTOR;

  if (Math.abs(sy - lastLoggedScrollY) > 10) {
    console.log("scrollY:", Math.round(sy), "-> scrollRotation:", scrollRotation.toFixed(4));
    lastLoggedScrollY = sy;
  }
}

function onWheel(e) {
  // Kan leeg blijven, behoud event voor eventuele fallback
}

// Window resize handler
function onWindowResize() {
  if (!container.value || !camera || !renderer) return;
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
}

// --- Mounted setup ---
onMounted(() => {
  // Setup ThreeJS scene
  scene = new THREE.Scene();

  const width = container.value.clientWidth || window.innerWidth;
  const height = container.value.clientHeight || window.innerHeight;

  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
  camera.position.set(0, 0, 6);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
  renderer.setClearColor(0x000000, 0);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  // Tone mapping & exposure
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.0;

  container.value.appendChild(renderer.domElement);

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 3);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 10);
  directionalLight.position.set(5, 10, 7);
  directionalLight.castShadow = true;
  directionalLight.shadow.camera.left = -5;
  directionalLight.shadow.camera.right = 5;
  directionalLight.shadow.camera.top = 5;
  directionalLight.shadow.camera.bottom = -5;
  directionalLight.shadow.camera.near = 1;
  directionalLight.shadow.camera.far = 20;
  directionalLight.shadow.mapSize.width = 2048;
  directionalLight.shadow.mapSize.height = 2048;
  scene.add(directionalLight);

  // Load HDR environment
  const hdrLoader = new HDRLoader();
  hdrLoader.load(
    "/lighting.hdr",
    (texture) => {
      texture.mapping = THREE.EquirectangularReflectionMapping;
      texture.encoding = THREE.LinearEncoding;
      scene.environment = texture;
      if (heartModel) {
        heartModel.traverse((child) => {
          if (child.isMesh && child.material) {
            child.material.envMap = texture;
            child.material.envMapIntensity = 5;
            child.material.needsUpdate = true;
          }
        });
      }
    },
    undefined,
    (err) => {
      console.warn("HDR load error", err);
    }
  );

  // Load 3D model
  const loader = new GLTFLoader();
  loader.load(
    "/the_heart.glb",
    (gltf) => {
      heartModel = gltf.scene;
      heartModel.scale.set(1.5, 1.5, 1.5);
      heartModel.castShadow = true;
      heartModel.receiveShadow = true;
      scene.add(heartModel);
    },
    undefined,
    (error) => {
      console.error("Error loading model:", error);
    }
  );

  // Event listeners for interaction
  window.addEventListener("mousemove", onMouseMove, { passive: true });
  const app = document.getElementById("app");
  if (app) app.addEventListener("scroll", onScroll, { passive: true });
  window.addEventListener("wheel", onWheel, { passive: true });

  // ResizeObserver voor container resize
  if (container.value && typeof ResizeObserver !== "undefined") {
    resizeObserver = new ResizeObserver(() => onWindowResize());
    resizeObserver.observe(container.value);
  } else {
    window.addEventListener("resize", onWindowResize);
  }

  // Initial resize call
  onWindowResize();

  // Animation loop voor 3D hart
  function animate() {
    animationFrameId = requestAnimationFrame(animate);
    const elapsed = clock.getElapsedTime();

    if (heartModel) {
      const combinedTargetY = scrollRotation + targetRotationY;
      heartModel.rotation.y += (combinedTargetY - heartModel.rotation.y) * 0.06;
      heartModel.rotation.x += (targetRotationX - heartModel.rotation.x) * 0.06;

      heartModel.position.x = Math.sin(elapsed * 0.5) * 0.05;
      heartModel.position.y = Math.cos(elapsed * 0.7) * 0.05;
      heartModel.position.z = Math.sin(elapsed * 0.3) * 0.05;

      const beat1 = Math.sin(elapsed * 6) * 0.03;
      const beat2 = Math.sin(elapsed * 2) * 0.015;
      const pulse = 1 + Math.max(0, beat1) + Math.max(0, beat2);
      heartModel.scale.set(pulse, pulse, pulse);
    }

    renderer.render(scene, camera);
  }
  animate();

  // --- GSAP kinetic text animatie ---
  CustomEase.create("customEase", "0.86, 0, 0.07, 1");

const kineticEl = kineticType.value;
const oddLines = kineticEl.querySelectorAll(".odd");
const evenLines = kineticEl.querySelectorAll(".even");
const TYPE_LINE_OPACITY = 0.99;

gsap.set(oddLines, { opacity: TYPE_LINE_OPACITY });
gsap.set(evenLines, { opacity: TYPE_LINE_OPACITY });

const timeline = gsap.timeline({ repeat: 0, defaults: { ease: "customEase" } }); // speel 1x af
timeline
  .to(kineticEl, { duration: 1.4, scale: 2.3, rotation: -90 })
  .to(oddLines, { x: "20%", duration: 1, stagger: 0.08 }, "<")
  .to(evenLines, { x: "-20%", duration: 1, stagger: 0.08 }, "<")
  .to(oddLines, { x: "-200%", duration: 1.5, stagger: 0.08 }, ">")
  .to(evenLines, { x: "200%", duration: 1.5, stagger: 0.08 }, "<")
  .to(kineticEl, { opacity: 0, duration: 1.5 })
  .set(kineticEl, { rotation: 0, scale: 1, x: 0, opacity: 1 })
  .eventCallback("onComplete", () => {
    showHeart.value = true;
  });


  // --- Cleanup ---
  onBeforeUnmount(() => {
    cancelAnimationFrame(animationFrameId);
    window.removeEventListener("mousemove", onMouseMove);
    if (app) app.removeEventListener("scroll", onScroll);
    window.removeEventListener("wheel", onWheel);

    if (resizeObserver && container.value) resizeObserver.unobserve(container.value);
    else window.removeEventListener("resize", onWindowResize);

    if (renderer) {
      renderer.dispose();
      if (renderer.domElement && renderer.domElement.parentNode)
        renderer.domElement.parentNode.removeChild(renderer.domElement);
      renderer.forceContextLoss();
      renderer.context = null;
      renderer.domElement = null;
    }
    // GSAP timeline stopt vanzelf met repeat: -1, maar wil je nog explicit killen dan moet je refen naar timeline
  });
});
</script>

<style>
@font-face {
  font-family: "Inter Tight";
  src: url("/InterTight.ttf") format("truetype");
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}

/* Algemene pagina en wrapper */
html,
body,
#app {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  background-color: black;
  overflow-y: auto; /* Scroll moet kunnen */
}

.wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: visible; /* zodat tekst niet clipped wordt */
  background-color: black;
}

/* 3D canvas container */
.three-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: transparent;
  z-index: 1; /* Onder tekst */
  display: flex;
  justify-content: center;
  align-items: center;
}

canvas {
  display: block;
  width: 100%;
  height: 100%;
  background-color: transparent;
}

/* Statistische tekst achter kinetic */
.text-background {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-family: "Inter Tight", sans-serif;
  font-weight: 900;
  font-size: clamp(5vw, 12vw, 16vw);
  user-select: none;
  pointer-events: none;
  z-index: 2; /* boven 3D canvas */
  white-space: nowrap;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  background: transparent;
  width: 100%;
  text-align: center;
}

/* Kinetische tekst animatie boven alles */
#kinetic-type {
  position: fixed;
  inset: 0;
  z-index: 10;
  pointer-events: none;
  font-family: "Inter Tight", sans-serif;
  color: white;
  user-select: none;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  will-change: transform, opacity;
  backface-visibility: hidden;
}

/* Kinetische tekst lijnen */
.type-line {
  display: block;
  width: max-content;
  white-space: nowrap;
  margin: 0;
  padding: 0;
  line-height: 0.8;
  font-weight: 800;
  color: #fff;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  letter-spacing: 0.06em;
  will-change: transform, opacity;
  opacity: 1 !important;
}

.type-line.odd {
  color: #fff;
}

.type-line.even {
  color: #f2f2f2;
}

#kinetic-type > .type-line {
  padding-top: 0;
  padding-bottom: 0;
  margin: 0;
}

/* Fade gradients boven en onder kinetic text */
#kinetic-type::before,
#kinetic-type::after {
  content: "";
  position: absolute;
  left: 0;
  right: 0;
  height: 8vh;
  pointer-events: none;
  z-index: 20;
}
#kinetic-type::before {
  top: 0;
  background: linear-gradient(to bottom, #000 0%, transparent 100%);
}
#kinetic-type::after {
  bottom: 0;
  background: linear-gradient(to top, #000 0%, transparent 100%);
}

/* Responsive font sizes voor kinetic text */
@media (min-width: 1400px) {
  #kinetic-type {
    font-size: clamp(3rem, 8vw, 12rem);
  }
}
@media (max-width: 1399px) {
  #kinetic-type {
    font-size: clamp(2.5rem, 9vw, 10.5rem);
  }
}
@media (max-width: 768px) {
  #kinetic-type {
    font-size: clamp(1.6rem, 8.5vw, 6rem);
  }
  #kinetic-type > .type-line {
    line-height: 0.85;
  }
}
</style>










