---
toc: false
theme: "light"
---

```js
import * as aq from 'npm:arquero';
import { op, table } from 'npm:arquero';
```


```js
/***************
 * CONTROLS
 ***************/

// CMIP6 scenario radios
const this_experiment = (() => {
  const wrapper = html`<div class="radio-inline"></div>`;
  const labelEl = html`<span class="radio-label">CMIP6 scenario:</span>`;
  const form = html`<form role="radiogroup" aria-label="CMIP6 scenario:">
    <label><input type="radio" name="exp" value="ssp1_2_6" checked> SSP1-2.6 (sustainability)</label>
    <label><input type="radio" name="exp" value="ssp2_4_5"> SSP2-4.5 (middle of the road)</label>
    <label><input type="radio" name="exp" value="ssp3_7_0"> SSP3-7.0 (regional rivalry)</label>
    <label><input type="radio" name="exp" value="ssp5_8_5"> SSP5-8.5 (fossil-fueled development)</label>
  </form>`;
  wrapper.append(labelEl, form);

  const setValue = () => {
    wrapper.value = form.querySelector('input[name="exp"]:checked')?.value;
    wrapper.dispatchEvent(new InputEvent('input', { bubbles: true }));
  };
  form.addEventListener('change', setValue);
  setValue();
  return wrapper;
})();

// Year slider
const this_year = (() => {
  const input = html`<input type="range" min="2015" max="2080" step="1" value="2025">`;
  const out = html`<span class="year-value">${input.value}</span>`;
  const box = html`<label class="year-inline">Year: ${input} ${out}</label>`;
  input.addEventListener("input", () => {
    out.textContent = input.value;
    box.dispatchEvent(new InputEvent("input", { bubbles: true }));
  });
  Object.defineProperty(box, "value", { get: () => +input.value });
  return box;
})();

// Senario radios
const this_scenario = (() => {
  const wrapper = html`<div class="radio-inline"></div>`;
  const labelEl = html`<span class="radio-label">National policy scenario:</span>`;
  const form = html`<form role="radiogroup" aria-label="National policy scenario">
    <label><input type="radio" name="scenario" value="BAU" checked> Business as usual</label>
    <label><input type="radio" name="scenario" value="Stress"> Climate Stress</label>
    <label><input type="radio" name="scenario" value="Sustainable"> Sustainability / Net Zero</label>
  </form>`;

  wrapper.append(labelEl, form);

  const setValue = () => {
    wrapper.value = form.querySelector('input[name="scenario"]:checked')?.value;
    wrapper.dispatchEvent(new InputEvent('input', { bubbles: true }));
  };

  form.addEventListener('change', setValue);
  setValue(); // initialize wrapper.value and emit first input

  return wrapper;
})();


// Helpers
const getYear = () => +this_year.value;
const onInputs = (fn) => {
  this_experiment.addEventListener("input", fn);
  this_year.addEventListener("input", fn);
  invalidation.then(() => {
    this_experiment.removeEventListener("input", fn);
    this_year.removeEventListener("input", fn);
  });
};
```


```js
/****************
 * DATA LOADING
 ****************/

// Small fetch helper
async function fetchCSV(url, map = (d) => d) {
  return d3.csv(url, map);
}

const iso3 = await (async () => {
  const txt = await d3.text("https://storage.googleapis.com/finrisk-demo/iso3.csv");
  return aq.fromCSV(txt);
})();

const climate_raw = await (async () => {
  const base = "https://storage.googleapis.com/finrisk-demo/climate.csv.gz";
  const url  = `${base}?v=${Date.now()}`;
  const txt  = await d3.text(url);
  return aq.fromCSV(txt);
})();

const fao_catch = await (async () => {
  const txt = await d3.text("https://storage.googleapis.com/finrisk-demo/catch_for_observable.csv");
  return aq.fromCSV(txt)
    .derive({year: d => d.year + 60})
    .derive({country: d => d.iso3 === 'PRK' ? 'Korea' : d.country});
})();

// Build EEZ select options (only those present in both climate & catch)
const foo = iso3
  .join(
    climate_raw
      .rename({ISO_TER1: 'iso3'})
      .select('iso3')
      .dedupe()
      .join(fao_catch.select('iso3').dedupe(), 'iso3'),
    'iso3'
  );

const eezOptions = [
  {label: "All", value: "ALL"},
  ...foo.objects().map(d => ({label: d.name, value: d.iso3}))
];

// EEZ selector
const this_eez = (() => {
  const row   = html`<div class="control-row"></div>`;
  const label = html`<span class="control-label">EEZ:</span>`;
  const select = html`<select></select>`;
  for (const opt of eezOptions) {
    const o = document.createElement("option");
    o.value = opt.value; o.textContent = opt.label;
    select.appendChild(o);
  }
  select.value = eezOptions[0]?.value ?? "ALL";
  select.addEventListener("input", () => {
    row.dispatchEvent(new InputEvent("input", { bubbles: true }));
  });
  Object.defineProperty(row, "value", {
    get: () => select.value,
    set: v => { select.value = v; select.dispatchEvent(new Event("input", { bubbles:true })); }
  });
  row.append(label, select);
  return row;
})();
```


