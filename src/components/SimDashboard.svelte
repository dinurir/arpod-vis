<script>
  const PLACEHOLDERS = [
    {
      id: 1,
      title: 'State Estimation Error Over Time',
      desc: 'EKF/UKF position & attitude error vs. ground truth across approach phases',
      axis_x: 'Time (s)',
      axis_y: 'Error (m / deg)',
      color: '#3b82f6',
      chartType: 'line',
    },
    {
      id: 2,
      title: 'Monte Carlo: Miss Distance Distribution',
      desc: 'Distribution of final approach miss distance across N = 1000 simulated runs with hardware uncertainty draws',
      axis_x: 'Miss Distance (m)',
      axis_y: 'Count',
      color: '#8b5cf6',
      chartType: 'histogram',
    },
    {
      id: 3,
      title: 'Trade Space: Safety vs. Fuel Use',
      desc: 'Pareto frontier across sensor / filter / control design parameter combinations',
      axis_x: 'ΔV Used (m/s)',
      axis_y: 'P(safe completion)',
      color: '#06b6d4',
      chartType: 'scatter',
    },
    {
      id: 4,
      title: 'STPA Unsafe Control Actions Heat Map',
      desc: 'Frequency and severity of UCAs mapped across approach phases and system states',
      axis_x: 'Approach Phase',
      axis_y: 'Subsystem',
      color: '#f59e0b',
      chartType: 'heatmap',
    },
    {
      id: 5,
      title: 'Sensor Failure Sensitivity Analysis',
      desc: 'Impact of individual sensor degradation (LiDAR, camera, IMU) on overall CPO success probability',
      axis_x: 'Degradation Level',
      axis_y: 'Success Rate (%)',
      color: '#10b981',
      chartType: 'bar',
    },
  ];

  const CHART_SKETCH = {
    line: (ctx, w, h, color) => {
      ctx.beginPath();
      const pts = [0.1,0.08,0.12,0.09,0.14,0.11,0.10,0.09,0.07,0.08,0.06,0.05,0.04,0.05,0.04,0.03,0.03,0.025,0.02,0.015];
      pts.forEach((v, i) => {
        const x = w * 0.08 + (i / (pts.length - 1)) * w * 0.84;
        const y = h * 0.85 - v * h * 5;
        i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
      });
      ctx.strokeStyle = color;
      ctx.lineWidth = 2;
      ctx.stroke();
    },
    histogram: (ctx, w, h, color) => {
      const bars = [2, 5, 11, 22, 38, 45, 38, 22, 11, 5, 2];
      const bw = w * 0.75 / bars.length;
      const max = Math.max(...bars);
      bars.forEach((v, i) => {
        const bh = (v / max) * h * 0.65;
        const x = w * 0.1 + i * bw;
        const y = h * 0.85 - bh;
        ctx.fillStyle = color + (i === 5 ? 'ff' : '66');
        ctx.fillRect(x, y, bw - 2, bh);
      });
    },
    scatter: (ctx, w, h, color) => {
      const pts = [[0.15,0.85],[0.22,0.75],[0.28,0.68],[0.35,0.60],[0.44,0.52],[0.55,0.42],[0.66,0.35],[0.76,0.30],[0.85,0.26]];
      pts.forEach(([px, py], i) => {
        ctx.beginPath();
        ctx.arc(w * px, h * py, 4 + i * 0.5, 0, Math.PI * 2);
        ctx.fillStyle = color + 'bb';
        ctx.fill();
      });
      // Pareto line
      ctx.beginPath();
      pts.forEach(([px, py], i) => { i === 0 ? ctx.moveTo(w * px, h * py) : ctx.lineTo(w * px, h * py); });
      ctx.strokeStyle = color + '55';
      ctx.lineWidth = 1.5;
      ctx.setLineDash([3, 4]);
      ctx.stroke();
      ctx.setLineDash([]);
    },
    heatmap: (ctx, w, h, color) => {
      const rows = 5, cols = 6;
      const cw = (w * 0.82) / cols, ch = (h * 0.65) / rows;
      for (let r = 0; r < rows; r++) {
        for (let c = 0; c < cols; c++) {
          const v = Math.random();
          const alpha = Math.round(v * 200).toString(16).padStart(2, '0');
          ctx.fillStyle = color + alpha;
          ctx.fillRect(w * 0.1 + c * cw, h * 0.15 + r * ch, cw - 1, ch - 1);
        }
      }
    },
    bar: (ctx, w, h, color) => {
      const vals = [0.92, 0.78, 0.55, 0.71, 0.88];
      const bw = w * 0.7 / vals.length;
      vals.forEach((v, i) => {
        const bh = v * h * 0.65;
        ctx.fillStyle = i < 2 ? '#ef4444aa' : color + 'cc';
        ctx.fillRect(w * 0.1 + i * (bw + 8), h * 0.85 - bh, bw, bh);
      });
    },
  };

  import { onMount } from 'svelte';
  let canvases = [];

  onMount(() => {
    canvases.forEach((canvas, i) => {
      if (!canvas) return;
      const p = PLACEHOLDERS[i];
      canvas.width = canvas.parentElement.clientWidth;
      canvas.height = canvas.parentElement.clientHeight;
      const ctx = canvas.getContext('2d');
      const W = canvas.width, H = canvas.height;

      // Background grid
      ctx.strokeStyle = 'rgba(99,179,237,0.06)';
      ctx.lineWidth = 0.5;
      for (let gx = 0; gx <= 4; gx++) {
        const x = W * 0.1 + gx * (W * 0.8 / 4);
        ctx.beginPath(); ctx.moveTo(x, H * 0.1); ctx.lineTo(x, H * 0.87); ctx.stroke();
      }
      for (let gy = 0; gy <= 4; gy++) {
        const y = H * 0.15 + gy * (H * 0.7 / 4);
        ctx.beginPath(); ctx.moveTo(W * 0.08, y); ctx.lineTo(W * 0.92, y); ctx.stroke();
      }

      // Axes
      ctx.beginPath();
      ctx.moveTo(W * 0.1, H * 0.1);
      ctx.lineTo(W * 0.1, H * 0.87);
      ctx.lineTo(W * 0.92, H * 0.87);
      ctx.strokeStyle = 'rgba(148,163,184,0.3)';
      ctx.lineWidth = 1;
      ctx.stroke();

      // Chart sketch
      CHART_SKETCH[p.chartType]?.(ctx, W, H, p.color);

      // Axis labels
      ctx.font = '500 9px var(--font-mono, monospace)';
      ctx.fillStyle = 'rgba(148,163,184,0.6)';
      ctx.textAlign = 'center';
      ctx.fillText(p.axis_x, W * 0.51, H * 0.96);
      ctx.save();
      ctx.translate(W * 0.03, H * 0.5);
      ctx.rotate(-Math.PI / 2);
      ctx.fillText(p.axis_y, 0, 0);
      ctx.restore();
    });
  });
