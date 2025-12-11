<template>
  <div class="wrapper" id="app">
    <!-- 3D hart canvas -->
    <div
      ref="container"
      class="three-container"
      :style="{ opacity: showHeart ? 1 : 0, transition: 'opacity 0.1s ease' }"
    ></div>

    <!-- Achtergrond tekstjes (zichtbaar ná kinetic text animatie) -->
    <div class="background-text-container" v-if="showHeart">
      <div
        v-for="item in textItems"
        :key="item.id"
        class="text-item"
        :style="itemStyle(item)"
        :class="{ highlight: item.highlighted || (autoHighlightActive && !userHoveringId) }"
        @mouseenter="onHover(item.id)"
        @mouseleave="onLeave(item.id)"
        :data-text="item.text"
        :ref="setTextItemRef"

      >
        {{ item.text }}
      </div>
    </div>

    <!-- Kinetische tekst animatie bovenop alles -->
    <div ref="kineticType" id="kinetic-type" v-if="!showHeart">
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
/* Full merged logic. Nothing of the core animations has been altered.
   You asked to fill textItems yourself, so there's a placeholder below. */

import { ref, reactive, onMounted, onBeforeUnmount, nextTick } from "vue";
import * as THREE from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { HDRLoader } from "three/examples/jsm/loaders/HDRLoader";
import gsap from "gsap";
import { CustomEase } from "gsap/CustomEase";
import { ScrambleTextPlugin } from "gsap/ScrambleTextPlugin";

gsap.registerPlugin(CustomEase, ScrambleTextPlugin);

const showHeart = ref(false);
const container = ref(null);
const kineticType = ref(null);

// ---------- BACKGROUND TEXTS (you fill textItems yourself) ----------
const textItems = reactive([
  { id: 1, text: "CREATE", top: "5%", left: "8%", right: null, highlighted: false, scrambleTl: null },
  { id: 2, text: "INSPIRE", top: "5%", left: "15%", right: null, highlighted: false, scrambleTl: null },
  { id: 3, text: "IMAGINE", top: "5%", left: "28%", right: null, highlighted: false, scrambleTl: null },
  { id: 4, text: "CONNECT", top: "5%", left: "42%", right: null, highlighted: false, scrambleTl: null },
  { id: 5, text: "FEEL", top: "5%", left: "55%", right: null, highlighted: false, scrambleTl: null },
  { id: 6, text: "&", top: "5%", left: "75%", right: null, highlighted: false, scrambleTl: null },
  { id: 7, text: "EXPLORE", top: "5%", left: "85%", right: null, highlighted: false, scrambleTl: null },
  { id: 8, text: "STAGE", top: "10%", left: "12%", right: null, highlighted: false, scrambleTl: null },
  { id: 9, text: "ART", top: "10%", left: "45%", right: null, highlighted: false, scrambleTl: null },
  { id: 10, text: "CULTURE", top: "10%", left: null, right: "20%", highlighted: false, scrambleTl: null },
  { id: 11, text: "THE", top: "15%", left: "8%", right: null, highlighted: false, scrambleTl: null },
  { id: 12, text: "CULTURE", top: "15%", left: "30%", right: null, highlighted: false, scrambleTl: null },
  { id: 13, text: "LAB", top: "15%", left: "55%", right: null, highlighted: false, scrambleTl: null },
  { id: 14, text: "EMBRACES", top: "15%", left: null, right: "20%", highlighted: false, scrambleTl: null },
  { id: 15, text: "INNOVATION", top: "15%", left: null, right: "5%", highlighted: false, scrambleTl: null },
  { id: 16, text: "P", top: "25%", left: "5%", right: null, highlighted: false, scrambleTl: null },
  { id: 17, text: "A", top: "25%", left: "10%", right: null, highlighted: false, scrambleTl: null },
  { id: 18, text: "S", top: "25%", left: "15%", right: null, highlighted: false, scrambleTl: null },
  { id: 19, text: "S", top: "25%", left: "20%", right: null, highlighted: false, scrambleTl: null },
  { id: 20, text: "I", top: "25%", left: "25%", right: null, highlighted: false, scrambleTl: null },
  { id: 21, text: "O", top: "25%", left: "30%", right: null, highlighted: false, scrambleTl: null },
  { id: 22, text: "N", top: "25%", left: "35%", right: null, highlighted: false, scrambleTl: null },
  { id: 23, text: "EXPRESSION", top: "25%", left: "40%", right: null, highlighted: false, scrambleTl: null },
  { id: 24, text: "IS", top: "25%", left: "45%", right: null, highlighted: false, scrambleTl: null },
  { id: 25, text: "THE", top: "25%", left: "50%", right: null, highlighted: false, scrambleTl: null },
  { id: 26, text: "KEY", top: "25%", left: null, right: "5%", highlighted: false, scrambleTl: null },
  { id: 27, text: "FIND YOUR VOICE", top: "35%", left: "25%", right: null, highlighted: false, scrambleTl: null },
  { id: 28, text: "TRUST INTUITION", top: "35%", left: "65%", right: null, highlighted: false, scrambleTl: null },
  { id: 29, text: "EMBRACE SILENCE", top: "50%", left: "5%", right: null, highlighted: false, scrambleTl: null },
  { id: 30, text: "QUESTION EVERYTHING", top: "50%", left: null, right: "5%", highlighted: false, scrambleTl: null },
  { id: 31, text: "TRUTH", top: "75%", left: "20%", right: null, highlighted: false, scrambleTl: null },
  { id: 32, text: "WISDOM", top: "75%", left: null, right: "20%", highlighted: false, scrambleTl: null },
  { id: 33, text: "FOCUS", top: "80%", left: "10%", right: null, highlighted: false, scrambleTl: null },
  { id: 34, text: "ATTENTION", top: "80%", left: "35%", right: null, highlighted: false, scrambleTl: null },
  { id: 35, text: "AWARENESS", top: "80%", left: "65%", right: null, highlighted: false, scrambleTl: null },
  { id: 36, text: "PRESENCE", top: "80%", left: null, right: "10%", highlighted: false, scrambleTl: null },
  { id: 37, text: "S", top: "85%", left: "25%", right: null, highlighted: false, scrambleTl: null },
  { id: 38, text: "I", top: "85%", left: null, right: "25%", highlighted: false, scrambleTl: null },
]);

