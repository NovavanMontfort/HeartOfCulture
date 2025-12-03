<template>
  <div ref="container" class="three-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'

const container = ref(null)

let scene, camera, renderer, heartModel, animationFrameId

// Muisrotatie variabelen
let mouseX = 0, mouseY = 0
let targetRotationX = 0, targetRotationY = 0

let clock = new THREE.Clock()

function onMouseMove(event) {
  mouseX = (event.clientX / window.innerWidth) * 2 - 1
  mouseY = -((event.clientY / window.innerHeight) * 2 - 1)

  targetRotationY = mouseX * 0.6
  targetRotationX = -mouseY * 0.6 
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

  // Gebruik window.innerWidth en innerHeight voor init grootte
  const width = window.innerWidth
  const height = window.innerHeight

  camera = new THREE.PerspectiveCamera(45, width / height, 0.1, 1000)
  camera.position.set(0, 0, 6)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setSize(width, height)
  renderer.setClearColor(0x000000, 0) // transparant
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
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

  // Model loader
  const loader = new GLTFLoader()
  loader.load(
    '/the_heart.glb',
    gltf => {
      heartModel = gltf.scene
      heartModel.scale.set(1.5, 1.5, 1.5)
      heartModel.castShadow = true
      heartModel.receiveShadow = true
      scene.add(heartModel)
    },
    undefined,
    error => {
      console.error('Error loading model:', error)
    }
  )

  window.addEventListener('resize', onWindowResize)
  window.addEventListener('mousemove', onMouseMove)

  // Zorg voor juiste sizing na mount
  onWindowResize()

  // Animatie loop
  function animate() {
    animationFrameId = requestAnimationFrame(animate)

    const elapsed = clock.getElapsedTime()

    if (heartModel) {
      heartModel.rotation.y += (targetRotationY - heartModel.rotation.y) * 0.05
      heartModel.rotation.x += (targetRotationX - heartModel.rotation.x) * 0.05

      heartModel.position.x = Math.sin(elapsed * 0.5) * 0.05
      heartModel.position.y = Math.cos(elapsed * 0.7) * 0.05
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
  if (renderer) renderer.dispose()
})
</script>

<style>
.three-container {
  /* Deze staat nu al in de global */
  /* width/height overflow etc staat in global CSS */
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