```js
/****************
 * VARIABLE PICKER + HEATMAP
 ****************/
const this_variable = (() => {
  const wrapper = html`<div class="radio-inline"></div>`;
  const form = html`<form role="radiogroup" aria-label="Variable">
    <label><input type="radio" name="exp" value="sst" checked> Sea surface temperature</label>
    <label><input type="radio" name="exp" value="wind"> Wind speed</label>
    <label><input type="radio" name="exp" value="seaice"> Sea-ice area</label>
    <label><input type="radio" name="exp" value="hdd"> Heating degree days</label>
  </form>`;
  wrapper.append(form);
  const setValue = () => {
    wrapper.value = form.querySelector('input[name="exp"]:checked')?.value;
    wrapper.dispatchEvent(new InputEvent('input', { bubbles: true }));
  };
  form.addEventListener('change', setValue);
  setValue();
  return wrapper;
})();

// Heatmap setup
import { feature } from "npm:topojson-client";
const world = await d3.json("https://cdn.jsdelivr.net/npm/world-atlas@2/countries-110m.json");
const land = feature(world, world.objects.land);
const lat = await fetchCSV("https://storage.googleapis.com/finrisk-demo/lat.csv", d => +d.lat);
const lon = await fetchCSV("https://storage.googleapis.com/finrisk-demo/lon.csv", d => +d.lon);

const aliasOf = { sst: "sst", wind: "wind", seaice: "seaice", hdd: "hdd" };

async function loadGrid(exp, varKey, year) {
  const alias = aliasOf[varKey];
  const url = `https://storage.googleapis.com/finrisk-demo/${exp}/${alias}/${alias}_${year}.csv.gz?v=${year}`;
  const res = await fetch(url, { cache: "no-store" });
  if (!res.ok) throw new Error(`Fetch failed ${res.status} for ${url}`);
  const txt = await res.text();
  return Float32Array.from(txt.trim().split(/\s*,\s*|\n/), Number);
}

const loadGridCached = (() => {
  const cache = new Map();
  return async (exp, varKey, year) => {
    const key = `${exp}/${varKey}/${year}`;
    if (!cache.has(key)) cache.set(key, loadGrid(exp, varKey, year));
    return cache.get(key);
  };
})();

function drawLegend(ctx, scale, x, y, w, h, vmin, vmax) {
  for (let i = 0; i < w; ++i) {
    const t = i / (w - 1);
    const v = vmin + t * (vmax - vmin);
    ctx.fillStyle = scale(v);
    ctx.fillRect(x + i, y, 1, h);
  }
  ctx.fillStyle = "#000";
  ctx.font = "11px system-ui";
  ctx.fillText(`${d3.format(".2f")(vmin)} – ${d3.format(".2f")(vmax)}`, x, y - 4);
}

