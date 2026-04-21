<template>
  <div ref="container" class="cube-container"></div>
</template>

<script setup>
import * as THREE from 'three'
import { LineSegments2 } from 'three/examples/jsm/lines/LineSegments2.js'
import { LineSegmentsGeometry } from 'three/examples/jsm/lines/LineSegmentsGeometry.js'
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js'
import { onMounted, onUnmounted, ref } from 'vue'
import gsap from 'gsap'

const container = ref(null)

let cube
let renderer
let camera
let scene
let fog
let animationFrameId = 0
let isDragging = false
let isSnapping = false
let activePointerId = null

let lastX = 0
let lastY = 0
let velocityX = 0
let velocityY = 0

let handlePointerDown
let handlePointerMove
let handlePointerUp
let handleResize
let outlineLineMat

function getViewSize() {
  const vv = window.visualViewport
  return {
    w: vv ? vv.width : window.innerWidth,
    h: vv ? vv.height : window.innerHeight,
  }
}

function computeCameraZ(aspect) {
  if (aspect >= 1) return 4
  const halfFovRad = THREE.MathUtils.degToRad(75 / 2)
  return Math.max(4, (3 * 1.5) / (2 * Math.tan(halfFovRad) * aspect))
}

function normalizeAngle(angle) {
  const twoPi = Math.PI * 2
  angle %= twoPi
  if (angle > Math.PI) angle -= twoPi
  if (angle < -Math.PI) angle += twoPi
  return angle
}

onMounted(() => {
  if (!process.client || !container.value) return

  const { w, h } = getViewSize()
  const aspect = w / h
  const camZ = computeCameraZ(aspect)

  scene = new THREE.Scene()
  fog = new THREE.Fog(0xfdf6e3, camZ + 1, camZ + 7)
  scene.fog = fog

  camera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000)
  camera.position.set(0, 0, camZ)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(w, h)
  renderer.setClearColor(0xfdf6e3)
  renderer.domElement.style.touchAction = 'none'
  container.value.appendChild(renderer.domElement)

  scene.add(new THREE.AmbientLight(0xffffff, 1.85))

  const keyLight = new THREE.DirectionalLight(0xfff4dd, 1.8)
  keyLight.position.set(3, 4, 5)
  scene.add(keyLight)

  const fillLight = new THREE.DirectionalLight(0xffe1d6, 0.9)
  fillLight.position.set(-3, -1, 2)
  scene.add(fillLight)

  const loader = new THREE.TextureLoader()
  const textureA = loader.load('/img/example.jpg')
  const textureB = loader.load('/img/example2.jpg')

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

  const edgesGeo = new THREE.EdgesGeometry(new THREE.BoxGeometry(3, 3, 3))
  const lineGeo = new LineSegmentsGeometry().fromEdgesGeometry(edgesGeo)
  edgesGeo.dispose()
  outlineLineMat = new LineMaterial({
    color: 0x000000,
    linewidth: 5,
    resolution: new THREE.Vector2(w, h),
  })
  const outlineLines = new LineSegments2(lineGeo, outlineLineMat)
  cube.add(outlineLines)

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
    if (activePointerId !== null) return
    activePointerId = event.pointerId
    gsap.killTweensOf(cube.rotation)
    isSnapping = false
    isDragging = true
    velocityX = 0
    velocityY = 0
    lastX = event.clientX
    lastY = event.clientY
  }

  handlePointerMove = (event) => {
    if (!isDragging || event.pointerId !== activePointerId) return

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

  handlePointerUp = (event) => {
    if (!isDragging || event.pointerId !== activePointerId) return
    isDragging = false
    activePointerId = null
    snapToFace()
  }

  handleResize = () => {
    const { w, h } = getViewSize()
    const newAspect = w / h
    const newCamZ = computeCameraZ(newAspect)

    camera.aspect = newAspect
    camera.position.z = newCamZ
    camera.updateProjectionMatrix()

    fog.near = newCamZ + 1
    fog.far = newCamZ + 7

    renderer.setSize(w, h)
    outlineLineMat?.resolution.set(w, h)
  }

  renderer.domElement.addEventListener('pointerdown', handlePointerDown)
  window.addEventListener('pointermove', handlePointerMove)
  window.addEventListener('pointerup', handlePointerUp)
  window.addEventListener('pointercancel', handlePointerUp)
  window.addEventListener('resize', handleResize)
  window.visualViewport?.addEventListener('resize', handleResize)

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

  if (handleResize) {
    window.removeEventListener('resize', handleResize)
    window.visualViewport?.removeEventListener('resize', handleResize)
  }

  if (Array.isArray(cube?.material)) {
    cube.material.forEach((material) => material.dispose())
  }

  const outlineChild = cube?.children?.[0]
  outlineChild?.geometry?.dispose()
  outlineLineMat?.dispose()

  cube?.geometry?.dispose()
  renderer?.dispose()
})
</script>

<style>
.cube-container {
  width: 100vw;
  height: 100dvh;
  overflow: hidden;
  touch-action: none;
}
</style>
