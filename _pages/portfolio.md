---
layout: archive
permalink: /portfolio/
title: "Projects"
author_profile: true
---

The same work, listed two ways: by the research question it asks, and by the grant or contract that paid for it.

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

<p class="proj-section-desc">Explore projects and their related papers. <strong>Theme</strong> and <strong>Modality</strong> organize each project under its primary focus, once per view. Use <strong>Filter</strong> to explore shared keywords across projects. Funding is listed below; all papers are on the <a href="/publications/">Publications</a> page.</p>

<div class="proj-grid" id="research-grid">
{%- for item in site.data.projects.research %}
{% include project-card.html card=item grouped=true %}
{%- endfor %}
</div>

<div class="proj-grouped" id="research-grouped" hidden></div>

<div class="proj-filter" id="research-filter" hidden>
<p class="proj-filter__hint">Explore by keyword. Select alternatives within a row; combine rows to narrow your results.</p>
<div class="proj-filter__bar" role="group" aria-label="Filter projects by keyword"></div>
<p class="proj-filter__status"><span role="status" aria-live="polite" aria-atomic="true"></span></p>
<div class="proj-grid proj-filter__grid"></div>
</div>

<h2 id="funded-projects">Funded Projects</h2>

<p class="proj-section-desc">The industry and government grants behind the work, newest first, with my own role in each.</p>

<div class="proj-grid">
{%- for item in site.data.projects.funded %}
{% include project-card.html card=item %}
{%- endfor %}
</div>

