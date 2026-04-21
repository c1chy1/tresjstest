<script setup>
import { shallowRef, ref, onMounted, onUnmounted } from 'vue'
import { OrbitControls } from '@tresjs/cientos'

const model = shallowRef(null)
const cameraZ = ref(3.2)

function updateCameraZ() {
  const vv = window.visualViewport
  const w = vv ? vv.width : window.innerWidth
  const h = vv ? vv.height : window.innerHeight
  const aspect = w / h
  cameraZ.value = aspect >= 1 ? 3.2 : Math.min(7, 3.2 / aspect)
}

onMounted(async () => {
  updateCameraZ()
  window.visualViewport?.addEventListener('resize', updateCameraZ)
  window.addEventListener('resize', updateCameraZ)

  const { GLTFLoader } = await import('three/examples/jsm/loaders/GLTFLoader.js')
  const loader = new GLTFLoader()
  loader.load('/models/heart.glb', (gltf) => {
    model.value = gltf.scene
    model.value.scale.setScalar(1.4)
    model.value.position.set(0, -0.15, 0)
  })
})

onUnmounted(() => {
  window.visualViewport?.removeEventListener('resize', updateCameraZ)
  window.removeEventListener('resize', updateCameraZ)
})
</script>

<template>
  <client-only>
    <div class="heart-scene">
      <TresCanvas clear-color="#fdf6e3" render-mode="always" window-size>
        <TresPerspectiveCamera :position="[0, 0, cameraZ]" :look-at="[0, 0, 0]" />

        <TresAmbientLight :intensity="2.2" />
        <TresDirectionalLight :intensity="2.4" :position="[2, 3, 4]" />
        <TresDirectionalLight :intensity="1.2" :position="[-2, -1, 2]" />

        <OrbitControls />
        <primitive v-if="model" :object="model" />
      </TresCanvas>
    </div>
  </client-only>
</template>

<style scoped>
.heart-scene {
  width: 100vw;
  height: 100dvh;
  background: #fdf6e3;
  touch-action: none;
}
</style>
