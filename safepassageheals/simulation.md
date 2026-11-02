---
layout: page
title: Support Simulation | Safe Passage Heals
permalink: /simulation/
safe_passage: true
hide_title: true
---

<!-- Support Simulation frontend (issue #10). Uses Yiming's Safe Passage styles
     (safe_passage: true). No data here: questions, hotline, and results come
     from the backend. -->
<main id="main-content" class="sim-page section-wrap">
  <div class="sim-top">
    <p id="sim-hotline" class="sim-hotline"></p>
    <a id="sim-exit" class="sim-exit" href="https://www.google.com" title="Leave this site now (or press Esc)">Quick exit</a>
  </div>

  <p class="eyebrow"><span aria-hidden="true">✳</span> Support simulation</p>
  <h1 class="sim-title">A few everyday questions. <em>No right or wrong answers.</em></h1>
  <p id="sim-progress" class="sim-progress"></p>

  <div id="sim-card" class="sim-card"><p>Loading…</p></div>
</main>

<style>
  .sim-page { max-width: 760px; padding: 40px 0 72px; }
  .sim-top { display: flex; align-items: flex-start; justify-content: space-between; gap: 16px;
             padding: 12px 16px; margin-bottom: 32px; background: var(--pale); border: 1px solid var(--line); border-radius: 14px; }
  .sim-hotline { margin: 0; font-size: 13px; color: var(--muted); }
  .sim-exit { position: relative; z-index: 10; flex-shrink: 0; padding: 8px 16px; border-radius: 999px;
              background: #b3261e; color: #fff; font-size: 12px; font-weight: 700; text-decoration: none; }
  .sim-title { font-family: var(--serif); font-weight: 500; font-size: clamp(30px, 4vw, 42px);
               letter-spacing: -1px; line-height: 1.15; margin: 0 0 12px; }
  .sim-title em { color: var(--purple); }
  .sim-progress { font-size: 12px; font-weight: 700; letter-spacing: 1.5px; text-transform: uppercase; color: var(--muted); }
  .sim-card { background: #fff; border: 1px solid var(--line); border-radius: 22px; padding: 28px 32px;
              box-shadow: 0 12px 30px rgba(104, 68, 161, 0.08); }
  .sim-card h2 { font-family: var(--serif); font-weight: 500; font-size: 24px; line-height: 1.3; margin: 0 0 18px; }
  .sim-choice { display: block; width: 100%; text-align: left; margin: 10px 0; padding: 14px 18px;
                border: 1px solid var(--line); border-radius: 14px; background: var(--pale);
                color: var(--ink); font: inherit; font-size: 15px; cursor: pointer; transition: background .2s, border-color .2s; }
  .sim-choice:hover { background: var(--lavender); border-color: #bca9d1; }
  .sim-actions { display: flex; flex-wrap: wrap; align-items: center; gap: 16px; margin-top: 20px; }
  .sim-restart { background: none; border: none; color: var(--purple); font: inherit; font-size: 13px;
                 font-weight: 700; text-decoration: underline; cursor: pointer; }
</style>

<script type="module">
  import { pythonURI, fetchOptions } from '{{ site.baseurl }}/assets/js/api/config.js';

  const API = `${pythonURI}/api/simulation`;
  const card = document.getElementById('sim-card');
  const progress = document.getElementById('sim-progress');
  let scenarios = [];   // filled from GET /scenarios
  let picked = [];      // choice ids the person picks, in order

  // Quick exit: replace() so the Back button doesn't return here.
  // The button is a real link, so it still leaves even if this script fails.
  function quickExit(e) {
    if (e) e.preventDefault();
    window.location.replace('https://www.google.com');
  }
  document.getElementById('sim-exit').addEventListener('click', quickExit);
  // "true" = capture phase: runs before other site scripts can stop the key
  window.addEventListener('keydown', e => { if (e.key === 'Escape') quickExit(e); }, true);

  // Small helper: make an element with text
  function el(tag, text, className) {
    const node = document.createElement(tag);
    if (text) node.textContent = text;
    if (className) node.className = className;
    return node;
  }

  function showHotline(h) {
    document.getElementById('sim-hotline').textContent =
      `Need to talk now? ${h.name}: ${h.phone} · ${h.text}`;
  }

  // 1. GET questions + hotline
  async function load() {
    try {
      const res = await fetch(`${API}/scenarios`, fetchOptions);
      if (!res.ok) throw new Error(res.status);
      const data = await res.json();
      scenarios = data.scenarios;
      showHotline(data.hotline);
      showQuestion(0);
    } catch (err) {
      card.replaceChildren(el('p', `Could not reach the backend (${err.message}). Is Flask running on ${pythonURI}?`));
    }
  }

  // 2. Show one question; clicking a choice moves to the next
  function showQuestion(i) {
    if (i >= scenarios.length) return submit();
    const s = scenarios[i];
    progress.textContent = `Question ${i + 1} of ${scenarios.length}`;
    const buttons = s.choices.map(c => {
      const b = el('button', c.text, 'sim-choice');
      b.type = 'button';
      b.addEventListener('click', () => { picked.push(c.id); showQuestion(i + 1); });
      return b;
    });
    card.replaceChildren(el('h2', s.prompt), ...buttons);
  }

  // 3. POST picks, show where to go next
  async function submit() {
    progress.textContent = 'Your next step';
    const res = await fetch(`${API}/result`, {
      ...fetchOptions,
      method: 'POST',
      body: JSON.stringify({ choice_ids: picked }),
    });
    const r = await res.json();
    const link = el('a', r.link_label + ' →', 'button button-primary');
    link.href = r.link_url;
    link.target = '_blank';
    link.rel = 'noopener noreferrer';
    const again = el('button', 'Start over', 'sim-restart');
    again.type = 'button';
    again.addEventListener('click', () => { picked = []; showQuestion(0); });
    const actions = el('div', '', 'sim-actions');
    actions.append(link, again);
    card.replaceChildren(el('h2', r.title), el('p', r.message), actions);
  }

  load();
</script>