const autoHighlightActive = ref(false);
const userHoveringId = ref(null);
const textItemRefs = ref([]); // will hold DOM elements
let scrambleIntervals = [];
let autoHighlightInterval = null;
let autoHighlightTimeout = null;

// helper used as ref setter in template to collect elements
function setTextItemRef(el) {
  if (!el) return;
  // Avoid duplicates
  if (!textItemRefs.value.includes(el)) textItemRefs.value.push(el);
}

// Position scaling for responsive layout
function scalePositions() {
  const width = window.innerWidth;
  textItems.forEach((item) => {
    const baseTop = parseFloat(item.top) || 0;
    const baseLeft = item.left ? parseFloat(item.left) : null;
    const baseRight = item.right ? parseFloat(item.right) : null;

    if (width < 480) {
      item._scaledTop = `${baseTop}%`;
      item._scaledLeft = baseLeft !== null ? `${baseLeft}%` : null;
      item._scaledRight = baseRight !== null ? `${baseRight}%` : null;
    } else if (width < 768) {
      item._scaledTop = `${baseTop * 1.1}%`;
      item._scaledLeft = baseLeft !== null ? `${baseLeft * 0.9 + 5}%` : null;
      item._scaledRight = baseRight !== null ? `${baseRight * 0.9 + 5}%` : null;
    } else {
      item._scaledTop = `${baseTop}%`;
      item._scaledLeft = baseLeft !== null ? `${baseLeft}%` : null;
      item._scaledRight = baseRight !== null ? `${baseRight}%` : null;
    }
  });
}

function itemStyle(item) {
  return {
    position: "absolute",
    top: item._scaledTop || item.top,
    left: item._scaledLeft || item.left,
    right: item._scaledRight || item.right,
  };
}

