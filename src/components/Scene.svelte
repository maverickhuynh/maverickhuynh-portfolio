<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let container;
  let scene, camera, renderer;
  let spheres = [];
  let mouse = { x: 0, y: 0 };
  let targetMouse = { x: 0, y: 0 };
  let envMap;

  onMount(() => {
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );
    camera.position.z = 5;

    renderer = new THREE.WebGLRenderer({
      alpha: true,
      antialias: true,
      powerPreference: 'high-performance',
    });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    renderer.toneMappingExposure = 1.05;
    container.appendChild(renderer.domElement);

    // Softbox studio with a bit more contrast so spheres separate from the page
    function createStudioEnvironment() {
      const envScene = new THREE.Scene();
      envScene.background = new THREE.Color(0xb8c8d8);

      const addPanel = (color, position, rotation, width, height) => {
        const mesh = new THREE.Mesh(
          new THREE.PlaneGeometry(width, height),
          new THREE.MeshBasicMaterial({ color, side: THREE.DoubleSide })
        );
        mesh.position.copy(position);
        mesh.rotation.set(rotation.x, rotation.y, rotation.z);
        envScene.add(mesh);
        return mesh;
      };

      // Large key softbox (upper left)
      addPanel(0xffffff, new THREE.Vector3(-4.2, 5.2, 3.2), { x: -0.55, y: 0.7, z: 0.1 }, 8, 6);
      // Soft white fill (upper right)
      addPanel(0xf0f6fb, new THREE.Vector3(5, 4, 2.2), { x: -0.4, y: -0.8, z: -0.08 }, 6.5, 5);
      // Sierra blue accent for reflections (no mint/lilac)
      addPanel(0x5fb0e8, new THREE.Vector3(5.2, 1.2, -2.5), { x: 0.1, y: -2.35, z: 0 }, 6, 7);
      // Cool sky bounce from above
      addPanel(0xa8d4f2, new THREE.Vector3(0, 7, 0), { x: -Math.PI / 2, y: 0, z: 0 }, 14, 14);
      // Cool underside bounce
      addPanel(0xa8c8e0, new THREE.Vector3(-1.5, -3.2, 1.5), { x: 1.05, y: 0.35, z: 0 }, 8, 5);
      // Soft rear rim
      addPanel(0xffffff, new THREE.Vector3(0.8, 3, -5.5), { x: 0, y: 0, z: 0 }, 9, 6);
      // Slightly darker floor for underside definition
      addPanel(0xc5d4e2, new THREE.Vector3(0, -4.2, 0), { x: Math.PI / 2, y: 0, z: 0 }, 18, 18);

      envScene.add(new THREE.AmbientLight(0xffffff, 0.28));
      const envKey = new THREE.DirectionalLight(0xffffff, 1.8);
      envKey.position.set(-4, 9, 4);
      envScene.add(envKey);

      const generator = new THREE.PMREMGenerator(renderer);
      const map = generator.fromScene(envScene, 0.18).texture;
      envScene.traverse((obj) => {
        if (obj.geometry) obj.geometry.dispose();
        if (obj.material) obj.material.dispose();
      });
      generator.dispose();
      return map;
    }

    envMap = createStudioEnvironment();
    scene.environment = envMap;

    const sphereCount = 50;
    const baseRadius = 0.5;
    const geometry = new THREE.SphereGeometry(baseRadius, 96, 96);
    const originalPositions = [];

    // Sierra Blue + pearl — force several clearly light balls on the outer edge
    const lightBallCount = 10;
    for (let i = 0; i < sphereCount; i++) {
      const isLightBall = i < lightBallCount;
      let color;
      let envMapIntensity;

      if (isLightBall) {
        // Guaranteed bright pearl — keep env bright so they don't go muddy grey
        color = new THREE.Color(0xffffff);
        envMapIntensity = 1.35 + Math.random() * 0.25;
      } else {
        const roll = Math.random();
        if (roll < 0.5) {
          // Light / icy Sierra blue — slight vibrance bump
          color = new THREE.Color(0xc8e4f6).lerp(new THREE.Color(0xa4d2f0), Math.random());
        } else if (roll < 0.85) {
          // Mid Sierra blue — clearer product blue
          color = new THREE.Color(0x8fc4ea).lerp(new THREE.Color(0x6eb4e4), Math.random());
        } else {
          // Soft deeper blue (still vibrant, not grey)
          color = new THREE.Color(0x6aaddc).lerp(new THREE.Color(0x5a9ed4), Math.random() * 0.5);
        }
        envMapIntensity = 1.85 + Math.random() * 0.35;
      }

      const roughness = isLightBall
        ? 0.18 + Math.random() * 0.12
        : 0.14 + Math.random() * 0.18;

      const material = new THREE.MeshPhysicalMaterial({
        color,
        metalness: isLightBall ? 0.01 : 0.03 + Math.random() * 0.04,
        roughness,
        clearcoat: 0.9 + Math.random() * 0.1,
        clearcoatRoughness: isLightBall ? 0.2 + Math.random() * 0.1 : 0.16 + Math.random() * 0.14,
        reflectivity: isLightBall ? 0.7 : 0.82,
        envMapIntensity,
        ior: 1.4,
        iridescence: isLightBall ? 0.03 : 0.08 + Math.random() * 0.1,
        iridescenceIOR: 1.2,
        iridescenceThicknessRange: [180, 400],
        sheen: isLightBall ? 0.08 : 0.1 + Math.random() * 0.1,
        sheenRoughness: 0.4,
        sheenColor: new THREE.Color(isLightBall ? 0xffffff : 0x9ec8e8),
        specularIntensity: isLightBall ? 0.55 : 0.65,
        specularColor: new THREE.Color(0xffffff),
      });

      const sphere = new THREE.Mesh(geometry, material);
      let x, y, z;
      if (isLightBall) {
        // Place light balls toward the outer ring so they read clearly
        const angle = (i / lightBallCount) * Math.PI * 2 + Math.random() * 0.35;
        const radiusXY = 0.85 + Math.random() * 0.55;
        x = Math.cos(angle) * radiusXY;
        y = Math.sin(angle) * radiusXY * 0.9;
        z = (Math.random() - 0.5) * 1.2;
      } else {
        x = (Math.random() - 0.5) * 2;
        y = (Math.random() - 0.5) * 2;
        z = (Math.random() - 0.5) * 1.5;
      }

      sphere.position.set(x, y, z);
      originalPositions.push(new THREE.Vector3(x, y, z));

      const targetScale = isLightBall
        ? 0.95 + Math.random() * 0.25
        : Math.random() * 0.3 + 0.8;
      sphere.scale.setScalar(0);
      const radius = baseRadius * targetScale;

      scene.add(sphere);
      const mass = Math.pow(radius / baseRadius, 3);
      spheres.push({
        mesh: sphere,
        originalPosition: originalPositions[i],
        radius,
        mass,
        velocity: new THREE.Vector3(0, 0, 0),
        targetScale,
        // Idle drift phase so each ball breathes differently
        driftPhase: Math.random() * Math.PI * 2,
        driftSpeed: 0.35 + Math.random() * 0.45,
        driftAmp: 0.012 + Math.random() * 0.018,
      });
    }

    // A bit more light contrast so spheres separate from each other and the page
    const hemi = new THREE.HemisphereLight(0xf2f7fb, 0xa8b9ca, 0.5);
    scene.add(hemi);

    const ambientLight = new THREE.AmbientLight(0xffffff, 0.18);
    scene.add(ambientLight);

    const keyLight = new THREE.DirectionalLight(0xffffff, 0.55);
    keyLight.position.set(3.5, 7, 2.5);
    scene.add(keyLight);

    const fillLight = new THREE.DirectionalLight(0xc5dff0, 0.32);
    fillLight.position.set(-5, 1.5, 2);
    scene.add(fillLight);

    const rimLight = new THREE.DirectionalLight(0xffffff, 0.35);
    rimLight.position.set(-1.5, 3.5, -5);
    scene.add(rimLight);

    setTimeout(() => {
      spheres.forEach((sphereData, i) => {
        const { mesh, targetScale } = sphereData;

        setTimeout(() => {
          const startTime = Date.now();
          const duration = 600;

          function animatePop() {
            const elapsed = Date.now() - startTime;
            const progress = Math.min(elapsed / duration, 1);
            const ease = 1 - Math.pow(1 - progress, 3);
            mesh.scale.setScalar(ease * targetScale);
            if (progress < 1) requestAnimationFrame(animatePop);
          }
          animatePop();
        }, i * 20);
      });
    }, 1000);

    const _toMouse = new THREE.Vector3();
    const _returnForce = new THREE.Vector3();
    const _drift = new THREE.Vector3();
    const _relVel = new THREE.Vector3();
    const _impulse = new THREE.Vector3();
    const _spinAxis = new THREE.Vector3();
    const _collisionNormal = new THREE.Vector3();
    let lastTime = performance.now();
    let elapsedTime = 0;
    const pointerActive = { value: false };

    function setPointerFromClient(clientX, clientY) {
      targetMouse.x = (clientX / window.innerWidth) * 2 - 1;
      targetMouse.y = -(clientY / window.innerHeight) * 2 + 1;
      pointerActive.value = true;
    }

    function handlePointerMove(event) {
      setPointerFromClient(event.clientX, event.clientY);
    }

    function handlePointerLeave() {
      pointerActive.value = false;
    }

    function handlePointerUp(event) {
      // Keep desktop hover active; clear touch after lift
      if (event.pointerType !== 'mouse') pointerActive.value = false;
    }

    function updateMousePosition(dt) {
      const lerp = 1 - Math.pow(0.001, dt);
      mouse.x += (targetMouse.x - mouse.x) * lerp;
      mouse.y += (targetMouse.y - mouse.y) * lerp;
    }

    function getWorldPosition(mouseX, mouseY) {
      const vector = new THREE.Vector3(mouseX, mouseY, 0.5);
      vector.unproject(camera);
      const dir = vector.sub(camera.position).normalize();
      const distance = -camera.position.z / dir.z;
      return camera.position.clone().add(dir.multiplyScalar(distance));
    }

    window.addEventListener('pointermove', handlePointerMove, { passive: true });
    window.addEventListener('pointerleave', handlePointerLeave);
    window.addEventListener('pointerup', handlePointerUp);
    window.addEventListener('pointercancel', handlePointerUp);

    function animate(now) {
      requestAnimationFrame(animate);
      const rawDt = (now - lastTime) / 1000;
      lastTime = now;
      // Clamp so tab-refocus doesn't explode the sim
      const dt = Math.min(Math.max(rawDt, 0), 0.05);
      elapsedTime += dt;

      updateMousePosition(dt);
      const worldMouse = getWorldPosition(mouse.x, mouse.y);

      // Soft plastic feel — tuned in units/sec so 60/120Hz match
      const damping = Math.pow(0.985, dt * 60);
      const springStrength = 1.0;
      const mouseForceMax = 80;
      const maxSpeed = 7;
      const restitution = 0.32;

      spheres.forEach((sphereData) => {
        const {
          mesh,
          originalPosition,
          velocity,
          radius,
          mass,
          driftPhase,
          driftSpeed,
          driftAmp,
        } = sphereData;

        // Soft repulsion field (force falloff) instead of stacking impulses
        if (pointerActive.value) {
          const distanceToMouse = mesh.position.distanceTo(worldMouse);
          const hitRadius = radius * 5.5;
          if (distanceToMouse < hitRadius && distanceToMouse > 1e-4) {
            const t = 1 - distanceToMouse / hitRadius;
            const falloff = t;
            _toMouse.subVectors(mesh.position, worldMouse).normalize();
            // Heavier balls move a bit less
            velocity.addScaledVector(_toMouse, (mouseForceMax * falloff * dt) / Math.max(mass, 0.35));
          }
        }

        // Idle drift around home so the cluster breathes at rest
        const phase = elapsedTime * driftSpeed + driftPhase;
        _drift.set(
          Math.sin(phase) * driftAmp,
          Math.cos(phase * 0.87) * driftAmp * 0.85,
          Math.sin(phase * 0.61) * driftAmp * 0.45
        );

        _returnForce
          .subVectors(originalPosition, mesh.position)
          .add(_drift)
          .multiplyScalar(springStrength * dt);
        velocity.add(_returnForce);

        velocity.multiplyScalar(damping);

        const speed = velocity.length();
        if (speed > maxSpeed) {
          velocity.multiplyScalar(maxSpeed / speed);
        }

        mesh.position.addScaledVector(velocity, dt);

        // Spin from velocity so clearcoat reflections sell mass
        const spinSpeed = velocity.length();
        if (spinSpeed > 1e-4) {
          _spinAxis.set(-velocity.y, velocity.x, -velocity.z).normalize();
          mesh.rotateOnWorldAxis(_spinAxis, spinSpeed * 0.55 * dt);
        } else {
          mesh.rotateOnWorldAxis(_spinAxis.set(0.25, 1, 0.12).normalize(), 0.12 * dt);
        }
      });

      for (let i = 0; i < spheres.length; i++) {
        for (let j = i + 1; j < spheres.length; j++) {
          const sphere1 = spheres[i];
          const sphere2 = spheres[j];
          const pos1 = sphere1.mesh.position;
          const pos2 = sphere2.mesh.position;

          const distance = pos1.distanceTo(pos2);
          const minDistance = sphere1.radius + sphere2.radius;

          if (distance < minDistance && distance > 1e-6) {
            const direction = _collisionNormal.subVectors(pos1, pos2).normalize();
            const overlap = minDistance - distance;
            const invMass1 = 1 / sphere1.mass;
            const invMass2 = 1 / sphere2.mass;
            const invMassSum = invMass1 + invMass2;

            // Separate fully by inverse mass so small balls yield more
            pos1.addScaledVector(direction, overlap * (invMass1 / invMassSum));
            pos2.addScaledVector(direction, -overlap * (invMass2 / invMassSum));

            _relVel.subVectors(sphere1.velocity, sphere2.velocity);
            const velocityAlongNormal = _relVel.dot(direction);
            if (velocityAlongNormal >= 0) continue;

            const jImpulse = (-(1 + restitution) * velocityAlongNormal) / invMassSum;
            _impulse.copy(direction).multiplyScalar(jImpulse);
            sphere1.velocity.addScaledVector(_impulse, invMass1);
            sphere2.velocity.addScaledVector(_impulse, -invMass2);
          }
        }
      }

      renderer.render(scene, camera);
    }
    requestAnimationFrame(animate);

    function handleResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    }
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('pointermove', handlePointerMove);
      window.removeEventListener('pointerleave', handlePointerLeave);
      window.removeEventListener('pointerup', handlePointerUp);
      window.removeEventListener('pointercancel', handlePointerUp);
      geometry.dispose();
      spheres.forEach(({ mesh }) => mesh.material?.dispose?.());
      envMap?.dispose?.();
      if (renderer) renderer.dispose();
    };
  });
</script>

<div bind:this={container} class="scene-container"></div>

<style>
  .scene-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 1;
  }

  :global(canvas) {
    display: block;
  }
</style>
