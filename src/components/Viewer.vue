<script setup lang="ts">
import { onMounted, onUnmounted, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/addons/controls/OrbitControls.js'
import { TilesRenderer } from '3d-tiles-renderer'
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js'
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js'

const canvas = ref<HTMLCanvasElement | null>(null)

const TILESET_URLS = [
  import.meta.env.VITE_TILESET_URL, // 中央区
  'https://assets.cms.plateau.reearth.io/assets/0d/7900b4-4d6c-4720-8edd-2f129b431235/01100_sapporo-shi_city_2020_citygml_7_op_bldg_3dtiles_01102_kita-ku_lod2/tileset.json', // 北区
]

let renderer: THREE.WebGLRenderer
let animationId: number

class GLTFCesiumRTCExtension {
  name = 'CESIUM_RTC'
  parser: any
  constructor(parser: any) {
    this.parser = parser
  }
  afterRoot(gltf: any): Promise<void> | null {
    const cesiumRTC = this.parser.json.extensions?.CESIUM_RTC
    if (!cesiumRTC) return null
    const [x, y, z] = cesiumRTC.center
    gltf.scene.position.set(x, y, z)
    return null
  }
}

onMounted(() => {
  if (!canvas.value) return

  renderer = new THREE.WebGLRenderer({ canvas: canvas.value, antialias: true })
  renderer.setSize(window.innerWidth, window.innerHeight)
  renderer.setPixelRatio(window.devicePixelRatio)
  renderer.setClearColor(0x87ceeb) // 水色

  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0x87ceeb)
  scene.fog = new THREE.Fog(0x87ceeb, 5000, 50000)

  const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 10, 1000000)
  camera.position.set(0, 5000, 10000)

  const controls = new OrbitControls(camera, canvas.value)
  controls.enableDamping = true

  scene.add(new THREE.AmbientLight(0xffffff, 2))
  const sun = new THREE.DirectionalLight(0xffffff, 3)
  sun.position.set(1000, 2000, 1000)
  scene.add(sun)

  const dracoLoader = new DRACOLoader()
  dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/')
  const gltfLoader = new GLTFLoader()
  gltfLoader.setDRACOLoader(dracoLoader)
  gltfLoader.register((parser: any) => new GLTFCesiumRTCExtension(parser))

  const tilesets = TILESET_URLS.map(url => {
    const tiles = new TilesRenderer(url)
    tiles.manager.addHandler(/\.gltf$/, gltfLoader)
    tiles.manager.addHandler(/\.glb$/, gltfLoader)
    tiles.setCamera(camera)
    tiles.setResolutionFromRenderer(camera, renderer)
    scene.add(tiles.group)
    return tiles
  })

  if (tilesets[0]) {
// 朝の環境光（明るめ）
scene.add(new THREE.AmbientLight(0xffffff, 3))

// 朝日（明るく・低い角度から）
const sun = new THREE.DirectionalLight(0xfffaee, 8)
sun.position.set(-3000, 1500, 2000)
scene.add(sun)

// 反対側からの補助光（影を柔らかく）
const fill = new THREE.DirectionalLight(0xaaccff, 2)
fill.position.set(3000, 1000, -2000)
scene.add(fill)

// 空の色
scene.background = new THREE.Color(0x87ceeb)
scene.fog = new THREE.Fog(0xc8e8ff, 8000, 60000)
renderer.setClearColor(0x87ceeb)
    tilesets[0].addEventListener('load-tileset', () => {
      const sphere = new THREE.Sphere()
      tilesets[0]?.getBoundingSphere(sphere)
    const { center, radius } = sphere

    const up = center.clone().normalize()
    const quaternion = new THREE.Quaternion()
    quaternion.setFromUnitVectors(up, new THREE.Vector3(0, 1, 0))

    tilesets.forEach(t => {
      t.group.quaternion.copy(quaternion)
      const rotatedCenter = center.clone().applyQuaternion(quaternion)
      t.group.position.copy(rotatedCenter).negate()
    })

    camera.position.set(0, radius * 0.8, radius * 1.5)
    camera.near = radius * 0.001
    camera.far = radius * 10
    camera.updateProjectionMatrix()

      controls.target.set(0, 0, 0)
      controls.update()
    })
  }

  // リサイズ対応
  const onResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth, window.innerHeight)
  }
  window.addEventListener('resize', onResize)

  const animate = () => {
    animationId = requestAnimationFrame(animate)
    controls.update()
    camera.updateMatrixWorld()
    tilesets.forEach(t => t.update())
    renderer.render(scene, camera)
  }
  animate()
})

onUnmounted(() => {
  cancelAnimationFrame(animationId)
  renderer?.dispose()
})

</script>

<template>
  <div class="viewer-wrap">
    <canvas ref="canvas" />
  </div>
</template>

<style scoped>
.viewer-wrap {
  position: fixed;
  inset: 0;
  overflow: hidden;
}

canvas {
  display: block;
  width: 100%;
  height: 100%;
}
</style>