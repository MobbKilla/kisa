<template>
  <TresCanvas
    window-size
    clear-color="#0d0020"
    :tone-mapping="ACESFilmicToneMapping"
    :tone-mapping-exposure="1.6"
  >
    <Suspense><Scene /></Suspense>

    <Suspense>
      <EffectComposer>
        <BloomPmndrs :intensity="1.4" :luminance-threshold="0.3" :luminance-smoothing="0.2" />
      </EffectComposer>
    </Suspense>
  </TresCanvas>
</template>

<script setup>
import { ACESFilmicToneMapping } from 'three'
import { TresCanvas } from '@tresjs/core'
import { EffectComposer, BloomPmndrs } from '@tresjs/post-processing'
import Scene from './components/Scene.vue'
</script>
