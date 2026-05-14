<template>
  <TresPerspectiveCamera ref="cameraRef" :position="[0, 1, 6]" :fov="55" make-default />

  <!-- ─── LIGHTS ─────────────────────────────────────────
       Студийный свет: мягкое фиолетовое заполнение + яркое
       розовое пятно снизу-сзади (как на рефе)             -->
  <TresAmbientLight color="#7700cc" :intensity="2.5" />
  <!-- яркое pink пятно у основания задней стены -->
  <TresPointLight color="#ff00cc" :position="[0, -1.5, -3]" :intensity="12" :distance="14" />
  <!-- холодный фиолетовый сверху -->
  <TresPointLight color="#4400aa" :position="[0, 6, 0]"    :intensity="5"  :distance="20" />
  <!-- тёплый розовый слева для объёма -->
  <TresPointLight color="#ff3399" :position="[-5, 2, 2]"   :intensity="4"  :distance="18" />
  <!-- подсветка граффити — белый свет прямо на стену -->
  <TresPointLight color="#ffffff" :position="[0, 1, 1]"    :intensity="3"  :distance="8"  />

  <!-- ─── BACKDROP ──────────────────────────────────────
       Большая плоскость за всем — градиент через шейдер   -->
  <TresMesh :position="[0, 1, -5]">
    <TresPlaneGeometry :args="[20, 12]" />
    <primitive :object="backdropMat" />
  </TresMesh>

  <!-- ─── FLOOR ─────────────────────────────────────────
       Тёмный глянцевый пол, плавно принимает розовый свет -->
  <TresMesh :position="[0, -2.5, -1]" :rotation="[-1.5708, 0, 0]">
    <TresPlaneGeometry :args="[20, 14]" />
    <TresMeshStandardMaterial
      color="#0a001a"
      :roughness="0.1"
      :metalness="0.85"
    />
  </TresMesh>

  <!-- ─── GRAFFITI ──────────────────────────────────────
       Главный элемент — белое граффити, spray-reveal       -->
  <TresMesh v-if="sprayMat" ref="graffitiRef" :position="[0, 0.5, -4.9]">
    <TresPlaneGeometry :args="graffitiArgs" />
    <primitive :object="sprayMat" />
  </TresMesh>

  <!-- ─── HEARTS ────────────────────────────────────────  -->
  <primitive v-if="hearts" :object="hearts" />
</template>

<script setup>
import { ref, shallowRef, computed, onMounted, onUnmounted } from 'vue'
import { useLoop } from '@tresjs/core'
import { gsap } from 'gsap'
import * as THREE from 'three'

// ─── refs ──────────────────────────────────────────────
const cameraRef   = ref(null)
const graffitiRef = ref(null)

const sprayMat    = shallowRef(null)
const backdropMat = shallowRef(makeBackdrop())   // синхронно — нет async-зависимостей
const hearts      = shallowRef(null)

const graffitiW    = ref(9)
const graffitiH    = ref(5)
const graffitiArgs = computed(() => [graffitiW.value, graffitiH.value])

// ─── input state ───────────────────────────────────────
const mouse  = { x: 0, y: 0 }
const target = { x: 0, y: 0 }
const device = { x: 0, y: 0 }
const LERP   = 0.055

// ─── backdrop gradient shader ──────────────────────────
// Имитируем реф: тёмный сине-фиолетовый сверху,
// яркий magenta/пурпурный в центре-снизу
const BACKDROP_VERT = /* glsl */`
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`
const BACKDROP_FRAG = /* glsl */`
  varying vec2 vUv;
  void main() {
    // top: тёмный синий-фиолет   #0d0030
    // mid: яркий пурпурный       #9900dd
    // bottom: тёмный             #150020
    vec3 top    = vec3(0.05, 0.00, 0.19);
    vec3 mid    = vec3(0.60, 0.00, 0.87);
    vec3 bot    = vec3(0.08, 0.00, 0.13);

    // яркое розовое пятно справа-снизу (как на рефе)
    vec3 spot   = vec3(0.95, 0.00, 0.55);
    float sDist = distance(vUv, vec2(0.45, 0.38));
    float sIntensity = smoothstep(0.55, 0.0, sDist) * 0.9;

    float t = vUv.y;
    vec3 grad = mix(bot, mix(mid, top, smoothstep(0.35, 0.85, t)), t);
    grad = mix(grad, spot, sIntensity);

    gl_FragColor = vec4(grad, 1.0);
  }
`

