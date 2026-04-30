<script>
  import { onMount } from 'svelte';
  import ParetoPlot       from './ParetoPlot.svelte';
  import SingleEventPlots from './SingleEventPlots.svelte';
  import MainCPOPlots     from './MainCPOPlots.svelte';

  // ── Data loading ──────────────────────────────────────────────────────────
  let paretoData      = null;
  let singleEventData = null;
  let mainCpoData     = null;
  let mainCpoMissing  = false;

  onMount(async () => {
    const [paretoRes, seRes, mainRes] = await Promise.allSettled([
      fetch('/data/pareto_latest.json'),
      fetch('/data/single_event_latest.json'),
      fetch('/data/main_cpo_latest.json'),
    ]);

    if (paretoRes.status === 'fulfilled' && paretoRes.value.ok)
      paretoData = await paretoRes.value.json();

    if (seRes.status === 'fulfilled' && seRes.value.ok)
      singleEventData = await seRes.value.json();

    if (mainRes.status === 'fulfilled' && mainRes.value.ok)
      mainCpoData = await mainRes.value.json();
    else
      mainCpoMissing = true;
  });

  // ── Placeholder canvas sketches (FIG 1 & 2) ──────────────────────────────
  const PLACEHOLDERS = [
    { id: 1, title: 'State Estimation Error Over Time',          color: '#3b82f6', chartType: 'line',      desc: 'EKF/UKF position & attitude error vs. ground truth across approach phases' },
    { id: 2, title: 'Monte Carlo: Miss Distance Distribution',   color: '#8b5cf6', chartType: 'histogram', desc: 'Distribution of final approach miss distance across N = 1000 simulated runs' },
  ];
  let canvases = [];

  function drawPlaceholder(canvas, type, color) {
    if (!canvas) return;
    canvas.width  = canvas.parentElement.clientWidth;
    canvas.height = canvas.parentElement.clientHeight;
    const ctx = canvas.getContext('2d');
    const W = canvas.width, H = canvas.height;
    ctx.strokeStyle = 'rgba(99,179,237,0.06)'; ctx.lineWidth = 0.5;
    for (let i = 0; i <= 4; i++) {
      const x = W*0.1+i*(W*0.8/4); ctx.beginPath(); ctx.moveTo(x,H*0.1); ctx.lineTo(x,H*0.87); ctx.stroke();
      const y = H*0.15+i*(H*0.7/4); ctx.beginPath(); ctx.moveTo(W*0.08,y); ctx.lineTo(W*0.92,y); ctx.stroke();
    }
    ctx.beginPath(); ctx.moveTo(W*0.1,H*0.1); ctx.lineTo(W*0.1,H*0.87); ctx.lineTo(W*0.92,H*0.87);
    ctx.strokeStyle='rgba(148,163,184,0.3)'; ctx.lineWidth=1; ctx.stroke();
    if (type === 'line') {
      const pts=[0.10,0.08,0.12,0.09,0.14,0.11,0.10,0.09,0.07,0.08,0.06,0.05,0.04,0.05,0.04,0.03,0.03,0.025,0.02,0.015];
      ctx.beginPath();
      pts.forEach((v,i)=>{ const x=W*0.1+(i/(pts.length-1))*W*0.82,y=H*0.85-v*H*5; i===0?ctx.moveTo(x,y):ctx.lineTo(x,y); });
      ctx.strokeStyle=color; ctx.lineWidth=2; ctx.stroke();
    }
    if (type === 'histogram') {
      const bars=[2,5,11,22,38,45,38,22,11,5,2], bw=W*0.75/bars.length, maxB=Math.max(...bars);
      bars.forEach((v,i)=>{ const bh=(v/maxB)*H*0.65; ctx.fillStyle=i===5?color+'ff':color+'55'; ctx.fillRect(W*0.1+i*bw,H*0.85-bh,bw-2,bh); });
    }
  }

  onMount(() => canvases.forEach((c,i) => drawPlaceholder(c, PLACEHOLDERS[i].chartType, PLACEHOLDERS[i].color)));

  // ── Main CPO placeholder figures ──────────────────────────────────────────
  const MAIN_CPO_FIGS = [
    { num: 10, title: 'Trajectory (3D / XY)',        color: '#3b82f6', desc: 'Servicer approach trajectory in relative frame — radial, along-track, cross-track' },
    { num: 11, title: 'Control History',             color: '#8b5cf6', desc: 'Thruster command magnitude vs. time across all three axes' },
    { num: 12, title: 'Covariance Envelope',         color: '#06b6d4', desc: 'EKF position & velocity covariance (σ) vs. time through approach phases' },
    { num: 13, title: 'Run Summary Table',           color: '#f59e0b', desc: 'Per-metric summary: mean, σ, min, max, p25/p75 for all key performance indicators' },
  ];
