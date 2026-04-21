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
let textures = []
let animationFrameId = 0
let isDragging = false
let isSnapping = false
let activePointerId = null

let lastX = 0
let lastY = 0
let velocityX = 0
let velocityY = 0
let pendingDeltaX = 0
let pendingDeltaY = 0

let handlePointerDown
let handlePointerMove
let handlePointerUp
let handleResize
let outlineLineMat
let snapTween

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
  return Math.max(4, (3.5 * 1.5) / (2 * Math.tan(halfFovRad) * aspect))
}

const _WORLD_Y = new THREE.Vector3(0, 1, 0)
const _WORLD_X = new THREE.Vector3(1, 0, 0)
const _tmpQ = new THREE.Quaternion()

function buildOrientations() {
  const list = []
  for (let xi = 0; xi < 4; xi++) {
    for (let yi = 0; yi < 4; yi++) {
      for (let zi = 0; zi < 4; zi++) {
        const q = new THREE.Quaternion().setFromEuler(
          new THREE.Euler(xi * Math.PI / 2, yi * Math.PI / 2, zi * Math.PI / 2)
        )
        if (!list.some(o => Math.abs(o.dot(q)) > 0.9999)) list.push(q)
      }
    }
  }
  return list
}

function nearestOrientation(q, orientations) {
  let best = orientations[0]
  let bestDot = -1
  for (const o of orientations) {
    const d = Math.abs(q.dot(o))
    if (d > bestDot) { bestDot = d; best = o }
  }
  return best
}

function rotateWorldSpace(mesh, angleY, angleX) {
  _tmpQ.setFromAxisAngle(_WORLD_Y, angleY)
  mesh.quaternion.premultiply(_tmpQ)
  _tmpQ.setFromAxisAngle(_WORLD_X, angleX)
  mesh.quaternion.premultiply(_tmpQ)
}

// Dla każdej ścianki BoxGeometry — kierunek V (góra tekstury) w lokalnej przestrzeni kostki
const _FACE_DATA = [
  { normal: new THREE.Vector3( 1, 0, 0), vDir: new THREE.Vector3(0, 1,  0) }, // +X
  { normal: new THREE.Vector3(-1, 0, 0), vDir: new THREE.Vector3(0, 1,  0) }, // -X
  { normal: new THREE.Vector3( 0, 1, 0), vDir: new THREE.Vector3(0, 0, -1) }, // +Y góra
  { normal: new THREE.Vector3( 0,-1, 0), vDir: new THREE.Vector3(0, 0,  1) }, // -Y dół
  { normal: new THREE.Vector3( 0, 0, 1), vDir: new THREE.Vector3(0, 1,  0) }, // +Z przód
  { normal: new THREE.Vector3( 0, 0,-1), vDir: new THREE.Vector3(0, 1,  0) }, // -Z tył
]

function correctTextureRoll() {
  // Znajdź ściankę zwróconą ku kamerze (kamera patrzy wzdłuż +Z)
  let maxDot = -Infinity
  let visibleVDir = new THREE.Vector3(0, 1, 0)
  for (const { normal, vDir } of _FACE_DATA) {
    const dot = normal.clone().applyQuaternion(cube.quaternion).z
    if (dot > maxDot) {
      maxDot = dot
      visibleVDir = vDir.clone().applyQuaternion(cube.quaternion)
    }
  }

  // Kąt rollu widocznej ścianki w płaszczyźnie XY
  const rawAngle = Math.atan2(visibleVDir.x, visibleVDir.y)
  const rollAngle = Math.round(rawAngle / (Math.PI / 2)) * (Math.PI / 2)
  const target = rollAngle

  textures.forEach(tex => {
    let cur = tex.rotation
    while (cur - target > Math.PI) cur -= 2 * Math.PI
    while (target - cur > Math.PI) cur += 2 * Math.PI
    if (Math.abs(cur - target) < 0.01) return
    tex.rotation = cur

    gsap.killTweensOf(tex)
    gsap.to(tex, {
      rotation: target,
      duration: 0.85,
      ease: 'elastic.out(1.4, 0.45)',
      onUpdate: () => { tex.needsUpdate = true },
    })
  })

}

