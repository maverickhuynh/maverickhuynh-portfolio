<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  let container;
  let scene, camera, renderer;
  let shoeModel = null;
  let teardown = null;
  let loadError = false;

  onMount(() => {
    // #region agent log
    fetch('http://127.0.0.1:7242/ingest/cc5c8dd2-c4de-4dd9-8239-351fb87bb34c',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({hypothesisId:'C',location:'ShoeScene.svelte:onMount',message:'onMount entered',data:{hasContainer:!!container},timestamp:Date.now()})}).catch(()=>{});
    // #endregion
    function init() {
      const w = container.clientWidth || 800;
      const h = Math.max(container.clientHeight, 400) || 400;
      // #region agent log
      fetch('http://127.0.0.1:7242/ingest/cc5c8dd2-c4de-4dd9-8239-351fb87bb34c',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({hypothesisId:'B',location:'ShoeScene.svelte:init',message:'container dimensions',data:{clientWidth:container.clientWidth,clientHeight:container.clientHeight,w,h},timestamp:Date.now()})}).catch(()=>{});
      // #endregion
      const aspect = w / h;

      scene = new THREE.Scene();
      camera = new THREE.PerspectiveCamera(50, aspect, 0.1, 1000);
      camera.position.set(0, 0, 2.5);

      renderer = new THREE.WebGLRenderer({ alpha: false, antialias: true });
      renderer.setSize(w, h);
      renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      renderer.outputColorSpace = THREE.SRGBColorSpace;
      renderer.setClearColor(0x0c0c0c, 1);
      container.appendChild(renderer.domElement);
      // #region agent log
      fetch('http://127.0.0.1:7242/ingest/cc5c8dd2-c4de-4dd9-8239-351fb87bb34c',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({hypothesisId:'E',location:'ShoeScene.svelte:canvasAppended',message:'canvas appended',data:{w,h,parentHeight:container.parentElement?.clientHeight},timestamp:Date.now()})}).catch(()=>{});
      // #endregion

      // Stronger lighting so model is visible on dark background
      const ambient = new THREE.AmbientLight(0xffffff, 1.0);
      scene.add(ambient);
      const dir = new THREE.DirectionalLight(0xffffff, 1.2);
      dir.position.set(2, 3, 2);
      scene.add(dir);
      const fill = new THREE.DirectionalLight(0xffffff, 0.6);
      fill.position.set(-2, 1, -1);
      scene.add(fill);
      const front = new THREE.DirectionalLight(0xffffff, 0.5);
      front.position.set(0, 0, 2);
      scene.add(front);

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
          const scale = 1.2 / maxDim;
          shoeModel.scale.setScalar(scale);

          shoeModel.rotation.y = Math.PI * 0.25;
          if (renderer && scene && camera) renderer.render(scene, camera);
          // #region agent log
          fetch('http://127.0.0.1:7242/ingest/cc5c8dd2-c4de-4dd9-8239-351fb87bb34c',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({hypothesisId:'A',location:'ShoeScene.svelte:gltfSuccess',message:'GLB load success',data:{sizeX:size.x,sizeY:size.y,sizeZ:size.z,centerX:center.x,centerY:center.y,centerZ:center.z,scale},timestamp:Date.now()})}).catch(()=>{});
          // #endregion
          console.log('trainer.glb loaded');
        },
        undefined,
        (err) => {
          loadError = true;
          // #region agent log
          fetch('http://127.0.0.1:7242/ingest/cc5c8dd2-c4de-4dd9-8239-351fb87bb34c',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({hypothesisId:'A',location:'ShoeScene.svelte:gltfError',message:'GLB load error',data:{errMessage:err?.message||String(err)},timestamp:Date.now()})}).catch(()=>{});
          // #endregion
          console.error('Error loading trainer.glb', err);
        }
      );

      let animating = true;
      function animate() {
        if (!animating || !renderer || !container?.parentElement) return;
        requestAnimationFrame(animate);
        renderer.render(scene, camera);
      }
      animate();

      function handleResize() {
        if (!container || !camera || !renderer) return;
        const w = container.clientWidth || 800;
        const h = Math.max(container.clientHeight, 400) || 400;
        camera.aspect = w / h;
        camera.updateProjectionMatrix();
        renderer.setSize(w, h);
      }
      window.addEventListener('resize', handleResize);

      return () => {
        animating = false;
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
    <p class="shoe-scene-message">3D model not found. Add <strong>trainer.glb</strong> to the folder <strong>public/models/</strong> in your project, then refresh.</p>
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
    display: block;
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
    width: 100% !important;
    height: 100% !important;
  }
</style>
