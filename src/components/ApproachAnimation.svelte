<script>
  import { onMount, onDestroy } from 'svelte';

  let canvas;
  let animFrame;
  let t = 0;

  // Servicer approaches from left, stops ~40px from target
  const TARGET_X_FRAC = 0.62;
  const STOP_DIST = 0.07; // fraction of width
  let servicerX = 0.05;
  let approachComplete = false;

  function draw() {
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const W = canvas.width, H = canvas.height;
    const cx = W / 2, cy = H / 2;

    ctx.clearRect(0, 0, W, H);

    // Star field
    ctx.save();
    const starSeed = 42;
    for (let i = 0; i < 120; i++) {
      const s = ((starSeed * (i * 7 + 1) * 1664525 + 1013904223) >>> 0) / 0xffffffff;
      const s2 = ((starSeed * (i * 13 + 3) * 1664525 + 1013904223) >>> 0) / 0xffffffff;
      const s3 = ((starSeed * (i * 17 + 7) * 1664525 + 1013904223) >>> 0) / 0xffffffff;
      const sx = s * W, sy = s2 * H;
      const br = 0.2 + s3 * 0.7;
      ctx.beginPath();
      ctx.arc(sx, sy, 0.6 + s3 * 0.9, 0, Math.PI * 2);
      ctx.fillStyle = `rgba(226,232,240,${br})`;
      ctx.fill();
    }
    ctx.restore();

    const tx = W * TARGET_X_FRAC;
    const ty = cy;

    // TARGET satellite body
    ctx.save();
    ctx.translate(tx, ty);
    // Body
    ctx.fillStyle = '#475569';
    ctx.fillRect(-18, -12, 36, 24);
    ctx.strokeStyle = 'rgba(148,163,184,0.5)';
    ctx.lineWidth = 1;
    ctx.strokeRect(-18, -12, 36, 24);
    // Solar panels
    ctx.fillStyle = '#1e3a5f';
    ctx.fillRect(-50, -5, 28, 10);
    ctx.fillRect(22, -5, 28, 10);
    ctx.strokeStyle = 'rgba(59,130,246,0.5)';
    ctx.strokeRect(-50, -5, 28, 10);
    ctx.strokeRect(22, -5, 28, 10);
    // Panel grid lines
    for (let gi = 1; gi < 4; gi++) {
      ctx.beginPath();
      ctx.moveTo(-50 + gi * 7, -5);
      ctx.lineTo(-50 + gi * 7,  5);
      ctx.strokeStyle = 'rgba(59,130,246,0.25)';
      ctx.lineWidth = 0.5;
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(22 + gi * 7, -5);
      ctx.lineTo(22 + gi * 7,  5);
      ctx.stroke();
    }
    // Antenna
    ctx.beginPath();
    ctx.moveTo(0, -12);
    ctx.lineTo(0, -28);
    ctx.strokeStyle = 'rgba(148,163,184,0.6)';
    ctx.lineWidth = 1.5;
    ctx.stroke();
    ctx.beginPath();
    ctx.arc(0, -28, 5, 0, Math.PI * 2);
    ctx.strokeStyle = 'rgba(148,163,184,0.4)';
    ctx.stroke();

    // Label
    ctx.fillStyle = 'rgba(148,163,184,0.8)';
    ctx.font = '600 9px var(--font-mono, monospace)';
    ctx.textAlign = 'center';
    ctx.fillText('CLIENT SAT', 0, 28);
    ctx.restore();

    // Range rings around target
    const dist = (tx - W * servicerX);
    const ringAlpha = Math.max(0, 1 - dist / (W * 0.4));
    [60, 90, 130].forEach((r) => {
      ctx.beginPath();
      ctx.arc(tx, ty, r, 0, Math.PI * 2);
      ctx.strokeStyle = `rgba(6,182,212,${ringAlpha * 0.15})`;
      ctx.lineWidth = 1;
      ctx.setLineDash([3, 5]);
      ctx.stroke();
      ctx.setLineDash([]);
    });

    // Approach corridor
    ctx.save();
    const corridorGrad = ctx.createLinearGradient(W * 0.05, cy, tx - 55, cy);
    corridorGrad.addColorStop(0, 'rgba(59,130,246,0)');
    corridorGrad.addColorStop(1, 'rgba(59,130,246,0.07)');
    ctx.fillStyle = corridorGrad;
    ctx.fillRect(W * 0.05, cy - 20, (tx - 55) - W * 0.05, 40);
    // Corridor borders
    ctx.beginPath();
    ctx.moveTo(W * 0.05, cy - 20);
    ctx.lineTo(tx - 55, cy - 20);
    ctx.moveTo(W * 0.05, cy + 20);
    ctx.lineTo(tx - 55, cy + 20);
    ctx.strokeStyle = 'rgba(59,130,246,0.2)';
    ctx.lineWidth = 1;
    ctx.setLineDash([4, 6]);
    ctx.stroke();
    ctx.setLineDash([]);
    ctx.restore();

    // Servicer movement
    const stopX = W * (TARGET_X_FRAC - STOP_DIST);
    if (W * servicerX < stopX) {
      servicerX += 0.0006;
    } else if (!approachComplete) {
      approachComplete = true;
    }
    // Oscillate slightly after stop
    const sxPx = approachComplete
      ? W * servicerX + Math.sin(t * 0.04) * 1.5
      : W * servicerX;
    const syPx = cy + Math.sin(t * 0.025) * 3;

    // Thruster plume
    if (!approachComplete || W * servicerX < stopX) {
      const plumeGrad = ctx.createLinearGradient(sxPx - 5, syPx, sxPx - 30, syPx);
      plumeGrad.addColorStop(0, 'rgba(59,130,246,0.55)');
      plumeGrad.addColorStop(1, 'rgba(59,130,246,0)');
      ctx.beginPath();
      ctx.moveTo(sxPx - 5, syPx - 5);
      ctx.lineTo(sxPx - 28, syPx);
      ctx.lineTo(sxPx - 5, syPx + 5);
      ctx.fillStyle = plumeGrad;
      ctx.fill();
    }

    // SERVICER body
    ctx.save();
    ctx.translate(sxPx, syPx);
    ctx.fillStyle = '#1e3a5f';
    ctx.fillRect(-10, -8, 20, 16);
    ctx.strokeStyle = '#3b82f6';
    ctx.lineWidth = 1.5;
    ctx.strokeRect(-10, -8, 20, 16);
    // Small solar panels
    ctx.fillStyle = '#0ea5e9';
    ctx.fillRect(-22, -3, 10, 6);
    ctx.fillRect(12, -3, 10, 6);
    ctx.strokeStyle = 'rgba(59,130,246,0.5)';
    ctx.strokeRect(-22, -3, 10, 6);
    ctx.strokeRect(12, -3, 10, 6);
    // Label
    ctx.fillStyle = 'rgba(59,130,246,0.9)';
    ctx.font = '700 9px var(--font-mono, monospace)';
    ctx.textAlign = 'center';
    ctx.fillText('SERVICER', 0, 22);
    ctx.restore();

    // Range readout
    const realDist = Math.max(0, tx - sxPx - 22);
    const distKm = (realDist * 0.8).toFixed(1);
    ctx.fillStyle = 'rgba(6,182,212,0.8)';
    ctx.font = '600 10px var(--font-mono, monospace)';
    ctx.textAlign = 'left';
    ctx.fillText(`RANGE: ${distKm} m`, 16, H - 20);

    if (approachComplete) {
      ctx.fillStyle = 'rgba(16,185,129,0.85)';
      ctx.font = '700 10px var(--font-mono, monospace)';
      ctx.fillText('STATIONKEEPING — DOCKING PHASE', 16, H - 36);
    }

    t++;
    animFrame = requestAnimationFrame(draw);
  }

  function resize() {
    if (!canvas) return;
    const p = canvas.parentElement;
    canvas.width = p.clientWidth;
    canvas.height = p.clientHeight;
    // Reset approach on resize
    servicerX = 0.05;
    approachComplete = false;
  }

  onMount(() => { resize(); draw(); window.addEventListener('resize', resize); });
  onDestroy(() => { cancelAnimationFrame(animFrame); window.removeEventListener('resize', resize); });
</script>

<div class="approach-wrap">
  <canvas bind:this={canvas}></canvas>
</div>

<style>
  .approach-wrap { width: 100%; height: 100%; }
  canvas { width: 100%; height: 100%; display: block; }
</style>