const heatmap = (async () => {
  const W = 800, H = 440;
  const MARGIN = { top: 10, right: 10, bottom: 10, left: 10 };
  const LEGEND_H = 12, LEGEND_BLOCK = LEGEND_H + 16;
  const canvas = html`<canvas width=${W} height=${H}></canvas>`;
  const ctx = canvas.getContext("2d");
  const proj = d3.geoEqualEarth().fitExtent(
    [[MARGIN.left, MARGIN.top], [W - MARGIN.right, H - MARGIN.bottom - LEGEND_BLOCK]],
    { type: "Sphere" }
  );
  const path = d3.geoPath(proj, ctx);

  let drawId = 0;
  async function render() {
    const id = ++drawId;
    const exp = this_experiment.value;
    const varKey = this_variable.value;
    const yr = getYear();

    const flat = await loadGridCached(exp, varKey, yr);
    if (id !== drawId) return;

    const nlat = lat.length, nlon = lon.length;
    const FIXED_DOMAINS = { sst:[-2,35], wind:[0.48,13.5], seaice:[0,100], hdd:[0,25600] };
    const [vmin, vmax] = FIXED_DOMAINS[varKey] ?? [0,1];
    const color = d3.scaleSequential(d3.interpolateTurbo).domain([vmin, vmax]);

    ctx.clearRect(0, 0, W, H);

    const latPx = new Float32Array(nlat);
    const dxAt = (φdeg) => {
      const a = proj([0, φdeg]), b = proj([1, φdeg]);
      return Math.abs((a && b) ? (b[0] - a[0]) : 1.5);
    };
    for (let i = 0; i < nlat; ++i) latPx[i] = dxAt(lat[i]);

    const dy = (() => {
      const a = proj([0, 0]), b = proj([0, 1]);
      return Math.abs((a && b) ? (b[1] - a[1]) : 1.5);
    })();

    const eps = 0.5;
    let k = 0;
    for (let i = 0; i < nlat; ++i) {
      const φ = lat[i];
      const dx = Math.max(1, latPx[i]);
      for (let j = 0; j < nlon; ++j, ++k) {
        const val = flat[k];
        if (!Number.isFinite(val)) continue;
        const λ = lon[j] > 180 ? ((lon[j] + 180) % 360) - 180 : lon[j];
        const p = proj([λ, φ]); if (!p) continue;
        ctx.fillStyle = color(val);
        ctx.fillRect(p[0] - dx/2 - eps, p[1] - dy/2 - eps, dx + 2*eps, dy + 2*eps);
      }
    }

    ctx.save(); ctx.fillStyle = "#666";
    ctx.beginPath(); path(land); ctx.fill();
    ctx.restore();
    ctx.strokeStyle = "#333"; ctx.lineWidth = 1;
    ctx.beginPath(); path({type:"Sphere"}); ctx.stroke();

    const legendY = H - MARGIN.bottom - LEGEND_H;
    drawLegend(ctx, color, MARGIN.left, legendY, 220, LEGEND_H, vmin, vmax);

    // warm cache for scrubbing
    loadGridCached(exp, varKey, yr - 1);
    loadGridCached(exp, varKey, yr + 1);
  }

  const onInput = () => render();
  this_experiment.addEventListener('input', onInput);
  this_variable.addEventListener('input', onInput);
  this_year.addEventListener('input', onInput);
  invalidation.then(() => {
    this_experiment.removeEventListener('input', onInput);
    this_variable.removeEventListener('input', onInput);
    this_year.removeEventListener('input', onInput);
  });

  await render();
  return canvas;
})();
```


```js
/****************
 * CATCH PANEL
 ****************/

// Build grouped catch table for current EEZ
function fao_catch2() {
  const iso = this_eez.value;
  const base = iso === 'ALL' ? fao_catch : fao_catch.params({iso}).filter(d => d.iso3 === iso);
  return base.groupby('ocean', 'year').rollup({ catch_tonne: op.sum('catch_tonne') });
}

// Deterministic seed + scenario model
function seedFromString(s){
  let h = 2166136261 >>> 0;
  for (let i = 0; i < s.length; i++) { h ^= s.charCodeAt(i); h = Math.imul(h, 16777619); }
  return (h >>> 0) / 4294967296;
}

function scenarioFactor(exp, year){
  const t0 = Math.max(0, Math.min(1, (year - 2015) / (2080 - 2015)));
  const e1 = Math.pow(t0, 2.2), e2 = Math.pow(t0, 4.0);
  const t  = 0.55 * e1 + 0.45 * e2;
  const targets = { ssp1_2_6: 0.70, ssp2_4_5: 0.50, ssp3_7_0: 0.33, ssp5_8_5: 0.22 };
  const end = targets[exp] ?? 0.50;
  return 1 + t * (end - 1);
}

function adjustCatchTable(table, exp, eezVal){
  const oceanK = {
    "Pacific": 0.95, "Atlantic": 0.90, "Indian Ocean": 0.85, "Arctic Sea": 0.70,
    "Mediterranean and Black Sea": 0.78
  };
  const arr = table.objects();
  return arr.map(d => {
    const base = d.catch_tonne;
    const fac  = scenarioFactor(exp, d.year) * (oceanK[d.ocean] ?? 1);
    const seed = seedFromString(`${exp}|${eezVal}|${d.ocean}|${d.year}`);
    const rng  = d3.randomLcg(seed);
    const noise = (rng() - 0.5) * 0.40;
    const t0 = Math.max(0, Math.min(1, (d.year - 2015) / (2080 - 2015)));
    const lateDrag = 1 - 0.06 * Math.pow(t0, 3.5);
    const adj = Math.max(0, base * fac * lateDrag * (1 + noise));
    return { ...d, catch_tonne_adj: adj };
  });
}

