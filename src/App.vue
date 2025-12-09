<template>
  <div class="wrapper">
    <div class="text-background">JOIN THE CULT</div>
    <div ref="container" class="three-container"></div>
  </div>
  <!-- <div style="height: 150vh; background-color: #FFE1EB;"></div> -->
</template>


<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { HDRLoader } from 'three/examples/jsm/loaders/HDRLoader'

const container = ref(null)

let scene, camera, renderer, heartModel, animationFrameId
let resizeObserver = null

// Muisrotatie variabelen
let mouseX = 0, mouseY = 0
let targetRotationX = 0, targetRotationY = 0

// Scroll rotatie variabele
let scrollRotation = 0
const SCROLL_FACTOR = 0.0025 // verhoog voor sterker effect (was 0.001)

// Klok voor animatie
let clock = new THREE.Clock()

// --- Helpers / event handlers ---

function onMouseMove(event) {
  if (!container.value) return
  const rect = container.value.getBoundingClientRect()

  mouseX = ((event.clientX - rect.left) / rect.width) * 2 - 1
  mouseY = -(((event.clientY - rect.top) / rect.height) * 2 - 1)

  targetRotationY = mouseX * 0.6
  targetRotationX = -mouseY * 0.6
}

let lastLoggedScrollY = -1
function onScroll() {
  // Gebruik documentElement.scrollTop als fallback
  const sy = window.scrollY ?? document.documentElement.scrollTop ?? 0
  scrollRotation = sy * SCROLL_FACTOR

  // Log alleen als verandering merkbaar is (vermijdt spam)
  if (Math.abs(sy - lastLoggedScrollY) > 10) {
    console.log('scrollY:', Math.round(sy), '-> scrollRotation:', scrollRotation.toFixed(4))
    lastLoggedScrollY = sy
  }
}

function onWheel(e) {
  // Fallback voor apparaten waar scroll events weird zijn
  // (hier niets veranderen behalve logging)
  // console.log('wheel delta', e.deltaY)
}

function onWindowResize() {
  if (!container.value || !camera || !renderer) return

  const width = container.value.clientWidth
  const height = container.value.clientHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()

  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2))
}

// --- Main mounted setup ---
onMounted(() => {
  scene = new THREE.Scene()

  const width = container.value.clientWidth || window.innerWidth
  const height = container.value.clientHeight || window.innerHeight

  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000)
  camera.position.set(0, 0, 6)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setSize(width, height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2))
  renderer.setClearColor(0x000000, 0)
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  // Tone mapping & exposure
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.0

  // Append renderer to container
  container.value.appendChild(renderer.domElement)

  // Lights
  const ambientLight = new THREE.AmbientLight(0xffffff, 3)
  scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 10)
  directionalLight.position.set(5, 10, 7)
  directionalLight.castShadow = true
  directionalLight.shadow.camera.left = -5
  directionalLight.shadow.camera.right = 5
  directionalLight.shadow.camera.top = 5
  directionalLight.shadow.camera.bottom = -5
  directionalLight.shadow.camera.near = 1
  directionalLight.shadow.camera.far = 20
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  scene.add(directionalLight)

  // Load HDR environment
  const hdrLoader = new HDRLoader()
  hdrLoader.load('/lighting.hdr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    texture.encoding = THREE.LinearEncoding
    scene.environment = texture

    if (heartModel) {
      heartModel.traverse((child) => {
        if (child.isMesh && child.material) {
          child.material.envMap = texture
          child.material.envMapIntensity = 5
          child.material.needsUpdate = true
        }
      })
    }
  }, undefined, (err) => {
    console.warn('HDR load error', err)
  })

  // Load model
  const loader = new GLTFLoader()
  loader.load(
    '/the_heart.glb',
    (gltf) => {
      heartModel = gltf.scene
      heartModel.scale.set(1.5, 1.5, 1.5)
      heartModel.castShadow = true
      heartModel.receiveShadow = true

      scene.add(heartModel)
    },
    undefined,
    (error) => {
      console.error('Error loading model:', error)
    }
  )

  // Event listeners
  window.addEventListener('mousemove', onMouseMove, { passive: true })
  const app = document.getElementById('app')
  app.addEventListener('scroll', onScroll, { passive: true })
  window.addEventListener('wheel', onWheel, { passive: true })
  
  function onScroll(e) {
  const sy = e.target.scrollTop
  scrollRotation = sy * SCROLL_FACTOR
//   console.log("SCROLLTOP:", sy)
}


  // ResizeObserver om betrouwbaar op container resize te reageren
  if (container.value && typeof ResizeObserver !== 'undefined') {
    resizeObserver = new ResizeObserver(() => onWindowResize())
    resizeObserver.observe(container.value)
  } else {
    window.addEventListener('resize', onWindowResize)
  }

  // Initial resize
  onWindowResize()

  // Animation loop
  function animate() {
    animationFrameId = requestAnimationFrame(animate)
    const elapsed = clock.getElapsedTime()

    if (heartModel) {
      // Combineer scroll en muis rotatie
      const combinedTargetY = scrollRotation + targetRotationY

      // Interpolatie (lerp) voor smooth beweging
      heartModel.rotation.y += (combinedTargetY - heartModel.rotation.y) * 0.06
      heartModel.rotation.x += (targetRotationX - heartModel.rotation.x) * 0.06

      // Subtiele zweef-animatie en scroll-vertical offset
      heartModel.position.x = Math.sin(elapsed * 0.5) * 0.05
      heartModel.position.y = Math.cos(elapsed * 0.7) * 0.05 
      heartModel.position.z = Math.sin(elapsed * 0.3) * 0.05

      // Pulsatie
      const beat1 = Math.sin(elapsed * 6) * 0.03
      const beat2 = Math.sin(elapsed * 2) * 0.015
      const pulse = 1 + Math.max(0, beat1) + Math.max(0, beat2)
      heartModel.scale.set(pulse, pulse, pulse)
    }

    renderer.render(scene, camera)
  }
  animate()
})

// Cleanup
onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrameId)
  window.removeEventListener('mousemove', onMouseMove)
  window.removeEventListener('scroll', onScroll)
  window.removeEventListener('wheel', onWheel)
  if (resizeObserver && container.value) resizeObserver.unobserve(container.value)
  else window.removeEventListener('resize', onWindowResize)
  if (renderer) renderer.dispose()
})
</script>

<style>
@font-face {
  font-family: 'Inter Tight';
  src: url('/InterTight.ttf') format('truetype');
  font-weight: 700;
  font-style: normal;
  font-display: swap;
}

html,
body,
#app {
  margin: 0;
  padding: 0;
  width: 100vw;
  height: 100vh;
  background-color: black;
  overflow-y: auto; /* zorg dat scrollen kan */
}

.wrapper {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: visible; /* zodat tekst niet clipped wordt */
}

.text-background {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
  font-family: 'Inter Tight', sans-serif;
  font-weight: 900;
  font-size: clamp(5vw, 12vw, 16vw);
  user-select: none;
  pointer-events: none;
  z-index: 0; /* achter canvas */
  white-space: nowrap;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  background: transparent;
  width: 100%;
  text-align: center;
}

.three-container {
  position: absolute;
  top: 0; left: 0;
  width: 100%;
  height: 100%;
  background-color: transparent;
  z-index: 1; /* boven tekst */
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

</style>








