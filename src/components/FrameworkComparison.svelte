<script>
  const STPA = {
    name: 'STPA',
    full: 'System-Theoretic Process Analysis',
    color: '#3b82f6',
    tagline: '"What control actions lead to unsafe outcomes?"',
    steps: [
      { label: 'Define Losses',         desc: 'Identify top-level safety constraints and unacceptable outcomes (e.g., collision, debris generation)' },
      { label: 'Model Control Loops',   desc: 'Map the feedback and control relationships between all system components' },
      { label: 'Identify UCAs',         desc: 'Find Unsafe Control Actions — commands that are missing, wrong, too early, too late, or applied too long' },
      { label: 'Trace Causal Scenarios',desc: 'Trace the conditions — sensor failures, software states, disturbances — that produce each UCA' },
    ],
    advantage: 'Captures emergent failures from subsystem interactions, not just component-level faults',
  };

  const EVDT = {
    name: 'EVDT',
    full: 'Exploration via Design and Trade Space',
    color: '#8b5cf6',
    tagline: '"How do design choices compound across performance dimensions?"',
    steps: [
      { label: 'Parameterize Design',   desc: 'Define the design variables: sensor types, filter architecture, control law gains, FSW fault logic' },
      { label: 'Define Metrics',        desc: 'Select performance dimensions: precision, safety, robustness, computational cost, fuel use' },
      { label: 'Sample Trade Space',    desc: 'Run Monte Carlo / DoE sweeps across the parameter space to map performance surfaces' },
      { label: 'Identify Frontiers',    desc: 'Find Pareto-optimal designs, sensitivity hotspots, and design regions with acceptable safety margins' },
    ],
    advantage: 'Reveals how sensor, filter, and control choices interact — and where the design space has acceptable safety margins',
  };
</script>

<div class="fw-wrap">
  {#each [STPA, EVDT] as fw}
    <div class="fw-card" style="--c:{fw.color}">
      <div class="fw-header">
        <div class="fw-badge">{fw.name}</div>
        <div class="fw-full">{fw.full}</div>
      </div>
      <div class="fw-tagline">"{fw.tagline.replace(/"/g,'')}"</div>

      <div class="fw-steps">
        {#each fw.steps as s, i}
          <div class="fw-step" style="animation-delay:{i*70}ms">
            <div class="step-num" style="color:{fw.color}">{String(i+1).padStart(2,'0')}</div>
            <div class="step-content">
              <div class="step-label">{s.label}</div>
              <div class="step-desc">{s.desc}</div>
            </div>
          </div>
        {/each}
      </div>

      <div class="fw-advantage" style="border-color:{fw.color}40; background:{fw.color}08">
        <span class="adv-icon">✦</span>
        <span class="adv-text">{fw.advantage}</span>
      </div>
    </div>
  {/each}

  <div class="fw-synergy">
    <span class="syn-icon">⟷</span>
    <div>
      <strong>Together:</strong> STPA provides the hazard vocabulary; EVDT provides the design exploration methodology. Both are required to develop autonomous systems that are not just functional, but <em>verifiably safe</em>.
    </div>
  </div>
</div>

<style>
  .fw-wrap {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1rem;
    align-items: start;
  }

  .fw-synergy {
    grid-column: 1 / -1;
    padding: 0.8rem 1rem;
    background: rgba(99,179,237,0.05);
    border: 1px solid rgba(99,179,237,0.15);
    border-radius: 4px;
    font-size: 0.82rem;
    line-height: 1.55;
    color: rgba(226,232,240,0.7);
    display: flex;
    align-items: flex-start;
    gap: 0.75rem;
  }
  .syn-icon { font-size: 1.1rem; flex-shrink: 0; padding-top: 0.1rem; }
  .fw-synergy strong { color: var(--ink); }
  .fw-synergy em { color: var(--cyan); font-style: italic; }

  .fw-card {
    background: var(--bg-card);
    border: 1px solid var(--c, var(--border));
    border-top: 3px solid var(--c);
    padding: 1.2rem;
    display: flex;
    flex-direction: column;
    gap: 0.85rem;
    border-radius: 0 0 6px 6px;
  }

  .fw-header {
    display: flex;
    align-items: baseline;
    gap: 0.65rem;
    flex-wrap: wrap;
  }

  .fw-badge {
    font-family: var(--font-mono);
    font-size: 1.2rem;
    font-weight: 700;
    color: var(--c);
    letter-spacing: 0.04em;
  }

  .fw-full {
    font-size: 0.72rem;
    color: var(--muted);
    line-height: 1.35;
  }

  .fw-tagline {
    font-size: 0.8rem;
    font-style: italic;
    color: rgba(226,232,240,0.55);
    line-height: 1.4;
    border-left: 2px solid var(--c);
    padding-left: 0.65rem;
  }

  .fw-steps {
    display: flex;
    flex-direction: column;
    gap: 0.6rem;
  }

  .fw-step {
    display: grid;
    grid-template-columns: 1.8rem 1fr;
    gap: 0.5rem;
    align-items: start;
    animation: fadeInUp 400ms ease-out both;
  }

  @keyframes fadeInUp {
    from { opacity: 0; transform: translateY(8px); }
    to   { opacity: 1; transform: translateY(0); }
  }

  .step-num {
    font-family: var(--font-mono);
    font-size: 0.68rem;
    font-weight: 700;
    padding-top: 0.12rem;
    letter-spacing: 0.04em;
  }

  .step-label {
    font-family: var(--font-ui);
    font-size: 0.8rem;
    font-weight: 700;
    color: var(--ink);
    margin-bottom: 0.18rem;
  }

  .step-desc {
    font-size: 0.73rem;
    line-height: 1.5;
    color: var(--muted);
  }

  .fw-advantage {
    padding: 0.6rem 0.75rem;
    border: 1px solid;
    border-radius: 3px;
    display: flex;
    align-items: flex-start;
    gap: 0.5rem;
  }

  .adv-icon { font-size: 0.75rem; flex-shrink: 0; padding-top: 0.15rem; }

  .adv-text {
    font-size: 0.75rem;
    line-height: 1.5;
    color: rgba(226,232,240,0.7);
    font-style: italic;
  }

  @media (max-width: 760px) {
    .fw-wrap { grid-template-columns: 1fr; }
  }
</style>
