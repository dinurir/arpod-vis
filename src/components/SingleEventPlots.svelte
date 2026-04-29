<script>
  import * as d3 from 'd3';
  import { onMount, afterUpdate } from 'svelte';

  export let data; // full single_event JSON

  // ── Derived ───────────────────────────────────────────────────────────────
  $: trajs   = data?.trajectories ?? [];
  $: metrics = data?.metrics ?? {};
  $: config  = data?.config ?? {};

  // Interpolate all trajectories onto a shared time grid
  function buildEnvelope(trajs, field) {
    if (!trajs.length) return null;
    const tMax  = d3.max(trajs, tr => tr.t[tr.t.length - 1]);
    const tMin  = d3.min(trajs, tr => tr.t[0]);
    const N     = 200;
    const tGrid = d3.range(N).map(i => tMin + (i / (N - 1)) * (tMax - tMin));

    const interped = trajs.map(tr => {
      const bisect = d3.bisector(d => d).left;
      return tGrid.map(tq => {
        const i = bisect(tr.t, tq);
        if (i <= 0) return tr[field][0];
        if (i >= tr.t.length) return tr[field][tr.t.length - 1];
        const t0 = tr.t[i - 1], t1 = tr.t[i];
        const v0 = tr[field][i - 1], v1 = tr[field][i];
        const alpha = (tq - t0) / (t1 - t0 || 1);
        return v0 + alpha * (v1 - v0);
      });
    });

    return tGrid.map((t, i) => {
      const vals = interped.map(r => r[i]).filter(v => v != null);
      const mean = d3.mean(vals);
      const std  = d3.deviation(vals) ?? 0;
      return { t, mean, lo: mean - std, hi: mean + std, min: d3.min(vals), max: d3.max(vals) };
    });
  }

  // Box stats from runs array
  function boxStats(runs) {
    const sorted = [...runs].filter(v => v != null).sort(d3.ascending);
    return {
      min: sorted[0],
      q1:  d3.quantile(sorted, 0.25),
      med: d3.quantile(sorted, 0.5),
      q3:  d3.quantile(sorted, 0.75),
      max: sorted[sorted.length - 1],
      mean: d3.mean(sorted),
      std:  d3.deviation(sorted),
    };
  }

  // ── Canvas/SVG refs ───────────────────────────────────────────────────────
  let rangeEl, durEl, energyEl, computeEl;

  const MARGIN = { top: 18, right: 20, bottom: 44, left: 54 };
  const M2     = { top: 18, right: 20, bottom: 44, left: 68 };

  function drawRangeEnvelope(el) {
    if (!el || !trajs.length) return;
    const W = el.clientWidth, H = el.clientHeight || 240;
    const iW = W - MARGIN.left - MARGIN.right;
    const iH = H - MARGIN.top  - MARGIN.bottom;
    d3.select(el).selectAll('*').remove();

    const env = buildEnvelope(trajs, 'range');
    if (!env) return;

    const xScale = d3.scaleLinear().domain([env[0].t, env[env.length-1].t]).range([0, iW]);
    const yScale = d3.scaleLinear().domain([0, d3.max(env, d => d.max) * 1.05]).range([iH, 0]);

    const svg = d3.select(el).append('g').attr('transform', `translate(${MARGIN.left},${MARGIN.top})`);

    // Grid
    svg.append('g').selectAll('line').data(yScale.ticks(5)).join('line')
      .attr('x1',0).attr('x2',iW).attr('y1',d=>yScale(d)).attr('y2',d=>yScale(d))
      .attr('stroke','rgba(99,179,237,0.07)').attr('stroke-width',0.8);
    svg.append('g').selectAll('line').data(xScale.ticks(6)).join('line')
      .attr('x1',d=>xScale(d)).attr('x2',d=>xScale(d)).attr('y1',0).attr('y2',iH)
      .attr('stroke','rgba(99,179,237,0.07)').attr('stroke-width',0.8);

    // KOZ line
    const kozR = config.keepout_radius ?? 2;
    svg.append('line')
      .attr('x1',0).attr('x2',iW)
      .attr('y1',yScale(kozR)).attr('y2',yScale(kozR))
      .attr('stroke','rgba(239,68,68,0.55)').attr('stroke-width',1).attr('stroke-dasharray','4 3');
    svg.append('text').attr('x',iW-2).attr('y',yScale(kozR)-4)
      .attr('text-anchor','end').attr('font-size','8px')
      .attr('fill','rgba(239,68,68,0.7)').attr('font-family','var(--font-mono,monospace)')
      .text(`KOZ ${kozR}m`);

    // Goal line
    const goalR = Math.abs(config.goal_r_des?.[1] ?? 5);
    svg.append('line')
      .attr('x1',0).attr('x2',iW)
      .attr('y1',yScale(goalR)).attr('y2',yScale(goalR))
      .attr('stroke','rgba(16,185,129,0.45)').attr('stroke-width',1).attr('stroke-dasharray','4 3');
    svg.append('text').attr('x',iW-2).attr('y',yScale(goalR)-4)
      .attr('text-anchor','end').attr('font-size','8px')
      .attr('fill','rgba(16,185,129,0.7)').attr('font-family','var(--font-mono,monospace)')
      .text(`Goal ${goalR}m`);

    // Individual run lines (faint)
    const line = d3.line().x((_,i)=>xScale(env[i]?.t??0)).y(d=>yScale(d)).curve(d3.curveBasis);
    const bisect = d3.bisector(d=>d).left;
    trajs.forEach(tr => {
      const pts = env.map(e => {
        const idx = bisect(tr.t, e.t);
        if (idx <= 0) return tr.range[0];
        if (idx >= tr.t.length) return tr.range[tr.t.length-1];
        const alpha = (e.t - tr.t[idx-1]) / ((tr.t[idx] - tr.t[idx-1]) || 1);
        return tr.range[idx-1] + alpha*(tr.range[idx]-tr.range[idx-1]);
      });
      svg.append('path')
        .datum(pts)
        .attr('fill','none')
        .attr('stroke','rgba(6,182,212,0.18)')
        .attr('stroke-width',0.8)
        .attr('d', line);
    });

    // ±1σ envelope
    const area = d3.area()
      .x(d => xScale(d.t)).y0(d => yScale(d.lo)).y1(d => yScale(d.hi))
      .curve(d3.curveBasis);
    svg.append('path').datum(env).attr('fill','rgba(6,182,212,0.1)').attr('d', area);

    // Mean line
    const meanLine = d3.line().x(d=>xScale(d.t)).y(d=>yScale(d.mean)).curve(d3.curveBasis);
    svg.append('path').datum(env)
      .attr('fill','none').attr('stroke','#06b6d4').attr('stroke-width',2).attr('d',meanLine);

    // Axes
    const xA = d3.axisBottom(xScale).ticks(6);
    const yA = d3.axisLeft(yScale).ticks(5);
    const xG = svg.append('g').attr('transform',`translate(0,${iH})`).call(xA);
    const yG = svg.append('g').call(yA);
    [xG,yG].forEach(g=>{
      g.selectAll('path,line').attr('stroke','rgba(148,163,184,0.2)');
      g.selectAll('text').attr('fill','rgba(148,163,184,0.65)').attr('font-size','9px').attr('font-family','var(--font-mono,monospace)');
    });
    svg.append('text').attr('x',iW/2).attr('y',iH+34).attr('text-anchor','middle')
      .attr('fill','rgba(148,163,184,0.6)').attr('font-size','9px').attr('font-family','var(--font-mono,monospace)')
      .text('Time [s]');
    svg.append('text').attr('transform','rotate(-90)').attr('x',-iH/2).attr('y',-42)
      .attr('text-anchor','middle').attr('fill','rgba(148,163,184,0.6)').attr('font-size','9px')
      .attr('font-family','var(--font-mono,monospace)').text('Range [m]');
  }

  function drawBoxStrip(el, runsArr, label, color, fmt) {
    if (!el || !runsArr?.length) return;
    const W = el.clientWidth, H = el.clientHeight || 200;
    const iW = W - M2.left - M2.right;
    const iH = H - M2.top  - M2.bottom;
    d3.select(el).selectAll('*').remove();

    const runs = runsArr.filter(v => v != null);
    const bs   = boxStats(runs);
    const yScale = d3.scaleLinear()
      .domain([bs.min - (bs.max - bs.min) * 0.15, bs.max + (bs.max - bs.min) * 0.15])
      .range([iH, 0]);

    const svg = d3.select(el).append('g').attr('transform',`translate(${M2.left},${M2.top})`);
    const cx  = iW * 0.38;
    const bw  = iW * 0.22;

    // Grid
    svg.append('g').selectAll('line').data(yScale.ticks(5)).join('line')
      .attr('x1',0).attr('x2',iW).attr('y1',d=>yScale(d)).attr('y2',d=>yScale(d))
      .attr('stroke','rgba(99,179,237,0.07)').attr('stroke-width',0.8);

    // Whiskers
    svg.append('line').attr('x1',cx).attr('x2',cx)
      .attr('y1',yScale(bs.min)).attr('y2',yScale(bs.q1))
      .attr('stroke',color).attr('stroke-width',1.5).attr('opacity',0.5);
    svg.append('line').attr('x1',cx).attr('x2',cx)
      .attr('y1',yScale(bs.q3)).attr('y2',yScale(bs.max))
      .attr('stroke',color).attr('stroke-width',1.5).attr('opacity',0.5);

    // Box
    svg.append('rect')
      .attr('x',cx-bw/2).attr('y',yScale(bs.q3))
      .attr('width',bw).attr('height',yScale(bs.q1)-yScale(bs.q3))
      .attr('fill',color).attr('opacity',0.15).attr('stroke',color).attr('stroke-width',1.2);

    // Median line
    svg.append('line').attr('x1',cx-bw/2).attr('x2',cx+bw/2)
      .attr('y1',yScale(bs.med)).attr('y2',yScale(bs.med))
      .attr('stroke',color).attr('stroke-width',2);

    // Mean diamond
    const dm = 5;
    svg.append('polygon')
      .attr('points',`${cx},${yScale(bs.mean)-dm} ${cx+dm},${yScale(bs.mean)} ${cx},${yScale(bs.mean)+dm} ${cx-dm},${yScale(bs.mean)}`)
      .attr('fill',color).attr('opacity',0.9);

    // Individual dots (jittered)
    const seed42 = runs.map((_,i)=>((i*1664525+1013904223)>>>0)/0xffffffff);
    svg.selectAll('.strip-dot').data(runs).join('circle')
      .attr('cx', (_,i) => cx + bw * 0.7 + (seed42[i]-0.5) * bw * 0.45)
      .attr('cy', d => yScale(d))
      .attr('r', 3).attr('fill', color).attr('opacity', 0.65);

    // Stats annotation
    svg.append('text').attr('x',cx+bw/2+8).attr('y',yScale(bs.mean))
      .attr('dominant-baseline','middle').attr('font-size','8px')
      .attr('fill',color).attr('font-family','var(--font-mono,monospace)').attr('opacity',0.85)
      .text(`μ=${fmt(bs.mean)}`);
    svg.append('text').attr('x',cx+bw/2+8).attr('y',yScale(bs.mean)+11)
      .attr('dominant-baseline','middle').attr('font-size','7.5px')
      .attr('fill','rgba(148,163,184,0.6)').attr('font-family','var(--font-mono,monospace)')
      .text(`σ=${fmt(bs.std)}`);

    // Y axis
    const yA = d3.axisLeft(yScale).ticks(5).tickFormat(d=>d3.format('.3~g')(d));
    const yG = svg.append('g').call(yA);
    yG.selectAll('path,line').attr('stroke','rgba(148,163,184,0.2)');
    yG.selectAll('text').attr('fill','rgba(148,163,184,0.65)').attr('font-size','9px').attr('font-family','var(--font-mono,monospace)');

    svg.append('text').attr('transform','rotate(-90)').attr('x',-iH/2).attr('y',-55)
      .attr('text-anchor','middle').attr('fill','rgba(148,163,184,0.6)').attr('font-size','9px')
      .attr('font-family','var(--font-mono,monospace)').text(label);

    svg.append('text').attr('x',iW/2).attr('y',iH+34).attr('text-anchor','middle')
      .attr('fill','rgba(148,163,184,0.4)').attr('font-size','8px').attr('font-family','var(--font-mono,monospace)')
      .text(`n = ${runs.length} runs  ·  box = IQR  ·  ◆ = mean`);
  }

  function drawComputeBreakdown(el) {
    if (!el || !metrics?.compute_load) return;
    const W = el.clientWidth, H = el.clientHeight || 200;
    const iW = W - MARGIN.left - MARGIN.right;
    const iH = H - MARGIN.top  - MARGIN.bottom;
    d3.select(el).selectAll('*').remove();

    const components = [
      { label: 'Sensors',    val: metrics.mean_runtime_sensors?.mean ?? 0, color: '#3b82f6' },
      { label: 'Estimator',  val: metrics.mean_runtime_est?.mean      ?? 0, color: '#8b5cf6' },
      { label: 'Navigation', val: metrics.mean_runtime_nav?.mean      ?? 0, color: '#06b6d4' },
      { label: 'Controller', val: metrics.mean_runtime_ctl?.mean      ?? 0, color: '#10b981' },
    ];
    const total = d3.sum(components, d => d.val);

    const svg = d3.select(el).append('g').attr('transform',`translate(${MARGIN.left},${MARGIN.top})`);
    const xScale = d3.scaleLinear().domain([0, total * 1.05]).range([0, iW]);

    // Stacked bar
    let xCur = 0;
    components.forEach(c => {
      const bw = xScale(c.val);
      svg.append('rect').attr('x',xCur).attr('y',iH*0.3).attr('width',bw).attr('height',iH*0.32)
        .attr('fill',c.color).attr('opacity',0.85);
      if (bw > 22) {
        svg.append('text').attr('x',xCur+bw/2).attr('y',iH*0.3+iH*0.32/2)
          .attr('dominant-baseline','middle').attr('text-anchor','middle')
          .attr('font-size','8px').attr('fill','rgba(226,232,240,0.9)')
          .attr('font-family','var(--font-mono,monospace)').attr('pointer-events','none')
          .text(`${(c.val/total*100).toFixed(1)}%`);
      }
      xCur += bw;
    });

    // Labels below
    xCur = 0;
    components.forEach(c => {
      const bw = xScale(c.val);
      svg.append('text').attr('x',xCur+bw/2).attr('y',iH*0.3+iH*0.32+14)
        .attr('text-anchor','middle').attr('font-size','8px').attr('fill',c.color)
        .attr('font-family','var(--font-mono,monospace)')
        .text(bw > 15 ? c.label : '');
      xCur += bw;
    });

    // Scatter of per-run compute_load
    const runs = metrics.compute_load?.runs?.filter(v=>v!=null) ?? [];
    const dt = config.dt ?? 1;
    const yLoad = d3.scaleLinear().domain([
      d3.min(runs)*0.999, d3.max(runs)*1.001
    ]).range([iH, iH*0.75]);
    runs.forEach((v, i) => {
      const seed = ((i*1664525+1013904223)>>>0)/0xffffffff;
      svg.append('circle').attr('cx', xScale(v))
        .attr('cy', iH*0.75 + (seed-0.5)*iH*0.08)
        .attr('r',2.5).attr('fill','rgba(6,182,212,0.7)');
    });
    svg.append('text').attr('x',0).attr('y',iH*0.77+12)
      .attr('font-size','7.5px').attr('fill','rgba(148,163,184,0.45)')
      .attr('font-family','var(--font-mono,monospace)')
      .text(`per-run compute load (s/step)  —  dt=${dt}s`);

    // X axis
    const xA = d3.axisBottom(xScale).ticks(5).tickFormat(d=>d3.format('.4f')(d));
    const xG = svg.append('g').attr('transform',`translate(0,${iH})`).call(xA);
    xG.selectAll('path,line').attr('stroke','rgba(148,163,184,0.2)');
    xG.selectAll('text').attr('fill','rgba(148,163,184,0.65)').attr('font-size','8px')
      .attr('font-family','var(--font-mono,monospace)');
    svg.append('text').attr('x',iW/2).attr('y',iH+36).attr('text-anchor','middle')
      .attr('fill','rgba(148,163,184,0.6)').attr('font-size','9px')
      .attr('font-family','var(--font-mono,monospace)').text('Runtime per step [s]');

    // Legend
    components.forEach((c,i) => {
      svg.append('rect').attr('x',iW-90).attr('y',i*12).attr('width',8).attr('height',8).attr('fill',c.color).attr('opacity',0.85);
      svg.append('text').attr('x',iW-78).attr('y',i*12+7)
        .attr('font-size','7.5px').attr('fill','rgba(148,163,184,0.7)')
        .attr('font-family','var(--font-mono,monospace)').text(`${c.label}: ${(c.val*1000).toFixed(3)}ms`);
    });
  }

  function renderAll() {
    if (!trajs.length) return;
    drawRangeEnvelope(rangeEl);
    drawBoxStrip(durEl,    metrics?.mission_duration?.runs, 'Mission Duration [s]',  '#3b82f6', v=>d3.format('.1f')(v));
    drawBoxStrip(energyEl, metrics?.control_energy?.runs,   'Control Energy',        '#10b981', v=>d3.format('.4f')(v));
    drawComputeBreakdown(computeEl);
  }

  onMount(renderAll);
  afterUpdate(renderAll);

  let ro;
  onMount(() => {
    ro = new ResizeObserver(renderAll);
    [rangeEl, durEl, energyEl, computeEl].forEach(el => el && ro.observe(el.parentElement));
    return () => ro?.disconnect();
  });
