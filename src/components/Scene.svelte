<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';

  let container;
  let scene, camera, renderer;
  let spheres = [];
  let mouse = { x: 0, y: 0 };
  let targetMouse = { x: 0, y: 0 };
  let raycaster = new THREE.Raycaster();

  onMount(() => {
    // Scene setup
    scene = new THREE.Scene();
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );
    camera.position.z = 5;

    renderer = new THREE.WebGLRenderer({ alpha: true, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    // Enable better blending
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    renderer.toneMapping = THREE.NoToneMapping;
    container.appendChild(renderer.domElement);

    // Create gradient texture with smoother interpolation
    function createGradientTexture() {
      const size = 1024;
      const canvas = document.createElement('canvas');
      canvas.width = size;
      canvas.height = size;
      const context = canvas.getContext('2d');
      
      // Enable better smoothing
      context.imageSmoothingEnabled = true;
      context.imageSmoothingQuality = 'high';
      
      // Helper function to interpolate colors smoothly in RGB space
      function lerpColor(color1, color2, factor) {
        // Convert hex to RGB
        const r1 = parseInt(color1.slice(1, 3), 16);
        const g1 = parseInt(color1.slice(3, 5), 16);
        const b1 = parseInt(color1.slice(5, 7), 16);
        const r2 = parseInt(color2.slice(1, 3), 16);
        const g2 = parseInt(color2.slice(3, 5), 16);
        const b2 = parseInt(color2.slice(5, 7), 16);
        
        // Smooth linear interpolation (no easing)
        const r = Math.round(r1 + (r2 - r1) * factor);
        const g = Math.round(g1 + (g2 - g1) * factor);
        const b = Math.round(b1 + (b2 - b1) * factor);
        
        return `#${r.toString(16).padStart(2, '0')}${g.toString(16).padStart(2, '0')}${b.toString(16).padStart(2, '0')}`;
      }
      
      // Define key colors - blue to light purplish with subtle white center, many small increments for smoothness
      const colors = [
        { pos: 0, color: '#6bb6ff' },      // Light blue top
        { pos: 0.05, color: '#73baff' },   // Smooth increment
        { pos: 0.10, color: '#7bbfff' },   // Smooth increment
        { pos: 0.15, color: '#84c4ff' },   // Smooth increment
        { pos: 0.20, color: '#8dc8ff' },   // Smooth increment
        { pos: 0.25, color: '#96ccff' },   // Smooth increment
        { pos: 0.30, color: '#a0d0ff' },   // Smooth increment
        { pos: 0.35, color: '#a8d4ff' },   // Smooth increment
        { pos: 0.38, color: '#b0c8f8' },   // Transition to purplish
        { pos: 0.40, color: '#b8c8f0' },   // Light purplish
        { pos: 0.42, color: '#c0c8e8' },   // Smooth increment
        { pos: 0.44, color: '#c4d0e8' },   // Smooth increment
        { pos: 0.46, color: '#c8d0e8' },   // Smooth increment
        { pos: 0.48, color: '#d0d8e8' },   // Transition to white
        { pos: 0.49, color: '#e8e8f0' },   // Smooth increment
        { pos: 0.50, color: '#f5f5f8' },   // Very subtle white center
        { pos: 0.51, color: '#f6f6f9' },   // Smooth increment
        { pos: 0.52, color: '#f7f7fa' },   // Smooth increment
        { pos: 0.54, color: '#f8f8fa' },   // Smooth increment
        { pos: 0.56, color: '#f8f8fb' },   // Smooth increment
        { pos: 0.58, color: '#f9f9fb' },   // Smooth increment
        { pos: 0.60, color: '#fafafa' },   // Smooth increment
        { pos: 0.62, color: '#fafafb' },   // Smooth increment
        { pos: 0.64, color: '#fbfbfc' },   // Smooth increment
        { pos: 0.66, color: '#fcfcfc' },   // Smooth increment
        { pos: 0.68, color: '#fdfdfd' },   // Smooth increment
        { pos: 0.70, color: '#fefefe' },   // Smooth increment
        { pos: 0.72, color: '#fefffe' },   // Smooth increment
        { pos: 0.74, color: '#fffeff' },   // Smooth increment
        { pos: 0.75, color: '#ffffff' },   // White - rest stays white
        { pos: 1, color: '#ffffff' }       // White bottom
      ];
      
      // Draw gradient pixel by pixel - radial gradient for circular white center
      const imageData = context.createImageData(size, size);
      const data = imageData.data;
      const centerX = size / 2;
      const centerY = size / 2;
      const maxRadius = Math.sqrt(centerX * centerX + centerY * centerY);
      
      for (let y = 0; y < size; y++) {
        for (let x = 0; x < size; x++) {
          // Calculate distance from center for radial gradient
          const dx = x - centerX;
          const dy = y - centerY;
          const distance = Math.sqrt(dx * dx + dy * dy);
          const t = Math.min(distance / maxRadius, 1); // Normalize to 0-1
          
          // Find which color stops to interpolate between
          let startColor, endColor, localT;
          for (let i = 0; i < colors.length - 1; i++) {
            if (t >= colors[i].pos && t <= colors[i + 1].pos) {
              startColor = colors[i].color;
              endColor = colors[i + 1].color;
              localT = (t - colors[i].pos) / (colors[i + 1].pos - colors[i].pos);
              break;
            }
          }
          
          // Get base color with linear interpolation
          const baseColor = lerpColor(startColor, endColor, localT);
          const r = parseInt(baseColor.slice(1, 3), 16);
          const g = parseInt(baseColor.slice(3, 5), 16);
          const b = parseInt(baseColor.slice(5, 7), 16);
          
          // Minimal dithering for ultra-smooth appearance
          const noise = (Math.random() - 0.5) * 2; // -1 to 1
          const ditherAmount = 0.4;
          
          const idx = (y * size + x) * 4;
          data[idx] = Math.max(0, Math.min(255, r + noise * ditherAmount));
          data[idx + 1] = Math.max(0, Math.min(255, g + noise * ditherAmount));
          data[idx + 2] = Math.max(0, Math.min(255, b + noise * ditherAmount));
          data[idx + 3] = 255; // Alpha
        }
      }
      
      context.putImageData(imageData, 0, 0);
      
      const texture = new THREE.CanvasTexture(canvas);
      // Better filtering to eliminate banding
      texture.minFilter = THREE.LinearMipmapLinearFilter;
      texture.magFilter = THREE.LinearFilter;
      texture.generateMipmaps = true;
      texture.anisotropy = renderer.capabilities.getMaxAnisotropy();
      // Use ClampToEdgeWrapping to prevent wrapping seams that cause dark slash
      texture.wrapS = THREE.ClampToEdgeWrapping;
      texture.wrapT = THREE.ClampToEdgeWrapping;
      // Ensure texture is not rotated for vertical alignment
      texture.rotation = 0;
      texture.needsUpdate = true;
      return texture;
    }

    const gradientTexture = createGradientTexture();

    // Create spheres - MORE BALLS and LESS CLUSTERED
    const sphereCount = 50;
    const baseRadius = 0.5;
    const geometry = new THREE.SphereGeometry(baseRadius, 64, 64);

    // Store original positions - MORE CENTERED (tighter clustering)
    const originalPositions = [];

    for (let i = 0; i < sphereCount; i++) {
      // Create material with iridescent swirling pattern
      const material = new THREE.ShaderMaterial({
        uniforms: {
          time: { value: 0.0 }, // For animation
          sphereSeed: { value: i * 123.456 } // Seed for per-sphere variation
        },
        vertexShader: `
          varying vec3 vNormal;
          varying vec3 vViewPosition;
          varying vec3 vWorldPosition;
          varying vec2 vUv;
          
          void main() {
            vNormal = normalize(normalMatrix * normal);
            vUv = uv;
            vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
            vViewPosition = -mvPosition.xyz;
            vWorldPosition = (modelMatrix * vec4(position, 1.0)).xyz;
            gl_Position = projectionMatrix * mvPosition;
          }
        `,
        fragmentShader: `
          uniform float time;
          uniform float sphereSeed;
          varying vec3 vNormal;
          varying vec3 vViewPosition;
          varying vec3 vWorldPosition;
          varying vec2 vUv;
          
          // Simple noise function
          float noise(vec2 p) {
            return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
          }
          
          // Smooth noise
          float smoothNoise(vec2 p) {
            vec2 i = floor(p);
            vec2 f = fract(p);
            f = f * f * (3.0 - 2.0 * f);
            
            float a = noise(i);
            float b = noise(i + vec2(1.0, 0.0));
            float c = noise(i + vec2(0.0, 1.0));
            float d = noise(i + vec2(1.0, 1.0));
            
            return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
          }
          
          // Fractal noise - smoother for better transitions
          float fractalNoise(vec2 p) {
            float value = 0.0;
            float amplitude = 0.5;
            for (int i = 0; i < 5; i++) {
              value += amplitude * smoothNoise(p);
              p *= 2.0;
              amplitude *= 0.5;
            }
            return value;
          }
          
          // Create swirling pattern using flow field - smoother
          vec2 flowField(vec2 uv, float t) {
            vec2 flow = vec2(0.0);
            flow.x = sin(uv.y * 2.5 + t * 0.4 + sphereSeed) * 0.5 + 0.5;
            flow.y = cos(uv.x * 2.5 + t * 0.25 + sphereSeed) * 0.5 + 0.5;
            return flow;
          }
          
          // Smooth interpolation function
          float smoothInterp(float t) {
            return t * t * (3.0 - 2.0 * t);
          }
          
          // Iridescent color based on angle and pattern - less vibrant, smoother
          vec3 iridescentColor(float pattern, vec3 normal, vec3 viewDir) {
            // Create color shift based on viewing angle - smoother
            float fresnel = 1.0 - abs(dot(normal, viewDir));
            
            // Less vibrant, more muted color palette
            vec3 color1 = vec3(0.5, 0.75, 0.7);   // Muted green
            vec3 color2 = vec3(0.55, 0.7, 0.8);   // Soft blue
            vec3 color3 = vec3(0.7, 0.65, 0.8);   // Soft purple
            vec3 color4 = vec3(0.8, 0.7, 0.75);   // Soft pink
            vec3 color5 = vec3(0.85, 0.75, 0.65);   // Soft orange
            vec3 color6 = vec3(0.9, 0.85, 0.75);   // Soft yellow
            
            // Combine pattern value with fresnel for iridescent effect - smoother transitions
            float colorIndex = fract(pattern * 0.5 + fresnel * 0.5 + sphereSeed * 0.1);
            colorIndex = smoothInterp(colorIndex); // Smooth the transitions
            
            // Smooth interpolation between colors
            vec3 color;
            float t;
            if (colorIndex < 0.17) {
              t = smoothInterp(colorIndex / 0.17);
              color = mix(color1, color2, t);
            } else if (colorIndex < 0.33) {
              t = smoothInterp((colorIndex - 0.17) / 0.16);
              color = mix(color2, color3, t);
            } else if (colorIndex < 0.5) {
              t = smoothInterp((colorIndex - 0.33) / 0.17);
              color = mix(color3, color4, t);
            } else if (colorIndex < 0.67) {
              t = smoothInterp((colorIndex - 0.5) / 0.17);
              color = mix(color4, color5, t);
            } else if (colorIndex < 0.83) {
              t = smoothInterp((colorIndex - 0.67) / 0.16);
              color = mix(color5, color6, t);
            } else {
              t = smoothInterp((colorIndex - 0.83) / 0.17);
              color = mix(color6, color1, t);
            }
            
            // Reduced highlight for less vibrancy
            float highlight = pow(fresnel, 3.0) * 0.2;
            color += vec3(highlight);
            
            return color;
          }
          
          void main() {
            vec3 normal = normalize(vNormal);
            vec3 viewDir = normalize(vViewPosition);
            
            // Calculate distance from center for radial gradient
            vec2 uv = vUv * 2.0 - 1.0;
            float distFromCenter = length(uv);
            
            // Calculate how much the surface faces the camera
            float facing = dot(normal, viewDir);
            float facing01 = (facing * 0.5 + 0.5); // 0 to 1
            
            // Create radial gradient from center to edge
            // Combine distance from center with facing for smooth gradient
            float gradient = mix(distFromCenter * 0.8, 1.0 - facing01 * 0.3, 0.7);
            gradient = smoothstep(0.0, 1.0, gradient);
            
            // Color palette matching the blurred gradient: warm to cool transition
            vec3 colorWhite = vec3(1.0, 1.0, 1.0);          // Bright white center
            vec3 colorOrange = vec3(0.9, 0.65, 0.45);       // Warm orange (from left)
            vec3 colorBrown = vec3(0.7, 0.5, 0.4);          // Earthy brown (from left)
            vec3 colorPurple = vec3(0.75, 0.6, 0.85);       // Purple/pink (transition zone)
            vec3 colorLavender = vec3(0.7, 0.75, 0.95);     // Light violet/lavender (middle)
            vec3 colorCyan = vec3(0.6, 0.9, 1.0);           // Bright light blue/cyan (right)
            vec3 colorDeepBlue = vec3(0.3, 0.5, 0.85);      // Deep saturated blue (right)
            
            // Smooth radial gradient transitions - warm to cool like the image
            vec3 color;
            float t;
            
            if (gradient < 0.25) {
              // White center to warm orange
              t = smoothInterp(gradient / 0.25);
              color = mix(colorWhite, colorOrange, t);
            } else if (gradient < 0.4) {
              // Orange to brown (warm earth tones)
              t = smoothInterp((gradient - 0.25) / 0.15);
              color = mix(colorOrange, colorBrown, t);
            } else if (gradient < 0.55) {
              // Brown to purple/pink (transition zone)
              t = smoothInterp((gradient - 0.4) / 0.15);
              color = mix(colorBrown, colorPurple, t);
            } else if (gradient < 0.7) {
              // Purple to lavender (cool transition)
              t = smoothInterp((gradient - 0.55) / 0.15);
              color = mix(colorPurple, colorLavender, t);
            } else if (gradient < 0.85) {
              // Lavender to cyan (bright blue)
              t = smoothInterp((gradient - 0.7) / 0.15);
              color = mix(colorLavender, colorCyan, t);
            } else {
              // Cyan to deep blue at edges
              t = smoothInterp((gradient - 0.85) / 0.15);
              color = mix(colorCyan, colorDeepBlue, t);
            }
            
            // Add subtle pearlescent highlight based on view angle
            float fresnel = 1.0 - abs(dot(normal, viewDir));
            float pearlescent = pow(fresnel, 2.0) * 0.15;
            color += vec3(pearlescent);
            
            // Add soft glossy highlight
            float highlight = pow(max(0.0, facing), 12.0) * 0.3;
            color += vec3(highlight);
            
            // Ensure colors stay soft and pastel
            color = pow(color, vec3(1.0));
            
            gl_FragColor = vec4(color, 1.0);
          }
        `
      });
      
      const sphere = new THREE.Mesh(geometry, material);
      // More centered - reduced spread for tighter clustering
      const x = (Math.random() - 0.5) * 2;
      const y = (Math.random() - 0.5) * 2;
      const z = (Math.random() - 0.5) * 1.5;
      
      sphere.position.set(x, y, z);
      originalPositions.push(new THREE.Vector3(x, y, z));
      
      // Rotate sphere to face the camera (screen)
      sphere.lookAt(camera.position);
      
      // Bigger scale range - store target scale but start at 0
      const targetScale = Math.random() * 0.3 + 0.8;
      sphere.scale.setScalar(0); // Start hidden
      const radius = baseRadius * targetScale; // Calculate actual radius
      
      scene.add(sphere);
      spheres.push({
        mesh: sphere,
        originalPosition: originalPositions[i],
        currentTarget: new THREE.Vector3(x, y, z),
        radius: radius, // Store radius for collision detection
        velocity: new THREE.Vector3(0, 0, 0), // Add velocity for dramatic bouncing
        speed: Math.random() * 0.02 + 0.01,
        targetScale: targetScale, // Store target scale for animation
        rotation: {
          x: (Math.random() - 0.5) * 0.02,
          y: (Math.random() - 0.5) * 0.02,
          z: (Math.random() - 0.5) * 0.02,
        },
      });
    }

    // Enhanced lighting for brighter, more uniform appearance - no shadows
    const ambientLight = new THREE.AmbientLight(0xffffff, 1.0); // Increased to maximum for even lighting
    scene.add(ambientLight);
    
    // Main directional light from above and slightly to the left - brighter for more prominent highlight
    const directionalLight = new THREE.DirectionalLight(0xffffff, 2.5); // Increased intensity
    directionalLight.position.set(3, 10, 2); // Above and slightly left
    directionalLight.castShadow = false;
    scene.add(directionalLight);
    
    // Bottom fill light to eliminate shadows on the bottom half
    const bottomLight = new THREE.DirectionalLight(0xffffff, 1.5); // Strong bottom light
    bottomLight.position.set(0, -10, 0); // From below
    scene.add(bottomLight);
    
    // Additional rim light for depth and edge definition
    const rimLight = new THREE.DirectionalLight(0xffffff, 0.5); // Slightly increased
    rimLight.position.set(-3, 5, -3);
    scene.add(rimLight);
    
    // Soft fill light for realistic shadows
    const fillLight = new THREE.DirectionalLight(0xffffff, 0.4); // Slightly increased
    fillLight.position.set(-2, 5, -2);
    scene.add(fillLight);

    // Animate spheres popping up after delay
    setTimeout(() => {
      spheres.forEach((sphereData, i) => {
        const { mesh, targetScale } = sphereData;
        
        // Animate scale from 0 to target scale with a slight delay per sphere
        setTimeout(() => {
          const startTime = Date.now();
          const duration = 600;
          
          function animatePop() {
            const elapsed = Date.now() - startTime;
            const progress = Math.min(elapsed / duration, 1);
            // Easing function for smooth pop
            const ease = 1 - Math.pow(1 - progress, 3);
            
            mesh.scale.setScalar(ease * targetScale);
            
            if (progress < 1) {
              requestAnimationFrame(animatePop);
            }
          }
          animatePop();
        }, i * 20); // Stagger each sphere by 20ms
      });
    }, 1000); // Start after 1 second (after nav/footer fade in)

    // Mouse tracking
    function handleMouseMove(event) {
      targetMouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      targetMouse.y = -(event.clientY / window.innerHeight) * 2 + 1;
    }

    // Smooth mouse position update
    function updateMousePosition() {
      mouse.x += (targetMouse.x - mouse.x) * 0.1;
      mouse.y += (targetMouse.y - mouse.y) * 0.1;
    }

    // Convert mouse position to 3D world coordinates
    function getWorldPosition(mouseX, mouseY) {
      const vector = new THREE.Vector3(mouseX, mouseY, 0.5);
      vector.unproject(camera);
      const dir = vector.sub(camera.position).normalize();
      const distance = -camera.position.z / dir.z;
      const pos = camera.position.clone().add(dir.multiplyScalar(distance));
      return pos;
    }

    window.addEventListener('mousemove', handleMouseMove);

    // Animation
    function animate() {
      requestAnimationFrame(animate);
      
      updateMousePosition();
      
      const worldMouse = getWorldPosition(mouse.x, mouse.y);

      // First, handle mouse interaction - knock individual spheres like rocks
      spheres.forEach((sphereData, i) => {
        const { mesh, originalPosition, currentTarget, velocity, radius, speed, rotation } = sphereData;
        
        // Update time uniform for iridescent animation (slow, smooth to avoid strobe)
        if (mesh.material.uniforms && mesh.material.uniforms.time) {
          mesh.material.uniforms.time.value += 0.004;
        }
        
        // Spheres are no longer rotating for stability

        // Calculate distance from mouse - only affect when very close to this specific sphere
        const distanceToMouse = mesh.position.distanceTo(worldMouse);
        const hitRadius = radius * 2.6; // Larger interaction area for more dispersed feel
        
        // Only apply impulse if mouse is very close to THIS sphere
        if (distanceToMouse < hitRadius && distanceToMouse > 0) {
          // Calculate direction from mouse to sphere (where the hit occurs)
          const hitDirection = new THREE.Vector3()
            .subVectors(mesh.position, worldMouse)
            .normalize();
          
          // Strong impulse - more dispersed when hovered
          const hitStrength = (1 - distanceToMouse / hitRadius); // Stronger when closer
          const impulseForce = hitStrength * 0.9; // Stronger push for more spread
          
          // Apply impulse to velocity (like knocking it)
          velocity.add(hitDirection.multiplyScalar(impulseForce));
        }

        // Less damping - drift longer before settling
        velocity.multiplyScalar(0.995); // Less friction, takes longer to settle
        
        // Very weak return force - takes longer to drift back to middle
        const returnForce = new THREE.Vector3()
          .subVectors(originalPosition, mesh.position)
          .multiplyScalar(0.002); // Weaker gravity, slower return to center
        velocity.add(returnForce);
        
        // Smooth out velocity changes to prevent jagged movement
        const maxVelocity = 0.15; // Limit max velocity for smoother movement
        if (velocity.length() > maxVelocity) {
          velocity.normalize().multiplyScalar(maxVelocity);
        }
        
        // Update position based on velocity (momentum-based movement)
        mesh.position.add(velocity);
        
        // Update target for collision detection
        currentTarget.copy(mesh.position);
      });

      // Collision detection and response - make spheres bounce off each other
      for (let i = 0; i < spheres.length; i++) {
        for (let j = i + 1; j < spheres.length; j++) {
          const sphere1 = spheres[i];
          const sphere2 = spheres[j];
          const pos1 = sphere1.mesh.position;
          const pos2 = sphere2.mesh.position;
          
          // Calculate distance between sphere centers
          const distance = pos1.distanceTo(pos2);
          const minDistance = sphere1.radius + sphere2.radius;
          
          // If spheres are overlapping or too close
          if (distance < minDistance && distance > 0) {
            // Calculate direction from sphere2 to sphere1
            const direction = new THREE.Vector3()
              .subVectors(pos1, pos2)
              .normalize();
            
            // Calculate overlap amount
            const overlap = minDistance - distance;
            
            // Push spheres apart based on their relative sizes
            const pushStrength = overlap * 0.5;
            const push1 = direction.clone().multiplyScalar(pushStrength * (sphere2.radius / minDistance));
            const push2 = direction.clone().multiplyScalar(-pushStrength * (sphere1.radius / minDistance));
            
            // Apply push to current positions
            pos1.add(push1);
            pos2.add(push2);
            
            // Also apply bounce to velocities
            const bounceStrength = 0.3;
            const relativeVelocity = new THREE.Vector3()
              .subVectors(sphere1.velocity, sphere2.velocity);
            const velocityAlongNormal = relativeVelocity.dot(direction);
            
            if (velocityAlongNormal > 0) continue; // Already separating - skip to next pair
            
            const restitution = 0.6; // Bounciness
            const impulse = -(1 + restitution) * velocityAlongNormal / 2;
            
            sphere1.velocity.add(direction.clone().multiplyScalar(impulse * bounceStrength));
            sphere2.velocity.sub(direction.clone().multiplyScalar(impulse * bounceStrength));
            
            // Update targets
            sphere1.currentTarget.copy(pos1);
            sphere2.currentTarget.copy(pos2);
          }
        }
      }

      renderer.render(scene, camera);
    }
    animate();

    // Handle resize
    function handleResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    }
    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
      window.removeEventListener('mousemove', handleMouseMove);
      if (renderer) {
        renderer.dispose();
      }
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

