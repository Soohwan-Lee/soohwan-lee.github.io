---
layout: archive
permalink: /portfolio/
title: "Projects"
author_profile: true
---

Two kinds of work: research I set the question for, and commissioned R&D I contribute to. Where a grant supports either one, it is credited on the card.

<!-- All project content lives in _data/projects.yml (instructions at the top of
     that file). Each card is rendered by _includes/project-card.html. -->

<div class="proj-page">

<div class="proj-section-head">
<h2 id="research-projects">Research Projects</h2>
<div class="pub-toggle proj-toggle" role="tablist" aria-label="Project view">
<span class="pub-toggle__slider" aria-hidden="true"></span>
<button type="button" role="tab" class="pub-toggle__btn is-active" data-view="latest" aria-selected="true">Latest</button>
{%- for v in site.data.projects.views %}
<button type="button" role="tab" class="pub-toggle__btn" data-view="{{ v.key }}" aria-selected="false">{{ v.label }}</button>
{%- endfor %}
<button type="button" role="tab" class="pub-toggle__btn" data-view="filter" aria-selected="false">Filter</button>
</div>
</div>

<p class="proj-section-desc">Work I drive myself, from research question to study design to a running system. Group by <strong>Theme</strong> or <strong>Modality</strong> to see how the lines connect; a project can sit in more than one. Papers are listed on the <a href="/publications/">Publications</a> page.</p>

<div class="proj-grid" id="research-grid">
{%- for item in site.data.projects.research %}
{% include project-card.html card=item grouped=true %}
{%- endfor %}
</div>

<div class="proj-grouped" id="research-grouped" hidden></div>

<div class="proj-filter" id="research-filter" hidden>
<div class="proj-filter__bar" role="group" aria-label="Filter projects by topic"></div>
<p class="proj-filter__status"><span></span></p>
<div class="proj-grid proj-filter__grid"></div>
</div>

<h2 id="funded-projects">Funded Projects</h2>

<p class="proj-section-desc">Industry- and government-commissioned R&amp;D, newest first, with my own role in each.</p>

<div class="proj-grid">
{%- for item in site.data.projects.funded %}
{% include project-card.html card=item %}
{%- endfor %}
</div>

