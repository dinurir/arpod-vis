<script>
  const MISSIONS = [
    {
      year: '2020',
      name: 'MEV-1',
      org: 'Northrop Grumman',
      color: '#3b82f6',
      icon: '🛰',
      tags: ['GEO', 'Life Extension', 'Docking'],
      body: 'First commercial satellite life extension in GEO. MEV-1 docked with Intelsat 901, assumed attitude and station-keeping control — proving servicing is commercially viable.',
    },
    {
      year: '2021',
      name: 'MEV-2',
      org: 'Northrop Grumman',
      color: '#3b82f6',
      icon: '🛰',
      tags: ['GEO', 'Life Extension', 'Repeat'],
      body: 'Second mission extension vehicle, docking with Intelsat 10-02. Confirmed MEV as a repeatable operational concept, not a one-off demonstration.',
    },
    {
      year: '2005',
      name: 'DART',
      org: 'NASA / Orbital Sciences',
      color: '#8b5cf6',
      icon: '⚠',
      tags: ['Autonomous Rendezvous', 'LEO', 'Partial Failure'],
      body: 'Demonstration for Autonomous Rendezvous Technology — the first U.S. attempt at fully autonomous spacecraft rendezvous, targeting the MUBLCOM satellite. A navigation system failure at ~200 m separation triggered an uncontrolled approach; DART collided with its target at 1.5 m/s and depleted propellant within minutes. Only 11 of 27 primary objectives were met, and none of the 14 proximity operations objectives succeeded. A landmark cautionary case: demonstrating both the promise of autonomous CPO and the critical consequences of navigation and fault-management failures.',
    },
    {
      year: 'TBD',
      name: 'Otter',
      org: 'Starfish Space',
      color: '#06b6d4',
      icon: '🦦',
      tags: ['LEO', 'Small Sat', 'Adhesive Capture'],
      body: 'Developing a servicer for non-cooperative LEO targets using visual navigation and gecko-inspired adhesive capture — illustrating the diversity of technical challenges across orbital regimes.',
    },
  ];
</script>

<div class="timeline-wrap">
  <div class="spine"></div>
  {#each MISSIONS as m, i}
    <div class="mission" style="--col:{m.color}; animation-delay:{i*120}ms">
      <div class="year-node">
        <span class="year">{m.year}</span>
        <div class="node-dot" style="background:{m.color}; box-shadow: 0 0 10px {m.color}88"></div>
      </div>
      <div class="card">
        <div class="card-header">
          <span class="icon">{m.icon}</span>
          <div>
            <div class="mission-name">{m.name}</div>
            <div class="mission-org">{m.org}</div>
          </div>
        </div>
        <div class="tags">
          {#each m.tags as tag}
            <span class="tag" style="border-color:{m.color}40; color:{m.color}">{tag}</span>
          {/each}
        </div>
        <p class="mission-body">{m.body}</p>
      </div>
    </div>
  {/each}
</div>

<style>
  .timeline-wrap {
    position: relative;
    padding: 1rem 0 1rem 0;
    display: flex;
    flex-direction: column;
    gap: 1.6rem;
  }

  .spine {
    position: absolute;
    left: 5.5rem;
    top: 0;
    bottom: 0;
    width: 1px;
    background: linear-gradient(to bottom, transparent, rgba(99,179,237,0.2) 10%, rgba(99,179,237,0.2) 90%, transparent);
  }

  .mission {
    display: grid;
    grid-template-columns: 6rem 1fr;
    gap: 1rem;
    align-items: start;
    animation: fadeInUp 500ms ease-out both;
  }

  @keyframes fadeInUp {
    from { opacity: 0; transform: translateY(14px); }
    to   { opacity: 1; transform: translateY(0); }
  }

  .year-node {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 0.5rem;
    padding-top: 0.85rem;
  }

  .year {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    font-weight: 700;
    color: var(--muted);
    letter-spacing: 0.06em;
  }

  .node-dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    flex-shrink: 0;
    position: relative;
    z-index: 1;
  }

  .card {
    background: var(--bg-card);
    border: 1px solid var(--border);
    border-left: 2px solid var(--col);
    padding: 1rem 1.2rem;
    border-radius: 0 6px 6px 0;
    transition: border-color 200ms ease;
  }

  .card:hover { border-color: var(--col); }

  .card-header {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    margin-bottom: 0.55rem;
  }

  .icon { font-size: 1.3rem; }

  .mission-name {
    font-family: var(--font-ui);
    font-size: 0.95rem;
    font-weight: 700;
    color: var(--ink);
  }

  .mission-org {
    font-size: 0.72rem;
    color: var(--muted);
    font-family: var(--font-mono);
    letter-spacing: 0.04em;
  }

  .tags {
    display: flex;
    flex-wrap: wrap;
    gap: 0.35rem;
    margin-bottom: 0.65rem;
  }

  .tag {
    font-size: 0.65rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    padding: 0.18rem 0.45rem;
    border: 1px solid;
    border-radius: 2px;
    font-family: var(--font-mono);
  }

  .mission-body {
    font-size: 0.84rem;
    line-height: 1.6;
    color: rgba(226,232,240,0.72);
  }
</style>
