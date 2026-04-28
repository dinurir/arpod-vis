<script>
  import { onMount, onDestroy } from 'svelte';

  let canvas;
  let animFrame;
  let t = 0;

  const GEO_SATS = [
    { name: 'Intelsat 901', angle: -0.6,  size: 5, highlight: true  },
    { name: 'SES-12',       angle: -0.2,  size: 4, highlight: false },
    { name: 'Astra 1N',     angle:  0.1,  size: 4, highlight: false },
    { name: 'EchoStar 3',   angle:  0.45, size: 4, highlight: false },
    { name: 'Intelsat 10',  angle:  0.75, size: 5, highlight: true  },
    { name: 'Galaxy 30',    angle:  1.05, size: 4, highlight: false },
    { name: 'AMC-11',       angle:  1.35, size: 5, highlight: true  },
    { name: 'NSS-7',        angle:  1.65, size: 4, highlight: false },
    { name: 'Intelsat 19',  angle:  1.95, size: 4, highlight: false },
    { name: 'SES-6',        angle:  2.25, size: 4, highlight: false },
    { name: 'Anik F2',      angle:  2.55, size: 5, highlight: true  },
    { name: 'WildBlue-1',   angle:  2.85, size: 4, highlight: false },
  ];

  const SERVICER = { angle: 0, orbitOffset: 0.022 };
  let servicerAngle = -0.8;

  function draw() {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const W = canvas.width, H = canvas.height;
    const cx = W * 0.5, cy = H * 0.72;
    const R = Math.min(W, H) * 0.55;

    ctx.clearRect(0, 0, W, H);

    // Earth arc (partial circle at bottom)
    const earthGrad = ctx.createRadialGradient(cx, cy, R * 0.25, cx, cy, R * 0.42);
    earthGrad.addColorStop(0, 'rgba(20, 70, 160, 0.9)');
    earthGrad.addColorStop(1, 'rgba(5, 10, 20, 0)');
    ctx.beginPath();
    ctx.arc(cx, cy, R * 0.38, 0, Math.PI * 2);
    ctx.fillStyle = earthGrad;
    ctx.fill();

    ctx.beginPath();
    ctx.arc(cx, cy, R * 0.28, 0, Math.PI * 2);
    ctx.fillStyle = '#1a3a6e';
    ctx.fill();
    ctx.strokeStyle = 'rgba(99,179,237,0.5)';
    ctx.lineWidth = 1.5;
    ctx.stroke();

    // Atmosphere ring
    ctx.beginPath();
    ctx.arc(cx, cy, R * 0.32, 0, Math.PI * 2);
    ctx.strokeStyle = 'rgba(59,130,246,0.2)';
    ctx.lineWidth = 6;
    ctx.stroke();

    // GEO arc (top half only)
    const arcGrad = ctx.createLinearGradient(cx - R, cy, cx + R, cy);
    arcGrad.addColorStop(0, 'rgba(6,182,212,0.08)');
    arcGrad.addColorStop(0.5, 'rgba(6,182,212,0.22)');
    arcGrad.addColorStop(1, 'rgba(6,182,212,0.08)');
    ctx.beginPath();
    ctx.arc(cx, cy, R, Math.PI * 1.05, Math.PI * 1.95);
    ctx.strokeStyle = arcGrad;
    ctx.lineWidth = 2;
    ctx.setLineDash([4, 6]);
    ctx.stroke();
    ctx.setLineDash([]);

    // GEO label
    ctx.save();
    ctx.translate(cx, cy - R - 14);
    ctx.fillStyle = 'rgba(6,182,212,0.65)';
    ctx.font = '600 10px var(--font-mono, monospace)';
    ctx.textAlign = 'center';
    ctx.fillText('GEO ARC — 35,786 km', 0, 0);
    ctx.restore();

    // Distance indicator line
    ctx.beginPath();
    ctx.moveTo(cx, cy - R * 0.28);
    ctx.lineTo(cx, cy - R);
    ctx.strokeStyle = 'rgba(6,182,212,0.2)';
    ctx.lineWidth = 1;
    ctx.setLineDash([2, 4]);
    ctx.stroke();
    ctx.setLineDash([]);

    // Satellites
    GEO_SATS.forEach((sat) => {
      const angle = Math.PI + sat.angle + t * 0.00015;
      const x = cx + Math.cos(angle) * R;
      const y = cy + Math.sin(angle) * R;

      if (sat.highlight) {
        // Glow
        const glow = ctx.createRadialGradient(x, y, 0, x, y, sat.size * 4);
        glow.addColorStop(0, 'rgba(245,158,11,0.5)');
        glow.addColorStop(1, 'rgba(245,158,11,0)');
        ctx.beginPath();
        ctx.arc(x, y, sat.size * 4, 0, Math.PI * 2);
        ctx.fillStyle = glow;
        ctx.fill();

        ctx.beginPath();
        ctx.arc(x, y, sat.size, 0, Math.PI * 2);
        ctx.fillStyle = '#f59e0b';
        ctx.fill();

        // Solar panel lines
        ctx.beginPath();
        ctx.moveTo(x - sat.size * 2.5, y);
        ctx.lineTo(x + sat.size * 2.5, y);
        ctx.strokeStyle = 'rgba(245,158,11,0.6)';
        ctx.lineWidth = 1.5;
        ctx.stroke();
      } else {
        ctx.beginPath();
        ctx.arc(x, y, sat.size * 0.8, 0, Math.PI * 2);
        ctx.fillStyle = 'rgba(148,163,184,0.7)';
        ctx.fill();
      }
    });

    // Animated servicer
    servicerAngle += 0.0008;
    const sAngle = Math.PI + servicerAngle;
    const sR = R + 10;
    const sx = cx + Math.cos(sAngle) * sR;
    const sy = cy + Math.sin(sAngle) * sR;

    const sGlow = ctx.createRadialGradient(sx, sy, 0, sx, sy, 12);
    sGlow.addColorStop(0, 'rgba(59,130,246,0.6)');
    sGlow.addColorStop(1, 'rgba(59,130,246,0)');
    ctx.beginPath();
    ctx.arc(sx, sy, 12, 0, Math.PI * 2);
    ctx.fillStyle = sGlow;
    ctx.fill();

    ctx.beginPath();
    ctx.arc(sx, sy, 3.5, 0, Math.PI * 2);
    ctx.fillStyle = '#3b82f6';
    ctx.fill();

    // Servicer label
    ctx.fillStyle = 'rgba(59,130,246,0.8)';
    ctx.font = '600 9px var(--font-mono, monospace)';
    ctx.fillText('SERVICER', sx + 8, sy - 6);

    t++;
    animFrame = requestAnimationFrame(draw);
  }

  function resize() {
    if (!canvas) return;
    const p = canvas.parentElement;
    canvas.width  = p.clientWidth;
    canvas.height = p.clientHeight;
  }

  onMount(() => { resize(); draw(); window.addEventListener('resize', resize); });
  onDestroy(() => { cancelAnimationFrame(animFrame); window.removeEventListener('resize', resize); });
</script>

<div class="geo-wrap">
  <canvas bind:this={canvas}></canvas>
  <div class="note">
    <span class="dot amber"></span> High-value GEO asset (life-extension candidate)
    <span class="dot blue" style="margin-left:1rem"></span> Servicer vehicle
  </div>
</div>

<style>
  .geo-wrap { width: 100%; height: 100%; position: relative; }
  canvas { width: 100%; height: 100%; display: block; }
  .note {
    position: absolute;
    bottom: 1rem;
    left: 50%;
    transform: translateX(-50%);
    font-size: 0.7rem;
    color: var(--muted);
    font-family: var(--font-mono);
    display: flex;
    align-items: center;
    gap: 0.4rem;
    white-space: nowrap;
  }
  .dot { width: 8px; height: 8px; border-radius: 50%; display: inline-block; }
  .amber { background: #f59e0b; }
  .blue  { background: #3b82f6; }
</style>