// Responsive catch card: widest possible chart, no "Year" label or year readout
// Responsive catch card: fixed-width readout to prevent chart resizing
function catch_card({ label = "Catch", unit = "tonnes" } = {}) {
  const stackOrder = [
    "Pacific",
    "Atlantic",
    "Indian Ocean",
    "Arctic Sea",
    "Mediterranean and Black Sea"
  ];

  // --- data prep ---
  const base   = fao_catch2().filter(d => d.year >= 2015 && d.year <= 2080);
  const exp    = this_experiment.value;
  const eezVal = this_eez.value;
  const dataAdj = adjustCatchTable(base, exp, eezVal);

  const year = getYear();
  const totalThisYear =
    d3.sum(dataAdj.filter(d => d.year === year), d => d.catch_tonne_adj) || 0;

  const fmtBig  = d3.format(".2s");
  const fmtYear = d3.format("d");

  // --- layout: left plot stretches; right readout fixed width (no jitter) ---
  const HEIGHT = 200;
  const el = html`<div style="
    display:grid;
    grid-template-columns: minmax(0,1fr) 8ch;  /* fixed readout column */
    column-gap: 20px;
    align-items:center;
  "></div>`;

  // Chart fills the left cell
  const plot = Plot.plot({
    height: HEIGHT,
    marginTop: 8,
    marginBottom: 28,
    marginRight: 10,
    y: { label: null, tickFormat: "s" },
    x: { tickFormat: fmtYear },
    color: { legend: true, domain: stackOrder, range: d3.schemeCategory10 },
    marks: [
      Plot.areaY(dataAdj, {
        x: "year",
        y: "catch_tonne_adj",
        fill: "ocean",
        order: stackOrder,
        clip: true
      }),
      Plot.ruleX([year], { stroke: "black", strokeWidth: 1.5, opacity: 0.7 }),
      Plot.ruleY([0])
    ]
  });
  plot.style.width = "100%";
  plot.style.display = "block";

  // Right readout: tabular numerals + fixed column width
  const right = html`<div style="
    text-align:right;
    white-space:nowrap;
    font-variant-numeric: tabular-nums;  /* equal-width digits */
  ">
    <div style="font:600 13px/1.2 system-ui,sans-serif;color:#666">${label}</div>
    <div style="font:700 26px/1.1 system-ui,sans-serif">${fmtBig(totalThisYear)}</div>
    <div style="font:12px/1.1 system-ui,sans-serif;color:#666">${unit}</div>
  </div>`;

  el.append(plot, right);
  return el;
}


const catch_panel = (() => {
  const host = html`<div></div>`;
  const render = () => { host.innerHTML = ""; host.append(catch_card()); };

  const onInput = () => render();
  this_eez.addEventListener("input", onInput);
  this_year.addEventListener("input", onInput);
  this_experiment.addEventListener("input", onInput);
  invalidation.then(() => {
    this_eez.removeEventListener("input", onInput);
    this_year.removeEventListener("input", onInput);
    this_experiment.removeEventListener("input", onInput);
  });

  render();
  return host;
})();
```


```js
/****************
 * CLIMATE SPARKS
 ****************/

function climate_sel() {
  const iso = typeof this_eez === "string" ? this_eez : this_eez?.value ?? "ALL";
  return climate_raw
    .params({ iso })
    .filter(d => iso === "ALL" ? d.ISO_TER1 === "ALL" : d.ISO_TER1 === iso)
    .filter(d => d.year <= 2080)
    .join_left(iso3.rename({ iso3: "ISO_TER1" }), "ISO_TER1")
    .derive({ name: d => d.ISO_TER1 === "ALL" ? "All" : (d.name ?? d.ISO_TER1) })
    .reify();
}