function startScramble(item, el) {
  if (item.scrambleTl) return;
  // Use GSAP ScrambleText on the DOM element
  item.scrambleTl = gsap.to(el, {
    duration: 1.2,
    scrambleText: {
      text: item.text,
      chars: "■▪▌▐▬",
      revealDelay: 0.5,
      speed: 0.2,
    },
    ease: "none",
    repeat: -1,
    yoyo: true,
    repeatDelay: 2,
  });
}

function stopScramble(item) {
  if (item.scrambleTl) {
    item.scrambleTl.kill();
    item.scrambleTl = null;
  }
}

function onHover(id) {
  userHoveringId.value = id;
  clearAutoHighlight();

  const item = textItems.find((i) => i.id === id);
  if (!item) return;

  // find the corresponding DOM element by data-text (safe if texts are unique)
  const el = textItemRefs.value.find((el) => el.dataset.text === item.text);
  if (el) {
    item.highlighted = true;
    startScramble(item, el);
  }
}

function onLeave(id) {
  userHoveringId.value = null;
  const item = textItems.find((i) => i.id === id);
  if (item) {
    item.highlighted = false;
    stopScramble(item);
  }
  startAutoHighlight();
}

function startAutoHighlight() {
  if (autoHighlightInterval) return;
  autoHighlightInterval = setInterval(() => {
    autoHighlightActive.value = true;
    autoHighlightTimeout = setTimeout(() => {
      autoHighlightActive.value = false;
    }, 1700);
  }, 5000);
}

function clearAutoHighlight() {
  autoHighlightActive.value = false;
  if (autoHighlightInterval) {
    clearInterval(autoHighlightInterval);
    autoHighlightInterval = null;
  }
  if (autoHighlightTimeout) {
    clearTimeout(autoHighlightTimeout);
    autoHighlightTimeout = null;
  }
}

function startRandomScrambles() {
  // clear any previous
  scrambleIntervals.forEach((i) => clearInterval(i));
  scrambleIntervals = [];

  textItems.forEach((item) => {
    const el = textItemRefs.value.find((el) => el.dataset.text === item.text);
    if (!el) return;

    const interval = setInterval(() => {
      if (userHoveringId.value === item.id) return;
      startScramble(item, el);
      setTimeout(() => {
        stopScramble(item);
      }, 1000 + Math.random() * 2000);
    }, 4000 + Math.random() * 8000);

    scrambleIntervals.push(interval);
  });
}

function stopAllScrambles() {
  textItems.forEach((item) => stopScramble(item));
  scrambleIntervals.forEach((i) => clearInterval(i));
  scrambleIntervals = [];
}

// ---------- THREE.JS + kinetic GSAP (originals, unchanged) ----------

let scene, camera, renderer, heartModel, animationFrameId;
let resizeObserver = null;

let mouseX = 0,
  mouseY = 0;
let targetRotationX = 0,
  targetRotationY = 0;

let scrollRotation = 0;
const SCROLL_FACTOR = 0.0025;
let clock = new THREE.Clock();

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
  const sy = e.target?.scrollTop ?? window.scrollY ?? document.documentElement.scrollTop ?? 0;
  scrollRotation = sy * SCROLL_FACTOR;

  if (Math.abs(sy - lastLoggedScrollY) > 10) {
    console.log("scrollY:", Math.round(sy), "-> scrollRotation:", scrollRotation.toFixed(4));
    lastLoggedScrollY = sy;
  }
}

function onWheel(e) {
  // reserved for future fallback
}

function onWindowResize() {
  if (!container.value || !camera || !renderer) return;
  const width = container.value.clientWidth;
  const height = container.value.clientHeight;
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
}

let timeline = null;

