<script>
  const BLOCKS = [
    { id: 'sensors',   label: 'Sensors',         sub: 'LiDAR · Camera · IMU · Star Tracker', color: '#06b6d4', col: 0 },
    { id: 'estimator', label: 'State Estimator',  sub: 'EKF / UKF · Pose · Velocity',         color: '#3b82f6', col: 1 },
    { id: 'guidance',  label: 'Guidance',         sub: 'Path Planning · Trajectory Gen',       color: '#8b5cf6', col: 2 },
    { id: 'control',   label: 'Controller',       sub: 'MPC / PD · Attitude Control',          color: '#a855f7', col: 3 },
    { id: 'actuators', label: 'Actuators',        sub: 'Thrusters · Reaction Wheels',          color: '#ec4899', col: 4 },
    { id: 'fsw',       label: 'Flight Software',  sub: 'Fault Management · State Machine',     color: '#f59e0b', col: 5 },
  ];

  const FAILURE_MODES = [
    { block: 'sensors',   text: 'Sensor bias / noise',   color: '#ef4444' },
    { block: 'estimator', text: 'Filter divergence',     color: '#ef4444' },
    { block: 'guidance',  text: 'Infeasible trajectory', color: '#ef4444' },
    { block: 'control',   text: 'Unmodeled dynamics',    color: '#ef4444' },
    { block: 'actuators', text: 'Saturation / failure',  color: '#ef4444' },
    { block: 'fsw',       text: 'Unhandled state',       color: '#ef4444' },
  ];
</script>

<div class="diagram-wrap">
  <div class="chain-label">AUTONOMOUS FLIGHT SOFTWARE STACK</div>

  <div class="chain">
    {#each BLOCKS as b, i}
      <div class="block-col">
        <div class="block" style="--c:{b.color}; animation-delay:{i*80}ms">
          <div class="block-name">{b.label}</div>
          <div class="block-sub">{b.sub}</div>
        </div>
        {#if i < BLOCKS.length - 1}
          <div class="arrow">→</div>
        {/if}
      </div>
    {/each}
  </div>

  <!-- Failure modes below each block -->
  <div class="failures">
    {#each BLOCKS as b, i}
      <div class="fail-col" style="animation-delay:{i*80 + 300}ms">
        <div class="fail-arrow">↓</div>
        <div class="fail-card">
          <span class="fail-icon">⚠</span>
          <span class="fail-text">{FAILURE_MODES[i].text}</span>
        </div>
      </div>
    {/each}
  </div>

  <!-- Cascade annotation -->
  <div class="cascade-banner">
    <span class="cascade-icon">⛓</span>
    <span>Failure at <em>any</em> point can cascade — sensor bias → filter divergence → guidance error → collision</span>
  </div>

  <!-- FSW overlay band -->
  <div class="fsw-band">
    <span class="fsw-label">FLIGHT SOFTWARE coordinates all subsystems in real time</span>
  </div>
</div>

<style>
  .diagram-wrap {
    width: 100%;
    padding: 1.25rem 0;
    display: flex;
    flex-direction: column;
    gap: 0;
  }

  .chain-label {
    font-family: var(--font-mono);
    font-size: 0.65rem;
    font-weight: 700;
    letter-spacing: 0.14em;
    text-transform: uppercase;
    color: var(--muted);
    margin-bottom: 0.9rem;
  }

  .chain {
    display: flex;
    align-items: center;
    gap: 0;
    overflow-x: auto;
    padding-bottom: 0.25rem;
  }

  .block-col {
    display: flex;
    align-items: center;
    gap: 0;
    flex-shrink: 0;
  }

  .block {
    background: var(--bg-card);
    border: 1px solid var(--c);
    border-top: 3px solid var(--c);
    padding: 0.75rem 0.85rem;
    min-width: 110px;
    animation: fadeInUp 400ms ease-out both;
    position: relative;
  }

  .block::after {
    content: '';
    position: absolute;
    inset: 0;
    background: var(--c);
    opacity: 0.04;
    pointer-events: none;
  }

  @keyframes fadeInUp {
    from { opacity: 0; transform: translateY(10px); }
    to   { opacity: 1; transform: translateY(0); }
  }

  .block-name {
    font-family: var(--font-ui);
    font-size: 0.82rem;
    font-weight: 700;
    color: var(--c);
    margin-bottom: 0.3rem;
  }

  .block-sub {
    font-size: 0.65rem;
    line-height: 1.45;
    color: var(--muted);
    font-family: var(--font-mono);
  }

  .arrow {
    font-size: 1rem;
    color: var(--muted);
    padding: 0 0.3rem;
    flex-shrink: 0;
  }

  .failures {
    display: flex;
    align-items: flex-start;
    gap: 0;
    margin-top: 0.25rem;
  }

  .fail-col {
    display: flex;
    flex-direction: column;
    align-items: center;
    min-width: 110px;
    gap: 0.2rem;
    flex-shrink: 0;
    animation: fadeInUp 400ms ease-out both;
  }

  /* account for arrow gaps */
  .fail-col:not(:first-child) { margin-left: 1.6rem; }

  .fail-arrow {
    font-size: 0.75rem;
    color: rgba(239,68,68,0.5);
    line-height: 1;
  }

  .fail-card {
    background: rgba(239,68,68,0.08);
    border: 1px solid rgba(239,68,68,0.2);
    border-radius: 3px;
    padding: 0.4rem 0.5rem;
    display: flex;
    align-items: flex-start;
    gap: 0.35rem;
    width: 100%;
  }

  .fail-icon { font-size: 0.7rem; line-height: 1.3; }

  .fail-text {
    font-size: 0.67rem;
    line-height: 1.45;
    color: rgba(239,68,68,0.85);
    font-family: var(--font-mono);
  }

  .cascade-banner {
    margin-top: 1.1rem;
    padding: 0.65rem 0.9rem;
    background: rgba(239,68,68,0.06);
    border: 1px solid rgba(239,68,68,0.18);
    border-radius: 4px;
    font-size: 0.8rem;
    line-height: 1.5;
    color: rgba(226,232,240,0.7);
    display: flex;
    align-items: flex-start;
    gap: 0.6rem;
  }
  .cascade-icon { font-size: 1rem; flex-shrink: 0; }
  .cascade-banner em { font-style: italic; color: #ef4444; font-weight: 600; }

  .fsw-band {
    margin-top: 0.75rem;
    padding: 0.5rem 0.85rem;
    background: rgba(245,158,11,0.06);
    border: 1px solid rgba(245,158,11,0.2);
    border-radius: 4px;
    text-align: center;
  }

  .fsw-label {
    font-size: 0.72rem;
    font-family: var(--font-mono);
    color: rgba(245,158,11,0.8);
    letter-spacing: 0.04em;
  }
</style>
