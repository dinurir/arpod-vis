<script>
  import { onMount, onDestroy, tick } from 'svelte';
  import { fade } from 'svelte/transition';

  import OrbitalDensity      from './components/OrbitalDensity.svelte';
  import GEOArc              from './components/GEOArc.svelte';
  import ApproachAnimation   from './components/ApproachAnimation.svelte';
  import MissionTimeline     from './components/MissionTimeline.svelte';
  import SystemBlockDiagram  from './components/SystemBlockDiagram.svelte';
  import FrameworkComparison from './components/FrameworkComparison.svelte';
  import SimDashboard        from './components/SimDashboard.svelte';

  // ── Scroll state ──────────────────────────────────────────────────────────
  let currentSection = -1;
  let scrollObserver = null;
  let heroVisible = true;

  const SECTIONS = [
    { id: 0, label: 'PROBLEM',     nav: 'Space Congestion'    },
    { id: 1, label: 'SOLUTION',    nav: 'Satellite Servicing' },
    { id: 2, label: 'CHALLENGE',   nav: 'Docking Problem'     },
    { id: 3, label: 'HISTORY',     nav: 'Historical Missions' },
    { id: 4, label: 'TRUST',       nav: 'Failure Modes'       },
    { id: 5, label: 'FRAMEWORKS',  nav: 'Analysis Frameworks' },
    { id: 6, label: 'SIMULATION',  nav: 'Simulation Results'  },
  ];

  // Which viz to show in the sticky panel — driven by currentSection
  $: activeViz = currentSection;

  function setupObserver() {
    if (scrollObserver) scrollObserver.disconnect();
    const els = document.querySelectorAll('[data-section]');
    scrollObserver = new IntersectionObserver(
      (entries) => {
        for (const e of entries) {
          if (e.isIntersecting) currentSection = +e.target.dataset.section;
        }
      },
      { threshold: 0.35 }
    );
    els.forEach((el) => scrollObserver.observe(el));

    // Hero observer
    const hero = document.querySelector('.hero');
    if (hero) {
      new IntersectionObserver(
        ([e]) => { heroVisible = e.isIntersecting; },
        { threshold: 0.1 }
      ).observe(hero);
    }
  }

  onMount(() => { tick().then(setupObserver); });
  onDestroy(() => scrollObserver?.disconnect());

  function scrollToSection(id) {
    document.querySelector(`[data-section="${id}"]`)?.scrollIntoView({ behavior: 'smooth' });
  }

  // ── Kessler counter animation ─────────────────────────────────────────────
  let debrisCount = 0;
  let debrisAnimId = null;

  $: if (currentSection === 0) {
    if (debrisAnimId) cancelAnimationFrame(debrisAnimId);
    debrisCount = 0;
    const target = 27000;
    const start = performance.now();
    const dur = 2400;
    function frame(now) {
      const t = Math.min((now - start) / dur, 1);
      const eased = 1 - Math.pow(1 - t, 3);
      debrisCount = Math.round(eased * target);
      if (t < 1) debrisAnimId = requestAnimationFrame(frame);
    }
    debrisAnimId = requestAnimationFrame(frame);
  }

  // ── GEO asset countup ─────────────────────────────────────────────────────
  let geoValue = 0;
  let geoAnimId = null;

  $: if (currentSection === 1) {
    if (geoAnimId) cancelAnimationFrame(geoAnimId);
    geoValue = 0;
    const target = 35786;
    const start = performance.now();
    const dur = 1800;
    function frame(now) {
      const t = Math.min((now - start) / dur, 1);
      const eased = 1 - Math.pow(1 - t, 3);
      geoValue = Math.round(eased * target);
      if (t < 1) geoAnimId = requestAnimationFrame(frame);
    }
    geoAnimId = requestAnimationFrame(frame);
  }
</script>

