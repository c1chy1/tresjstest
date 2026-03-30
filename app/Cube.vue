<template>
  <div ref="container" class="cube-container"></div>
</template>

<script setup>
import * as THREE from 'three'
import { onMounted, onUnmounted, ref } from 'vue'
import gsap from 'gsap'

const container = ref(null)

let cube
let renderer
let camera
let animationFrameId = 0
let isDragging = false
let isSnapping = false

let lastX = 0
let lastY = 0
let velocityX = 0
let velocityY = 0

let handlePointerDown
let handlePointerMove
let handlePointerUp
let handleResize

function normalizeAngle(angle) {
  const twoPi = Math.PI * 2
  angle %= twoPi
  if (angle > Math.PI) angle -= twoPi
  if (angle < -Math.PI) angle += twoPi
  return angle
}

onMounted(() => {
  if (!process.client || !container.value) return

  const scene = new THREE.Scene()
  scene.fog = new THREE.Fog(0xfdf6e3, 5, 10)

  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000)
  camera.position.set(0, 0, 4)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setClearColor(0xfdf6e3)
  container.value.appendChild(renderer.domElement)

  scene.add(new THREE.AmbientLight(0xffffff, 1.85))

  const keyLight = new THREE.DirectionalLight(0xfff4dd, 1.8)
  keyLight.position.set(3, 4, 5)
  scene.add(keyLight)

  const fillLight = new THREE.DirectionalLight(0xffe1d6, 0.9)
  fillLight.position.set(-3, -1, 2)
  scene.add(fillLight)

  const loader = new THREE.TextureLoader()
  const textureA = loader.load('/img/wyjebongo.jpg')
  const textureB = loader.load('/img/wiosna.jpg')

  const materials = [
    new THREE.MeshLambertMaterial({ map: textureA, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureB, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureA, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureB, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureA, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureB, color: 0xffffff, flatShading: true }),
  ]

  cube = new THREE.Mesh(new THREE.BoxGeometry(3, 3, 3), materials)
  cube.rotation.set(-0.18, 0.55, 0)
  scene.add(cube)

  function snapToFace() {
    const step = Math.PI / 2
    const targetY = Math.round(normalizeAngle(cube.rotation.y) / step) * step
    const targetX = Math.round(normalizeAngle(cube.rotation.x) / step) * step

    isSnapping = true

    gsap.to(cube.rotation, {
      x: targetX,
      y: targetY,
      duration: 0.6,
      ease: 'power3.out',
      onComplete: () => {
        isSnapping = false
      },
    })

    velocityX = 0
    velocityY = 0
  }

  function animate() {
    animationFrameId = requestAnimationFrame(animate)

    if (!isDragging && !isSnapping) {
      cube.rotation.y += velocityX
      cube.rotation.x += velocityY

      velocityX *= 0.92
      velocityY *= 0.92

      if (Math.abs(velocityX) < 0.0001) velocityX = 0
      if (Math.abs(velocityY) < 0.0001) velocityY = 0
    }

    const time = Date.now() * 0.0005
    camera.position.x = Math.sin(time) * 0.12
    camera.position.y = Math.cos(time * 0.8) * 0.08
    camera.lookAt(0, 0, 0)

    renderer.render(scene, camera)
  }

  handlePointerDown = (event) => {
    isDragging = true
    lastX = event.clientX
    lastY = event.clientY
  }

  handlePointerMove = (event) => {
    if (!isDragging) return

    const deltaX = event.clientX - lastX
    const deltaY = event.clientY - lastY
    const sensitivity = 0.005

    velocityX = deltaX * sensitivity
    velocityY = deltaY * sensitivity

    cube.rotation.y += velocityX
    cube.rotation.x += velocityY

    const maxX = Math.PI / 2 - 0.01
    cube.rotation.x = Math.max(-maxX, Math.min(maxX, cube.rotation.x))

    lastX = event.clientX
    lastY = event.clientY
  }

  handlePointerUp = () => {
    if (!isDragging) return
    isDragging = false
    snapToFace()
  }

  handleResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  }

  renderer.domElement.addEventListener('pointerdown', handlePointerDown)
  window.addEventListener('pointermove', handlePointerMove)
  window.addEventListener('pointerup', handlePointerUp)
  window.addEventListener('pointercancel', handlePointerUp)
  window.addEventListener('resize', handleResize)

  animate()
})

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId)

  if (renderer?.domElement && handlePointerDown) {
    renderer.domElement.removeEventListener('pointerdown', handlePointerDown)
  }

  if (handlePointerMove) window.removeEventListener('pointermove', handlePointerMove)

  if (handlePointerUp) {
    window.removeEventListener('pointerup', handlePointerUp)
    window.removeEventListener('pointercancel', handlePointerUp)
  }

  if (handleResize) window.removeEventListener('resize', handleResize)

  if (Array.isArray(cube?.material)) {
    cube.material.forEach((material) => material.dispose())
  }

  cube?.geometry?.dispose()
  renderer?.dispose()
})
</script>

<style>
.cube-container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>
