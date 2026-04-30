<script>
  import * as d3 from 'd3';
  import { onMount, afterUpdate } from 'svelte';

  export let data;

  $: traj    = data?.trajectory ?? {};
  $: metrics = data?.metrics    ?? {};
  $: config  = data?.config     ?? {};
  $: T       = traj.t           ?? [];

  // ── SVG refs ──────────────────────────────────────────────────────────────
  let trajEl, ctrlEl, covEl;

  const M = { top: 22, right: 22, bottom: 46, left: 58 };

  // ── Helpers ───────────────────────────────────────────────────────────────
  function clearSVG(el) { d3.select(el).selectAll('*').remove(); }

  function baseAxes(svg, xScale, yScale, iW, iH, xLabel, yLabel) {
    // Grid
    svg.append('g').selectAll('line').data(yScale.ticks(5)).join('line')
      .attr('x1',0).attr('x2',iW).attr('y1',d=>yScale(d)).attr('y2',d=>yScale(d))
      .attr('stroke','rgba(99,179,237,0.07)').attr('stroke-width',0.7);
    svg.append('g').selectAll('line').data(xScale.ticks(6)).join('line')
      .attr('x1',d=>xScale(d)).attr('x2',d=>xScale(d)).attr('y1',0).attr('y2',iH)
      .attr('stroke','rgba(99,179,237,0.07)').attr('stroke-width',0.7);
    // Axes
    const xA = d3.axisBottom(xScale).ticks(6).tickFormat(d=>d3.format('.3~g')(d));
    const yA = d3.axisLeft(yScale).ticks(5).tickFormat(d=>d3.format('.3~g')(d));
    const xG = svg.append('g').attr('transform',`translate(0,${iH})`).call(xA);
    const yG = svg.append('g').call(yA);
    [xG, yG].forEach(g => {
      g.selectAll('path,line').attr('stroke','rgba(148,163,184,0.2)');
      g.selectAll('text').attr('fill','rgba(148,163,184,0.65)').attr('font-size','9px').attr('font-family','var(--font-mono,monospace)');
    });
    svg.append('text').attr('x',iW/2).attr('y',iH+36).attr('text-anchor','middle')
      .attr('fill','rgba(148,163,184,0.6)').attr('font-size','9px').attr('font-family','var(--font-mono,monospace)')
      .text(xLabel);
    svg.append('text').attr('transform','rotate(-90)').attr('x',-iH/2).attr('y',-44)
      .attr('text-anchor','middle').attr('fill','rgba(148,163,184,0.6)').attr('font-size','9px')
      .attr('font-family','var(--font-mono,monospace)').text(yLabel);
  }

  // ── FIG 10: Trajectory ───────────────────────────────────────────────────
  function drawTrajectory(el) {
    if (!el || !T.length) return;
    const W = el.clientWidth, H = el.clientHeight || 260;
    const iW = W - M.left - M.right;
    const iH = H - M.top  - M.bottom;
    clearSVG(el);

    const { x, y, z } = traj;
    const goalY  = config.goal_r_des?.[1] ?? -5;
    const kozR   = config.keepout_radius  ?? 2;
    const tColor = d3.scaleSequential(d3.interpolatePlasma).domain([T[0], T[T.length-1]]);

    // Split into two panels: XY approach (left) + range vs t (right)
    const lW = iW * 0.46, rW = iW * 0.46, gap = iW * 0.08;

    // ── Left: XY trajectory ─────────────────────────────────────────────────
    const xExt = d3.extent(x), yExt = d3.extent(y);
    const xPad = Math.max(Math.abs(xExt[1]-xExt[0])*0.25, 0.5);
    const yPad = Math.abs(yExt[1]-yExt[0])*0.08;
    const xScL = d3.scaleLinear().domain([xExt[0]-xPad, xExt[1]+xPad]).range([0, lW]);
    const yScL = d3.scaleLinear().domain([yExt[0]-yPad, yExt[1]+yPad]).range([iH, 0]);

    const svgL = d3.select(el).append('g').attr('transform',`translate(${M.left},${M.top})`);

    // KOZ circle
    const kozCx = xScL(0), kozCy = yScL(goalY);
    svgL.append('circle').attr('cx',kozCx).attr('cy',kozCy)
      .attr('r', Math.abs(xScL(kozR)-xScL(0)))
      .attr('fill','rgba(239,68,68,0.08)').attr('stroke','rgba(239,68,68,0.4)')
      .attr('stroke-width',1).attr('stroke-dasharray','3 3');

    // Goal point
    svgL.append('circle').attr('cx',xScL(0)).attr('cy',yScL(goalY))
      .attr('r',5).attr('fill','rgba(16,185,129,0.7)').attr('stroke','#10b981').attr('stroke-width',1.5);
    svgL.append('text').attr('x',xScL(0)+8).attr('y',yScL(goalY)-6)
      .attr('font-size','8px').attr('fill','rgba(16,185,129,0.8)').attr('font-family','var(--font-mono,monospace)')
      .text('GOAL');

    // Time-colored path (segments)
    for (let i = 1; i < T.length; i++) {
      svgL.append('line')
        .attr('x1',xScL(x[i-1])).attr('y1',yScL(y[i-1]))
        .attr('x2',xScL(x[i])).attr('y2',yScL(y[i]))
        .attr('stroke',tColor(T[i])).attr('stroke-width',1.8).attr('opacity',0.85);
    }

    // Start marker
    svgL.append('circle').attr('cx',xScL(x[0])).attr('cy',yScL(y[0]))
      .attr('r',5).attr('fill','none').attr('stroke','rgba(148,163,184,0.7)').attr('stroke-width',1.5);
    svgL.append('text').attr('x',xScL(x[0])+8).attr('y',yScL(y[0])-6)
      .attr('font-size','8px').attr('fill','rgba(148,163,184,0.7)').attr('font-family','var(--font-mono,monospace)')
      .text('START');

    baseAxes(svgL, xScL, yScL, lW, iH, 'Cross-track X [m]', 'Along-track Y [m]');

    // Colorbar (time)
    const cbH = iH * 0.55, cbX = lW + 4, cbY = iH * 0.22;
    const cbScale = d3.scaleLinear().domain([0,1]).range([cbY+cbH, cbY]);
    const grad = d3.select(el).append('defs').append('linearGradient')
      .attr('id','traj-grad').attr('x1','0%').attr('y1','100%').attr('x2','0%').attr('y2','0%');
    d3.schemeTableau10; // just to load d3-color
    [[0,'#0d0221'],[0.33,'#7c3fbc'],[0.66,'#f08e3b'],[1,'#f0f921']].forEach(([o,c])=>
      grad.append('stop').attr('offset',`${o*100}%`).attr('stop-color',c));
    d3.select(el).append('rect')
      .attr('transform',`translate(${M.left},${M.top})`)
      .attr('x',cbX).attr('y',cbY).attr('width',6).attr('height',cbH)
      .attr('fill','url(#traj-grad)').attr('opacity',0.7);
    const cbAxis = d3.axisRight(cbScale.copy().range([cbY,cbY+cbH]).domain([T[0],T[T.length-1]])).ticks(4).tickFormat(d=>`${d}s`);
    d3.select(el).append('g')
      .attr('transform',`translate(${M.left+cbX+6},${M.top})`).call(cbAxis)
      .selectAll('path,line').attr('stroke','rgba(148,163,184,0.2)');
    d3.select(el).selectAll('.traj-cb-text').data([0]).join('text').attr('class','traj-cb-text')
      .attr('transform',`translate(${M.left+cbX+22},${M.top + cbY + cbH/2}) rotate(90)`)
      .attr('text-anchor','middle').attr('font-size','7.5px').attr('fill','rgba(148,163,184,0.5)')
      .attr('font-family','var(--font-mono,monospace)').text('time');
    d3.select(el).select('.traj-cb-text ~ g').selectAll('text')
      .attr('fill','rgba(148,163,184,0.55)').attr('font-size','8px').attr('font-family','var(--font-mono,monospace)');

    // ── Right: range vs time ────────────────────────────────────────────────
    const rOffset = M.left + lW + gap;
    const tSc = d3.scaleLinear().domain([T[0],T[T.length-1]]).range([0,rW]);
    const rSc = d3.scaleLinear().domain([0, d3.max(traj.range)*1.05]).range([iH,0]);

    const svgR = d3.select(el).append('g').attr('transform',`translate(${rOffset},${M.top})`);

    // KOZ line
    svgR.append('line').attr('x1',0).attr('x2',rW).attr('y1',rSc(kozR)).attr('y2',rSc(kozR))
      .attr('stroke','rgba(239,68,68,0.5)').attr('stroke-width',1).attr('stroke-dasharray','4 3');
    svgR.append('text').attr('x',rW-2).attr('y',rSc(kozR)-4).attr('text-anchor','end')
      .attr('font-size','8px').attr('fill','rgba(239,68,68,0.7)').attr('font-family','var(--font-mono,monospace)')
      .text(`KOZ ${kozR}m`);

    // Range line (time-colored)
    for (let i = 1; i < T.length; i++) {
      svgR.append('line')
        .attr('x1',tSc(T[i-1])).attr('y1',rSc(traj.range[i-1]))
        .attr('x2',tSc(T[i])).attr('y2',rSc(traj.range[i]))
        .attr('stroke',tColor(T[i])).attr('stroke-width',1.8).attr('opacity',0.9);
    }

    baseAxes(svgR, tSc, rSc, rW, iH, 'Time [s]', 'Range [m]');
  }

  // ── FIG 11: Control History ───────────────────────────────────────────────
  function drawControl(el) {
    if (!el || !T.length) return;
    const W = el.clientWidth, H = el.clientHeight || 240;
    const iW = W - M.left - M.right;
    const iH = H - M.top  - M.bottom;
    clearSVG(el);

    const series = [
      { key: 'ux', label: 'u_x', color: '#3b82f6' },
      { key: 'uy', label: 'u_y', color: '#06b6d4' },
      { key: 'uz', label: 'u_z', color: '#8b5cf6' },
    ];

    const allVals = series.flatMap(s => traj[s.key] ?? []);
    const vExt = d3.extent(allVals);
    const vPad = Math.max(Math.abs(vExt[1]-vExt[0])*0.12, 0.001);

    const tSc = d3.scaleLinear().domain([T[0],T[T.length-1]]).range([0,iW]);
    const vSc = d3.scaleLinear().domain([vExt[0]-vPad, vExt[1]+vPad]).range([iH,0]);

    const svg = d3.select(el).append('g').attr('transform',`translate(${M.left},${M.top})`);

    // Zero line
    svg.append('line').attr('x1',0).attr('x2',iW).attr('y1',vSc(0)).attr('y2',vSc(0))
      .attr('stroke','rgba(148,163,184,0.18)').attr('stroke-width',1);

    // Thrust total as filled area behind
    const thrustSc = d3.scaleLinear().domain([0, d3.max(traj.thrust??[0])*1.1]).range([iH,0]);
    const thrustArea = d3.area().x((_,i)=>tSc(T[i])).y0(iH).y1((_,i)=>thrustSc(traj.thrust[i]));
    svg.append('path').datum(traj.thrust).attr('d',thrustArea)
      .attr('fill','rgba(245,158,11,0.08)').attr('stroke','rgba(245,158,11,0.3)').attr('stroke-width',1);

    // Component lines
    const lineGen = d3.line().x((_,i)=>tSc(T[i])).y(d=>vSc(d)).curve(d3.curveMonotoneX);
    series.forEach(s => {
      if (!traj[s.key]?.length) return;
      svg.append('path').datum(traj[s.key])
        .attr('fill','none').attr('stroke',s.color).attr('stroke-width',1.5).attr('opacity',0.85)
        .attr('d',lineGen);
    });

    baseAxes(svg, tSc, vSc, iW, iH, 'Time [s]', 'Control Input [N]');

    // Legend
    const legItems = [...series, { key:'thrust', label:'|u| total', color:'#f59e0b' }];
    legItems.forEach((s,i) => {
      svg.append('line').attr('x1',iW-110+i*0).attr('y1',-10+i*11).attr('x2',iW-98+i*0).attr('y2',-10+i*11)
        .attr('stroke',s.color).attr('stroke-width',2).attr('opacity',0.85);
      svg.append('text').attr('x',iW-94+i*0).attr('y',-6+i*11)
        .attr('font-size','8px').attr('fill',s.color).attr('font-family','var(--font-mono,monospace)')
        .attr('opacity',0.8).text(s.label);
    });
  }

  // ── FIG 12: Covariance / Estimation Error ─────────────────────────────────
  function drawCovariance(el) {
    if (!el || !T.length) return;
    const W = el.clientWidth, H = el.clientHeight || 240;
    const iW = W - M.left - M.right;
    const iH = H - M.top  - M.bottom;
    clearSVG(el);

    const { pos_sigma, est_pos_err, est_vel_err } = traj;
    if (!pos_sigma?.length) return;

    const allVals = [...(pos_sigma??[]), ...(est_pos_err??[])].filter(v => v != null);
    const yMax = d3.max(allVals) * 1.1;

    const tSc = d3.scaleLinear().domain([T[0],T[T.length-1]]).range([0,iW]);
    const ySc = d3.scaleLinear().domain([0, yMax]).range([iH,0]);

    const svg = d3.select(el).append('g').attr('transform',`translate(${M.left},${M.top})`);

    // 1σ position covariance — filled band (0 to pos_sigma)
    const sigmaArea = d3.area().x((_,i)=>tSc(T[i])).y0(iH).y1((_,i)=>ySc(pos_sigma[i])).curve(d3.curveBasis);
    svg.append('path').datum(pos_sigma)
      .attr('d',sigmaArea).attr('fill','rgba(59,130,246,0.12)').attr('stroke','none');

    // 3σ band
    const sigma3Area = d3.area().x((_,i)=>tSc(T[i])).y0(iH).y1((_,i)=>ySc(pos_sigma[i]*3)).curve(d3.curveBasis);
    svg.append('path').datum(pos_sigma)
      .attr('d',sigma3Area).attr('fill','rgba(59,130,246,0.05)').attr('stroke','none');

    const lineGen = d3.line().x((_,i)=>tSc(T[i])).y(d=>ySc(d)).curve(d3.curveBasis);

    // 1σ line
    svg.append('path').datum(pos_sigma)
      .attr('fill','none').attr('stroke','#3b82f6').attr('stroke-width',1.8).attr('d',lineGen);

    // EKF pos error
    if (est_pos_err?.length) {
      svg.append('path').datum(est_pos_err)
        .attr('fill','none').attr('stroke','#06b6d4').attr('stroke-width',1.5)
        .attr('stroke-dasharray','5 3').attr('d',lineGen);
    }

    // Vel error on secondary scale (right axis)
    if (est_vel_err?.length) {
      const vMax = d3.max(est_vel_err) * 1.1;
      const vSc = d3.scaleLinear().domain([0,vMax]).range([iH,0]);
      const velLine = d3.line().x((_,i)=>tSc(T[i])).y((_,i)=>vSc(est_vel_err[i])).curve(d3.curveBasis);
      svg.append('path').datum(est_vel_err)
        .attr('fill','none').attr('stroke','#8b5cf6').attr('stroke-width',1.2).attr('opacity',0.7).attr('d',velLine);
      // Right axis
      const rA = d3.axisRight(vSc).ticks(4).tickFormat(d=>d3.format('.3f')(d));
      const rG = svg.append('g').attr('transform',`translate(${iW},0)`).call(rA);
      rG.selectAll('path,line').attr('stroke','rgba(139,92,246,0.2)');
      rG.selectAll('text').attr('fill','rgba(139,92,246,0.55)').attr('font-size','8px').attr('font-family','var(--font-mono,monospace)');
      svg.append('text').attr('transform',`rotate(90)`).attr('x',iH/2).attr('y',-(iW+42))
        .attr('text-anchor','middle').attr('fill','rgba(139,92,246,0.5)').attr('font-size','8px')
        .attr('font-family','var(--font-mono,monospace)').text('Vel error [m/s]');
    }

    baseAxes(svg, tSc, ySc, iW, iH, 'Time [s]', 'Position σ / error [m]');

    // Legend
    const legData = [
      { label: '1σ pos cov', color: '#3b82f6', dash: null },
      { label: 'pos error',  color: '#06b6d4', dash: '5 3' },
      { label: 'vel error',  color: '#8b5cf6', dash: '3 2' },
    ];
    legData.forEach((l,i) => {
      const lx = 6, ly = 4 + i*12;
      if (l.dash)
        svg.append('line').attr('x1',lx).attr('y1',ly).attr('x2',lx+14).attr('y2',ly)
          .attr('stroke',l.color).attr('stroke-width',1.8).attr('stroke-dasharray',l.dash);
      else {
        svg.append('rect').attr('x',lx).attr('y',ly-4).attr('width',14).attr('height',5)
          .attr('fill',l.color).attr('opacity',0.3);
        svg.append('line').attr('x1',lx).attr('y1',ly).attr('x2',lx+14).attr('y2',ly)
          .attr('stroke',l.color).attr('stroke-width',1.8);
      }
      svg.append('text').attr('x',lx+18).attr('y',ly+4)
        .attr('font-size','8px').attr('fill',l.color).attr('font-family','var(--font-mono,monospace)').attr('opacity',0.85)
        .text(l.label);
    });
  }

  function renderAll() {
    if (!T.length) return;
    drawTrajectory(trajEl);
    drawControl(ctrlEl);
    drawCovariance(covEl);
  }

  onMount(renderAll);
  afterUpdate(renderAll);

  let ro;
  onMount(() => {
    ro = new ResizeObserver(renderAll);
    [trajEl, ctrlEl, covEl].forEach(el => el && ro.observe(el.parentElement));
    return () => ro?.disconnect();
  });

  // ── FIG 13 — Summary table ────────────────────────────────────────────────
  const TABLE_ROWS = [
    { label: 'Goal Reached',            key: 'goal_reached_flag', unit: '',     fmt: v => v ? '✓ YES' : '✗ NO', good: v => v === true,  val: () => data?.goal_reached },
    { label: 'Mission Duration',        key: 'mission_duration',  unit: 's',    fmt: v => v?.toFixed(0),          good: v => v < 500       },
    { label: 'Settling Time',           key: 'settling_time',     unit: 's',    fmt: v => v?.toFixed(0),          good: v => v < 500       },
    { label: 'Steady-State Error',      key: 'steady_state_error',unit: 'm',    fmt: v => v?.toFixed(4),          good: v => v < 0.3       },
    { label: 'Total Impulse',           key: 'total_impulse',     unit: 'm/s',  fmt: v => v?.toFixed(4),          good: v => v != null     },
    { label: 'Control Energy',          key: 'control_energy',    unit: '',     fmt: v => v?.toFixed(5),          good: v => v != null     },
    { label: 'Peak Thrust',             key: 'peak_thrust',       unit: 'N',    fmt: v => v?.toFixed(5),          good: v => v != null     },
    { label: 'Min Distance to Target',  key: 'min_distance',      unit: 'm',    fmt: v => v?.toFixed(4),          good: v => v > (config?.keepout_radius ?? 2) },
    { label: 'KOZ Violation',           key: 'koz_violation',     unit: '',     fmt: v => v > 0 ? `⚠ ${v}` : '✓ None', good: v => v === 0 },
    { label: 'Violation Depth',         key: 'violation_depth',   unit: 'm',    fmt: v => v?.toFixed(4),          good: v => v === 0       },
    { label: 'Mahalanobis (CPA)',        key: 'mahalanobis_cpa',   unit: '',     fmt: v => v?.toFixed(2),          good: v => v > 10        },
    { label: '3σ Miss Distance LB',     key: 'miss_distance_3sigma_lb', unit:'m', fmt: v => v?.toFixed(4),        good: v => v > (config?.keepout_radius ?? 2) },
    { label: 'Time to Closest Approach',key: 'time_to_closest_approach', unit:'s', fmt: v => v?.toFixed(0),      good: v => v != null     },
    { label: 'Radial σ (CPA)',          key: 'radial_sigma_cpa',  unit: 'm',    fmt: v => v?.toFixed(5),          good: v => v < 0.05      },
  ];

  $: tableRows = TABLE_ROWS.map(r => ({
    ...r,
    value: r.val ? r.val() : metrics[r.key],
  }));