<!-- ── Nav bar ──────────────────────────────────────────────────────────── -->
<nav class="top-nav" class:scrolled={!heroVisible}>
  <div class="nav-inner">
    <span class="nav-wordmark">CPO RESEARCH</span>
    <div class="nav-links">
      {#each SECTIONS as s}
        <button
          class="nav-link"
          class:active={currentSection === s.id}
          on:click={() => scrollToSection(s.id)}
        >{s.nav}</button>
      {/each}
    </div>
  </div>
</nav>

<!-- ── Hero ─────────────────────────────────────────────────────────────── -->
<header class="hero">
  <div class="hero-bg">
    <OrbitalDensity />
  </div>
  <div class="hero-content">
    <div class="hero-eyebrow">MIT MEDIA LAB · SPACE ENABLED RESEARCH GROUP</div>
    <h1 class="hero-title">Autonomous<br>Close Proximity<br>Operations</h1>
    <p class="hero-dek">
      A rigorous investigation into the safety, failure modes, and design trade space
      of autonomous satellite servicing systems — from hazard analysis to high-fidelity simulation.
    </p>
    <div class="hero-meta">
      <span class="hero-meta-item">Master's Thesis Research</span>
      <span class="hero-sep">·</span>
      <span class="hero-meta-item">STPA + EVDT Frameworks</span>
      <span class="hero-sep">·</span>
      <span class="hero-meta-item">GEO/LEO Servicing</span>
    </div>
    <button class="hero-cta" on:click={() => scrollToSection(0)}>
      Explore the Research ↓
    </button>
  </div>
</header>

<!-- ── Scrollytelling body ───────────────────────────────────────────────── -->
<main class="scroll-body">

  <!-- SECTION 1 — Space Congestion ──────────────────────────────────────── -->
  <div class="scroll-section" data-section="0">
    <div class="narrative-col">
      <div class="section-eyebrow">SECTION 01</div>
      <h2 class="section-heading">Space is Getting Crowded</h2>

      <div class="prose-block active">
        <p>
          Space sustainability refers to the long-term ability to conduct safe and responsible
          activities in Earth orbit — preserving the space environment for future generations
          and missions. Today, that environment is under serious strain.
        </p>
      </div>

      <div class="prose-block">
        <p>
          The number of active satellites has grown exponentially, driven by large commercial
          constellations from operators like SpaceX, Amazon, and OneWeb. Alongside active
          spacecraft, thousands of defunct satellites, rocket bodies, and debris fragments
          occupy critical orbital regimes.
        </p>
      </div>

      <div class="stat-callout">
        <div class="stat-num" style="color: var(--blue)">{debrisCount.toLocaleString()}</div>
        <div class="stat-label">tracked debris objects in Earth orbit</div>
      </div>

      <div class="prose-block">
        <p>
          Collisions between objects — even small ones — can trigger cascading debris fields
          in a phenomenon known as the <span class="highlight">Kessler Syndrome</span>, which
          could render entire orbital bands unusable for decades.
        </p>
      </div>

      <div class="callout-box callout-box--warn">
        <span class="callout-icon">⚠</span>
        <span>Space congestion is not a distant threat. It is a <strong>present and accelerating reality</strong> that demands new approaches to how we operate, maintain, and retire spacecraft.</span>
      </div>
    </div>

    <div class="viz-col-sticky">
      <div class="viz-panel">
        <div class="viz-label">ORBITAL POPULATION — LIVE SIMULATION</div>
        <OrbitalDensity />
      </div>
    </div>
  </div>

  <!-- SECTION 2 — Satellite Servicing ───────────────────────────────────── -->
  <div class="scroll-section" data-section="1">
    <div class="narrative-col">
      <div class="section-eyebrow">SECTION 02</div>
      <h2 class="section-heading">A Solution: Satellite Servicing</h2>

      <div class="prose-block">
        <p>
          One of the most promising tools for improving space sustainability is
          <span class="highlight">on-orbit satellite servicing</span> — the ability to inspect,
          repair, refuel, or reposition spacecraft already in orbit.
        </p>
      </div>

      <div class="stat-callout">
        <div class="stat-num" style="color: var(--cyan)">{geoValue.toLocaleString()}</div>
        <div class="stat-label">km above the equator — the GEO belt</div>
      </div>

      <div class="prose-block">
        <p>
          The benefits are especially significant in Geostationary Orbit (GEO), where satellites
          are large, expensive, and occupy a limited and highly congested arc. A single GEO
          communications satellite can represent a <strong>billion-dollar asset</strong> with a
          design life of 15 years.
        </p>
      </div>

      <div class="prose-block">
        <p>
          Today, when these satellites run low on propellant or experience a subsystem anomaly,
          they are typically retired — drifting into a graveyard orbit and permanently occupying
          spectrum and orbital slots.
        </p>
      </div>

      <div class="callout-box callout-box--blue">
        <span class="callout-icon">✦</span>
        <span>Satellite servicing changes this calculus entirely. Life extension of even a few years per satellite translates to enormous <strong>economic and sustainability value</strong> across a fleet.</span>
      </div>
    </div>

    <div class="viz-col-sticky">
      <div class="viz-panel">
        <div class="viz-label">GEO ARC — 35,786 km</div>
        <GEOArc />
      </div>
    </div>
  </div>

  <!-- SECTION 3 — Docking Problem ────────────────────────────────────────── -->
  <div class="scroll-section" data-section="2">
    <div class="narrative-col">
      <div class="section-eyebrow">SECTION 03</div>
      <h2 class="section-heading">The Docking Problem:<br>Getting Close Safely</h2>

      <div class="prose-block">
        <p>
          Before any servicing can occur, the servicer spacecraft must do something deceptively
          difficult: it must <span class="highlight">find, approach, and physically dock</span>
          with another vehicle in the harsh and unforgiving environment of space.
        </p>
        <p class="prose-sub">This is the close proximity operations (CPO) problem.</p>
      </div>

      <div class="prose-block">
        <p>
          As orbital environments grow more congested and missions grow more complex — involving
          non-cooperative targets, legacy spacecraft with no docking ports, and simultaneous
          multi-vehicle operations — these proximity operations can no longer rely on continuous
          human oversight and ground-in-the-loop commanding.
        </p>
      </div>

      <div class="three-pillars">
        {#each [
          { icon: '⚡', label: 'Latency', desc: 'Communication delays make real-time ground control infeasible at critical moments' },
          { icon: '📊', label: 'Volume',  desc: 'The rate of decisions required exceeds human operator capacity' },
          { icon: '⏱',  label: 'Speed',   desc: 'Situations evolve faster than round-trip comms allow' },
        ] as p}
          <div class="pillar">
            <span class="pillar-icon">{p.icon}</span>
            <span class="pillar-label">{p.label}</span>
            <span class="pillar-desc">{p.desc}</span>
          </div>
        {/each}
      </div>

      <div class="callout-box callout-box--purple">
        <span class="callout-icon">→</span>
        <span>Autonomy is not just a convenience here. It is an <strong>operational necessity</strong>.</span>
      </div>
    </div>

    <div class="viz-col-sticky">
      <div class="viz-panel">
        <div class="viz-label">TERMINAL APPROACH SIMULATION</div>
        <ApproachAnimation />
      </div>
    </div>
  </div>

  <!-- SECTION 4 — Historical Missions ────────────────────────────────────── -->
  <div class="scroll-section scroll-section--wide" data-section="3">
    <div class="narrative-col narrative-col--wide">
      <div class="section-eyebrow">SECTION 04</div>
      <h2 class="section-heading">Historical Missions:<br>Proving the Concept</h2>

      <div class="prose-block">
        <p>
          These missions collectively illustrate an arc: from human-supervised proximity
          operations to increasingly autonomous, capable, and commercially-driven servicing.
          Each represents a milestone in establishing the technical and commercial viability
          of on-orbit operations.
        </p>
      </div>

      <MissionTimeline />

      <div class="callout-box callout-box--blue" style="margin-top: 1.5rem">
        <span class="callout-icon">◈</span>
        <span>From MEV's first commercial docking in 2020 to DART's autonomous terminal guidance in 2022, the industry has moved from proof-of-concept to operational capability — with fully autonomous servicing as the clear next frontier.</span>
      </div>
    </div>
  </div>

  <!-- SECTION 5 — Trust Problem ──────────────────────────────────────────── -->
  <div class="scroll-section" data-section="4">
    <div class="narrative-col">
      <div class="section-eyebrow">SECTION 05</div>
      <h2 class="section-heading">The Trust Problem:<br>How Do Autonomous<br>Systems Fail?</h2>

      <div class="prose-block">
        <p>
          For autonomous satellite servicing to become routine, we must be able to
          <span class="highlight">trust these systems</span> — and trust is built on a thorough,
          rigorous understanding of how they can fail.
        </p>
      </div>

      <div class="prose-block">
        <p>
          An autonomous servicer is not a single technology. It is a tightly integrated stack
          of subsystems: sensors, estimators, guidance algorithms, controllers, actuators, and
          flight software that coordinates everything in real time.
        </p>
      </div>

      <div class="prose-block">
        <p>
          A failure at <em>any point</em> in this chain — a biased sensor, a diverging filter,
          a saturated actuator, an unhandled software state — can cascade into mission loss or,
          worse, a <span class="highlight-red">collision that generates debris</span> and
          threatens neighboring assets.
        </p>
      </div>

      <div class="callout-box callout-box--red">
        <span class="callout-icon">⛓</span>
        <span>Understanding how this system fails holistically is not optional. It is the <strong>foundation of safe, certifiable autonomous operations</strong>.</span>
      </div>
    </div>

    <div class="viz-col-sticky">
      <div class="viz-panel viz-panel--scroll">
        <div class="viz-label">AUTONOMOUS FLIGHT SOFTWARE STACK</div>
        <div class="viz-inner-scroll">
          <SystemBlockDiagram />
        </div>
      </div>
    </div>
  </div>

  <!-- SECTION 6 — Frameworks ──────────────────────────────────────────────── -->
  <div class="scroll-section scroll-section--wide" data-section="5">
    <div class="narrative-col narrative-col--wide">
      <div class="section-eyebrow">SECTION 06</div>
      <h2 class="section-heading">Frameworks for<br>Rigorous Analysis</h2>

      <div class="prose-block">
        <p>
          Two analytical frameworks anchor this research. Together, they provide both the
          hazard vocabulary and the design exploration methodology needed to develop autonomous
          systems that are not just functional, but <span class="highlight">verifiably safe</span>.
        </p>
      </div>

      <FrameworkComparison />
    </div>
  </div>

  <!-- SECTION 7 — Simulation ──────────────────────────────────────────────── -->
  <div class="scroll-section scroll-section--wide" data-section="6">
    <div class="narrative-col narrative-col--wide">
      <div class="section-eyebrow">SECTION 07</div>
      <h2 class="section-heading">The Simulation:<br>Where Theory Meets Reality</h2>

      <div class="prose-block">
        <p>
          Theoretical analysis is a starting point, not a destination. At the
          <span class="highlight">Space Enabled Research Group at MIT Media Lab</span>, we have
          developed a rigorous, high-fidelity simulation environment that puts our STPA hazard
          analysis, EVDT trade space exploration, and sensor/actuator modeling to the test in a
          closed-loop, dynamic setting.
        </p>
      </div>

      <div class="prose-block">
        <p>
          This simulation spans the full autonomous flight software stack — from raw sensor
          outputs through state estimation, guidance, control, and actuator execution — and
          incorporates realistic hardware uncertainties drawn directly from
          <span class="highlight">NASA's Small Spacecraft Technology State-of-the-Art database</span>
          and flight-heritage GNC component specifications.
        </p>
      </div>

      <div class="callout-box callout-box--cyan" style="margin-bottom: 1.75rem">
        <span class="callout-icon">↓</span>
        <span>Each visualization below tells part of the story of how an autonomous servicer sees, thinks, decides, and acts — and where, under what conditions, it struggles.</span>
      </div>

      <SimDashboard />
    </div>
  </div>

