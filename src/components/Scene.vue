<template>
  <TresPerspectiveCamera ref="cameraRef" :position="[0, 1, 6]" :fov="55" make-default />

  <!-- ─── LIGHTS ──────────────────────────────────────── -->
  <TresAmbientLight color="#7700cc" :intensity="2.5" />
  <TresPointLight color="#ff00cc" :position="[0, -1.5, -3]" :intensity="12" :distance="14" />
  <TresPointLight color="#4400aa" :position="[0, 6, 0]"    :intensity="5"  :distance="20" />
  <TresPointLight color="#ff3399" :position="[-5, 2, 2]"   :intensity="4"  :distance="18" />
  <TresPointLight color="#ffffff" :position="[0, 1, 1]"    :intensity="3"  :distance="8"  />

  <!-- ─── BACKDROP ─────────────────────────────────────── -->
  <TresMesh :position="[0, 1, -5]">
    <TresPlaneGeometry :args="[20, 12]" />
    <primitive :object="backdropMat" />
  </TresMesh>

  <!-- ─── FLOOR ────────────────────────────────────────── -->
  <TresMesh :position="[0, -2.5, -1]" :rotation="[-1.5708, 0, 0]">
    <TresPlaneGeometry :args="[20, 14]" />
    <TresMeshStandardMaterial color="#0a001a" :roughness="0.1" :metalness="0.85" />
  </TresMesh>

  <!-- ─── GLOW LAYER — за граффити, отдельная глубина ─── -->
  <TresMesh v-if="glowMat" ref="glowRef" :position="[0, 0.5, -5.05]">
    <TresPlaneGeometry :args="glowArgs" />
    <primitive :object="glowMat" />
  </TresMesh>

  <!-- ─── GRAFFITI — главный слой ──────────────────────── -->
  <TresMesh v-if="sprayMat" ref="graffitiRef" :position="[0, 0.5, -4.9]">
    <TresPlaneGeometry :args="graffitiArgs" />
    <primitive :object="sprayMat" />
  </TresMesh>

  <!-- ─── HEARTS — ближе к камере, максимальный параллакс  -->
  <primitive v-if="hearts" :object="hearts" />
</template>

<script setup>
import { ref, shallowRef, computed, onMounted, onUnmounted } from 'vue'
import { useLoop } from '@tresjs/core'
import { gsap } from 'gsap'
import * as THREE from 'three'

// ─── refs ───────────────────────────────────────────────
const cameraRef   = ref(null)
const graffitiRef = ref(null)
const glowRef     = ref(null)

const sprayMat    = shallowRef(null)
const glowMat     = shallowRef(null)
const backdropMat = shallowRef(makeBackdrop())
const hearts      = shallowRef(null)

const graffitiW    = ref(8)
const graffitiH    = ref(4.5)
const graffitiArgs = computed(() => [graffitiW.value, graffitiH.value])
const glowArgs     = computed(() => [graffitiW.value * 1.5, graffitiH.value * 1.5])

// ─── tilt state ─────────────────────────────────────────
// smoothed mouse (-1..1 per axis)
const sm = { x: 0, y: 0 }
// raw target
const tg = { x: 0, y: 0, vx: 0, vy: 0 }
const device = { x: 0, y: 0 }
const LERP = 0.06

// tilt strengths per layer
const TILT = {
  cam:  { rotY: 0.06,  rotX: 0.04,  tx: 0.18,  ty: 0.12 },
  glow: { rotY: 0.08,  rotX: 0.055, tx: -0.08, ty: -0.06 },
  graf: { rotY: 0.13,  rotX: 0.09,  tx: -0.14, ty: -0.10 },
  hrt:  {                            tx: -0.28, ty: -0.16 },
}

// ─── backdrop shader ────────────────────────────────────
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
    vec3 top  = vec3(0.05, 0.00, 0.19);
    vec3 mid  = vec3(0.60, 0.00, 0.87);
    vec3 bot  = vec3(0.08, 0.00, 0.13);
    vec3 spot = vec3(0.95, 0.00, 0.55);

    float sDist      = distance(vUv, vec2(0.45, 0.38));
    float sIntensity = smoothstep(0.55, 0.0, sDist) * 0.9;

    float t    = vUv.y;
    vec3 grad  = mix(bot, mix(mid, top, smoothstep(0.35, 0.85, t)), t);
    grad = mix(grad, spot, sIntensity);

    gl_FragColor = vec4(grad, 1.0);
  }