// Spark card: unit moved to the right (under value), no unit on left, no year shown
function sparkCard({
  table,
  label,
  unit,
  color = d3.schemeCategory10[0],
  baseline = "axis"   // "axis" (default) or "series"
} = {}) {
  // --- data prep ---
  const raw = table.objects?.() ?? table?.objects?.() ?? [];
  const data = raw
    .filter(d => Number.isFinite(d.year) && Number.isFinite(d.mean))
    .sort((a, b) => d3.ascending(a.year, b.year));

  if (!data.length) {
    const empty = html`<div style="
      display:grid;
      grid-template-columns: 15% 70% 15%;
      align-items:center;
      gap:12px;
    "></div>`;
    empty.append(
      html`<div style="text-align:right">
        <div style="font:600 14px/1.1 system-ui,sans-serif">${label}</div>
      </div>`,
      html`<div style="height:70px;display:flex;align-items:center;justify-content:center;color:#999;font:12px system-ui">No data</div>`,
      html`<div></div>`
    );
    return empty;
  }

  const [y0, y1] = d3.extent(data, d => d.mean);
  const span = (y1 ?? 0) - (y0 ?? 0);
  const pad  = span ? span * 0.06 : ((Math.abs(y1 ?? 1) || 1) * 0.06);
  const yDomain = [(y0 ?? 0) - pad, (y1 ?? 0) + pad];

  // snap to current slider value (used only for the value readout and dot)
  const year = getYear();
  const snapped = d3.least(data, d => Math.abs(d.year - year));
  const v = snapped ? snapped.mean : null;
  const dotData = snapped ? [{year: snapped.year, mean: snapped.mean}] : [];
  const fmt = d3.format(".1f");

  // --- layout: 15% | 70% | 15% ---
  const CARD_HEIGHT = 70;
  const el = html`<div style="
    display:grid;
    grid-template-columns: 15% 70% 15%;
    align-items:center;
    gap:12px;
  "></div>`;
  //el.style.minWidth = "420px";

  // Left: label only (no unit here anymore)
  const left = html`<div style="text-align:right">
    <div style="font:600 14px/1.1 system-ui,sans-serif">${label}</div>
  </div>`;

  // Fill baseline choice
  const yMin = baseline === "series"
    ? (d3.min(data, d => d.mean) ?? yDomain[0])
    : yDomain[0];

  // Center: spark plot
  const plot = Plot.plot({
    height: CARD_HEIGHT,
    marginTop: 4,
    marginBottom: 4,
    x: { axis: null },
    y: { axis: null, domain: yDomain },
    marks: [
      Plot.areaY(data, { x: "year", y1: "mean", y2: yMin, fill: color, opacity: 0.1, clip: true }),
      Plot.lineY(data, { x: "year", y: "mean", stroke: color, clip: true }),
      Plot.dot(dotData, { x: "year", y: "mean", r: 4, fill: "#000", clip: false })
    ]
  });
  plot.style.width = "100%";
  plot.style.display = "block";

  // Right: value + unit (no year here anymore)
  const right = html`<div style="text-align:left">
    <div style="font:700 20px/1.1 system-ui,sans-serif">${v == null ? "" : fmt(v)}</div>
    <div style="font:12px/1.1 system-ui,sans-serif;color:#666">${unit ?? ""}</div>
  </div>`;

  el.append(left, plot, right);
  return el;
}


const spark_panel = (() => {
  const host = html`<div></div>`;
  function render() {
    host.innerHTML = "";
    const base = climate_sel();
    const exp = this_experiment.value;
    const sst_t    = base.params({exp}).filter(d => d.variable === "sst"    && d.experiment === exp);
    const wind_t   = base.params({exp}).filter(d => d.variable === "wind"   && d.experiment === exp);
    const seaice_t = base.params({exp}).filter(d => d.variable === "seaice" && d.experiment === exp);
    const hdd_t    = base.params({exp}).filter(d => d.variable === "hdd"    && d.experiment === exp);
    host.append(
      sparkCard({ table: sst_t,    label: "Sea surface temperature", unit: "°C" }),
      sparkCard({ table: wind_t,   label: "Wind speed",              unit: "m/s" }),
      sparkCard({ table: seaice_t, label: "Sea-ice area",            unit: "%" }),
      sparkCard({ table: hdd_t,    label: "Heating degree days",     unit: "°C·days" }),
    );
  }

  const onInput = () => render();
  this_eez.addEventListener("input", onInput);
  this_experiment.addEventListener("input", onInput);
  this_year.addEventListener("input", onInput);
  invalidation.then(() => {
    this_eez.removeEventListener("input", onInput);
    this_experiment.removeEventListener("input", onInput);
    this_year.removeEventListener("input", onInput);
  });

  render();
  return host;
})();
```

```js
/****************
 * KOBE PLOT
 ****************/