<script>
(function () {
  var page = document.querySelector('.proj-page');
  if (!page) return;

  /* Move the same cards between views: one DOM node per research project.
     The first valid group key is primary; tags preserve cross-cutting topics. */
  var VIEWS = {{ site.data.projects.views | jsonify }};
  var FILTERS = {{ site.data.projects.filters | jsonify }};

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

  function buildView(view) {
    var attr = 'data-' + view.key;
    var wrap = document.createElement('div');
    wrap.className = 'proj-view';
    var groups = (view.groups || []).slice();
    var known = {};
    groups.forEach(function (g) { known[g.key] = true; });
    groups.push({ key: '__other__', title: 'Other' });

    function primaryGroup(c) {
      var valid = valuesOf(c, attr).filter(function (v) { return known[v]; });
      return valid[0] || '__other__';
    }

    groups.forEach(function (g) {
      var members = fileOrder.filter(function (c) {
        return primaryGroup(c) === g.key;
      });
      if (!members.length) return;
      var s = section(g.title, g.desc, members.length);
      members.forEach(function (c) { c.hidden = false; s.grid.appendChild(c); });
      wrap.appendChild(s.el);
    });
    grouped.textContent = '';
    grouped.appendChild(wrap);
  }

  function showLatest() {
    fileOrder.slice().sort(function (a, b) {
      var ia = +a.getAttribute('data-index'), ib = +b.getAttribute('data-index');
      return (sortYear[ib] - sortYear[ia]) || (ia - ib);
    }).forEach(function (c) { c.hidden = false; grid.appendChild(c); });
    grid.hidden = false;
    grouped.hidden = true;
    filterWrap.hidden = true;
  }

  function showGrouped(view) {
    buildView(view);
    grid.hidden = true;
    grouped.hidden = false;
    filterWrap.hidden = true;
  }

  /* ---- Filter view -----------------------------------------------------
     OR within each facet, AND across facets. Counts preview the number of
     results after toggling a keyword, including alternatives in one row. */
  var filterBar = filterWrap.querySelector('.proj-filter__bar');
  var filterStatus = filterWrap.querySelector('.proj-filter__status');
  var filterGrid = filterWrap.querySelector('.proj-filter__grid');
  var picked = [];
  var filterReady = false;
  var facetOf = {};

  FILTERS.forEach(function (f, i) {
    f.tags.forEach(function (t) { facetOf[t] = 'facet-' + i; });
  });

  function matches(card, tags) {
    var has = tagsOf(card);
    var facets = {};
    tags.forEach(function (t) {
      var facet = facetOf[t] || 'other';
      if (!facets[facet]) facets[facet] = [];
      facets[facet].push(t);
    });
    return Object.keys(facets).every(function (facet) {
      return facets[facet].some(function (t) { return has.indexOf(t) !== -1; });
    });
  }

  function buildFilter() {
    var counts = {};
    fileOrder.forEach(function (c) {
      tagsOf(c).forEach(function (t) { counts[t] = (counts[t] || 0) + 1; });
    });

    function chip(t) {
      var b = document.createElement('button');
      b.type = 'button';
      b.className = 'proj-filter__tag';
      b.setAttribute('data-tag', t);
      b.setAttribute('aria-pressed', 'false');
      var hash = document.createElement('span');
      hash.className = 'proj-filter__hash';
      hash.textContent = '#';
      hash.setAttribute('aria-hidden', 'true');
      b.appendChild(hash);
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
      return b;
    }

    /* One row per facet, in the order _data/projects.yml lists them. The
       label and its chips are separate grid cells, so the label column
       lines up across rows without a hand-set width. */
    function row(name, tags) {
      if (!tags.length) return;
      var label = document.createElement('span');
      label.className = 'proj-filter__label';
      label.id = 'proj-facet-' + filterBar.childElementCount;
      label.textContent = name;
      var chips = document.createElement('span');
      chips.className = 'proj-filter__chips';
      chips.setAttribute('role', 'group');
      chips.setAttribute('aria-labelledby', label.id);
      tags.forEach(function (t) { chips.appendChild(chip(t)); });
      filterBar.appendChild(label);
      filterBar.appendChild(chips);
    }

    var placed = {};
    (FILTERS || []).forEach(function (f) {
      var tags = (f.tags || []).filter(function (t) { return counts[t] && !placed[t]; });
      tags.forEach(function (t) { placed[t] = true; });
      row(f.label, tags);
    });
    row('Other', Object.keys(counts).filter(function (t) { return !placed[t]; }).sort());

    var clear = document.createElement('button');
    clear.type = 'button';
    clear.className = 'proj-filter__clear';
    clear.textContent = 'Clear filters';
    clear.addEventListener('click', function () { picked = []; applyFilter(); });
    filterStatus.appendChild(clear);

    filterReady = true;
  }

  function applyFilter() {
    var shown = 0;
    Array.prototype.forEach.call(filterGrid.children, function (c) {
      var ok = matches(c, picked);
      c.hidden = !ok;
      if (ok) shown++;
    });

    Array.prototype.forEach.call(filterBar.querySelectorAll('.proj-filter__tag'), function (b) {
      var t = b.getAttribute('data-tag');
      var on = picked.indexOf(t) !== -1;
      b.classList.toggle('is-on', on);
      b.setAttribute('aria-pressed', on ? 'true' : 'false');
      var next = on ? picked.filter(function (tag) { return tag !== t; }) : picked.concat([t]);
      var count = fileOrder.filter(function (c) { return matches(c, next); }).length;
      b.querySelector('.proj-filter__count').textContent = count;
      b.querySelector('.proj-filter__count').setAttribute('aria-hidden', 'true');
      var action = (on ? 'Remove ' : 'Add ') + t + ': ' + count + (count === 1 ? ' project' : ' projects');
      b.setAttribute('aria-label', action);
      b.title = action;
      b.disabled = !on && count === 0;
      b.classList.toggle('is-empty', b.disabled);
    });

    filterStatus.firstChild.textContent = picked.length
      ? shown + ' of ' + fileOrder.length + ' projects'
      : 'All ' + fileOrder.length + ' projects · Choose keywords to explore';
    filterStatus.lastChild.hidden = !picked.length;
  }

  function showFilter() {
    if (!filterReady) buildFilter();
    fileOrder.forEach(function (c) { filterGrid.appendChild(c); });
    applyFilter();
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
