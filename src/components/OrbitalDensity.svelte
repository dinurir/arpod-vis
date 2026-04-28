<script>
  import { onMount, onDestroy } from 'svelte';

  let canvas;
  let animFrame;
  let t = 0;

  // Orbital shells: [radius_fraction, label, color, satellite_count, speed]
  const SHELLS = [
    { r: 0.28, label: 'LEO',  color: '#3b82f6', count: 180, speed: 0.0018 },
    { r: 0.38, label: 'MEO',  color: '#8b5cf6', count: 40,  speed: 0.0008 },
    { r: 0.52, label: 'GEO',  color: '#06b6d4', count: 24,  speed: 0.0003 },
    { r: 0.62, label: 'GYO',  color: '#94a3b8', count: 18,  speed: 0.0001 },
  ];

  // Seeded pseudo-random for stable dot placement
  function seededRand(seed) {
    let s = seed;
    return () => { s = (s * 1664525 + 1013904223) & 0xffffffff; return (s >>> 0) / 0xffffffff; };
  }

  let dots = [];

  function buildDots(cx, cy, R) {
    dots = [];
    SHELLS.forEach((shell, si) => {
      const rand = seededRand(si * 9999 + 1);
      for (let i = 0; i < shell.count; i++) {
        const phase = rand() * Math.PI * 2;
        const radiusJitter = (rand() - 0.5) * R * 0.025;
        dots.push({ shell: si, phase, radiusJitter, size: 1.5 + rand() * 1.5 });
      }
    });
  }

  function draw() {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const W = canvas.width, H = canvas.height;
    const cx = W / 2, cy = H / 2;
    const R = Math.min(W, H) * 0.46;

    ctx.clearRect(0, 0, W, H);

    // Earth glow
    const earthGlow = ctx.createRadialGradient(cx, cy, 0, cx, cy, R * 0.17);
    earthGlow.addColorStop(0, 'rgba(30, 80, 180, 0.9)');
    earthGlow.addColorStop(0.6, 'rgba(16, 50, 120, 0.6)');
    earthGlow.addColorStop(1, 'rgba(5, 10, 20, 0)');
    ctx.beginPath();
    ctx.arc(cx, cy, R * 0.17, 0, Math.PI * 2);
    ctx.fillStyle = earthGlow;
    ctx.fill();

    // Earth body
    ctx.beginPath();
    ctx.arc(cx, cy, R * 0.13, 0, Math.PI * 2);
    ctx.fillStyle = '#1a3a6e';
    ctx.fill();
    ctx.strokeStyle = 'rgba(99,179,237,0.4)';
    ctx.lineWidth = 1.5;
    ctx.stroke();

    // Orbital shells
    SHELLS.forEach((shell) => {
      const r = shell.r * R;
      ctx.beginPath();
      ctx.arc(cx, cy, r, 0, Math.PI * 2);
      ctx.strokeStyle = shell.color.replace(')', ', 0.18)').replace('rgb(', 'rgba(').replace('#', '');
      // Use hex-based approach
      ctx.strokeStyle = shell.color + '30';
      ctx.lineWidth = 0.8;
      ctx.setLineDash([3, 5]);
      ctx.stroke();
      ctx.setLineDash([]);

      // Label
      ctx.fillStyle = shell.color + 'aa';
      ctx.font = `600 10px var(--font-mono, monospace)`;
      ctx.fillText(shell.label, cx + r + 6, cy - 4);
    });

    // Dots
    dots.forEach((dot) => {
      const shell = SHELLS[dot.shell];
      const angle = dot.phase + t * shell.speed;
      const r = shell.r * R + dot.radiusJitter;
      const x = cx + Math.cos(angle) * r;
      const y = cy + Math.sin(angle) * r;

      ctx.beginPath();
      ctx.arc(x, y, dot.size, 0, Math.PI * 2);
      ctx.fillStyle = shell.color;
      ctx.globalAlpha = 0.75;
      ctx.fill();
      ctx.globalAlpha = 1;
    });

    t++;
    animFrame = requestAnimationFrame(draw);
  }

  function resize() {
    if (!canvas) return;
    const parent = canvas.parentElement;
    canvas.width  = parent.clientWidth;
    canvas.height = parent.clientHeight;
    buildDots(canvas.width / 2, canvas.height / 2, Math.min(canvas.width, canvas.height) * 0.46);
  }

  onMount(() => {
    resize();
    draw();
    window.addEventListener('resize', resize);
  });

  onDestroy(() => {
    cancelAnimationFrame(animFrame);
    window.removeEventListener('resize', resize);
  });
</script>

<div class="orbital-wrap">
  <canvas bind:this={canvas}></canvas>
  <div class="legend">
    {#each [['LEO','#3b82f6','Low Earth Orbit'],['MEO','#8b5cf6','Medium Earth Orbit'],['GEO','#06b6d4','Geostationary'],['GYO','#94a3b8','Graveyard']] as [l,c,full]}
      <div class="legend-item">
        <span class="swatch" style="background:{c}"></span>
        <span class="lbl">{l}</span>
        <span class="full">{full}</span>
      </div>
    {/each}
  </div>
</div>

<style>
  .orbital-wrap {
    width: 100%;
    height: 100%;
    position: relative;
  }
  canvas {
    width: 100%;
    height: 100%;
    display: block;
  }
  .legend {
    position: absolute;
    bottom: 1rem;
    left: 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
  }
  .legend-item {
    display: flex;
    align-items: center;
    gap: 0.45rem;
    font-size: 0.72rem;
    color: var(--muted);
    font-family: var(--font-mono);
  }
  .swatch {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .lbl { font-weight: 700; color: var(--ink); }
  .full { color: var(--muted); }
</style>
