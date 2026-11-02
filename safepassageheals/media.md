---
layout: page
title: Share Media | Safe Passage Heals
permalink: /media/
safe_passage: true
hide_title: true
---

<!-- Media Submissions frontend (Luke Sanders, issue #14). Uses Yiming's Safe Passage
     styles (safe_passage: true). No data here: media types, the pending list, and the
     accept/reject result all come from the backend. -->
<main id="main-content" class="media-page section-wrap">
  <p class="eyebrow"><span aria-hidden="true">✳</span> Share with the community</p>
  <h1 class="media-title">Help us tell the story. <em>Share a photo, video, or article.</em></h1>
  <p class="media-intro">Every submission is reviewed by the Safe Passage Heals team before it appears on the site.</p>

  <form id="media-form" class="media-card">
    <div class="media-field">
      <label for="media-title">Title</label>
      <input id="media-title" type="text" placeholder="e.g. Walk a Mile in Her Shoes recap" autocomplete="off">
    </div>
    <div class="media-field">
      <label for="media-type">Type</label>
      <select id="media-type"></select>
    </div>
    <button type="submit" class="button button-primary">Submit for review <span aria-hidden="true">→</span></button>
    <p id="media-message" class="media-message" role="status" aria-live="polite"></p>
  </form>

  <h2 class="media-subtitle">Pending review</h2>
  <ul id="media-pending" class="media-pending"><li>Loading…</li></ul>
</main>

<style>
  .media-page { max-width: 760px; padding: 40px 0 72px; }
  .media-title { font-family: var(--serif); font-weight: 500; font-size: clamp(30px, 4vw, 42px);
                 letter-spacing: -1px; line-height: 1.15; margin: 0 0 12px; }
  .media-title em { color: var(--purple); }
  .media-intro { color: var(--muted); margin: 0 0 28px; }
  .media-card { background: #fff; border: 1px solid var(--line); border-radius: 22px; padding: 28px 32px;
                box-shadow: 0 12px 30px rgba(104, 68, 161, 0.08); }
  .media-field { display: flex; flex-direction: column; gap: 6px; margin-bottom: 18px; }
  .media-field label { font-size: 12px; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; color: var(--muted); }
  .media-field input, .media-field select { padding: 12px 14px; border: 1px solid var(--line); border-radius: 12px;
                background: var(--pale); color: var(--ink); font: inherit; font-size: 15px; }
  .media-card .button { border: none; cursor: pointer; }
  .media-message { margin: 14px 0 0; font-size: 14px; font-weight: 600; min-height: 1.4em; }
  .media-message.ok { color: #2e7d4f; }
  .media-message.bad { color: #b3261e; }
  .media-subtitle { font-family: var(--serif); font-weight: 500; font-size: 24px; margin: 40px 0 12px; }
  .media-pending { list-style: none; padding: 0; margin: 0; }
  .media-pending li { display: flex; justify-content: space-between; gap: 12px; padding: 12px 16px; margin: 8px 0;
                      background: var(--pale); border: 1px solid var(--line); border-radius: 12px; }
  .media-tag { font-size: 11px; font-weight: 700; letter-spacing: 1px; text-transform: uppercase; color: var(--purple); }
</style>

<script type="module">
  import { pythonURI, fetchOptions } from '{{ site.baseurl }}/assets/js/api/config.js';

  const API = `${pythonURI}/api/media/submissions`;
  const form = document.getElementById('media-form');
  const titleInput = document.getElementById('media-title');
  const typeSelect = document.getElementById('media-type');
  const message = document.getElementById('media-message');
  const pendingList = document.getElementById('media-pending');

  // Small helper: make an element with text
  function el(tag, text, className) {
    const node = document.createElement(tag);
    if (text) node.textContent = text;
    if (className) node.className = className;
    return node;
  }

  function showPending(items) {
    if (items.length === 0) {
      pendingList.replaceChildren(el('li', 'Nothing waiting for review.'));
      return;
    }
    pendingList.replaceChildren(...items.map(item => {
      const li = el('li');
      li.append(el('span', item.title), el('span', item.media_type, 'media-tag'));
      return li;
    }));
  }

  // 1. GET allowed types (for the dropdown) and the pending list
  async function load() {
    try {
      const res = await fetch(API, fetchOptions);
      if (!res.ok) throw new Error(res.status);
      const data = await res.json();
      typeSelect.replaceChildren(...data.allowed_types.map(t => {
        const opt = el('option', t);
        opt.value = t;
        return opt;
      }));
      showPending(data.pending);
    } catch (err) {
      pendingList.replaceChildren(el('li', `Could not reach the backend (${err.message}). Is Flask running on ${pythonURI}?`));
    }
  }

  // 2. POST a submission; the backend runs validateSubmission and says why if it's rejected
  form.addEventListener('submit', async (e) => {
    e.preventDefault();
    const res = await fetch(API, {
      ...fetchOptions,
      method: 'POST',
      body: JSON.stringify({ title: titleInput.value, media_type: typeSelect.value }),
    });
    const data = await res.json();
    message.textContent = data.result;
    message.className = 'media-message ' + (data.accepted ? 'ok' : 'bad');
    if (data.accepted) {
      titleInput.value = '';
      load();
    }
  });

  load();
</script>