// ─── spray reveal shader ───────────────────────────────
const SPRAY_VERT = /* glsl */`
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`
const SPRAY_FRAG = /* glsl */`
  uniform sampler2D uTexture;
  uniform float     uProgress;
  uniform float     uTime;
  varying vec2      vUv;

  float rand(vec2 c) {
    return fract(sin(dot(c, vec2(12.9898, 78.233))) * 43758.5453);
  }
  float noise(vec2 p) {
    vec2 i = floor(p), f = fract(p);
    f = f * f * (3.0 - 2.0 * f);
    float a = rand(i), b = rand(i + vec2(1,0));
    float c2 = rand(i + vec2(0,1)), d = rand(i + vec2(1,1));
    return mix(mix(a, b, f.x), mix(c2, d, f.x), f.y);
  }
  float fbm(vec2 p) {
    return noise(p)*0.5 + noise(p*2.1)*0.3 + noise(p*4.3)*0.2;
  }

  void main() {
    vec4 col = texture2D(uTexture, vUv);
    if (col.a < 0.01) discard;

    float n          = fbm(vUv * 7.0 + uTime * 0.04);
    float threshold  = 1.0 - uProgress;
    float localT     = threshold - n * 0.18;
    float reveal     = smoothstep(localT - 0.04, localT + 0.04, vUv.y);

    col.a *= reveal;
    if (col.a < 0.01) discard;
    gl_FragColor = col;
  }
`

// ─── helpers ───────────────────────────────────────────
function makeBackdrop() {
  return new THREE.ShaderMaterial({
    vertexShader:   BACKDROP_VERT,
    fragmentShader: BACKDROP_FRAG,
    side:           THREE.FrontSide,
    depthWrite:     false,
  })
}

function processGraffiti() {
  return new Promise((resolve) => {
    new THREE.TextureLoader().load(import.meta.env.BASE_URL + 'graffiti.png', (tex) => {
      const img = tex.image
      const cv  = document.createElement('canvas')
      cv.width  = img.width
      cv.height = img.height
      const ctx = cv.getContext('2d')
      ctx.drawImage(img, 0, 0)

      const data = ctx.getImageData(0, 0, cv.width, cv.height)
      const d    = data.data

      for (let i = 0; i < d.length; i += 4) {
        const brightness = (d[i] + d[i + 1] + d[i + 2]) / 3
        if (brightness > 200) {
          // белый фон → прозрачный
          d[i + 3] = 0
        } else {
          // тёмный пиксель → белый (максимальный контраст + bloom)
          const t  = 1 - brightness / 210
          d[i]     = 255
          d[i + 1] = 255
          d[i + 2] = 255
          d[i + 3] = Math.round(255 * t)
        }
      }
      ctx.putImageData(data, 0, 0)

      const aspect    = img.width / img.height
      graffitiW.value = 9
      graffitiH.value = parseFloat((9 / aspect).toFixed(3))

      resolve(new THREE.CanvasTexture(cv))
    })
  })
}

