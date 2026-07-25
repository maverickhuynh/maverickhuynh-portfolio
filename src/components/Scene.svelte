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
      spheres.push({
        mesh: sphere,
        originalPosition: originalPositions[i],
        currentTarget: new THREE.Vector3(x, y, z),
        radius,
        velocity: new THREE.Vector3(0, 0, 0),
        speed: Math.random() * 0.02 + 0.01,
        targetScale,
        rotation: {
          x: (Math.random() - 0.5) * 0.02,
          y: (Math.random() - 0.5) * 0.02,
          z: (Math.random() - 0.5) * 0.02,
        },
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

    function handleMouseMove(event) {
      targetMouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      targetMouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    }

    function updateMousePosition() {
      mouse.x += (targetMouse.x - mouse.x) * 0.1;
      mouse.y += (targetMouse.y - mouse.y) * 0.1;
    }

    function getWorldPosition(mouseX, mouseY) {
      const vector = new THREE.Vector3(mouseX, mouseY, 0.5);
      vector.unproject(camera);
      const dir = vector.sub(camera.position).normalize();
      const distance = -camera.position.z / dir.z;
      return camera.position.clone().add(dir.multiplyScalar(distance));
    }

    window.addEventListener('mousemove', handleMouseMove);

    function animate() {
      requestAnimationFrame(animate);
      updateMousePosition();
      const worldMouse = getWorldPosition(mouse.x, mouse.y);

      spheres.forEach((sphereData) => {
        const { mesh, originalPosition, currentTarget, velocity, radius } = sphereData;

        const distanceToMouse = mesh.position.distanceTo(worldMouse);
        const hitRadius = radius * 3.3;

        if (distanceToMouse < hitRadius && distanceToMouse > 0) {
          const hitDirection = new THREE.Vector3()
            .subVectors(mesh.position, worldMouse)
            .normalize();
          const hitStrength = 1 - distanceToMouse / hitRadius;
          const impulseForce = hitStrength * 1.5;
          velocity.add(hitDirection.multiplyScalar(impulseForce));
        }

        velocity.multiplyScalar(0.997);

        const returnForce = new THREE.Vector3()
          .subVectors(originalPosition, mesh.position)
          .multiplyScalar(0.001);
        velocity.add(returnForce);

        const maxVelocity = 0.15;
        if (velocity.length() > maxVelocity) {
          velocity.normalize().multiplyScalar(maxVelocity);
        }

        mesh.position.add(velocity);
        currentTarget.copy(mesh.position);
      });

      for (let i = 0; i < spheres.length; i++) {
        for (let j = i + 1; j < spheres.length; j++) {
          const sphere1 = spheres[i];
          const sphere2 = spheres[j];
          const pos1 = sphere1.mesh.position;
          const pos2 = sphere2.mesh.position;

          const distance = pos1.distanceTo(pos2);
          const minDistance = sphere1.radius + sphere2.radius;

          if (distance < minDistance && distance > 0) {
            const direction = new THREE.Vector3().subVectors(pos1, pos2).normalize();
            const overlap = minDistance - distance;
            const pushStrength = overlap * 0.5;
            const push1 = direction.clone().multiplyScalar(pushStrength * (sphere2.radius / minDistance));
            const push2 = direction.clone().multiplyScalar(-pushStrength * (sphere1.radius / minDistance));

            pos1.add(push1);
            pos2.add(push2);

            const bounceStrength = 0.3;
            const relativeVelocity = new THREE.Vector3().subVectors(sphere1.velocity, sphere2.velocity);
            const velocityAlongNormal = relativeVelocity.dot(direction);

            if (velocityAlongNormal > 0) continue;

            const restitution = 0.6;
            const impulse = -(1 + restitution) * velocityAlongNormal / 2;

            sphere1.velocity.add(direction.clone().multiplyScalar(impulse * bounceStrength));
            sphere2.velocity.sub(direction.clone().multiplyScalar(impulse * bounceStrength));

            sphere1.currentTarget.copy(pos1);
            sphere2.currentTarget.copy(pos2);
          }
        }
      }

      renderer.render(scene, camera);
    }
    animate();

    function handleResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    }
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('mousemove', handleMouseMove);
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