</script>

<div class="sim-wrap">
  <div class="sim-header">
    <div class="sim-badge">SIMULATION OUTPUT</div>
    <div class="sim-sub">Space Enabled Research Group · MIT Media Lab · High-Fidelity Closed-Loop CPO Simulation</div>
  </div>

  <div class="figures-grid">
    {#each PLACEHOLDERS as p, i}
      <div class="figure-slot" style="--c:{p.color}; animation-delay:{i*100}ms">
        <div class="figure-header">
          <span class="figure-num" style="color:{p.color}">FIG. {p.id}</span>
          <span class="figure-title">{p.title}</span>
        </div>
        <p class="figure-desc">{p.desc}</p>
        <div class="chart-area">
          <canvas bind:this={canvases[i]}></canvas>
          <div class="placeholder-overlay">
            <span class="ph-label">FIGURE PLACEHOLDER</span>
            <span class="ph-type">{p.chartType.toUpperCase()}</span>
          </div>
        </div>
      </div>
    {/each}
  </div>
</div>

<style>
  .sim-wrap {
    display: flex;
    flex-direction: column;
    gap: 1.5rem;
  }

  .sim-header {
    display: flex;
    flex-direction: column;
    gap: 0.3rem;
  }

  .sim-badge {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    font-weight: 700;
    letter-spacing: 0.18em;
    text-transform: uppercase;
    color: var(--cyan);
  }

  .sim-sub {
    font-size: 0.75rem;
    color: var(--muted);
    font-family: var(--font-mono);
  }

  .figures-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 1rem;
  }

  .figure-slot:nth-child(5) {
    grid-column: 1 / -1;
  }

  .figure-slot {
    background: var(--bg-card);
    border: 1px solid var(--border);
    border-top: 2px solid var(--c);
    padding: 1rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    animation: fadeInUp 500ms ease-out both;
  }

  @keyframes fadeInUp {
    from { opacity: 0; transform: translateY(12px); }
    to   { opacity: 1; transform: translateY(0); }
  }

  .figure-header {
    display: flex;
    align-items: baseline;
    gap: 0.65rem;
  }

  .figure-num {
    font-family: var(--font-mono);
    font-size: 0.68rem;
    font-weight: 700;
    letter-spacing: 0.06em;
    flex-shrink: 0;
  }

  .figure-title {
    font-family: var(--font-ui);
    font-size: 0.82rem;
    font-weight: 700;
    color: var(--ink);
    line-height: 1.3;
  }

  .figure-desc {
    font-size: 0.73rem;
    line-height: 1.5;
    color: var(--muted);
    margin: 0;
  }

  .chart-area {
    position: relative;
    height: 160px;
    background: rgba(5,10,20,0.6);
    border: 1px solid rgba(99,179,237,0.08);
    overflow: hidden;
  }

  canvas { width: 100%; height: 100%; display: block; }

  .placeholder-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 0.3rem;
    pointer-events: none;
  }

  .ph-label {
    font-family: var(--font-mono);
    font-size: 0.6rem;
    font-weight: 700;
    letter-spacing: 0.14em;
    color: rgba(148,163,184,0.25);
  }

  .ph-type {
    font-family: var(--font-mono);
    font-size: 0.55rem;
    letter-spacing: 0.1em;
    color: rgba(148,163,184,0.15);
  }

  @media (max-width: 760px) {
    .figures-grid { grid-template-columns: 1fr; }
    .figure-slot:nth-child(5) { grid-column: 1; }
  }
</style>