</script>

<!-- FIG 6 — Range Envelope -->
<div class="se-grid se-grid--wide">
  <div class="figure-slot" style="--c:#06b6d4">
    <div class="figure-header">
      <span class="figure-num" style="color:#06b6d4">FIG. 6</span>
      <span class="figure-title">Range Envelope — {data?.n_runs ?? 0} Monte Carlo Runs</span>
    </div>
    <p class="figure-desc">
      Approach range [m] vs. time for all {data?.n_runs ?? 0} runs. Bold = mean trajectory, shaded band = ±1σ,
      faint lines = individual runs. Red dashed = KOZ ({data?.config?.keepout_radius ?? 2}m).
      Green dashed = goal ({Math.abs(data?.config?.goal_r_des?.[1] ?? 5)}m).
    </p>
    <div class="chart-area chart-area--tall">
      <svg bind:this={rangeEl} width="100%" height="100%"></svg>
    </div>
  </div>
</div>

<!-- FIG 7–9 — Mission Duration, Control Energy, Compute Load -->
<div class="se-grid se-grid--three">
  <div class="figure-slot" style="--c:#3b82f6">
    <div class="figure-header">
      <span class="figure-num" style="color:#3b82f6">FIG. 7</span>
      <span class="figure-title">Mission Duration</span>
    </div>
    <p class="figure-desc">
      Distribution of total mission duration across {data?.n_runs ?? 0} runs.
      Mean {data?.metrics?.mission_duration?.mean?.toFixed(0)}s ±{data?.metrics?.mission_duration?.std?.toFixed(1)}s.
    </p>
    <div class="chart-area chart-area--med">
      <svg bind:this={durEl} width="100%" height="100%"></svg>
    </div>
  </div>

  <div class="figure-slot" style="--c:#10b981">
    <div class="figure-header">
      <span class="figure-num" style="color:#10b981">FIG. 8</span>
      <span class="figure-title">Control Energy</span>
    </div>
    <p class="figure-desc">
      Control energy (∑||u||²·dt) across {data?.n_runs ?? 0} runs.
      Mean {data?.metrics?.control_energy?.mean?.toFixed(4)} ±{data?.metrics?.control_energy?.std?.toFixed(4)}.
    </p>
    <div class="chart-area chart-area--med">
      <svg bind:this={energyEl} width="100%" height="100%"></svg>
    </div>
  </div>

  <div class="figure-slot" style="--c:#f59e0b">
    <div class="figure-header">
      <span class="figure-num" style="color:#f59e0b">FIG. 9</span>
      <span class="figure-title">Compute Load Breakdown</span>
    </div>
    <p class="figure-desc">
      Mean runtime per GNC step by subsystem. Total ≈
      {((data?.metrics?.compute_load?.mean ?? 0) * 1000).toFixed(2)}ms /
      {data?.config?.dt ?? 1}s dt =
      {((data?.metrics?.compute_load?.mean ?? 0) / (data?.config?.dt ?? 1) * 100).toFixed(2)}% budget.
    </p>
    <div class="chart-area chart-area--med">
      <svg bind:this={computeEl} width="100%" height="100%"></svg>
    </div>
  </div>
</div>

<style>
  .se-grid { display: grid; gap: 1rem; }
  .se-grid--wide  { grid-template-columns: 1fr; }
  .se-grid--three { grid-template-columns: repeat(3, 1fr); }

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

  .figure-header { display: flex; align-items: baseline; gap: 0.65rem; }
  .figure-num {
    font-family: var(--font-mono); font-size: 0.68rem; font-weight: 700;
    letter-spacing: 0.06em; flex-shrink: 0;
  }
  .figure-title {
    font-family: var(--font-ui); font-size: 0.82rem; font-weight: 700;
    color: var(--ink); line-height: 1.3;
  }
  .figure-desc { font-size: 0.73rem; line-height: 1.5; color: var(--muted); margin: 0; }

  .chart-area {
    background: rgba(5,10,20,0.6);
    border: 1px solid rgba(99,179,237,0.08);
    overflow: hidden;
  }
  .chart-area--tall { height: 260px; }
  .chart-area--med  { height: 210px; }

  @media (max-width: 900px) {
    .se-grid--three { grid-template-columns: 1fr; }
  }
</style>
