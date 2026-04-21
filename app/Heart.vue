<script setup>
import { shallowRef, onMounted } from 'vue'
import { OrbitControls } from '@tresjs/cientos'

const model = shallowRef(null)

onMounted(async () => {
  const { GLTFLoader } = await import('three/examples/jsm/loaders/GLTFLoader.js')

  const loader = new GLTFLoader()

  loader.load('/models/heart.glb', (gltf) => {
    model.value = gltf.scene
    model.value.scale.setScalar(1.4)
    model.value.position.set(0, -0.15, 0)
  })
})
</script>

<template>
  <client-only>
    <div class="heart-scene">
      <TresCanvas clear-color="#fdf6e3" render-mode="always" window-size>
        <TresPerspectiveCamera :position="[0, 0, 3.2]" :look-at="[0, 0, 0]" />

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