onMounted(() => {
  if (!process.client || !container.value) return

  const isMobile = /Mobi|Android|iPhone|iPad/i.test(navigator.userAgent)
  const { w, h } = getViewSize()
  const aspect = w / h
  const camZ = computeCameraZ(aspect)

  scene = new THREE.Scene()
  if (!isMobile) {
    fog = new THREE.Fog(0xfdf6e3, camZ + 1, camZ + 7)
    scene.fog = fog
  }

  camera = new THREE.PerspectiveCamera(75, aspect, 0.1, 1000)
  camera.position.set(0, 0, camZ)

  renderer = new THREE.WebGLRenderer({ antialias: !isMobile, alpha: false, powerPreference: 'high-performance' })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, isMobile ? 1.5 : 2))
  renderer.setSize(w, h)
  renderer.setClearColor(0xfdf6e3)
  renderer.domElement.style.touchAction = 'none'
  container.value.appendChild(renderer.domElement)

  if (!isMobile) {
    scene.add(new THREE.AmbientLight(0xffffff, 1.85))
    const keyLight = new THREE.DirectionalLight(0xfff4dd, 1.8)
    keyLight.position.set(3, 4, 5)
    scene.add(keyLight)
    const fillLight = new THREE.DirectionalLight(0xffe1d6, 0.9)
    fillLight.position.set(-3, -1, 2)
    scene.add(fillLight)
  }

  const loader = new THREE.TextureLoader()
  const imgPaths = [
    '/img/narzekanie.jpg',
    '/img/wiosna2.jpg',
    '/img/wiosna.jpg',
    '/img/niepotrzebnie.jpg',
    '/img/cyrk.jpg',
    '/img/wyjebongo.jpg',
  ]
  textures = imgPaths.map(path => {
    const tex = loader.load(path)
    tex.center.set(0.5, 0.5)
    return tex
  })

  const materials = textures.map(tex =>
    isMobile
      ? new THREE.MeshBasicMaterial({ map: tex })
      : new THREE.MeshLambertMaterial({ map: tex, color: 0xffffff, flatShading: true })
  )

  const orientations = buildOrientations()

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
    velocityX = 0
    velocityY = 0
    isSnapping = true

    const target = nearestOrientation(cube.quaternion, orientations)
    const start = cube.quaternion.clone()
    const proxy = { t: 0 }

    snapTween = gsap.to(proxy, {
      t: 1,
      duration: 0.6,
      ease: 'power3.out',
      onUpdate() {
        cube.quaternion.slerpQuaternions(start, target, proxy.t)
      },
      onComplete() {
        cube.quaternion.copy(target)
        isSnapping = false
        snapTween = null
        correctTextureRoll()
      },
    })
  }

  function animate() {
    animationFrameId = requestAnimationFrame(animate)

    if (isDragging) {
      if (pendingDeltaX !== 0 || pendingDeltaY !== 0) {
        rotateWorldSpace(cube, pendingDeltaX, pendingDeltaY)
        pendingDeltaX = 0
        pendingDeltaY = 0
      }
    } else if (!isSnapping) {
      rotateWorldSpace(cube, velocityX, velocityY)
      velocityX *= 0.92
      velocityY *= 0.92
      if (Math.abs(velocityX) < 0.0001) velocityX = 0
      if (Math.abs(velocityY) < 0.0001) velocityY = 0
    }

    renderer.render(scene, camera)
  }

  handlePointerDown = (event) => {
    if (activePointerId !== null) return
    activePointerId = event.pointerId
    snapTween?.kill()
    snapTween = null
    isSnapping = false
    isDragging = true
    velocityX = 0
    velocityY = 0
    pendingDeltaX = 0
    pendingDeltaY = 0
    lastX = event.clientX
    lastY = event.clientY
  }

  handlePointerMove = (event) => {
    if (!isDragging || event.pointerId !== activePointerId) return

    const sensitivity = 0.005
    const deltaX = (event.clientX - lastX) * sensitivity
    const deltaY = (event.clientY - lastY) * sensitivity

    pendingDeltaX += deltaX
    pendingDeltaY += deltaY
    velocityX = deltaX
    velocityY = deltaY

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

    if (fog) {
      fog.near = newCamZ + 1
      fog.far = newCamZ + 7
    }

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