function makeHearts() {
  const cv = document.createElement('canvas')
  cv.width = cv.height = 64
  const ctx = cv.getContext('2d')
  ctx.shadowColor = '#ff69b4'
  ctx.shadowBlur  = 14
  ctx.fillStyle   = '#ffffff'
  ctx.font        = '44px serif'
  ctx.textAlign   = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText('♥', 32, 34)

  const count = 60
  const pos   = new Float32Array(count * 3)
  for (let i = 0; i < count; i++) {
    pos[i * 3]     = (Math.random() - 0.5) * 16
    pos[i * 3 + 1] = (Math.random() - 0.5) * 8
    pos[i * 3 + 2] = -5 + Math.random() * 5
  }
  const geo = new THREE.BufferGeometry()
  geo.setAttribute('position', new THREE.BufferAttribute(pos, 3))
  const mat = new THREE.PointsMaterial({
    map:        new THREE.CanvasTexture(cv),
    size:       0.22,
    transparent: true,
    alphaTest:  0.05,
    depthWrite: false,
    opacity:    0.7,
  })
  return new THREE.Points(geo, mat)
}

// ─── init ──────────────────────────────────────────────
onMounted(async () => {
  backdropMat.value = makeBackdrop()
  hearts.value      = makeHearts()

  const tex = await processGraffiti()

  sprayMat.value = new THREE.ShaderMaterial({
    uniforms: {
      uTexture:  { value: tex },
      uProgress: { value: 0   },
      uTime:     { value: 0   },
    },
    vertexShader:   SPRAY_VERT,
    fragmentShader: SPRAY_FRAG,
    transparent:    true,
    depthWrite:     false,
    side:           THREE.DoubleSide,
  })

  gsap.to(sprayMat.value.uniforms.uProgress, {
    value:    1.3,
    duration: 3.5,
    delay:    0.5,
    ease:     'power1.inOut',
  })

  // iOS gyro permission
  if (
    typeof DeviceOrientationEvent !== 'undefined' &&
    typeof DeviceOrientationEvent.requestPermission === 'function'
  ) {
    window.addEventListener('touchstart', async () => {
      const perm = await DeviceOrientationEvent.requestPermission().catch(() => 'denied')
      if (perm === 'granted') window.addEventListener('deviceorientation', onDeviceOrientation)
    }, { once: true })
  } else {
    window.addEventListener('deviceorientation', onDeviceOrientation)
  }

  window.addEventListener('mousemove', onMouseMove)
})

onUnmounted(() => {
  window.removeEventListener('mousemove', onMouseMove)
  window.removeEventListener('deviceorientation', onDeviceOrientation)
})

// ─── input handlers ────────────────────────────────────
function onMouseMove(e) {
  target.x =  (e.clientX / window.innerWidth  - 0.5) * 2
  target.y = -(e.clientY / window.innerHeight - 0.5) * 2
}
function onDeviceOrientation(e) {
  if (e.beta === null) return
  device.x = (e.gamma        / 45) * 0.9
  device.y = ((e.beta - 45)  / 45) * 0.9
}

// ─── render loop ───────────────────────────────────────
const { onRender } = useLoop()

onRender(({ elapsed }) => {
  mouse.x += (target.x + device.x - mouse.x) * LERP
  mouse.y += (target.y + device.y - mouse.y) * LERP

  const tx = mouse.x
  const ty = mouse.y

  if (cameraRef.value) {
    cameraRef.value.position.x = tx * 0.3
    cameraRef.value.position.y = ty * 0.2 + 1
    cameraRef.value.lookAt(0, 0.5, -3)
  }

  if (graffitiRef.value) {
    graffitiRef.value.position.x = -tx * 0.22
    graffitiRef.value.position.y = -ty * 0.15 + 0.5
  }

  if (sprayMat.value) {
    sprayMat.value.uniforms.uTime.value = elapsed
  }

  if (hearts.value) {
    const pos = hearts.value.geometry.attributes.position
    for (let i = 0; i < pos.count; i++) {
      pos.array[i * 3 + 1] += 0.003
      if (pos.array[i * 3 + 1] > 5) pos.array[i * 3 + 1] = -5
    }
    pos.needsUpdate = true
    hearts.value.position.x = -tx * 0.25
    hearts.value.position.y = -ty * 0.1
  }
})
</script>