// === module-scope helpers (put in their own cell) ===
function seeded01(str) {
  const h = Array.from(str).reduce(
    (acc, ch) => (acc * 1664525 + ch.charCodeAt(0) + 1013904223) >>> 0,
    2166136261
  );
  return (h % 1e6) / 1e6;
}

const KOBE_START_CACHE = new Map();
function kobeStartForEEZ(eezKey) {
  let s = KOBE_START_CACHE.get(eezKey);
  if (!s) {
    const rng = d3.randomLcg(seeded01(`${eezKey}|start.v2`));  // EEZ-only seed
    s = { b0: 0.9 + rng()*0.8, f0: 0.9 + rng()*0.8 };
    KOBE_START_CACHE.set(eezKey, s);
  }
  return s;
}

function makeKobePlot() {
  // ---- deterministic seed helper (module-scope copy OK too) ----
  function seeded01(str) {
    const h = Array.from(str).reduce(
      (acc, ch) => (acc * 1664525 + ch.charCodeAt(0) + 1013904223) >>> 0,
      2166136261
    );
    return (h % 1e6) / 1e6;
  }

  // ---- config ----
  const years = d3.range(2010, 2081, 10);
  const HEIGHT = 400;

  // Fixed domains (symmetric around 1)
  const BAND_B = 0.8; // tighten if you want even less motion
  const BAND_F = 0.8;
  const X_DOMAIN = [Math.max(0, 1 - BAND_B), 1 + BAND_B];
  const Y_DOMAIN = [Math.max(0, 1 - BAND_F), 1 + BAND_F];

  // ---- scenario & experiment effects (kept modest) ----
  const scenarioParams = {
    BAU:         { muB:  0.00, muF:  0.00, vol: 0.18, kappa: 0.18 },
    Stress:      { muB: -0.02, muF:  0.04, vol: 0.22, kappa: 0.12 },
    Sustainable: { muB:  0.03, muF: -0.05, vol: 0.14, kappa: 0.28 }
  };
  const scen = scenarioParams[this_scenario.value] ?? scenarioParams.BAU;

  const experimentMods = {
    ssp1_2_6: { mu_mult: 1.00, vol_mult: 0.85, kappa_mult: 1.15 },
    ssp2_4_5: { mu_mult: 1.00, vol_mult: 1.00, kappa_mult: 1.00 },
    ssp3_7_0: { mu_mult: 1.10, vol_mult: 1.15, kappa_mult: 0.90 },
    ssp5_8_5: { mu_mult: 1.05, vol_mult: 1.25, kappa_mult: 0.95 }
  };
  const expm = experimentMods[this_experiment.value] ?? experimentMods.ssp2_4_5;

  const muB   = scen.muB * expm.mu_mult;
  const muF   = scen.muF * expm.mu_mult;
  const sigma = scen.vol * expm.vol_mult;
  const kappa = scen.kappa * expm.kappa_mult; // mean reversion toward 1

  // ---- EEZ-fixed start (cache-friendly) ----
  const eezKey = String(this_eez?.value ?? this_eez?.label ?? this_eez ?? "ALL");

  // Start depends ONLY on EEZ so it never jumps when toggling scenario/experiment
  const rngStart = d3.randomLcg(seeded01(`${eezKey}|start.v3`));
  const b0 = 0.9 + rngStart() * 0.8;  // ~[0.9,1.7]
  const f0 = 0.9 + rngStart() * 0.8;

  // Convert start into bounded latent states via atanh so we can evolve in ℝ
  function clampToBand(v, band) { return Math.min(1 + band - 1e-6, Math.max(1 - band + 1e-6, v)); }
  const b0c = clampToBand(b0, BAND_B);
  const f0c = clampToBand(f0, BAND_F);
  let zb = Math.atanh((b0c - 1) / BAND_B); // latent for B
  let zf = Math.atanh((f0c - 1) / BAND_F); // latent for F

  // Path RNG can depend on everything
  const rngPath = d3.randomLcg(seeded01(`${eezKey}|${this_scenario.value}|${this_experiment.value}|path.v3`));

  // ---- evolve latent AR(1)-like mean-reverting process, then squash with tanh ----
  const kobe = years.map((y, i) => {
    if (i) {
      // Ornstein–Uhlenbeck-ish in discrete time (mean = 0), plus small directional mu
      zb = (1 - kappa) * zb + muB + (rngPath() - 0.5) * sigma;
      zf = (1 - kappa) * zf + muF + (rngPath() - 0.5) * sigma;
    }
    const B = 1 + BAND_B * Math.tanh(zb);
    const F = 1 + BAND_F * Math.tanh(zf);
    return { year: y, B_BMSY: B, F_FMSY: F };
  });

  // ---- highlight nearest decade to slider ----
  const nearestYear = Math.max(2010, Math.min(2080, Math.round(this_year.value / 10) * 10));
  const sel = kobe.find(d => d.year === nearestYear);

  const padX = 0.03 * (X_DOMAIN[1] - X_DOMAIN[0]);
  const padY = 0.04 * (Y_DOMAIN[1] - Y_DOMAIN[0]);

  // ---- plot ----
  return Plot.plot({
    width: 700,
    height: HEIGHT,
    x: { label: "B / B\u2098\u209C\u2099", domain: X_DOMAIN, ticks: d3.ticks(X_DOMAIN[0], X_DOMAIN[1], 8) },
    y: { label: "F / F\u2098\u209C\u2099", domain: Y_DOMAIN, ticks: d3.ticks(Y_DOMAIN[0], Y_DOMAIN[1], 8) },
    marks: [
      // quadrants (fixed to domain so the background never shifts)
      Plot.rect([{}], { x1: 1, x2: X_DOMAIN[1], y1: Y_DOMAIN[0], y2: 1, fill: "#9FD8A3", opacity: 0.30 }),
      Plot.rect([{}], { x1: X_DOMAIN[0], x2: 1, y1: Y_DOMAIN[0], y2: 1, fill: "#F6E27F", opacity: 0.30 }),
      Plot.rect([{}], { x1: 1, x2: X_DOMAIN[1], y1: 1, y2: Y_DOMAIN[1], fill: "#F6B26B", opacity: 0.30 }),
      Plot.rect([{}], { x1: X_DOMAIN[0], x2: 1, y1: 1, y2: Y_DOMAIN[1], fill: "#EA9999", opacity: 0.30 }),

      Plot.ruleX([1], { stroke: "#444" }),
      Plot.ruleY([1], { stroke: "#444" }),

      // corner labels
      Plot.text([{ x: X_DOMAIN[0] + padX, y: Y_DOMAIN[1] - padY, text: "Unsustainable" }],
                { x: "x", y: "y", text: "text", textAnchor: "start", fontSize: 14 }),
      Plot.text([{ x: X_DOMAIN[1] - padX, y: Y_DOMAIN[1] - padY, text: "Overfishing (high F)" }],
                { x: "x", y: "y", text: "text", textAnchor: "end",   fontSize: 14 }),
      Plot.text([{ x: X_DOMAIN[0] + padX, y: Y_DOMAIN[0] + padY, text: "Overfished (low B)" }],
                { x: "x", y: "y", text: "text", textAnchor: "start", fontSize: 14 }),
      Plot.text([{ x: X_DOMAIN[1] - padX, y: Y_DOMAIN[0] + padY, text: "Sustainable" }],
                { x: "x", y: "y", text: "text", textAnchor: "end",   fontSize: 14 }),

      // path + points
      Plot.line(kobe, { x: "B_BMSY", y: "F_FMSY", stroke: d3.schemeCategory10[0], strokeWidth: 2, curve: "catmull-rom" }),
      Plot.dot(kobe,  { x: "B_BMSY", y: "F_FMSY", r: 3.5, fill: d3.schemeCategory10[0], opacity: 0.5 }),

      ...(sel ? [
        Plot.dot([sel], { x: "B_BMSY", y: "F_FMSY", r: 8, fill: "black", stroke: "white", strokeWidth: 1 }),
        Plot.text([sel], {
          x: "B_BMSY", y: "F_FMSY", text: d => d3.format("d")(d.year),
          dx: 30, dy: 0, fontSize: 14, fontWeight: 700,
          fill: "black", stroke: "white", strokeWidth: 2, paintOrder: "stroke"
        })
      ] : [])
    ]
  });
}



