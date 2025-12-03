<template>
  <div ref="container" class="three-container"></div>
  <h1>test</h1>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'

const container = ref(null)

let scene, camera, renderer, heartModel, animationFrameId

onMounted(() => {
  // Scene
  scene = new THREE.Scene()

  // Camera - perspective, fov 45, aspect ratio dynamic
  const width = container.value.clientWidth
  const height = container.value.clientHeight
  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000)
  camera.position.set(0, 0, 10) // beetje afstand

  // Renderer
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setSize(width, height)
  renderer.setClearColor(0x000000, 0) // transparant achtergrond
  container.value.appendChild(renderer.domElement)

  // Belichting en schaduw 

  // Zachte ambient light voor basisverlichting
  const ambientLight = new THREE.AmbientLight(0xffffff, 3)
  scene.add(ambientLight)

  // Directional light als zonlicht, met schaduwen
  const directionalLight = new THREE.DirectionalLight(0xffffff, 10)
  directionalLight.position.set(5, 10, 7)
  directionalLight.castShadow = true

  // Grotere schaduwcamera zodat het model helemaal in de schaduw valt
  directionalLight.shadow.camera.left = -5
  directionalLight.shadow.camera.right = 5
  directionalLight.shadow.camera.top = 5
  directionalLight.shadow.camera.bottom = -5
  directionalLight.shadow.camera.near = 1
  directionalLight.shadow.camera.far = 20

  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048

  scene.add(directionalLight)

  // extra invullicht
  const pointLight = new THREE.PointLight(0xffffff, 0.4)
  pointLight.position.set(-3, 3, 5)
  scene.add(pointLight)

  // GLTF loader
  const loader = new GLTFLoader()
  loader.load(
    '/the_heart.glb', // zorg dat dit in je public map staat
    gltf => {
      heartModel = gltf.scene
      heartModel.scale.set(1.5, 1.5, 1.5) // wat groter
      scene.add(heartModel)
    },
    undefined,
    error => {
      console.error('Error loading model:', error)
    }
  )

  // Responsiveness
  window.addEventListener('resize', onWindowResize)

  // Animatie loop
  function animate() {
    animationFrameId = requestAnimationFrame(animate)
    if (heartModel) heartModel.rotation.y += 0.005
    renderer.render(scene, camera)
  }
  animate()
})

function onWindowResize() {
  const width = container.value.clientWidth
  const height = container.value.clientHeight
  camera.aspect = width / height
  camera.updateProjectionMatrix()
  renderer.setSize(width, height)
}

onBeforeUnmount(() => {
  cancelAnimationFrame(animationFrameId)
  window.removeEventListener('resize', onWindowResize)
  renderer.dispose()
})
</script>

<style scoped>
.three-container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}
canvas {
  display: block;
  /* voorkom scrollbars */
  max-width: 100%;
  max-height: 100%;
}
</style>

