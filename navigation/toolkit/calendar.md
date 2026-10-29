---
title: Community Calendar
permalink: /student/calendar
layout: page
active_tab: calendar
safe_passage: true
safe_passage_calendar: true
hide_title: true
---
<section class="safe-passage-calendar section-wrap" aria-labelledby="calendar-page-title">
  <header class="safe-passage-calendar-heading">
    <p class="eyebrow">A place to come together</p>
    <h1 id="calendar-page-title">Community <em>calendar</em></h1>
    <p>Find programs, support, and ways to connect with Safe Passage Heals.</p>
  </header>

  <div class="community-calendar-tools" aria-label="Calendar filters">
    <label class="community-calendar-search">
      <input id="communityEventSearch" type="search" aria-label="Search community events" placeholder="Search events" autocomplete="off">
    </label>
    <div class="community-calendar-filters" role="group" aria-label="Filter events by type">
      <button class="community-filter is-active" type="button" data-category="all" aria-pressed="true">All events</button>
      <button class="community-filter" type="button" data-category="programs" aria-pressed="false">Programs</button>
      <button class="community-filter" type="button" data-category="support" aria-pressed="false">Support</button>
      <button class="community-filter" type="button" data-category="community" aria-pressed="false">Community</button>
      <button class="community-filter" type="button" data-category="fundraising" aria-pressed="false">Fundraising</button>
      <button class="community-filter" type="button" data-category="saved" aria-pressed="false">Saved <span id="savedEventCount">0</span></button>
    </div>
  </div>

  <p id="communityCalendarStatus" class="community-calendar-status" role="status" aria-live="polite">Loading community events…</p>
  <div id="communityCalendar" aria-label="Community events calendar"></div>

  <p class="community-calendar-note">Select an event to see its details. Saved events stay in this browser.</p>
</section>

<div id="communityEventModal" class="community-event-backdrop" role="presentation" hidden>
  <section class="community-event-dialog" role="dialog" aria-modal="true" aria-labelledby="communityEventTitle" aria-describedby="communityEventDescription">
    <button id="closeCommunityEvent" class="community-event-close" type="button" aria-label="Close event details">&times;</button>
    <p id="communityEventCategory" class="eyebrow">Community event</p>
    <h2 id="communityEventTitle"></h2>
    <p id="communityEventDate" class="community-event-date"></p>
    <p id="communityEventDescription"></p>
    <div class="community-event-actions">
      <button id="saveCommunityEvent" type="button" class="community-event-primary">Save event</button>
      <button id="downloadCommunityEvent" type="button" class="community-event-secondary">Add to calendar</button>
    </div>
  </section>