// Reactive wrapper
const kobe_panel = (() => {
  const host = html`<div></div>`;
  const render = () => { host.innerHTML = ""; host.append(makeKobePlot()); };

  const onInput = () => render();
  // re-render on all relevant controls
  this_experiment.addEventListener("input", onInput);
  this_scenario.addEventListener("input", onInput);
  this_year.addEventListener("input", onInput);
  this_eez.addEventListener("input", onInput);

  invalidation.then(() => {
    this_experiment.removeEventListener("input", onInput);
    this_scenario.removeEventListener("input", onInput);
    this_year.removeEventListener("input", onInput);
    this_eez.removeEventListener("input", onInput);
  });

  render();
  return host;
})();
```


<div class="hero">
  <h1>Demo 1</h1>
  <div class="controls">
    ${this_experiment}
    ${this_year}
    ${this_eez}
  </div>
  <div class="grid grid-cols-2 full-bleed tight-rows" style="gap: 1rem;">
    <div class="card">
      ${spark_panel}
    </div>
    <div class="card" style="position:relative;">
      ${catch_panel}
    </div>
    <div class="card controls-vertical no-gap">
      ${this_variable}
      ${heatmap}
    </div>
    <div class="card" style="position:relative;">
      ${this_scenario}
      ${kobe_panel}
    </div>
  </div>
</div>




<style>
:root {
  --control-gap: 1rem;
  --control-font-size: 14px;
}

/* ---------- Page framework ---------- */
.hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: var(--sans-serif);
  text-align: center;
  margin: 0;
  text-wrap: balance;
  overflow: hidden;
}
.hero h1 {
  margin: 0 0 .75rem;
  line-height: 1;
}
@media (min-width: 640px) {
  .hero h1 { font-size: 50px; }
}

.full-bleed { width: 100%; margin: 0; }

/* ---------- Cards / stacked content ---------- */
.controls-vertical { display: flex; flex-direction: column; gap: var(--control-gap); }
.controls-vertical.no-gap { gap: 0; }
.controls-vertical.no-gap > .radio-inline { margin-bottom: 0; }

/* ---------- Controls (tight rows, shared left rail) ---------- */
.controls {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: .375rem;
  width: 100%;
}
.controls > * { margin: 0; font-size: var(--control-font-size); }

/* Each control row: label left, control right */
.control-row {
  display: flex;
  align-items: center;
  gap: .5rem;
  width: 100%;
}
.control-label {
  font-weight: 700;
  white-space: nowrap;
}

/* Extra space before the EEZ row (it's the only .control-row) */
.controls > .control-row { margin-bottom: 1.5rem; }

/* ---------- CMIP6 radios ---------- */
.radio-inline {
  display: flex !important;
  align-items: center;
  gap: .5rem 1rem;
  flex-wrap: wrap;
  overflow: visible;
  line-height: 1.2;
}
.radio-inline .radio-label {
  margin-right: .5rem;
  white-space: nowrap;
  font-weight: 700;
}
.radio-inline [role="radiogroup"],
.radio-inline > form {
  display: flex !important;
  flex-wrap: wrap !important;
  align-items: center;
  gap: .5rem 1rem;
  margin: 0;
  padding: 0;
  border: 0;
}
.radio-inline label {
  display: inline-flex !important;
  align-items: center;
  gap: .35rem;
  margin: 0;
  white-space: nowrap !important;
}

/* ---------- Year slider ---------- */
.year-inline {
  display: flex;
  align-items: center;
  gap: .75rem;
  width: 100%;
  font: 700 var(--control-font-size)/1.2 var(--sans-serif);
}
.year-inline input[type="range"] {
  flex: 1 1 0%;
  min-width: 240px;
  max-width: 50%;
  font-size: var(--control-font-size);
}
.year-inline .year-value {
  font: 700 var(--control-font-size)/1.2 var(--sans-serif);
  width: 4ch;
  text-align: right;
}

/* ---------- EEZ dropdown (native select) ---------- */
.eez-inline { display: flex; align-items: center; gap: .5rem; }
.eez-label { font-weight: 700; white-space: nowrap; }
.eez-inline select { font-weight: 400; }

/* ---------- Grid rows: each row takes tallest item; no overlap ---------- */
.tight-rows {
  /* default grid-auto-rows: auto → row height = tallest item in that row */
  align-items: stretch;        /* children stretch to fill the row’s track */
}

/* Two columns, auto row height = tallest card in that row */
.grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr)); /* <- key change */
  grid-auto-rows: auto;
  align-items: stretch;
  gap: 1rem;
}


.card {
  display: flex;
  flex-direction: column;
  /* no fixed height! let grid control it */
  min-height: 0;  /* safety for flex content */
}


/* If any inner content is absolute/fixed-size, keep it from forcing the row */
.card > * {
  min-height: 0;
}



</style>

