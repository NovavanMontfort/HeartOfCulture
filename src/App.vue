<template>
  <div ref="container" class="three-container"></div>
  <div style="height: 150vh; background-color: #FFE1EB;"></div> <!-- voor scroll -->
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { EXRLoader } from 'three/examples/jsm/loaders/EXRLoader'

const container = ref(null)

let scene, camera, renderer, heartModel, animationFrameId

// Muisrotatie variabelen
let mouseX = 0, mouseY = 0
let targetRotationX = 0, targetRotationY = 0

// Scroll rotatie variabele
let scrollRotation = 0

let clock = new THREE.Clock()

function onMouseMove(event) {
  mouseX = (event.clientX / window.innerWidth) * 2 - 1
  mouseY = -((event.clientY / window.innerHeight) * 2 - 1)

  targetRotationY = mouseX * 0.6
  targetRotationX = -mouseY * 0.6 
}

function onScroll() {
  scrollRotation = window.scrollY * 0.001  // Pas factor aan voor rotatie snelheid
}

function onWindowResize() {
  const width = window.innerWidth
  const height = window.innerHeight

  camera.aspect = width / height
  camera.updateProjectionMatrix()

  renderer.setSize(width, height)
}

onMounted(() => {
  scene = new THREE.Scene()

  const width = window.innerWidth
  const height = window.innerHeight

  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000)
  camera.position.set(0, 0, 6)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setSize(width, height)
  renderer.setClearColor(0x000000, 0) // transparant
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap

  // Tone mapping & exposure voor realistischere HDR effecten
  renderer.toneMapping = THREE.ACESFilmicToneMapping
  renderer.toneMappingExposure = 1.0 // Speel hiermee voor helderheid

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

  const pointLight = new THREE.PointLight(0xffffff, 0.4)
  pointLight.position.set(-3, 3, 5)
  scene.add(pointLight)

  // HDRI environment map laden en instellen
  const exrLoader = new EXRLoader()
  exrLoader.load('/lighting.exr', (texture) => {
    texture.mapping = THREE.EquirectangularReflectionMapping
    scene.environment = texture
    // scene.background = texture // NIET zetten, want geen achtergrond

    // Als model al geladen is, update dan materialen met envMap
    if (heartModel) {
      heartModel.traverse((child) => {
        if (child.isMesh && child.material) {
          child.material.envMap = texture
          child.material.envMapIntensity = 5
          child.material.needsUpdate = true
        }
      })
    }
  })

  // Model loader
  const loader = new GLTFLoader()
  loader.load(
    '/the_heart.glb',
    gltf => {
      heartModel = gltf.scene
      heartModel.scale.set(1.5, 1.5, 1.5)
      heartModel.castShadow = true
      heartModel.receiveShadow = true

      // Zodra de HDRI geladen is (of zodra deze geladen wordt),
      // kun je hier ook envMap aan materialen koppelen,
      // maar dat is beter in de exrLoader callback hierboven

      scene.add(heartModel)
    },
    undefined,
    error => {
      console.error('Error loading model:', error)
    }
  )

  window.addEventListener('resize', onWindowResize)
  window.addEventListener('mousemove', onMouseMove)
  window.addEventListener('scroll', onScroll)

  onWindowResize()

  function animate() {
    animationFrameId = requestAnimationFrame(animate)

    const elapsed = clock.getElapsedTime()

    if (heartModel) {
      // Combineer scroll rotatie en muis rotatie smooth
      const combinedTargetY = scrollRotation + targetRotationY

      heartModel.rotation.y += (combinedTargetY - heartModel.rotation.y) * 0.05
      heartModel.rotation.x += (targetRotationX - heartModel.rotation.x) * 0.05

      heartModel.position.x = Math.sin(elapsed * 0.5) * 0.05
      heartModel.position.y = Math.cos(elapsed * 0.7) * 0.05 - window.scrollY * 0.002
      heartModel.position.z = Math.sin(elapsed * 0.3) * 0.05

      const beat1 = Math.sin(elapsed * 6) * 0.03
      const beat2 = Math.sin(elapsed * 2) * 0.015
      const pulse = 1 + Math.max(0, beat1) + Math.max(0, beat2)

      heartModel.scale.set(pulse, pulse, pulse)
    }

    renderer.render(scene, camera)
  }
  animate()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrameId)
  window.removeEventListener('resize', onWindowResize)
  window.removeEventListener('mousemove', onMouseMove)
  window.removeEventListener('scroll', onScroll)
  if (renderer) renderer.dispose()
})
</script>

<style>
.three-container {
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: black;
}

canvas {
  display: block;
  width: 100%;
  height: 100%;
}
</style>