`

// ─── spray reveal shader ────────────────────────────────
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

    float n         = fbm(vUv * 7.0 + uTime * 0.04);
    float threshold = 1.0 - uProgress;
    float localT    = threshold - n * 0.18;
    float reveal    = smoothstep(localT - 0.04, localT + 0.04, vUv.y);

    col.a *= reveal;
    if (col.a < 0.01) discard;
    gl_FragColor = col;
  }
`

// ─── helpers ────────────────────────────────────────────
function makeBackdrop() {
  return new THREE.ShaderMaterial({
    vertexShader:   BACKDROP_VERT,
    fragmentShader: BACKDROP_FRAG,
    side:           THREE.FrontSide,
    depthWrite:     false,
  })
}

function makeGlowMat() {
  const cv  = document.createElement('canvas')
  cv.width  = cv.height = 512
  const ctx = cv.getContext('2d')
  const g   = ctx.createRadialGradient(256, 256, 0, 256, 256, 256)
  g.addColorStop(0,   'rgba(255, 20, 147, 0.55)')
  g.addColorStop(0.4, 'rgba(160, 0, 220, 0.30)')
  g.addColorStop(1,   'rgba(0, 0, 0, 0)')
  ctx.fillStyle = g
  ctx.fillRect(0, 0, 512, 512)
  return new THREE.MeshBasicMaterial({
    map:         new THREE.CanvasTexture(cv),
    transparent: true,
    depthWrite:  false,
    blending:    THREE.AdditiveBlending,
  })
}

// Вычисляем размер плоскости так, чтобы граффити влезало в экран полностью.
// camera z=6, plane z=-4.9 → distance=10.9, fov=55
function calcGraffitiSize(imgW, imgH) {
  const fovRad   = 55 * Math.PI / 180
  const dist     = 6 - (-4.9)                                    // 10.9
  const visH     = 2 * Math.tan(fovRad / 2) * dist               // видимая высота
  const visW     = visH * (window.innerWidth / window.innerHeight)

  const maxW     = visW * 0.88
  const maxH     = visH * 0.80
  const aspect   = imgW / imgH

  // вписываем изображение в maxW × maxH сохраняя пропорции
  if (maxW / aspect <= maxH) {
    return { w: maxW, h: maxW / aspect }
  }
  return { w: maxH * aspect, h: maxH }
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
        const br = (d[i] + d[i+1] + d[i+2]) / 3
        if (br > 200) {
          d[i+3] = 0
        } else {
          const t  = 1 - br / 210
          d[i]     = 255
          d[i+1]   = 255
          d[i+2]   = 255
          d[i+3]   = Math.round(255 * t)
        }
      }
      ctx.putImageData(data, 0, 0)

      const { w, h }   = calcGraffitiSize(img.width, img.height)
      graffitiW.value  = parseFloat(w.toFixed(3))
      graffitiH.value  = parseFloat(h.toFixed(3))

      resolve(new THREE.CanvasTexture(cv))
    })
  })
}

function makeHearts() {
  const cv  = document.createElement('canvas')
  cv.width  = cv.height = 64
  const ctx = cv.getContext('2d')
  ctx.shadowColor   = '#ff69b4'
  ctx.shadowBlur    = 14
  ctx.fillStyle     = '#ffffff'
  ctx.font          = '44px serif'
  ctx.textAlign     = 'center'
  ctx.textBaseline  = 'middle'
  ctx.fillText('♥', 32, 34)

  const count = 60
  const pos   = new Float32Array(count * 3)
  for (let i = 0; i < count; i++) {
    pos[i*3]   = (Math.random() - 0.5) * 16
    pos[i*3+1] = (Math.random() - 0.5) * 8
    pos[i*3+2] = -3.5 + Math.random() * 2.5   // ближе к камере для параллакса
  }
  const geo = new THREE.BufferGeometry()
  geo.setAttribute('position', new THREE.BufferAttribute(pos, 3))
  return new THREE.Points(geo, new THREE.PointsMaterial({
    map:         new THREE.CanvasTexture(cv),
    size:        0.22,
    transparent: true,
    alphaTest:   0.05,
    depthWrite:  false,
    opacity:     0.75,
  }))
}

