<template>
  <div ref="container" class="exhibition-container"></div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { onMounted, ref } from 'vue'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader'
import { color, screenUV, hue, reflector, time, Fn, vec2, length, atan, float, sin, cos, vec3, sub, mul, pow, blendDodge, normalWorld } from 'three/tsl';
import { WebGPURenderer, NodeMaterial } from 'three/webgpu'
export default {
  setup() {
    const container = ref(null)
    let scene, camera, renderer, controls

    onMounted(async () => {
      initScene()
      await renderer.init();
      animate()
      window.addEventListener('resize', onWindowResize)
    })

    function onWindowResize() {
      camera.aspect = container.value.clientWidth / container.value.clientHeight
      camera.updateProjectionMatrix()
      renderer.setSize(container.value.clientWidth, container.value.clientHeight)
    }

    function initScene() {

      scene = new THREE.Scene()


      camera = new THREE.PerspectiveCamera(
        40,
        container.value.clientWidth / container.value.clientHeight,
        0.25,
        50
      )
      camera.position.set(-2, 1, 4)

      renderer = new WebGPURenderer({ antialias: true })
      renderer.setSize(container.value.clientWidth, container.value.clientHeight)
      renderer.toneMapping = THREE.NeutralToneMapping;
      renderer.setAnimationLoop(animate);
      renderer.setPixelRatio(window.devicePixelRatio);
      container.value.appendChild(renderer.domElement)


      controls = new OrbitControls(camera, renderer.domElement)
      controls.minDistance = 3;
      controls.maxDistance = 12;
      controls.target.set(0, 0, 0);
      controls.maxPolarAngle = Math.PI / 2;



      const lightSpeed = /*#__PURE__*/ Fn(([suv_immutable]) => {

        const suv = vec2(suv_immutable);
        const uv = vec2(length(suv), atan(suv.y, suv.x));
        const offset = float(float(.1).mul(sin(uv.y.mul(10.).sub(time.mul(.6)))).mul(cos(uv.y.mul(48.).add(time.mul(.3)))).mul(cos(uv.y.mul(3.7).add(time))));
        const rays = vec3(vec3(sin(uv.y.mul(150.).add(time)).mul(.5).add(.5)).mul(vec3(sin(uv.y.mul(80.).sub(time.mul(0.6))).mul(.5).add(.5))).mul(vec3(sin(uv.y.mul(45.).add(time.mul(0.8))).mul(.5).add(.5))).mul(vec3(sub(1., cos(uv.y.add(mul(22., time).sub(pow(uv.x.add(offset), .3).mul(60.))))))).mul(vec3(uv.x.mul(2.))));

        return rays;

      }).setLayout({
        name: 'lightSpeed',
        type: 'vec3',
        inputs: [
          { name: 'suv', type: 'vec2' }
        ]
      });


      const coloredVignette = screenUV.distance(.5).mix(hue(color(0x0175ad), time.mul(.1)), hue(color(0x02274f), time.mul(.5)));
      const lightSpeedEffect = lightSpeed(normalWorld).clamp();
      const lightSpeedSky = normalWorld.y.remapClamp(- .1, 1).mix(0, lightSpeedEffect);
      const composedBackground = blendDodge(coloredVignette, lightSpeedSky);

      scene.backgroundNode = composedBackground;


      const ambientLight = new THREE.AmbientLight(0x404040)
      scene.add(ambientLight)

      const sunLight = new THREE.DirectionalLight(0xffffff, 1)
      scene.add(sunLight)

      const directionalLight = new THREE.DirectionalLight(0xffffff, 1)
      directionalLight.position.set(1, 1, 1)
      scene.add(directionalLight)

      const loader = new GLTFLoader()
      loader.load(
        '/飞行器.glb',
        (gltf) => {
          const aircraft = gltf.scene
          aircraft.rotation.y = Math.PI
          // scale
          aircraft.scale.set(.2, .2, .2)
          // rotate
          aircraft.rotation.y = Math.PI / 2
          scene.add(aircraft)

          // 添加飞行器灯光效果
          const engineLightL = new THREE.PointLight(0x00aaff, 5, 5)
          engineLightL.position.set(-0.8, 0.5, -1.5)
          aircraft.add(engineLightL)

          const engineLightR = new THREE.PointLight(0x00aaff, 5, 5)
          engineLightR.position.set(0.8, 0.5, -1.5)
          aircraft.add(engineLightR)

          // 飞行器上下移动
          let i = 0
          setInterval(() => {
            i += 0.1
            aircraft.position.y = 0.2 + 0.05 * Math.sin(i)
          }, 100)
        },
        undefined,
        (error) => {
          console.error('加载飞行器模型出错:', error)
        }
      )

      const reflection = reflector();
      reflection.target.rotateX(- Math.PI / 2);
      scene.add(reflection.target);

      const floorMaterial = new NodeMaterial();
      floorMaterial.colorNode = reflection;
      floorMaterial.opacity = .2;
      floorMaterial.transparent = true;

      const floor = new THREE.Mesh(new THREE.BoxGeometry(50, .001, 50), floorMaterial);
      floor.receiveShadow = true;

      floor.position.set(0, 0, 0);
      scene.add(floor);
    }

    function animate() {
      // requestAnimationFrame(animate)
      controls.update()
      renderer.render(scene, camera)
    }

    return { container }
  }
}
</script>

<style scoped>
.exhibition-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}
</style>