</script>

<!-- FIG 10 — Trajectory -->
<div class="cpo-section">
  <div class="figure-slot" style="--c:#3b82f6">
    <div class="figure-header">
      <span class="figure-num" style="color:#3b82f6">FIG. 10</span>
      <span class="figure-title">
        Approach Trajectory — XY Frame &amp; Range vs. Time
        {#if data?.goal_reached}<span class="badge-success">GOAL REACHED</span>{/if}
      </span>
    </div>
    <p class="figure-desc">
      Left: servicer path in relative Hill frame (cross-track vs. along-track), colored by time.
      Right: range vs. time with KOZ and goal markers.
      Config: {config.estimator?.toUpperCase()} / {config.navigator} · dt={config.dt}s
    </p>
    <div class="chart-area chart-area--tall">
      <svg bind:this={trajEl} width="100%" height="100%"></svg>
    </div>
  </div>
</div>

<!-- FIG 11 & 12 — Control + Covariance side by side -->
<div class="cpo-grid cpo-grid--two">
  <div class="figure-slot" style="--c:#06b6d4">
    <div class="figure-header">
      <span class="figure-num" style="color:#06b6d4">FIG. 11</span>
      <span class="figure-title">Control History — Thrust Components</span>
    </div>
    <p class="figure-desc">
      u_x (radial), u_y (along-track), u_z (cross-track) control inputs vs. time.
      Amber fill = total thrust magnitude |u|.
      Peak thrust: {metrics.peak_thrust?.toFixed(5)} N · Total ΔV: {metrics.total_impulse?.toFixed(4)} m/s
    </p>
    <div class="chart-area chart-area--med">
      <svg bind:this={ctrlEl} width="100%" height="100%"></svg>
    </div>
  </div>

  <div class="figure-slot" style="--c:#8b5cf6">
    <div class="figure-header">
      <span class="figure-num" style="color:#8b5cf6">FIG. 12</span>
      <span class="figure-title">EKF Covariance &amp; Estimation Error</span>
    </div>
    <p class="figure-desc">
      Blue = 1σ position covariance (EKF); cyan dashed = actual position estimation error;
      purple = velocity estimation error (right axis).
      Final σ: {traj.pos_sigma?.[traj.pos_sigma.length-1]?.toFixed(4) ?? '—'} m
    </p>
    <div class="chart-area chart-area--med">
      <svg bind:this={covEl} width="100%" height="100%"></svg>
    </div>
  </div>
</div>

<!-- FIG 13 — Summary Table -->
<div class="cpo-section">
  <div class="figure-slot" style="--c:#f59e0b">
    <div class="figure-header">
      <span class="figure-num" style="color:#f59e0b">FIG. 13</span>
      <span class="figure-title">Run Summary — Key Performance Indicators</span>
    </div>
    <p class="figure-desc">
      Generated {data?.generated_at ? new Date(data.generated_at).toLocaleString() : '—'} ·
      Single deterministic run · {config.estimator?.toUpperCase()} estimator ·
      Sensors: {config.sensor_types?.join(', ')}
    </p>
    <div class="table-wrap">
      <table class="summary-table">
        <thead>
          <tr>
            <th>Metric</th>
            <th>Value</th>
            <th>Unit</th>
            <th>Status</th>
          </tr>
        </thead>
        <tbody>
          {#each tableRows as row}
            {@const val = row.value}
            {@const isGood = val != null && row.good(val)}
            <tr>
              <td class="metric-label">{row.label}</td>
              <td class="metric-value">{val != null ? row.fmt(val) : '—'}</td>
              <td class="metric-unit">{row.unit}</td>
              <td class="metric-status">
                <span class="status-dot" class:good={isGood} class:bad={!isGood && val != null}></span>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  </div>
</div>

<style>
  .cpo-section { display: grid; grid-template-columns: 1fr; gap: 1rem; }
  .cpo-grid { display: grid; gap: 1rem; }
  .cpo-grid--two { grid-template-columns: repeat(2, 1fr); }

  .figure-slot {
    background: var(--bg-card); border: 1px solid var(--border);
    border-top: 2px solid var(--c); padding: 1rem;
    display: flex; flex-direction: column; gap: 0.5rem;
    animation: fadeInUp 500ms ease-out both;
  }
  @keyframes fadeInUp { from { opacity:0; transform:translateY(12px); } to { opacity:1; transform:translateY(0); } }

  .figure-header { display: flex; align-items: baseline; gap: 0.65rem; flex-wrap: wrap; }
  .figure-num { font-family: var(--font-mono); font-size: 0.68rem; font-weight: 700; letter-spacing: 0.06em; flex-shrink: 0; }
  .figure-title { font-family: var(--font-ui); font-size: 0.82rem; font-weight: 700; color: var(--ink); line-height: 1.3; }
  .figure-desc { font-size: 0.73rem; line-height: 1.5; color: var(--muted); margin: 0; }

  .badge-success {
    font-family: var(--font-mono); font-size: 0.6rem; font-weight: 700;
    letter-spacing: 0.1em; padding: 0.15rem 0.4rem;
    background: rgba(16,185,129,0.15); color: #10b981;
    border: 1px solid rgba(16,185,129,0.3); border-radius: 2px;
  }

  .chart-area {
    background: rgba(5,10,20,0.6); border: 1px solid rgba(99,179,237,0.08); overflow: hidden;
  }
  .chart-area--tall { height: 280px; }
  .chart-area--med  { height: 230px; }

  /* ── Summary table ───────────────────────────────────────────────────── */
  .table-wrap { overflow-x: auto; }

  .summary-table {
    width: 100%; border-collapse: collapse;
    font-size: 0.78rem; font-family: var(--font-mono);
  }

  .summary-table thead tr {
    border-bottom: 1px solid var(--border-bright);
  }

  .summary-table th {
    text-align: left; padding: 0.45rem 0.75rem;
    font-size: 0.62rem; font-weight: 700; letter-spacing: 0.1em;
    color: var(--muted); text-transform: uppercase;
  }

  .summary-table tbody tr {
    border-bottom: 1px solid rgba(99,179,237,0.06);
    transition: background 150ms ease;
  }
  .summary-table tbody tr:hover { background: rgba(99,179,237,0.04); }

  .summary-table td { padding: 0.42rem 0.75rem; }

  .metric-label { color: rgba(226,232,240,0.75); font-size: 0.77rem; }
  .metric-value { color: var(--ink); font-weight: 700; font-variant-numeric: tabular-nums; white-space: nowrap; }
  .metric-unit  { color: var(--muted); font-size: 0.7rem; }
  .metric-status { text-align: center; }

  .status-dot {
    display: inline-block; width: 7px; height: 7px; border-radius: 50%;
    background: rgba(148,163,184,0.3);
  }
  .status-dot.good { background: #10b981; box-shadow: 0 0 5px rgba(16,185,129,0.5); }
  .status-dot.bad  { background: #ef4444; box-shadow: 0 0 5px rgba(239,68,68,0.5); }

  @media (max-width: 900px) { .cpo-grid--two { grid-template-columns: 1fr; } }
</style>
