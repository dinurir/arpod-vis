<script>
  import * as d3 from 'd3';
  import { onMount, afterUpdate } from 'svelte';

  export let pair;    // { id, title, x_field, y_field, x_label, y_label, x_lower_better, y_lower_better }
  export let stacks;  // full stacks array

  // Color by estimator type parsed from label
  const ESTIMATOR_COLOR = {
    'EKF': '#3b82f6',
    'UKF': '#06b6d4',
    'KF':  '#8b5cf6',
  };

  function estimatorOf(label) {
    if (label.startsWith('UKF')) return 'UKF';
    if (label.startsWith('EKF')) return 'EKF';
    return 'KF';
  }

  // Build point list: one point per stack (mean), plus individual runs
  function buildPoints(stacks, pair) {
    const pts = [];
    for (const s of stacks) {
      const mx = s.metrics[pair.x_field];
      const my = s.metrics[pair.y_field];
      if (!mx || !my || mx.mean == null || my.mean == null) continue;
      pts.push({
        key:   s.key,
        label: s.label,
        estimator: estimatorOf(s.label),
        x:     mx.mean,
        y:     my.mean,
        x_std: mx.std ?? 0,
        y_std: my.std ?? 0,
        runs:  (mx.runs ?? []).map((rx, i) => ({ rx, ry: (my.runs ?? [])[i] }))
                               .filter(r => r.rx != null && r.ry != null),
      });
    }
    return pts;
  }

  // Non-dominated (Pareto) front — both axes lower-is-better
  function paretoFront(pts) {
    const sorted = [...pts].sort((a, b) => a.x - b.x);
    const front = [];
    let minY = Infinity;
    for (const p of sorted) {
      if (p.y < minY) {
        front.push(p);
        minY = p.y;
      }
    }
    return front;
  }

  let svgEl;
  const MARGIN = { top: 24, right: 16, bottom: 52, left: 58 };

  function render() {
    if (!svgEl || !pair || !stacks) return;

    const W = svgEl.clientWidth  || 400;
    const H = svgEl.clientHeight || 280;
    const iW = W - MARGIN.left - MARGIN.right;
    const iH = H - MARGIN.top  - MARGIN.bottom;

    d3.select(svgEl).selectAll('*').remove();

    const pts = buildPoints(stacks, pair);
    if (!pts.length) {
      d3.select(svgEl).append('text')
        .attr('x', W / 2).attr('y', H / 2)
        .attr('text-anchor', 'middle')
        .attr('fill', 'rgba(148,163,184,0.5)')
        .attr('font-size', '11px')
        .text('No data for this metric');
      return;
    }

    const xExt = d3.extent(pts, d => d.x);
    const yExt = d3.extent(pts, d => d.y);
    const xPad = (xExt[1] - xExt[0]) * 0.12 || 1;
    const yPad = (yExt[1] - yExt[0]) * 0.15 || 1;

    const xScale = d3.scaleLinear()
      .domain([xExt[0] - xPad, xExt[1] + xPad])
      .range([0, iW]);
    const yScale = d3.scaleLinear()
      .domain([yExt[0] - yPad, yExt[1] + yPad])
      .range([iH, 0]);

    const svg = d3.select(svgEl).append('g')
      .attr('transform', `translate(${MARGIN.left},${MARGIN.top})`);

    // Grid
    svg.append('g').attr('class', 'grid')
      .selectAll('line.gx')
      .data(xScale.ticks(5))
      .join('line')
        .attr('x1', d => xScale(d)).attr('x2', d => xScale(d))
        .attr('y1', 0).attr('y2', iH)
        .attr('stroke', 'rgba(99,179,237,0.07)').attr('stroke-width', 1);

    svg.append('g').attr('class', 'grid')
      .selectAll('line.gy')
      .data(yScale.ticks(5))
      .join('line')
        .attr('x1', 0).attr('x2', iW)
        .attr('y1', d => yScale(d)).attr('y2', d => yScale(d))
        .attr('stroke', 'rgba(99,179,237,0.07)').attr('stroke-width', 1);

    // Axes
    const xAxis = d3.axisBottom(xScale).ticks(5)
      .tickFormat(d => d3.format('.3~g')(d));
    const yAxis = d3.axisLeft(yScale).ticks(5)
      .tickFormat(d => d3.format('.3~g')(d));

    const xG = svg.append('g').attr('transform', `translate(0,${iH})`).call(xAxis);
    const yG = svg.append('g').call(yAxis);

    [xG, yG].forEach(g => {
      g.selectAll('path, line').attr('stroke', 'rgba(148,163,184,0.25)');
      g.selectAll('text')
        .attr('fill', 'rgba(148,163,184,0.7)')
        .attr('font-size', '9px')
        .attr('font-family', 'var(--font-mono, monospace)');
    });

    // Axis labels
    svg.append('text')
      .attr('x', iW / 2).attr('y', iH + 40)
      .attr('text-anchor', 'middle')
      .attr('fill', 'rgba(148,163,184,0.65)')
      .attr('font-size', '9px')
      .attr('font-family', 'var(--font-mono, monospace)')
      .text(pair.x_label);

    svg.append('text')
      .attr('transform', `rotate(-90)`)
      .attr('x', -iH / 2).attr('y', -44)
      .attr('text-anchor', 'middle')
      .attr('fill', 'rgba(148,163,184,0.65)')
      .attr('font-size', '9px')
      .attr('font-family', 'var(--font-mono, monospace)')
      .text(pair.y_label);

    // Individual run dots (faint)
    for (const p of pts) {
      const color = ESTIMATOR_COLOR[p.estimator] ?? '#94a3b8';
      svg.selectAll(null)
        .data(p.runs)
        .join('circle')
          .attr('cx', r => xScale(r.rx))
          .attr('cy', r => yScale(r.ry))
          .attr('r', 2)
          .attr('fill', color)
          .attr('opacity', 0.18);
    }

    // Std ellipses
    for (const p of pts) {
      const color = ESTIMATOR_COLOR[p.estimator] ?? '#94a3b8';
      const rx = Math.abs(xScale(p.x + p.x_std) - xScale(p.x));
      const ry = Math.abs(yScale(p.y + p.y_std) - yScale(p.y));
      if (rx > 0.5 && ry > 0.5) {
        svg.append('ellipse')
          .attr('cx', xScale(p.x)).attr('cy', yScale(p.y))
          .attr('rx', rx).attr('ry', ry)
          .attr('fill', color).attr('opacity', 0.07)
          .attr('stroke', color).attr('stroke-width', 0.7).attr('stroke-opacity', 0.25);
      }
    }

    // Pareto frontier
    const front = paretoFront(pts);
    if (front.length > 1) {
      // Step-line
      const lineData = [];
      front.forEach((p, i) => {
        lineData.push([xScale(p.x), yScale(p.y)]);
        if (i < front.length - 1) {
          lineData.push([xScale(front[i + 1].x), yScale(p.y)]);
        }
      });
      svg.append('polyline')
        .attr('points', lineData.map(d => d.join(',')).join(' '))
        .attr('fill', 'none')
        .attr('stroke', 'rgba(16,185,129,0.55)')
        .attr('stroke-width', 1.5)
        .attr('stroke-dasharray', '4 3');

      // Pareto frontier dots highlighted
      svg.selectAll('.pareto-dot')
        .data(front)
        .join('circle')
          .attr('class', 'pareto-dot')
          .attr('cx', d => xScale(d.x))
          .attr('cy', d => yScale(d.y))
          .attr('r', 4)
          .attr('fill', 'rgba(16,185,129,0.7)')
          .attr('stroke', 'rgba(16,185,129,0.9)')
          .attr('stroke-width', 1);
    }

    // Mean points (all stacks, colored by estimator)
    const tooltip = d3.select(svgEl.parentElement).select('.pareto-tooltip');

    svg.selectAll('.mean-dot')
      .data(pts)
      .join('circle')
        .attr('class', 'mean-dot')
        .attr('cx', d => xScale(d.x))
        .attr('cy', d => yScale(d.y))
        .attr('r', 4.5)
        .attr('fill', d => ESTIMATOR_COLOR[d.estimator] ?? '#94a3b8')
        .attr('opacity', 0.85)
        .attr('stroke', '#050a14')
        .attr('stroke-width', 0.8)
        .attr('cursor', 'pointer')
        .on('mouseenter', function(event, d) {
          d3.select(this).attr('r', 6.5).attr('opacity', 1);
          tooltip
            .style('display', 'block')
            .html(`<strong>${d.label}</strong><br>${pair.x_label}: ${d.x.toFixed(3)}<br>${pair.y_label}: ${d.y.toFixed(3)}`);
        })
        .on('mousemove', function(event) {
          const rect = svgEl.getBoundingClientRect();
          tooltip
            .style('left', (event.clientX - rect.left + 12) + 'px')
            .style('top',  (event.clientY - rect.top  - 8)  + 'px');
        })
        .on('mouseleave', function() {
          d3.select(this).attr('r', 4.5).attr('opacity', 0.85);
          tooltip.style('display', 'none');
        });
  }

  onMount(render);
  afterUpdate(render);

  // Resize
  let resizeObs;
  onMount(() => {
    resizeObs = new ResizeObserver(() => render());
    if (svgEl?.parentElement) resizeObs.observe(svgEl.parentElement);
    return () => resizeObs?.disconnect();
  });