onMounted(() => {
  // THREE.JS scene setup
  scene = new THREE.Scene();

  const width = container.value?.clientWidth || window.innerWidth;
  const height = container.value?.clientHeight || window.innerHeight;

  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000);
  camera.position.set(0, 0, 6);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(width, height);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));
  renderer.setClearColor(0x000000, 0);
  renderer.shadowMap.enabled = true;
  renderer.shadowMap.type = THREE.PCFSoftShadowMap;

  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  renderer.toneMappingExposure = 1.0;

  container.value.appendChild(renderer.domElement);

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

  window.addEventListener("mousemove", onMouseMove, { passive: true });
  const appEl = document.getElementById("app");
  if (appEl) appEl.addEventListener("scroll", onScroll, { passive: true });
  window.addEventListener("wheel", onWheel, { passive: true });

  if (container.value && typeof ResizeObserver !== "undefined") {
    resizeObserver = new ResizeObserver(() => onWindowResize());
    resizeObserver.observe(container.value);
  } else {
    window.addEventListener("resize", onWindowResize);
  }

  onWindowResize();

  // Animation loop for 3D heart
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

  // --- GSAP kinetic text animatie (unchanged) ---
  CustomEase.create("customEase", "0.86, 0, 0.07, 1");

  const kineticEl = kineticType.value;
  const oddLines = kineticEl?.querySelectorAll(".odd") || [];
  const evenLines = kineticEl?.querySelectorAll(".even") || [];
  const TYPE_LINE_OPACITY = 0.99;

  gsap.set(oddLines, { opacity: TYPE_LINE_OPACITY });
  gsap.set(evenLines, { opacity: TYPE_LINE_OPACITY });

  timeline = gsap.timeline({ repeat: 0, defaults: { ease: "customEase" } });
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

      // wait nextTick so background items mount and refs collect
      nextTick(() => {
        scalePositions();
        startAutoHighlight();
        startRandomScrambles();
      });
    });
});

// Cleanup
onBeforeUnmount(() => {
  // 3D cleanup
  cancelAnimationFrame(animationFrameId);
  window.removeEventListener("mousemove", onMouseMove);
  const appEl = document.getElementById("app");
  if (appEl) appEl.removeEventListener("scroll", onScroll);
  window.removeEventListener("wheel", onWheel);

  if (resizeObserver && container.value) resizeObserver.unobserve(container.value);
  else window.removeEventListener("resize", onWindowResize);

  if (renderer) {
    try {
      renderer.dispose();
      if (renderer.domElement && renderer.domElement.parentNode)
        renderer.domElement.parentNode.removeChild(renderer.domElement);
      // clear WebGL context references
      renderer.forceContextLoss && renderer.forceContextLoss();
      renderer.context = null;
      renderer.domElement = null;
    } catch (e) {
      // ignore
    }
  }

  // GSAP & background cleanup
  clearAutoHighlight();
  stopAllScrambles();
  if (timeline) timeline.kill();
});
</script>

<style>
@font-face {
  font-family: "Inter Tight";
  src: url("/InterTight.ttf") format("truetype");
  /* font-weight: 700; */
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

/* Achtergrond tekstjes container */
.background-text-container {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 0; /* Tussen 3D canvas (1) en kinetic text (10) */
  user-select: none;
}

/* Achtergrond tekstjes */
.text-item {
  color: #ffe1eb;
  font-size: clamp(0.4rem, 0.9vw, 0.8rem);
  text-transform: uppercase;
  opacity: 1;
  white-space: nowrap;
  font-family: "Inter Tight", sans-serif;
  font-weight: 400;
  z-index: 1;
  cursor: pointer;
  user-select: none;
  position: relative;
  transition: color 0.3s ease;
  pointer-events: auto; /* Hover events mogen */
}

.text-item::after {
  content: "";
  position: absolute;
  top: -2px;
  left: -4px;
  width: 0;
  height: calc(100% + 4px);
  background-color: #ffe1eb; /* tc8l kleur = #ffe1eb */
  z-index: -1;
  transition: width 0.5s cubic-bezier(0.34, 1.56, 0.64, 1);
  will-change: width;
  pointer-events: none;
}

.text-item.highlight {
  color: #000000 !important;
}

.text-item.highlight::after {
  width: calc(100% + 8px) !important;
  transition-timing-function: cubic-bezier(0.34, 1.56, 0.64, 1) !important;
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

/* Small screens text-item adjustments */
@media (max-width: 480px) {
  .text-item {
    font-size: clamp(0.5rem, 2vw, 0.9rem);
  }
}
</style>
