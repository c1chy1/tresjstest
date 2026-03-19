<script setup>
import { shallowRef, onMounted } from 'vue'
import { OrbitControls } from '@tresjs/cientos'

const model = shallowRef(null)

onMounted(async () => {
  const { GLTFLoader } = await import('three/examples/jsm/loaders/GLTFLoader.js')

  const loader = new GLTFLoader()

  loader.load('/models/heart.glb', (gltf) => {
    model.value = gltf.scene


  })


})
</script>

<template>
  <client-only>
    <div style="height:100vh">
      <TresCanvas window-size>
        <TresPerspectiveCamera :position="[0,0,2]" />

        <TresAmbientLight />
        <TresDirectionalLight :position="[1,3,3]" />

        <OrbitControls />
        <primitive v-if="model" :object="model" />
      </TresCanvas>
    </div>
  </client-only>
</template>