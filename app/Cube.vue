<template>
  <div ref="container" class="cube-container">
    <div class="cloud-layer">
      <div class="cloud cloud-1" :class="{ puff: cloudPuff }"></div>
      <div class="cloud cloud-2" :class="{ puff: cloudPuff }"></div>
      <div class="cloud cloud-3" :class="{ puff: cloudPuff }"></div>
      <div class="cloud cloud-4" :class="{ puff: cloudPuff }"></div>
    </div>
  </div>
</template>

<script setup>
import * as THREE from 'three'
import { LineSegments2 } from 'three/examples/jsm/lines/LineSegments2.js'
import { LineSegmentsGeometry } from 'three/examples/jsm/lines/LineSegmentsGeometry.js'
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js'
import { onMounted, onUnmounted, ref, nextTick } from 'vue'
import gsap from 'gsap'

const container = ref(null)
const cloudPuff = ref(false)

let cube
let renderer
let camera
let scene
let fog
let textureA
let textureB
let animationFrameId = 0
let isDragging = false
let isSnapping = false
let activePointerId = null
let cloudTimeout = null

let lastX = 0
let lastY = 0
let velocityX = 0
let velocityY = 0

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

// Sprawdza o ile stopni tekstury są "przekręcone" po snapie
// i animuje TYLKO tekstury z powrotem do pionu — kostka się nie rusza
function correctTextureRoll() {
  const localY = new THREE.Vector3(0, 1, 0).applyQuaternion(cube.quaternion)
  const rawAngle = Math.atan2(localY.x, localY.y)
  const rollAngle = Math.round(rawAngle / (Math.PI / 2)) * (Math.PI / 2)

  // Cel: obrót tekstury kompensuje roll kostki
  const target = rollAngle

  ;[textureA, textureB].forEach(tex => {
    // Najkrótsza droga
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

  if (Math.abs(rollAngle) > 0.01) {
    // Cartoon squash & stretch podczas korekcji
    gsap.timeline()
      .to(cube.scale, { x: 1.1, y: 0.88, z: 1.1, duration: 0.1, ease: 'power2.in' })
      .to(cube.scale, { x: 1, y: 1, z: 1, duration: 0.65, ease: 'elastic.out(1.3, 0.4)' })

    showCloudEffect()
  }
}

function showCloudEffect() {
  cloudPuff.value = false
  clearTimeout(cloudTimeout)
  nextTick(() => {
    cloudPuff.value = true
    cloudTimeout = setTimeout(() => { cloudPuff.value = false }, 1000)
  })
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
  textureA = loader.load('/img/example.jpg')
  textureB = loader.load('/img/example2.jpg')
  textureA.center.set(0.5, 0.5)
  textureB.center.set(0.5, 0.5)

  const materials = [
    new THREE.MeshLambertMaterial({ map: textureA, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureB, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureA, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureB, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureA, color: 0xffffff, flatShading: true }),
    new THREE.MeshLambertMaterial({ map: textureB, color: 0xffffff, flatShading: true }),
  ]

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

    if (!isDragging && !isSnapping) {
      rotateWorldSpace(cube, velocityX, velocityY)

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
    snapTween?.kill()
    snapTween = null
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

    rotateWorldSpace(cube, velocityX, velocityY)

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
  clearTimeout(cloudTimeout)

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
  position: relative;
}

/* ── Cartoon clouds overlay ── */
.cloud-layer {
  position: absolute;
  inset: 0;
  pointer-events: none;
  z-index: 10;
}

.cloud {
  position: absolute;
  background: #fff;
  border: 3px solid #111;
  border-radius: 60px;
  opacity: 0;
}

.cloud::before,
.cloud::after {
  content: '';
  position: absolute;
  background: #fff;
  border: 3px solid #111;
  border-radius: 50%;
}

/* Duża chmurka — lewa góra */
.cloud-1 {
  width: 110px; height: 46px;
  top: calc(50% - 90px); left: calc(50% - 160px);
}
.cloud-1::before { width: 56px; height: 56px; top: -32px; left: 10px; }
.cloud-1::after  { width: 40px; height: 40px; top: -22px; left: 52px; }

/* Mała chmurka — prawa góra */
.cloud-2 {
  width: 80px; height: 34px;
  top: calc(50% - 110px); left: calc(50% + 60px);
}
.cloud-2::before { width: 40px; height: 40px; top: -24px; left: 8px; }
.cloud-2::after  { width: 30px; height: 30px; top: -16px; left: 40px; }

/* Duża chmurka — prawy dół */
.cloud-3 {
  width: 100px; height: 42px;
  top: calc(50% + 60px); left: calc(50% + 50px);
}
.cloud-3::before { width: 50px; height: 50px; top: -28px; left: 12px; }
.cloud-3::after  { width: 36px; height: 36px; top: -20px; left: 50px; }

/* Mała chmurka — lewy dół */
.cloud-4 {
  width: 70px; height: 30px;
  top: calc(50% + 50px); left: calc(50% - 140px);
}
.cloud-4::before { width: 36px; height: 36px; top: -20px; left: 6px; }
.cloud-4::after  { width: 26px; height: 26px; top: -14px; left: 34px; }

/* Animacja: puff in & out z lekkim bounce */
.puff {
  animation: cloudPuff 0.9s ease-out forwards;
}
.cloud-2.puff { animation-delay: 0.07s; }
.cloud-3.puff { animation-delay: 0.12s; }
.cloud-4.puff { animation-delay: 0.05s; }

@keyframes cloudPuff {
  0%   { opacity: 0;    transform: scale(0.3) translateY(10px); }
  35%  { opacity: 1;    transform: scale(1.05) translateY(-4px); }
  65%  { opacity: 0.85; transform: scale(1);   }
  100% { opacity: 0;    transform: scale(1.3) translateY(-8px); }
}
</style>
