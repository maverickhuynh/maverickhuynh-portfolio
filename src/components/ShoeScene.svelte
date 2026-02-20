<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  let container;
  let scene, camera, renderer;
  let shoeModel = null;
  let teardown = null;
  let loadError = false;

  function applySceneMaterials(meshGroup) {
    meshGroup.traverse((child) => {
      if (child.isMesh && child.material) {
        const mats = Array.isArray(child.material) ? child.material : [child.material];
        const newMats = mats.map((m) => {
          let hex = 0x888888;
          if (m.color && typeof m.color.getHex === 'function') hex = m.color.getHex();
          const r = (hex >> 16) & 0xff, g = (hex >> 8) & 0xff, b = hex & 0xff;
          const brightness = (r + g + b) / 3;
          const color = brightness < 140 ? 0xff4444 : 0xffffff;
          return new THREE.MeshStandardMaterial({
            color,
            metalness: 0.12,
            roughness: 0.6,
            envMapIntensity: 0.5,
          });
        });
        child.material = newMats.length === 1 ? newMats[0] : newMats;
      } else if (child.isMesh) {
        child.material = new THREE.MeshStandardMaterial({
          color: 0xff4444,
          metalness: 0.12,
          roughness: 0.6,
        });
      }
    });
  }

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
      camera.position.set(-61, -1.6, 4);
      camera.lookAt(-55, -1.6, 0);

      renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
      renderer.setSize(w, h);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      renderer.outputColorSpace = THREE.SRGBColorSpace;
      renderer.setClearColor(0x000000, 0);
      container.appendChild(renderer.domElement);

      const controls = new OrbitControls(camera, renderer.domElement);
      controls.target.set(-63, -1.6, 0);
      controls.enableZoom = false;
      controls.enablePan = false;

      const raycaster = new THREE.Raycaster();
      const mouse = new THREE.Vector2();
      let isHovering = false;
      let baseScale = 1;

      // Strong lighting for shading and detail on dark background
      const ambient = new THREE.AmbientLight(0xffffff, 1.6);
      scene.add(ambient);
      const hemi = new THREE.HemisphereLight(0xffffff, 0x666666, 1.1);
      scene.add(hemi);
      const key = new THREE.DirectionalLight(0xffffff, 1.9);
      key.position.set(3, 4, 3);
      scene.add(key);
      const fill = new THREE.DirectionalLight(0xffffff, 1.1);
      fill.position.set(-2, 2, -1);
      scene.add(fill);
      const rim = new THREE.DirectionalLight(0xffffff, 0.85);
      rim.position.set(-1, 0.5, -2);
      scene.add(rim);

      const loader = new GLTFLoader();
      loader.load(
        '/models/Trainer.glb',
        (gltf) => {
          shoeModel = gltf.scene;
          scene.add(shoeModel);

          const box = new THREE.Box3().setFromObject(shoeModel);
          const center = box.getCenter(new THREE.Vector3());
          const size = box.getSize(new THREE.Vector3());
          shoeModel.position.sub(center);

          const maxDim = Math.max(size.x, size.y, size.z) || 1;
          baseScale = 3.8 / maxDim;
          shoeModel.scale.setScalar(baseScale);
          applySceneMaterials(shoeModel);

          shoeModel.rotation.y = Math.PI * 0.25;
          shoeModel.position.x = -66.0;
          shoeModel.position.y = -1.6;
          controls.target.copy(shoeModel.position);
          if (renderer && scene && camera) renderer.render(scene, camera);
        },
        undefined,
        (err) => {
          loadError = true;
          console.error('Error loading Trainer.glb', err);
        }
      );

      const canvas = renderer.domElement;
      function onMouseMove(event) {
        const rect = canvas.getBoundingClientRect();
        mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
        mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
        raycaster.setFromCamera(mouse, camera);
        if (shoeModel) {
          const intersects = raycaster.intersectObject(shoeModel, true);
          isHovering = intersects.length > 0;
        }
      }
      function onMouseLeave() {
        isHovering = false;
      }
      canvas.addEventListener('mousemove', onMouseMove);
      canvas.addEventListener('mouseleave', onMouseLeave);

      let animating = true;
      function animate() {
        if (!animating || !renderer || !container?.parentElement) return;
        requestAnimationFrame(animate);
        controls.update();
        if (shoeModel) {
          shoeModel.rotation.y += 0.002;
          shoeModel.scale.setScalar(isHovering ? baseScale * 1.08 : baseScale);
        }
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
        canvas.removeEventListener('mousemove', onMouseMove);
        canvas.removeEventListener('mouseleave', onMouseLeave);
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

<div class="shoe-scene">
  <div bind:this={container} class="shoe-scene-inner"></div>
  {#if loadError}
    <p class="shoe-scene-message">3D model not found. Add <strong>Trainer.glb</strong> to the folder <strong>public/models/</strong> in your project, then refresh.</p>
  {/if}
</div>

<style>
  .shoe-scene {
    width: 100%;
    height: 100%;
    min-height: 55vh;
    display: block;
    position: relative;
  }

  .shoe-scene-inner {
    width: 100%;
    height: 100%;
    min-height: 55vh;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .shoe-scene-message {
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

  :global(.shoe-scene canvas) {
    display: block;
    flex: 0 0 auto;
    max-width: 100%;
    max-height: 100%;
  }
</style>