</script>

<div class="pareto-wrap">
  <svg bind:this={svgEl} width="100%" height="100%"></svg>
  <div class="pareto-tooltip"></div>

  <!-- Legend -->
  <div class="pareto-legend">
    {#each Object.entries(ESTIMATOR_COLOR) as [est, col]}
      <div class="legend-item">
        <span class="legend-dot" style="background:{col}"></span>
        <span>{est}</span>
      </div>
    {/each}
    <div class="legend-item">
      <span class="legend-dot pareto-front-dot"></span>
      <span>Pareto front</span>
    </div>
  </div>
</div>

<style>
  .pareto-wrap {
    width: 100%;
    height: 100%;
    position: relative;
  }

  svg { display: block; width: 100%; height: 100%; }

  .pareto-tooltip {
    display: none;
    position: absolute;
    background: rgba(13,25,41,0.95);
    border: 1px solid rgba(99,179,237,0.25);
    padding: 0.5rem 0.65rem;
    font-size: 0.72rem;
    font-family: var(--font-mono, monospace);
    color: var(--ink, #e2e8f0);
    line-height: 1.5;
    pointer-events: none;
    z-index: 10;
    white-space: nowrap;
    border-radius: 3px;
  }

  :global(.pareto-tooltip strong) { color: #e2e8f0; display: block; margin-bottom: 0.15rem; }

  .pareto-legend {
    position: absolute;
    top: 6px;
    right: 8px;
    display: flex;
    gap: 0.75rem;
    flex-wrap: wrap;
    justify-content: flex-end;
  }

  .legend-item {
    display: flex;
    align-items: center;
    gap: 0.3rem;
    font-size: 0.65rem;
    font-family: var(--font-mono, monospace);
    color: rgba(148,163,184,0.75);
  }

  .legend-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .pareto-front-dot {
    background: rgba(16,185,129,0.8);
  }
</style>