// ─── init ───────────────────────────────────────────────
onMounted(async () => {
  glowMat.value  = makeGlowMat()
  hearts.value   = makeHearts()

  const tex = await processGraffiti()
  sprayMat.value = new THREE.ShaderMaterial({
    uniforms:       { uTexture: { value: tex }, uProgress: { value: 0 }, uTime: { value: 0 } },
    vertexShader:   SPRAY_VERT,
    fragmentShader: SPRAY_FRAG,
    transparent:    true,
    depthWrite:     false,
    side:           THREE.DoubleSide,
  })

  gsap.to(sprayMat.value.uniforms.uProgress, {
    value: 1.3, duration: 3.5, delay: 0.5, ease: 'power1.inOut',
  })

  // iOS gyro
  if (
    typeof DeviceOrientationEvent !== 'undefined' &&
    typeof DeviceOrientationEvent.requestPermission === 'function'
  ) {
    window.addEventListener('touchstart', async () => {
      const p = await DeviceOrientationEvent.requestPermission().catch(() => 'denied')
      if (p === 'granted') window.addEventListener('deviceorientation', onDeviceOrientation)
    }, { once: true })
  } else {
    window.addEventListener('deviceorientation', onDeviceOrientation)
  }

  window.addEventListener('mousemove',  onMouseMove)
  window.addEventListener('mouseleave', onMouseLeave)
})

onUnmounted(() => {
  window.removeEventListener('mousemove',  onMouseMove)
  window.removeEventListener('mouseleave', onMouseLeave)
  window.removeEventListener('deviceorientation', onDeviceOrientation)
})

// ─── input ──────────────────────────────────────────────
function onMouseMove(e) {
  tg.x =  (e.clientX / window.innerWidth  - 0.5) * 2
  tg.y = -(e.clientY / window.innerHeight - 0.5) * 2
}
function onMouseLeave() {
  // плавный возврат в центр при уходе курсора
  gsap.to(tg, { x: 0, y: 0, duration: 1.4, ease: 'elastic.out(1, 0.4)' })
}
function onDeviceOrientation(e) {
  if (e.beta === null) return
  device.x = (e.gamma       / 45) * 0.9
  device.y = ((e.beta - 45) / 45) * 0.9
}

// ─── render loop ────────────────────────────────────────
const { onRender } = useLoop()

onRender(({ elapsed }) => {
  // lerp к target (для gyro lerp работает; для мыши GSAP берёт на себя возврат)
  sm.x += (tg.x + device.x - sm.x) * LERP
  sm.y += (tg.y + device.y - sm.y) * LERP

  const tx = sm.x
  const ty = sm.y

  // камера
  if (cameraRef.value) {
    cameraRef.value.position.x  = tx * TILT.cam.tx
    cameraRef.value.position.y  = ty * TILT.cam.ty + 1
    cameraRef.value.lookAt(0, 0.5, -3)
  }

  // glow — чуть меньший наклон
  if (glowRef.value) {
    glowRef.value.rotation.y    = tx * TILT.glow.rotY
    glowRef.value.rotation.x    = -ty * TILT.glow.rotX
    glowRef.value.position.x    = tx * TILT.glow.tx
    glowRef.value.position.y    = ty * TILT.glow.ty + 0.5
  }

  // граффити — основной 3D наклон
  if (graffitiRef.value) {
    graffitiRef.value.rotation.y = tx * TILT.graf.rotY
    graffitiRef.value.rotation.x = -ty * TILT.graf.rotX
    graffitiRef.value.position.x = tx * TILT.graf.tx
    graffitiRef.value.position.y = ty * TILT.graf.ty + 0.5
  }

  // шейдер время
  if (sprayMat.value) {
    sprayMat.value.uniforms.uTime.value = elapsed ?? 0
  }

  // сердечки — ближний слой, максимальный параллакс
  if (hearts.value) {
    const pos = hearts.value.geometry.attributes.position
    for (let i = 0; i < pos.count; i++) {
      pos.array[i*3+1] += 0.003
      if (pos.array[i*3+1] > 5) pos.array[i*3+1] = -5
    }
    pos.needsUpdate     = true
    hearts.value.position.x = tx * TILT.hrt.tx
    hearts.value.position.y = ty * TILT.hrt.ty
  }
})
</script>