</main>

<!-- ── Footer ────────────────────────────────────────────────────────────── -->
<footer class="site-footer">
  <div class="footer-inner">
    <div class="footer-left">
      <div class="footer-wordmark">AUTONOMOUS CPO RESEARCH</div>
      <div class="footer-sub">Space Enabled Research Group · MIT Media Lab</div>
    </div>
    <div class="footer-right">
      <div class="footer-note">Master's Thesis · {new Date().getFullYear()}</div>
    </div>
  </div>
</footer>

<style>
  /* ── Nav ──────────────────────────────────────────────────────────────── */
  .top-nav {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 900;
    padding: 0.85rem 2rem;
    transition: background 300ms ease, border-color 300ms ease;
    border-bottom: 1px solid transparent;
  }

  .top-nav.scrolled {
    background: rgba(5, 10, 20, 0.92);
    backdrop-filter: blur(12px);
    border-bottom-color: var(--border);
  }

  .nav-inner {
    display: flex;
    align-items: center;
    justify-content: space-between;
    max-width: 1400px;
    margin: 0 auto;
  }

  .nav-wordmark {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    font-weight: 700;
    letter-spacing: 0.16em;
    color: var(--cyan);
  }

  .nav-links {
    display: flex;
    gap: 0.25rem;
    flex-wrap: wrap;
  }

  .nav-link {
    background: none;
    border: none;
    font-family: var(--font-mono);
    font-size: 0.65rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    color: var(--muted);
    cursor: pointer;
    padding: 0.3rem 0.55rem;
    border-radius: 3px;
    transition: color 150ms ease, background 150ms ease;
  }

  .nav-link:hover { color: var(--ink); background: rgba(99,179,237,0.08); }
  .nav-link.active { color: var(--cyan); }

  /* ── Hero ─────────────────────────────────────────────────────────────── */
  .hero {
    position: relative;
    height: 100vh;
    min-height: 600px;
    display: flex;
    align-items: center;
    overflow: hidden;
  }

  .hero-bg {
    position: absolute;
    inset: 0;
    opacity: 0.6;
  }

  .hero-content {
    position: relative;
    z-index: 10;
    max-width: 1400px;
    margin: 0 auto;
    padding: 0 2rem;
    display: flex;
    flex-direction: column;
    gap: 1.2rem;
  }

  .hero-eyebrow {
    font-family: var(--font-mono);
    font-size: 0.68rem;
    font-weight: 700;
    letter-spacing: 0.18em;
    color: var(--cyan);
    text-transform: uppercase;
  }

  .hero-title {
    font-family: var(--font-ui);
    font-size: clamp(2.8rem, 6vw, 5.5rem);
    font-weight: 700;
    line-height: 1.08;
    letter-spacing: -0.025em;
    color: var(--ink);
    margin: 0;
  }

  .hero-dek {
    font-size: clamp(0.9rem, 1.3vw, 1.05rem);
    line-height: 1.65;
    color: rgba(226,232,240,0.65);
    max-width: 52ch;
    margin: 0;
  }

  .hero-meta {
    display: flex;
    align-items: center;
    gap: 0.6rem;
    flex-wrap: wrap;
  }

  .hero-meta-item {
    font-family: var(--font-mono);
    font-size: 0.7rem;
    color: var(--muted);
    letter-spacing: 0.04em;
  }

  .hero-sep { color: var(--faint); }

  .hero-cta {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    background: none;
    border: 1px solid var(--border-bright);
    color: var(--cyan);
    font-family: var(--font-mono);
    font-size: 0.78rem;
    font-weight: 700;
    letter-spacing: 0.08em;
    padding: 0.65rem 1.25rem;
    cursor: pointer;
    transition: background 200ms ease, border-color 200ms ease;
    width: fit-content;
    margin-top: 0.5rem;
    border-radius: 3px;
  }

  .hero-cta:hover {
    background: rgba(6, 182, 212, 0.08);
    border-color: var(--cyan);
  }

  /* ── Main body ────────────────────────────────────────────────────────── */
  .scroll-body {
    max-width: 1400px;
    margin: 0 auto;
    padding: 0 2rem 4rem;
  }

  /* ── Scroll sections ──────────────────────────────────────────────────── */
  .scroll-section {
    display: grid;
    grid-template-columns: 44% 56%;
    gap: 0;
    min-height: 100vh;
    align-items: start;
    padding: 6rem 0 4rem;
    border-bottom: 1px solid var(--border);
    position: relative;
  }

  .scroll-section--wide {
    grid-template-columns: 1fr;
  }

  /* ── Narrative column ─────────────────────────────────────────────────── */
  .narrative-col {
    padding-right: 3rem;
    padding-top: 2rem;
    display: flex;
    flex-direction: column;
    gap: 1.4rem;
  }

  .narrative-col--wide {
    padding-right: 0;
    max-width: 100%;
  }

  .section-eyebrow {
    font-family: var(--font-mono);
    font-size: 0.62rem;
    font-weight: 700;
    letter-spacing: 0.2em;
    color: var(--cyan);
    text-transform: uppercase;
  }

  .section-heading {
    font-family: var(--font-ui);
    font-size: clamp(1.8rem, 2.8vw, 2.6rem);
    font-weight: 700;
    line-height: 1.1;
    letter-spacing: -0.02em;
    color: var(--ink);
    margin: 0;
  }

  .prose-block {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }

  .prose-block p {
    font-size: clamp(0.88rem, 1.1vw, 0.96rem);
    line-height: 1.72;
    color: rgba(226,232,240,0.72);
    margin: 0;
  }

  .prose-sub {
    font-family: var(--font-mono) !important;
    font-size: 0.82rem !important;
    color: var(--cyan) !important;
    font-weight: 600;
    letter-spacing: 0.02em;
  }

  .highlight {
    color: var(--cyan);
    font-weight: 600;
  }

  .highlight-red {
    color: #ef4444;
    font-weight: 600;
  }

  /* ── Stat callouts ────────────────────────────────────────────────────── */
  .stat-callout {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    padding: 1.1rem 1.4rem;
    background: var(--bg-card);
    border: 1px solid var(--border);
    border-left: 3px solid var(--blue);
  }

  .stat-num {
    font-family: var(--font-mono);
    font-size: clamp(2.8rem, 5vw, 4.2rem);
    font-weight: 700;
    line-height: 1;
    letter-spacing: -0.03em;
  }

  .stat-label {
    font-size: 0.78rem;
    color: var(--muted);
    font-family: var(--font-mono);
    letter-spacing: 0.04em;
  }

  /* ── Callout boxes ────────────────────────────────────────────────────── */
  .callout-box {
    padding: 0.85rem 1rem;
    border: 1px solid;
    border-radius: 4px;
    font-size: 0.84rem;
    line-height: 1.6;
    display: flex;
    align-items: flex-start;
    gap: 0.7rem;
  }

  .callout-box strong { font-weight: 700; }

  .callout-box--warn   { background: rgba(245,158,11,0.07); border-color: rgba(245,158,11,0.25); color: rgba(226,232,240,0.8); }
  .callout-box--blue   { background: rgba(59,130,246,0.07);  border-color: rgba(59,130,246,0.25);  color: rgba(226,232,240,0.8); }
  .callout-box--purple { background: rgba(139,92,246,0.07); border-color: rgba(139,92,246,0.25); color: rgba(226,232,240,0.8); }
  .callout-box--red    { background: rgba(239,68,68,0.07);  border-color: rgba(239,68,68,0.25);  color: rgba(226,232,240,0.8); }
  .callout-box--cyan   { background: rgba(6,182,212,0.07);  border-color: rgba(6,182,212,0.25);  color: rgba(226,232,240,0.8); }

  .callout-icon { font-size: 1rem; flex-shrink: 0; padding-top: 0.1rem; }

  /* ── Three pillars ────────────────────────────────────────────────────── */
  .three-pillars {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.75rem;
  }

  .pillar {
    display: flex;
    flex-direction: column;
    gap: 0.35rem;
    padding: 0.85rem;
    background: var(--bg-card);
    border: 1px solid var(--border);
    border-top: 2px solid var(--purple);
  }

  .pillar-icon { font-size: 1.2rem; }
  .pillar-label { font-family: var(--font-ui); font-size: 0.82rem; font-weight: 700; color: var(--ink); }
  .pillar-desc  { font-size: 0.73rem; line-height: 1.5; color: var(--muted); }

  /* ── Sticky viz column ────────────────────────────────────────────────── */
  .viz-col-sticky {
    position: sticky;
    top: 0;
    height: 100vh;
    align-self: start;
    display: flex;
    align-items: stretch;
    padding: 1.5rem 0 1.5rem 1rem;
  }

  .viz-panel {
    width: 100%;
    background: var(--bg-panel);
    border: 1px solid var(--border);
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  .viz-panel--scroll {
    overflow-y: auto;
  }

  .viz-label {
    padding: 0.65rem 1rem;
    font-family: var(--font-mono);
    font-size: 0.62rem;
    font-weight: 700;
    letter-spacing: 0.14em;
    color: var(--muted);
    border-bottom: 1px solid var(--border);
    flex-shrink: 0;
    text-transform: uppercase;
  }

  .viz-panel > :global(*:not(.viz-label)) {
    flex: 1;
    min-height: 0;
  }

  .viz-inner-scroll {
    padding: 1rem;
    overflow-y: auto;
  }

  /* ── Footer ───────────────────────────────────────────────────────────── */
  .site-footer {
    border-top: 1px solid var(--border);
    padding: 2rem;
  }

  .footer-inner {
    max-width: 1400px;
    margin: 0 auto;
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .footer-wordmark {
    font-family: var(--font-mono);
    font-size: 0.72rem;
    font-weight: 700;
    letter-spacing: 0.14em;
    color: var(--cyan);
  }

  .footer-sub {
    font-size: 0.7rem;
    color: var(--muted);
    font-family: var(--font-mono);
    margin-top: 0.25rem;
  }

  .footer-note {
    font-family: var(--font-mono);
    font-size: 0.68rem;
    color: var(--faint);
  }

  /* ── Responsive ───────────────────────────────────────────────────────── */
  @media (max-width: 900px) {
    .scroll-section {
      grid-template-columns: 1fr;
    }
    .viz-col-sticky {
      position: relative;
      height: 60vw;
      min-height: 300px;
      padding: 0 0 1rem 0;
    }
    .narrative-col { padding-right: 0; }
    .three-pillars { grid-template-columns: 1fr; }
    .nav-links { display: none; }
  }
</style>