</script>

<div class="sim-wrap">

  <!-- ── Header ────────────────────────────────────────────────────────── -->
  <div class="sim-header">
    <div class="sim-badge">SIMULATION OUTPUT</div>
    <div class="sim-sub">Space Enabled Research Group · MIT Media Lab · High-Fidelity Closed-Loop CPO Simulation</div>
  </div>

  <!-- ── FIG 1 & 2 — awaiting separate JSON ────────────────────────────── -->
  <div class="section-divider">
    <span class="section-tag">GNC STACK OVERVIEW</span>
  </div>
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
          <div class="placeholder-overlay"><span class="ph-label">AWAITING DATA</span></div>
        </div>
      </div>
    {/each}
  </div>

  <!-- ── FIG 3–5 — Pareto trade space ──────────────────────────────────── -->
  <div class="section-divider">
    <span class="section-tag">TRADE SPACE ANALYSIS — {paretoData?.n_stacks ?? '…'} GNC STACKS</span>
  </div>
  {#if !paretoData}
    <div class="loading-msg">Loading pareto data…</div>
  {:else}
    <div class="figures-grid figures-grid--three">
      {#each paretoData.pareto_pairs as pair, i}
        {@const col = ['#06b6d4','#f59e0b','#10b981'][i]}
        <div class="figure-slot figure-slot--pareto" style="--c:{col}">
          <div class="figure-header">
            <span class="figure-num" style="color:{col}">FIG. {i + 3}</span>
            <span class="figure-title">{pair.title}</span>
          </div>
          <p class="figure-desc">
            {pair.x_label} vs. {pair.y_label} across {paretoData.n_stacks} configurations.
            Green = Pareto front · Ellipses = ±1σ · Dots = individual runs.
          </p>
          <div class="chart-area chart-area--pareto">
            <ParetoPlot {pair} stacks={paretoData.stacks} />
          </div>
        </div>
      {/each}
    </div>
    <div class="data-note">
      Generated {new Date(paretoData.generated_at).toLocaleString()} ·
      {paretoData.n_stacks} stacks · 3 Pareto dimensions
    </div>
  {/if}

  <!-- ── FIG 6–9 — Single-event Monte Carlo ────────────────────────────── -->
  <div class="section-divider">
    <span class="section-tag">
      SINGLE-EVENT MONTE CARLO
      {#if singleEventData}
        — {singleEventData.n_runs} RUNS ·
        {singleEventData.config.estimator?.toUpperCase()}
        [{singleEventData.config.sensor_types?.map(s=>s.split('_')[0].toUpperCase()).join('+')}]
      {/if}
    </span>
  </div>

  {#if !singleEventData}
    <div class="loading-msg">Loading single-event data…</div>
  {:else}
    <SingleEventPlots data={singleEventData} />
    <div class="data-note">
      Generated {new Date(singleEventData.generated_at).toLocaleString()} ·
      {singleEventData.n_runs} runs · dt = {singleEventData.config.dt}s ·
      max mission time = {singleEventData.config.max_mission_time}s
    </div>
  {/if}

  <!-- ── FIG 10–13 — Main CPO run ───────────────────────────────────────── -->
  <div class="section-divider">
    <span class="section-tag">MAIN CPO RUN — DETAILED ANALYSIS</span>
  </div>

  {#if mainCpoMissing}
    <div class="export-prompt">
      <div class="export-prompt-header">
        <span class="export-icon">⚙</span>
        <span><code>main_cpo_latest.json</code> not found — run the following in MATLAB to export:</span>
      </div>
      <pre class="export-code">loaded = load('results/gnc_stack_results_latest.mat');
export_main_cpo_json(loaded.stack_results, 'results/main_cpo_latest.json');</pre>
      <p class="export-note">
        Then copy: <code>cp results/main_cpo_latest.json arpod_vis/public/data/</code> and reload.
      </p>
    </div>
  {:else if mainCpoData}
    <MainCPOPlots data={mainCpoData} />
    <div class="data-note">
      Generated {new Date(mainCpoData.generated_at).toLocaleString()} ·
      Single deterministic run · {mainCpoData.config.estimator?.toUpperCase()} ·
      {mainCpoData.trajectory.t?.length} timesteps
    </div>
  {/if}

</div>

<style>
  .sim-wrap { display: flex; flex-direction: column; gap: 1.5rem; }

  .sim-header { display: flex; flex-direction: column; gap: 0.3rem; }
  .sim-badge {
    font-family: var(--font-mono); font-size: 0.65rem; font-weight: 700;
    letter-spacing: 0.18em; color: var(--cyan); text-transform: uppercase;
  }
  .sim-sub { font-size: 0.75rem; color: var(--muted); font-family: var(--font-mono); }

  /* ── Section dividers ─────────────────────────────────────────────────── */
  .section-divider {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    margin-top: 0.5rem;
  }
  .section-divider::before, .section-divider::after {
    content: '';
    flex: 1;
    height: 1px;
    background: var(--border);
  }
  .section-tag {
    font-family: var(--font-mono); font-size: 0.6rem; font-weight: 700;
    letter-spacing: 0.14em; color: var(--muted); white-space: nowrap;
  }

  /* ── Figure grids ─────────────────────────────────────────────────────── */
  .figures-grid { display: grid; gap: 1rem; }
  .figures-grid--two   { grid-template-columns: repeat(2, 1fr); }
  .figures-grid--three { grid-template-columns: repeat(3, 1fr); }

  .figure-slot {
    background: var(--bg-card); border: 1px solid var(--border);
    border-top: 2px solid var(--c); padding: 1rem;
    display: flex; flex-direction: column; gap: 0.5rem;
    animation: fadeInUp 500ms ease-out both;
    transition: opacity 300ms ease;
  }
  @keyframes fadeInUp { from { opacity:0; transform:translateY(12px); } to { opacity:1; transform:translateY(0); } }

  .figure-header { display: flex; align-items: baseline; gap: 0.65rem; }
  .figure-num { font-family: var(--font-mono); font-size: 0.68rem; font-weight: 700; letter-spacing: 0.06em; flex-shrink: 0; }
  .figure-title { font-family: var(--font-ui); font-size: 0.82rem; font-weight: 700; color: var(--ink); line-height: 1.3; }
  .figure-desc { font-size: 0.73rem; line-height: 1.5; color: var(--muted); margin: 0; }

  .chart-area {
    position: relative; height: 160px;
    background: rgba(5,10,20,0.6); border: 1px solid rgba(99,179,237,0.08); overflow: hidden;
  }
  .chart-area--pareto { height: 240px; }
  .chart-area--med    { height: 200px; }

  canvas { width: 100%; height: 100%; display: block; }

  .placeholder-overlay {
    position: absolute; inset: 0;
    display: flex; align-items: center; justify-content: center; pointer-events: none;
  }
  .ph-label {
    font-family: var(--font-mono); font-size: 0.6rem; font-weight: 700;
    letter-spacing: 0.14em; color: rgba(148,163,184,0.2);
  }

  /* ── Export prompt ────────────────────────────────────────────────────── */
  .export-prompt {
    background: rgba(245,158,11,0.05); border: 1px solid rgba(245,158,11,0.2);
    padding: 1rem 1.1rem; display: flex; flex-direction: column; gap: 0.65rem;
  }
  .export-prompt-header {
    display: flex; align-items: flex-start; gap: 0.6rem;
    font-size: 0.8rem; color: rgba(226,232,240,0.75); line-height: 1.5;
  }
  .export-icon { font-size: 0.95rem; flex-shrink: 0; padding-top: 0.05rem; }
  .export-prompt-header code, .export-note code {
    font-family: var(--font-mono); background: rgba(245,158,11,0.12);
    padding: 0.1em 0.35em; border-radius: 2px; font-size: 0.85em; color: #f59e0b;
  }
  .export-code {
    font-family: var(--font-mono); font-size: 0.78rem; line-height: 1.65;
    background: rgba(5,10,20,0.7); border: 1px solid rgba(99,179,237,0.1);
    padding: 0.75rem 1rem; color: var(--cyan); overflow-x: auto; margin: 0;
  }
  .export-note { font-size: 0.73rem; color: var(--muted); margin: 0; line-height: 1.5; }

  /* ── Misc ─────────────────────────────────────────────────────────────── */
  .loading-msg { font-family: var(--font-mono); font-size: 0.78rem; color: var(--muted); padding: 1.5rem 0; text-align: center; }
  .data-note { font-family: var(--font-mono); font-size: 0.65rem; color: rgba(148,163,184,0.4); text-align: right; letter-spacing: 0.03em; }

  @media (max-width: 900px) {
    .figures-grid--two   { grid-template-columns: 1fr; }
    .figures-grid--three { grid-template-columns: 1fr; }
  }
</style>
