<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  let container;
  let scene, camera, renderer;
  let model = null;
  let teardown = null;
  let loadError = false;
  let loading = true;

  onMount(() => {
    const targetAspect = 2.0;
    function getSize() {
      const cw = container.clientWidth || 800;
      const ch = Math.max(container.clientHeight, 400) || 400;
      let w, h;
      if (cw / ch > targetAspect) {
        h = ch;
        w = Math.round(ch * targetAspect);
      } else {
        w = cw;
        h = Math.round(cw / targetAspect);
      }
      return { w, h };
    }
    function init() {
      const { w, h } = getSize();
      const aspect = w / h;

      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(60, aspect, 0.1, 1000);
      camera.position.set(0, 0, 4);
      camera.lookAt(0, 0, 0);

      renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
      renderer.setSize(w, h);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      renderer.outputColorSpace = THREE.SRGBColorSpace;
      renderer.setClearColor(0x000000, 0);
      container.appendChild(renderer.domElement);

      const controls = new OrbitControls(camera, renderer.domElement);
      controls.target.set(0, 0, 0);
      controls.enableZoom = false;
      controls.enablePan = false;

      const ambient = new THREE.AmbientLight(0xffffff, 0.9);
      scene.add(ambient);
      const hemi = new THREE.HemisphereLight(0xffffff, 0x333333, 0.7);
      scene.add(hemi);
      const key = new THREE.DirectionalLight(0xffffff, 1.2);
      key.position.set(3, 4, 3);
      scene.add(key);
      const fill = new THREE.DirectionalLight(0xffffff, 0.6);
      fill.position.set(-2, 2, -1);
      scene.add(fill);
      const rim = new THREE.DirectionalLight(0xffffff, 0.4);
      rim.position.set(-1, 0.5, -2);
      scene.add(rim);

      const loader = new GLTFLoader();
      loader.load(
        '/models/Generic Male.glb',
        (gltf) => {
          model = gltf.scene;
          scene.add(model);

          const box = new THREE.Box3().setFromObject(model);
          const center = box.getCenter(new THREE.Vector3());
          const size = box.getSize(new THREE.Vector3());
          model.position.sub(center);

          const maxDim = Math.max(size.x, size.y, size.z) || 1;
          const scale = Math.max(200 / maxDim, 50);
          model.scale.setScalar(scale);

          model.position.y -= 2.0;

          const dist = 220;
          const cy = model.position.y;
          camera.position.set(0, cy, dist);
          controls.target.set(0, cy, 0);

          loading = false;
          if (renderer && scene && camera) renderer.render(scene, camera);
          requestAnimationFrame(() => handleResize());
          console.log('Generic Male.glb loaded', { maxDim, scale });
        },
        undefined,
        (err) => {
          loading = false;
          loadError = true;
          console.error('Error loading Generic Male.glb', err);
        }
      );

      let animating = true;
      function animate() {
        if (!animating || !renderer || !container?.parentElement) return;
        requestAnimationFrame(animate);
        controls.update();
        if (model) model.rotation.y += 0.002;
        renderer.render(scene, camera);
      }
      animate();

      function handleResize() {
        if (!container || !camera || !renderer) return;
        const { w, h } = getSize();
        camera.aspect = w / h;
        camera.updateProjectionMatrix();
        renderer.setSize(w, h);
      }
      window.addEventListener('resize', handleResize);

      return () => {
        animating = false;
        controls.dispose();
        window.removeEventListener('resize', handleResize);
        if (renderer) {
          renderer.dispose();
          if (container && renderer.domElement.parentNode === container) {
            container.removeChild(renderer.domElement);
          }
        }
      };
    }

    function runInit() {
      teardown = init();
    }
    const raf = requestAnimationFrame(runInit);
    return () => {
      cancelAnimationFrame(raf);
      if (teardown) teardown();
    };
  });
</script>

<div class="generic-male-scene">
  <div bind:this={container} class="generic-male-scene-inner"></div>
  {#if loading && !loadError}
    <p class="generic-male-scene-message">Loading…</p>
  {:else if loadError}
    <p class="generic-male-scene-message">3D model not found. Add <strong>Generic Male.glb</strong> to <strong>public/models/</strong>, then refresh.</p>
  {/if}
</div>

<style>
  .generic-male-scene {
    width: 100%;
    height: 100%;
    min-height: 55vh;
    display: block;
    position: relative;
  }

  .generic-male-scene-inner {
    width: 100%;
    height: 100%;
    min-height: 55vh;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .generic-male-scene-message {
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);
    margin: 0;
    padding: 1rem 1.5rem;
    background: rgba(20, 20, 20, 0.95);
    border: 1px solid rgba(255, 255, 255, 0.2);
    border-radius: 8px;
    color: rgba(255, 255, 255, 0.9);
    font-size: 0.95rem;
    max-width: 90%;
    text-align: center;
  }

  :global(.generic-male-scene canvas) {
    display: block;
    flex: 0 0 auto;
    max-width: 100%;
    max-height: 100%;
  }
</style>