<script>
(function () {
  var page = document.querySelector('.proj-page');
  if (!page) return;

  /* Grouped views come from _data/projects.yml -> views.
     Each view reads the card attribute data-<key>, which holds one or more
     space-separated group keys, and shows its groups in order. A card listed
     in several groups is cloned into each; a card matching none goes to an
     "Other" section. The cards in #research-grid are never moved, so the
     "Latest" list stays intact. */
  var VIEWS = {{ site.data.projects.views | jsonify }};

  var grid = document.getElementById('research-grid');
  var grouped = document.getElementById('research-grouped');
  var filterWrap = document.getElementById('research-filter');
  var toggle = page.querySelector('.proj-toggle');
  if (!grid || !grouped || !filterWrap || !toggle) return;

  var fileOrder = Array.prototype.slice.call(grid.querySelectorAll('.proj-card'));

  /* Sort key for "Latest": data-year, or the previous card's year when a
     card has none, so cards without a year stay where they are in the file. */
  var sortYear = [];
  var carry = 9999;
  fileOrder.forEach(function (c, i) {
    var y = parseInt(c.getAttribute('data-year'), 10);
    if (!isNaN(y)) carry = y;
    sortYear[i] = carry;
    c.setAttribute('data-index', i);
  });

  function valuesOf(card, attr) {
    return (card.getAttribute(attr) || '').split(/\s+/).filter(Boolean);
  }

  function tagsOf(card) {
    return (card.getAttribute('data-tags') || '').split('|').filter(Boolean);
  }

  function section(title, desc, count) {
    var s = document.createElement('section');
    s.className = 'proj-theme';
    var head = document.createElement('div');
    head.className = 'proj-theme__head';
    var h = document.createElement('h3');
    h.className = 'proj-theme__title';
    h.textContent = title + ' ';
    var n = document.createElement('span');
    n.className = 'proj-theme__count';
    n.textContent = count;
    h.appendChild(n);
    head.appendChild(h);
    if (desc) {
      var p = document.createElement('p');
      p.className = 'proj-theme__desc';
      p.textContent = desc;
      head.appendChild(p);
    }
    var g = document.createElement('div');
    g.className = 'proj-grid';
    s.appendChild(head);
    s.appendChild(g);
    return { el: s, grid: g };
  }

  /* Build each view's sections once (lazily), from clones of the cards. */
  var built = {};
  function buildView(view) {
    var attr = 'data-' + view.key;
    var wrap = document.createElement('div');
    wrap.className = 'proj-view';
    var groups = (view.groups || []).slice();
    var known = {};
    groups.forEach(function (g) { known[g.key] = true; });
    groups.push({ key: '__other__', title: 'Other' });

    groups.forEach(function (g) {
      var members = fileOrder.filter(function (c) {
        var vals = valuesOf(c, attr);
        if (g.key === '__other__') {
          return !vals.some(function (v) { return known[v]; });
        }
        return vals.indexOf(g.key) !== -1;
      });
      if (!members.length) return;
      var s = section(g.title, g.desc, members.length);
      members.forEach(function (c) { s.grid.appendChild(c.cloneNode(true)); });
      wrap.appendChild(s.el);
    });
    grouped.appendChild(wrap);
    built[view.key] = wrap;
    return wrap;
  }

  function hideAllViews(except) {
    Object.keys(built).forEach(function (k) { built[k].hidden = (k !== except); });
  }

  function showLatest() {
    fileOrder.slice().sort(function (a, b) {
      var ia = +a.getAttribute('data-index'), ib = +b.getAttribute('data-index');
      return (sortYear[ib] - sortYear[ia]) || (ia - ib);
    }).forEach(function (c) { grid.appendChild(c); });
    hideAllViews(null);
    grid.hidden = false;
    grouped.hidden = true;
    filterWrap.hidden = true;
  }

  function showGrouped(view) {
    if (!built[view.key]) buildView(view);
    hideAllViews(view.key);
    grid.hidden = true;
    grouped.hidden = false;
    filterWrap.hidden = true;
  }

  /* ---- Filter view -----------------------------------------------------
     A chip per topic tag. Picking several narrows to the cards carrying all
     of them, and a chip that would leave nothing is dimmed out. */
  var filterBar = filterWrap.querySelector('.proj-filter__bar');
  var filterStatus = filterWrap.querySelector('.proj-filter__status');
  var filterGrid = filterWrap.querySelector('.proj-filter__grid');
  var picked = [];
  var filterReady = false;

  function matches(card, tags) {
    var has = tagsOf(card);
    return tags.every(function (t) { return has.indexOf(t) !== -1; });
  }

  function buildFilter() {
    var counts = {};
    fileOrder.forEach(function (c) {
      tagsOf(c).forEach(function (t) { counts[t] = (counts[t] || 0) + 1; });
    });

    Object.keys(counts)
      .sort(function (a, b) { return (counts[b] - counts[a]) || a.localeCompare(b); })
      .forEach(function (t) {
        var b = document.createElement('button');
        b.type = 'button';
        b.className = 'proj-filter__tag';
        b.setAttribute('data-tag', t);
        b.setAttribute('aria-pressed', 'false');
        b.appendChild(document.createTextNode(t));
        var n = document.createElement('span');
        n.className = 'proj-filter__count';
        n.textContent = counts[t];
        b.appendChild(n);
        b.addEventListener('click', function () {
          var i = picked.indexOf(t);
          if (i === -1) picked.push(t); else picked.splice(i, 1);
          applyFilter();
        });
        filterBar.appendChild(b);
      });

    var clear = document.createElement('button');
    clear.type = 'button';
    clear.className = 'proj-filter__clear';
    clear.textContent = 'Clear';
    clear.addEventListener('click', function () { picked = []; applyFilter(); });
    filterStatus.appendChild(clear);

    fileOrder.forEach(function (c) { filterGrid.appendChild(c.cloneNode(true)); });
    filterReady = true;
  }

  function applyFilter() {
    var shown = 0;
    Array.prototype.forEach.call(filterGrid.children, function (c) {
      var ok = matches(c, picked);
      c.hidden = !ok;
      if (ok) shown++;
    });

    Array.prototype.forEach.call(filterBar.children, function (b) {
      var t = b.getAttribute('data-tag');
      var on = picked.indexOf(t) !== -1;
      b.classList.toggle('is-on', on);
      b.setAttribute('aria-pressed', on ? 'true' : 'false');
      b.classList.toggle('is-empty', !on && !fileOrder.some(function (c) {
        return matches(c, picked.concat([t]));
      }));
    });

    filterStatus.firstChild.textContent = picked.length
      ? shown + ' of ' + fileOrder.length + ' projects'
      : 'Pick one or more topics to narrow ' + fileOrder.length + ' projects.';
    filterStatus.lastChild.hidden = !picked.length;
  }

  function showFilter() {
    if (!filterReady) buildFilter();
    applyFilter();
    hideAllViews(null);
    grid.hidden = true;
    grouped.hidden = true;
    filterWrap.hidden = false;
  }

  var buttons = toggle.querySelectorAll('.pub-toggle__btn');
  var slider = toggle.querySelector('.pub-toggle__slider');

  function moveSlider() {
    var active = toggle.querySelector('.pub-toggle__btn.is-active');
    if (!active || !slider) return;
    slider.style.width = active.offsetWidth + 'px';
    slider.style.transform = 'translateX(' + (active.offsetLeft - slider.offsetLeft) + 'px)';
  }

  function findView(key) {
    for (var i = 0; i < VIEWS.length; i++) if (VIEWS[i].key === key) return VIEWS[i];
    return null;
  }

  function show(key) {
    var view = key === 'filter' ? null : findView(key);
    if (key !== 'filter' && !view) key = 'latest';
    if (key === 'filter') showFilter();
    else if (view) showGrouped(view);
    else showLatest();
    Array.prototype.forEach.call(buttons, function (b) {
      var on = b.getAttribute('data-view') === key;
      b.classList.toggle('is-active', on);
      b.setAttribute('aria-selected', on ? 'true' : 'false');
    });
    moveSlider();
    try { localStorage.setItem('projView', key); } catch (e) {}
  }

  Array.prototype.forEach.call(buttons, function (b) {
    b.addEventListener('click', function () { show(b.getAttribute('data-view')); });
  });

  var saved = 'latest';
  try { saved = localStorage.getItem('projView') || 'latest'; } catch (e) {}
  show(saved);
  window.addEventListener('resize', moveSlider);
  if (document.fonts && document.fonts.ready) { document.fonts.ready.then(moveSlider); }
})();
</script>

</div>
