<script>
  import { onMount } from 'svelte';
  import ParetoPlot from './ParetoPlot.svelte';

  // ── Pareto data ───────────────────────────────────────────────────────────
  let paretoData = null;
  let loadError  = null;

  onMount(async () => {
    try {
      const res = await fetch('/data/pareto_latest.json');
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      paretoData = await res.json();
    } catch (e) {
      loadError = e.message;
    }
  });

  // ── Placeholder canvas sketches (FIG 1 & 2) ──────────────────────────────
  const PLACEHOLDERS = [
    {
      id: 1,
      title: 'State Estimation Error Over Time',
      desc: 'EKF/UKF position & attitude error vs. ground truth across approach phases',
      color: '#3b82f6',
      chartType: 'line',
    },
    {
      id: 2,
      title: 'Monte Carlo: Miss Distance Distribution',
      desc: 'Distribution of final approach miss distance across N = 1000 simulated runs',
      color: '#8b5cf6',
      chartType: 'histogram',
    },
  ];

  let canvases = [];

  function drawPlaceholder(canvas, type, color) {
    if (!canvas) return;
    canvas.width  = canvas.parentElement.clientWidth;
    canvas.height = canvas.parentElement.clientHeight;
    const ctx = canvas.getContext('2d');
    const W = canvas.width, H = canvas.height;

    // Grid
    ctx.strokeStyle = 'rgba(99,179,237,0.06)';
    ctx.lineWidth = 0.5;
    for (let i = 0; i <= 4; i++) {
      const x = W * 0.1 + i * (W * 0.8 / 4);
      ctx.beginPath(); ctx.moveTo(x, H * 0.1); ctx.lineTo(x, H * 0.87); ctx.stroke();
      const y = H * 0.15 + i * (H * 0.7 / 4);
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

    if (type === 'line') {
      const pts = [0.10,0.08,0.12,0.09,0.14,0.11,0.10,0.09,0.07,0.08,0.06,0.05,0.04,0.05,0.04,0.03,0.03,0.025,0.02,0.015];
      ctx.beginPath();
      pts.forEach((v, i) => {
        const x = W * 0.1 + (i / (pts.length - 1)) * W * 0.82;
        const y = H * 0.85 - v * H * 5;
        i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
      });
      ctx.strokeStyle = color;
      ctx.lineWidth = 2;
      ctx.stroke();
    }

    if (type === 'histogram') {
      const bars = [2, 5, 11, 22, 38, 45, 38, 22, 11, 5, 2];
      const bw   = W * 0.75 / bars.length;
      const maxB = Math.max(...bars);
      bars.forEach((v, i) => {
        const bh = (v / maxB) * H * 0.65;
        ctx.fillStyle = i === 5 ? color + 'ff' : color + '55';
        ctx.fillRect(W * 0.1 + i * bw, H * 0.85 - bh, bw - 2, bh);
      });
    }
  }

  onMount(() => {
    canvases.forEach((c, i) => drawPlaceholder(c, PLACEHOLDERS[i].chartType, PLACEHOLDERS[i].color));
  });
</script>

<div class="sim-wrap">
  <div class="sim-header">
    <div class="sim-badge">SIMULATION OUTPUT</div>
    <div class="sim-sub">Space Enabled Research Group · MIT Media Lab · High-Fidelity Closed-Loop CPO Simulation</div>
  </div>

  <!-- FIG 1 & 2 — placeholders awaiting separate JSON -->
  <div class="figures-grid figures-grid--two">
    {#each PLACEHOLDERS as p, i}
      <div class="figure-slot" style="--c:{p.color}">
        <div class="figure-header">
          <span class="figure-num" style="color:{p.color}">FIG. {p.id}</span>
          <span class="figure-title">{p.title}</span>
        </div>
        <p class="figure-desc">{p.desc}</p>
        <div class="chart-area">
          <canvas bind:this={canvases[i]}></canvas>
          <div class="placeholder-overlay">
            <span class="ph-label">AWAITING DATA</span>
          </div>
        </div>
      </div>
    {/each}
  </div>

  <!-- FIG 3–5 — live Pareto plots -->
  {#if loadError}
    <div class="load-error">
      Failed to load pareto_latest.json: {loadError}
    </div>
  {:else if !paretoData}
    <div class="loading-msg">Loading simulation data…</div>
  {:else}
    <div class="figures-grid figures-grid--three">
      {#each paretoData.pareto_pairs as pair, i}
        <div class="figure-slot figure-slot--pareto" style="--c:{['#06b6d4','#f59e0b','#10b981'][i]}">
          <div class="figure-header">
            <span class="figure-num" style="color:{['#06b6d4','#f59e0b','#10b981'][i]}">FIG. {i + 3}</span>
            <span class="figure-title">{pair.title}</span>
          </div>
          <p class="figure-desc">
            {pair.x_label} vs. {pair.y_label} across {paretoData.n_stacks} GNC stack configurations.
            Green = Pareto-optimal. Ellipses = ±1σ across {paretoData.stacks[0]?.metrics[pair.x_field]?.runs?.length ?? 3} runs.
          </p>
          <div class="chart-area chart-area--pareto">
            <ParetoPlot {pair} stacks={paretoData.stacks} />
          </div>
        </div>
      {/each}
    </div>

    <div class="data-note">
      Generated {new Date(paretoData.generated_at).toLocaleString()} ·
      {paretoData.n_stacks} GNC stacks · 3 Pareto dimensions
    </div>
  {/if}
</div>

<style>
  .sim-wrap {
    display: flex;
    flex-direction: column;
    gap: 1.25rem;
  }

  .sim-header { display: flex; flex-direction: column; gap: 0.3rem; }

  .sim-badge {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    font-weight: 700;
    letter-spacing: 0.18em;
    color: var(--cyan);
    text-transform: uppercase;
  }

  .sim-sub {
    font-size: 0.75rem;
    color: var(--muted);
    font-family: var(--font-mono);
  }

  .figures-grid {
    display: grid;
    gap: 1rem;
  }

  .figures-grid--two   { grid-template-columns: repeat(2, 1fr); }
  .figures-grid--three { grid-template-columns: repeat(3, 1fr); }

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

  .chart-area--pareto { height: 240px; }

  canvas { width: 100%; height: 100%; display: block; }

  .placeholder-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
  }

  .ph-label {
    font-family: var(--font-mono);
    font-size: 0.6rem;
    font-weight: 700;
    letter-spacing: 0.14em;
    color: rgba(148,163,184,0.2);
  }

  .loading-msg {
    font-family: var(--font-mono);
    font-size: 0.78rem;
    color: var(--muted);
    padding: 1.5rem 0;
    text-align: center;
  }

  .load-error {
    font-family: var(--font-mono);
    font-size: 0.78rem;
    color: #ef4444;
    padding: 0.75rem 1rem;
    background: rgba(239,68,68,0.08);
    border: 1px solid rgba(239,68,68,0.2);
  }

  .data-note {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    color: rgba(148,163,184,0.4);
    text-align: right;
    letter-spacing: 0.03em;
  }

  @media (max-width: 900px) {
    .figures-grid--two   { grid-template-columns: 1fr; }
    .figures-grid--three { grid-template-columns: 1fr; }
  }
</style>