</div>

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.0/main.min.css">
<script src="https://cdn.jsdelivr.net/npm/fullcalendar@5.11.0/main.min.js"></script>
<script type="module">
  import { javaURI, fetchOptions } from '{{ site.baseurl }}/assets/js/api/config.js';

  const calendarRoot = document.getElementById('communityCalendar');
  const statusEl = document.getElementById('communityCalendarStatus');
  const searchEl = document.getElementById('communityEventSearch');
  const modal = document.getElementById('communityEventModal');
  const savedKey = 'safePassageSavedCommunityEvents';
  let allEvents = [];
  let selectedCategory = 'all';
  let calendar;
  let activeEvent;

  function getSavedEvents() {
    try { return new Set(JSON.parse(localStorage.getItem(savedKey) || '[]')); }
    catch (_) { return new Set(); }
  }

  function setSavedEvents(saved) {
    localStorage.setItem(savedKey, JSON.stringify([...saved]));
    document.getElementById('savedEventCount').textContent = saved.size;
  }

  function categoryFor(event) {
    const explicit = String(event.category || event.type || '').toLowerCase();
    if (/program/.test(explicit)) return 'programs';
    if (/support|counsel|healing|group/.test(explicit)) return 'support';
    if (/fundrais|donat|walk|gala/.test(explicit)) return 'fundraising';
    if (/community|event|outreach/.test(explicit)) return 'community';
    return 'community';
  }

  function isLegacySchoolEvent(event) {
    const schoolClass = String(event.class || event.period || '').toUpperCase();
    return ['CSA', 'CSP', 'CSSE'].includes(schoolClass) || /\b(?:CSA|CSP|CSSE)\s*$/i.test(String(event.title || ''));
  }

  function visibleEvents() {
    const query = searchEl.value.trim().toLowerCase();
    const saved = getSavedEvents();
    return allEvents.filter(event => {
      const matchesCategory = selectedCategory === 'all' ||
        (selectedCategory === 'saved' ? saved.has(event.id) : event.category === selectedCategory);
      const matchesQuery = !query || `${event.title} ${event.description} ${event.category}`.toLowerCase().includes(query);
      return matchesCategory && matchesQuery;
    });
  }

  function refreshEvents() {
    if (!calendar) return;
    calendar.removeAllEvents();
    calendar.addEventSource(visibleEvents());
    const shown = visibleEvents().length;
    statusEl.textContent = shown ? `${shown} ${shown === 1 ? 'event' : 'events'} shown` : 'No events match these filters yet.';
  }

  function displayDate(date) {
    return new Intl.DateTimeFormat('en-US', {
      weekday: 'long', month: 'long', day: 'numeric', year: 'numeric',
      hour: 'numeric', minute: '2-digit'
    }).format(date);
  }

  function openEvent(event) {
    activeEvent = event;
    document.getElementById('communityEventCategory').textContent = event.extendedProps.categoryLabel;
    document.getElementById('communityEventTitle').textContent = event.title;
    document.getElementById('communityEventDate').textContent = displayDate(event.start);
    document.getElementById('communityEventDescription').textContent = event.extendedProps.description || 'More details will be shared soon.';
    const saved = getSavedEvents().has(event.id);
    const saveButton = document.getElementById('saveCommunityEvent');
    saveButton.textContent = saved ? 'Remove saved event' : 'Save event';
    modal.hidden = false;
    document.getElementById('closeCommunityEvent').focus();
  }

  function closeEvent() {
    modal.hidden = true;
    activeEvent = null;
  }

  function downloadEvent(event) {
    const toICS = date => date.toISOString().replace(/[-:]/g, '').replace(/\.\d{3}/, '');
    const escapeICS = value => String(value || '').replace(/\\/g, '\\\\').replace(/\n/g, '\\n').replace(/,/g, '\\,').replace(/;/g, '\\;');
    const body = [
      'BEGIN:VCALENDAR', 'VERSION:2.0', 'PRODID:-//Safe Passage Heals//Community Calendar//EN',
      'BEGIN:VEVENT', `UID:${escapeICS(event.id)}@safepassageheals.org`, `DTSTAMP:${toICS(new Date())}`,
      `DTSTART:${toICS(event.start)}`, `SUMMARY:${escapeICS(event.title)}`,
      `DESCRIPTION:${escapeICS(event.extendedProps.description)}`, 'END:VEVENT', 'END:VCALENDAR'
    ].join('\r\n');
    const link = document.createElement('a');
    link.href = URL.createObjectURL(new Blob([body], { type: 'text/calendar;charset=utf-8' }));
    link.download = `${event.title.toLowerCase().replace(/[^a-z0-9]+/g, '-') || 'community-event'}.ics`;
    link.click();
    setTimeout(() => URL.revokeObjectURL(link.href), 1000);
  }

  calendar = new FullCalendar.Calendar(calendarRoot, {
    initialView: 'dayGridMonth',
    height: 'auto',
    nowIndicator: true,
    dayMaxEvents: true,
    navLinks: true,
    selectable: false,
    headerToolbar: { left: 'prev,next today', center: 'title', right: 'dayGridMonth,timeGridWeek,listMonth' },
    views: { dayGridMonth: { buttonText: 'Month' }, timeGridWeek: { buttonText: 'Week' }, listMonth: { buttonText: 'Agenda' } },
    events: [],
    eventClick(info) { openEvent(info.event); },
    eventDidMount(info) { info.el.title = info.event.extendedProps.description || info.event.title; }
  });
  calendar.render();
  setSavedEvents(getSavedEvents());

  document.querySelectorAll('.community-filter').forEach(button => {
    button.addEventListener('click', () => {
      selectedCategory = button.dataset.category;
      document.querySelectorAll('.community-filter').forEach(filter => {
        const active = filter === button;
        filter.classList.toggle('is-active', active);
        filter.setAttribute('aria-pressed', String(active));
      });
      refreshEvents();
    });
  });
  searchEl.addEventListener('input', refreshEvents);
  document.getElementById('closeCommunityEvent').addEventListener('click', closeEvent);
  modal.addEventListener('click', event => { if (event.target === modal) closeEvent(); });
  document.addEventListener('keydown', event => { if (event.key === 'Escape' && !modal.hidden) closeEvent(); });
  document.getElementById('saveCommunityEvent').addEventListener('click', () => {
    if (!activeEvent) return;
    const saved = getSavedEvents();
    if (saved.has(activeEvent.id)) saved.delete(activeEvent.id); else saved.add(activeEvent.id);
    setSavedEvents(saved);
    document.getElementById('saveCommunityEvent').textContent = saved.has(activeEvent.id) ? 'Remove saved event' : 'Save event';
    refreshEvents();
  });
  document.getElementById('downloadCommunityEvent').addEventListener('click', () => {
    if (activeEvent) downloadEvent(activeEvent);
  });

  fetch(`${javaURI}/api/calendar/events`, fetchOptions)
    .then(response => {
      if (!response.ok) throw new Error(`Calendar service returned ${response.status}`);
      return response.json();
    })
    .then(events => {
      allEvents = (Array.isArray(events) ? events : [])
        .filter(event => event && event.title && event.date && !isLegacySchoolEvent(event))
        .map(event => {
          const category = categoryFor(event);
          return {
            id: String(event.id || `${event.title}-${event.date}`),
            title: event.title,
            start: event.date,
            color: '#8b6aa8',
            extendedProps: {
              description: event.description || '',
              category,
              categoryLabel: category[0].toUpperCase() + category.slice(1)
            }
          };
        });
      refreshEvents();
    })
    .catch(error => {
      console.error('Could not load community calendar:', error);
      statusEl.textContent = 'Community events are temporarily unavailable. Please check back soon.';
    });
</script>
