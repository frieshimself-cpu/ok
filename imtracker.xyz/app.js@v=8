(function () {
  function initIntroAnimation() {
    const overlay = document.getElementById("introOverlay");
    if (!overlay) return;
    if (window.matchMedia("(prefers-reduced-motion: reduce)").matches) {
      overlay.remove();
      return;
    }

    const MAN_DELAY = 2000;
    const MAN_ENTER_MS = 1000;
    const BUBBLE_SHOW_MS = 3000;
    const HIDE_MS = 550;

    const bubbleStart = MAN_DELAY + MAN_ENTER_MS;
    const hideStart = bubbleStart + BUBBLE_SHOW_MS;

    window.setTimeout(() => {
      overlay.setAttribute("aria-hidden", "false");
      overlay.classList.add("is-man-in");
    }, MAN_DELAY);

    window.setTimeout(() => {
      overlay.classList.add("is-bubble-in");
    }, bubbleStart);

    window.setTimeout(() => {
      overlay.classList.add("is-hiding");
    }, hideStart);

    window.setTimeout(() => {
      overlay.classList.add("is-done");
      window.setTimeout(() => overlay.remove(), 450);
    }, hideStart + HIDE_MS);
  }

  initIntroAnimation();

  const { meta, countries } = window.EUROPE_DATA;
  const byId = new Map(countries.map((c) => [c.id, c]));

  const NUM_TO_ISO3 = {
    8: "ALB", 20: "AND", 40: "AUT", 112: "BLR", 56: "BEL", 70: "BIH", 100: "BGR",
    191: "HRV", 196: "CYP", 203: "CZE", 208: "DNK", 233: "EST", 246: "FIN",
    250: "FRA", 276: "DEU", 300: "GRC", 348: "HUN", 352: "ISL", 372: "IRL",
    380: "ITA", 398: "KAZ", 417: "KGZ", 428: "LVA", 438: "LIE", 440: "LTU",
    442: "LUX", 807: "MKD", 498: "MDA", 492: "MCO", 499: "MNE", 528: "NLD",
    578: "NOR", 616: "POL", 620: "PRT", 642: "ROU", 643: "RUS", 674: "SMR",
    688: "SRB", 703: "SVK", 705: "SVN", 724: "ESP", 752: "SWE", 756: "CHE",
    792: "TUR", 804: "UKR", 826: "GBR", 336: "VAT",
  };

  const ISO3_TO_NAME = {
    ALB: "Albania", AND: "Andorra", AUT: "Austria", BLR: "Belarus", BEL: "Belgium",
    BIH: "Bosnia & Herzegovina", BGR: "Bulgaria", HRV: "Croatia", CYP: "Cyprus",
    CZE: "Czechia", DNK: "Denmark", EST: "Estonia", FIN: "Finland", FRA: "France",
    DEU: "Germany", GRC: "Greece", HUN: "Hungary", ISL: "Iceland", IRL: "Ireland",
    ITA: "Italy", LVA: "Latvia", LIE: "Liechtenstein", LTU: "Lithuania", LUX: "Luxembourg",
    MKD: "North Macedonia", MDA: "Moldova", MCO: "Monaco", MNE: "Montenegro",
    NLD: "Netherlands", NOR: "Norway", POL: "Poland", PRT: "Portugal", ROU: "Romania",
    RUS: "Russia", SMR: "San Marino", SRB: "Serbia", SVK: "Slovakia", SVN: "Slovenia",
    ESP: "Spain", SWE: "Sweden", CHE: "Switzerland", TUR: "Türkiye", UKR: "Ukraine",
    GBR: "United Kingdom", VAT: "Vatican City", XKX: "Kosovo",
  };

  const svg = d3.select("#europeMap");
  const g = svg.append("g");
  const gCountries = g.append("g").attr("class", "countries-layer");
  const gFaces = g.append("g").attr("class", "faces-layer");
  const tooltip = document.getElementById("mapTooltip");
  const detailEmpty = document.getElementById("detailEmpty");
  const detailContent = document.getElementById("detailContent");

  document.getElementById("disclaimerText").textContent = meta.disclaimer;
  document.getElementById("sourcesText").textContent =
    "Sources: " + meta.sources.join(" · ");
  const countdownNoteEl = document.getElementById("countdownNote");
  if (countdownNoteEl && meta.countdownDefinition) {
    countdownNoteEl.textContent = meta.countdownDefinition;
  }

  const rates = countries.map((c) => c.murdersPerMillion).sort(d3.ascending);
  const rateMin = rates[0];
  const rateMax = rates[rates.length - 1];
  const colorInterpolator = d3.interpolateRgbBasis([
    "#1f8f4a",
    "#4fa85c",
    "#c9a227",
    "#e84a1a",
    "#8b0000",
  ]);

  /** Bias colors toward red (low exponent + floor = less green on map). */
  function colorScale(v) {
    const span = rateMax - rateMin || 1;
    const linear = (v - rateMin) / span;
    const t = Math.min(1, 0.08 + 0.92 * Math.pow(linear, 0.28));
    return colorInterpolator(t);
  }

  const fmt = new Intl.NumberFormat("en-GB");
  let selectedIso = null;
  let caseFetchController = null;
  let caseRequestId = 0;
  let countdownTimer = null;
  let countdownTargetMs = null;
  let countdownRow = null;
  let lastCountdownParts = null;

  const COUNTDOWN_UNIT_IDS = {
    days: "cdDays",
    hours: "cdHours",
    mins: "cdMins",
    secs: "cdSecs",
  };

  const pad2 = (n) => String(n).padStart(2, "0");

  /** Resolve countdown from data.js or estimate from foreign-born share. */
  function resolveCountdown(row) {
    const first = row.firstGenShare ?? row.foreignBornShare ?? 0;
    const second = row.secondGenShare ?? 2;
    const third = row.thirdGenShare ?? 0.5;
    const bg =
      row.immigrantBackgroundShare ??
      Math.round((first + second + third) * 10) / 10;
    const growth = row.backgroundGrowthPerYear ?? 0.2;
    let status = row.fiftyPercentStatus;
    let targetMs = row.fiftyPercentDate ? Date.parse(row.fiftyPercentDate) : NaN;
    let yearsToFifty = row.yearsToFifty;

    if (!status || (status === "projected" && !Number.isFinite(targetMs))) {
      if (bg >= 50) {
        status = "reached";
      } else if (growth <= 0) {
        status = "unavailable";
      } else {
        yearsToFifty = (50 - bg) / growth;
        if (yearsToFifty > 500) {
          status = "beyond";
        } else {
          status = "projected";
          targetMs = Date.now() + yearsToFifty * 365.25 * 24 * 60 * 60 * 1000;
        }
      }
    }

    return { bg, first, second, third, growth, status, targetMs, yearsToFifty };
  }

  function setDigitsVisible(visible) {
    const digits = document.getElementById("countdownDigits");
    if (!digits) return;
    digits.classList.toggle("is-hidden", !visible);
    digits.setAttribute("aria-hidden", visible ? "false" : "true");
  }

  function setCountdownHeadline(countryName) {
    const el = document.getElementById("countdownHeadline");
    if (!el) return;
    if (!countryName) {
      el.textContent = "TIME UNTIL MORE THAN 50% OF — ARE IMMIGRANTS";
      return;
    }
    const upper = countryName.toUpperCase();
    el.innerHTML =
      'TIME UNTIL MORE THAN 50% OF <span class="countdown-country-name">' +
      upper +
      "</span> ARE IMMIGRANTS";
  }

  function stopCountdownTimer() {
    if (countdownTimer) {
      clearInterval(countdownTimer);
      countdownTimer = null;
    }
    lastCountdownParts = null;
  }

  function pulseCountdownEl(id, alsoSepBefore) {
    const el = document.getElementById(id);
    if (!el) return;
    el.classList.remove("is-ticking");
    void el.offsetWidth;
    el.classList.add("is-ticking");
    if (alsoSepBefore) {
      const sep = el.closest(".countdown-unit")?.previousElementSibling;
      if (sep?.classList.contains("countdown-sep")) {
        sep.classList.remove("is-ticking");
        void sep.offsetWidth;
        sep.classList.add("is-ticking");
      }
    }
  }

  function renderCountdownTick() {
    if (!countdownRow || selectedIso !== countdownRow.id) return;
    const statusEl = document.getElementById("countdownStatus");
    if (!Number.isFinite(countdownTargetMs)) return;

    const left = countdownTargetMs - Date.now();
    if (left <= 0) {
      setDigitsVisible(false);
      if (statusEl) statusEl.textContent = "50% threshold reached (projection)";
      return;
    }

    setDigitsVisible(true);
    const totalSec = Math.floor(left / 1000);
    const days = Math.floor(totalSec / 86400);
    const hours = Math.floor((totalSec % 86400) / 3600);
    const mins = Math.floor((totalSec % 3600) / 60);
    const secs = totalSec % 60;
    const parts = { days, hours, mins, secs };

    if (lastCountdownParts) {
      for (const key of ["secs", "mins", "hours", "days"]) {
        if (parts[key] !== lastCountdownParts[key]) {
          pulseCountdownEl(COUNTDOWN_UNIT_IDS[key], key !== "days");
        }
      }
    } else {
      pulseCountdownEl("cdSecs", true);
    }
    lastCountdownParts = parts;

    document.getElementById("cdDays").textContent = fmt.format(days);
    document.getElementById("cdHours").textContent = pad2(hours);
    document.getElementById("cdMins").textContent = pad2(mins);
    document.getElementById("cdSecs").textContent = pad2(secs);
    if (statusEl) {
      const targetYear = new Date(countdownTargetMs).getUTCFullYear();
      statusEl.textContent = `Est. year ${targetYear} · 1st/2nd/3rd gen immigrant background`;
    }
  }

  function updateCountdownPanel(row) {
    stopCountdownTimer();
    countdownRow = row;
    countdownTargetMs = null;

    const statusEl = document.getElementById("countdownStatus");
    const nowEl = document.getElementById("countdownNow");
    const noteEl = document.getElementById("countdownNote");

    if (!document.getElementById("countdownHeadline")) return;

    if (!row) {
      setCountdownHeadline(null);
      setDigitsVisible(false);
      if (statusEl) statusEl.textContent = "Click a country on the map";
      if (nowEl) nowEl.textContent = "";
      if (noteEl) noteEl.textContent = meta.countdownDefinition || "";
      return;
    }

    const name = row.name || ISO3_TO_NAME[row.id] || row.id;
    setCountdownHeadline(name);
    const cd = resolveCountdown(row);

    if (nowEl) {
      nowEl.textContent =
        `Now (est.): ${cd.bg}% — 1st ${cd.first}% · 2nd ${cd.second}% · 3rd ${cd.third}% · +${cd.growth} pp/year`;
    }

    if (noteEl) {
      noteEl.textContent = meta.countdownDefinition || "";
    }

    if (cd.status === "reached") {
      setDigitsVisible(false);
      if (statusEl) statusEl.textContent = `Already above 50% (${cd.bg}% est. immigrant background)`;
      return;
    }

    if (cd.status === "beyond") {
      setDigitsVisible(false);
      const yr = 2026 + Math.round(cd.yearsToFifty || 0);
      if (statusEl) {
        statusEl.textContent = `Not projected before ~${yr} (${Math.round(cd.yearsToFifty || 0)} years at current pace)`;
      }
      return;
    }

    if (cd.status === "unavailable") {
      setDigitsVisible(false);
      if (statusEl) statusEl.textContent = "Projection unavailable";
      return;
    }

    if (cd.status === "projected" && Number.isFinite(cd.targetMs)) {
      countdownTargetMs = cd.targetMs;
      renderCountdownTick();
      countdownTimer = setInterval(renderCountdownTick, 1000);
      return;
    }

    setDigitsVisible(false);
    statusEl.textContent = "Projection unavailable for this country";
  }

  const maxImmigrants = d3.max(countries, (c) => c.immigrants2025) || 1;
  const FACE_SIZE = 32;

  function faceCount(immigrants) {
    if (!immigrants || immigrants <= 0) return 0;
    const scaled = Math.ceil((immigrants / maxImmigrants) * 5);
    return Math.max(1, Math.min(5, scaled));
  }

  function deadlyLineHtml(row) {
    const country = (row.name || "this country").toUpperCase();
    const x = row.deadlyMultiplier;
    if (x == null || !Number.isFinite(x)) {
      return `AN IMMIGRANT IS <span class="deadly-num">—</span> TIMES MORE DEADLY THAN THE AVERAGE ${country} CITIZEN`;
    }
    if (x >= 1) {
      return `AN IMMIGRANT IS <span class="deadly-num">${x.toFixed(1)}</span> TIMES MORE DEADLY THAN THE AVERAGE ${country} CITIZEN`;
    }
    const less = (1 / x).toFixed(1);
    return `AN IMMIGRANT IS <span class="deadly-num">${less}</span> TIMES LESS DEADLY THAN THE AVERAGE ${country} CITIZEN`;
  }

  function showCountry(iso) {
    const row = byId.get(iso);
    if (!row) return;

    selectedIso = iso;
    d3.selectAll(".country").classed("selected", function () {
      return d3.select(this).attr("data-iso") === iso;
    });

    detailEmpty.hidden = true;
    detailContent.hidden = false;

    document.getElementById("countryName").textContent =
      row.name || ISO3_TO_NAME[iso] || iso;

    document.getElementById("statImmigrants").textContent = fmt.format(row.immigrants2025);
    document.getElementById("statImmMurders").textContent = fmt.format(row.immigrantMurders);
    document.getElementById("statDeadly").innerHTML = deadlyLineHtml(row);

    document.getElementById("detailNote").textContent =
      "Deadly multiplier = (immigrant homicide share) ÷ (foreign-born population share, Eurostat 2025). " +
      (row.foreignBornShare != null
        ? `This country: ${row.foreignSuspectShare}% of homicides vs ${row.foreignBornShare}% foreign-born residents.`
        : "");

    loadHighlightedCase(iso);
    updateCountdownPanel(row);
  }

  function stripHtml(text) {
    const el = document.createElement("div");
    el.innerHTML = text || "";
    return (el.textContent || "").replace(/\s+/g, " ").trim();
  }

  function parseGoogleRssXml(xmlText) {
    const doc = new DOMParser().parseFromString(xmlText, "text/xml");
    if (doc.querySelector("parsererror")) return null;
    const item = doc.querySelector("item");
    if (!item) return null;
    const title = item.querySelector("title")?.textContent?.trim();
    const url = item.querySelector("link")?.textContent?.trim();
    if (!title || !url) return null;
    return {
      ok: true,
      title,
      url,
      published: item.querySelector("pubDate")?.textContent?.trim() || "",
      snippet: stripHtml(item.querySelector("description")?.textContent || ""),
      source: item.querySelector("source")?.textContent?.trim() || "",
      image: null,
    };
  }

  let curatedCasesCache = null;

  async function loadCuratedCases(signal) {
    if (curatedCasesCache) return curatedCasesCache;
    try {
      const res = await fetch("./data/highlighted_cases.json", { signal });
      curatedCasesCache = res.ok ? await res.json() : {};
    } catch {
      curatedCasesCache = {};
    }
    return curatedCasesCache;
  }

  const COUNTRY_MARKERS = {
    POL: ["poland", "polish", "polska", "warsaw", "warszawa", "krakow", "kraków", "gdansk", "gdańsk"],
    DEU: ["germany", "german", "deutschland", "berlin", "munich", "hamburg"],
    FRA: ["france", "french", "paris", "lyon", "marseille"],
    GBR: ["united kingdom", "britain", "british", "england", "london", "southampton", "henry nowak"],
  };
  const RSS_LOCALE = {
    POL: ["pl", "PL", "PL:pl"],
    DEU: ["de", "DE", "DE:de"],
    FRA: ["fr", "FR", "FR:fr"],
    GBR: ["en", "GB", "GB:en"],
  };
  const UK_LEAK = /\b(henry\s+nowak|southampton|united\s+kingdom|british\s+student)\b/i;

  function isAboutCountry(iso, name, title, snippet) {
    const blob = `${title} ${snippet}`.toLowerCase();
    const markers = COUNTRY_MARKERS[iso] || [name.toLowerCase(), name.split(" ")[0].toLowerCase()];
    if (!markers.some((m) => blob.includes(m))) return false;
    if (iso !== "GBR" && UK_LEAK.test(blob)) return false;
    return true;
  }

  function parseGoogleRssItems(xmlText) {
    const doc = new DOMParser().parseFromString(xmlText, "text/xml");
    if (doc.querySelector("parsererror")) return [];
    return Array.from(doc.querySelectorAll("item")).map((item) => {
      const title = item.querySelector("title")?.textContent?.trim();
      const url = item.querySelector("link")?.textContent?.trim();
      if (!title || !url) return null;
      return {
        ok: true,
        title,
        url,
        published: item.querySelector("pubDate")?.textContent?.trim() || "",
        snippet: stripHtml(item.querySelector("description")?.textContent || ""),
        source: item.querySelector("source")?.textContent?.trim() || "",
        image: null,
      };
    }).filter(Boolean);
  }

  async function fetchHighlightedCaseClient(iso, signal) {
    const curated = await loadCuratedCases(signal);
    const hit = curated[iso];
    if (hit?.title) {
      return { ok: true, curated: true, ...hit };
    }

    const name = ISO3_TO_NAME[iso] || iso;
    const queries = [
      `${name} murder 2025`,
      `${name} stabbing killed 2025`,
      `${name} homicide victim 2025`,
    ];
    if (iso === "POL") queries.push("Polska morderstwo 2025");
    const scoreArticle = (title, snippet) => {
      if (!isAboutCountry(iso, name, title, snippet)) return -100;
      const blob = `${title} ${snippet}`.toLowerCase();
      if (
        /\b(netflix|trailer|teaser|movie|film|serie|series|doku|dokumentation|true[- ]crime|stream(?:ing)?|disney|prime video|hbo|kinostart|serienstart|staffel|episode|recap|podcast|roman|fiktion|fiction|hallo deutschland|serienjunkies)\b/.test(
          blob,
        )
      ) {
        return -50;
      }
      if (
        /\b(kriminalstatistik|ranking|memorial|monument|diese kriminalf[aä]lle bewegten|roundup|mordrate in|morde in den usa)\b/.test(
          blob,
        )
      ) {
        return -50;
      }
      if (
        !/\b(murder|homicide|killed|stab(bed|bing)?|knife|victim|sentenced|life sentence|attempted murder|mord|messermord|messerangriff|t[oö]dlich)\b/.test(
          blob,
        )
      ) {
        return -50;
      }
      let s = 8;
      if (/\bmurder\b/.test(blob)) s += 5;
      if (/\bhomicide\b/.test(blob)) s += 5;
      if (/\bkilled\b/.test(blob)) s += 4;
      if (/\bstab/.test(blob)) s += 4;
      if (/\b2025\b/.test(blob)) s += 4;
      if (/\b2026\b/.test(blob)) s += 3;
      return s;
    };

    const [hl, gl, ceid] = RSS_LOCALE[iso] || ["en", "US", "US:en"];
    let best = null;
    let bestScore = 0;
    for (const query of queries) {
      const q = encodeURIComponent(query);
      const rssUrl = `https://news.google.com/rss/search?q=${q}&hl=${hl}&gl=${gl}&ceid=${ceid}`;
      const proxyUrl = `https://api.allorigins.win/raw?url=${encodeURIComponent(rssUrl)}`;
      const res = await fetch(proxyUrl, { signal });
      if (!res.ok) continue;
      for (const article of parseGoogleRssItems(await res.text())) {
        const s = scoreArticle(article.title, article.snippet);
        if (s > bestScore) {
          bestScore = s;
          best = article;
        }
      }
    }

    if (best && bestScore >= 3) return best;
    const q = encodeURIComponent(`${name} murder 2025`);
    return {
      ok: false,
      error: "No major 2025–today case found in automated search",
      searchUrl: `https://news.google.com/search?q=${q}`,
    };
  }

  async function fetchHighlightedCaseServer(iso, signal) {
    const url = `${window.location.origin}/api/highlighted-case?iso=${encodeURIComponent(iso)}`;
    const res = await fetch(url, { signal, cache: "no-store" });
    const ct = (res.headers.get("content-type") || "").toLowerCase();
    if (!res.ok || !ct.includes("json")) {
      throw new Error("Server API unavailable — restart with python serve.py");
    }
    return res.json();
  }

  function renderHighlightedCase(data, iso, requestId) {
    const loading = document.getElementById("caseLoading");
    const content = document.getElementById("caseContent");
    const empty = document.getElementById("caseEmpty");
    const img = document.getElementById("caseImage");

    if (requestId !== caseRequestId || iso !== selectedIso) return;
    loading.hidden = true;

    if (!data.ok) {
      content.hidden = true;
      empty.hidden = false;
      img.hidden = true;
      const search = data.searchUrl
        ? ` <a href="${data.searchUrl}" target="_blank" rel="noopener noreferrer">Search Google News</a>`
        : "";
      empty.innerHTML = (data.error || "No highlighted case found.") + search;
      return;
    }

    empty.hidden = true;
    content.hidden = false;
    document.getElementById("caseTitle").textContent = data.title;
    document.getElementById("caseMeta").textContent = [data.source, data.published]
      .filter(Boolean)
      .join(" · ");
    document.getElementById("caseSnippet").textContent =
      data.snippet || "Open the article for full details.";
    document.getElementById("caseLink").href = data.url;
    img.hidden = true;
    if (data.image) {
      img.src = data.image;
      img.alt = data.title;
      img.hidden = false;
      img.onerror = () => {
        img.hidden = true;
      };
    }
  }

  async function loadHighlightedCase(iso) {
    const loading = document.getElementById("caseLoading");
    const content = document.getElementById("caseContent");
    const empty = document.getElementById("caseEmpty");
    const requestId = ++caseRequestId;

    loading.hidden = false;
    loading.textContent = "Loading highlighted case…";
    content.hidden = true;
    empty.hidden = true;

    if (caseFetchController) caseFetchController.abort();
    caseFetchController = new AbortController();
    const { signal } = caseFetchController;

    try {
      let data = null;
      if (window.location.protocol.startsWith("http")) {
        try {
          data = await fetchHighlightedCaseServer(iso, signal);
        } catch {
          if (requestId !== caseRequestId || iso !== selectedIso) return;
          loading.textContent = "Loading highlighted case (fallback)…";
          data = await fetchHighlightedCaseClient(iso, signal);
        }
      } else {
        data = await fetchHighlightedCaseClient(iso, signal);
      }
      renderHighlightedCase(data, iso, requestId);
    } catch (err) {
      if (err.name === "AbortError" || requestId !== caseRequestId || iso !== selectedIso) return;
      loading.hidden = true;
      content.hidden = true;
      empty.hidden = false;
      empty.innerHTML =
        'Could not load news. Start the site with <code>python serve.py</code> from the <code>europe-map-site</code> folder, then open <a href="http://127.0.0.1:8779/">http://127.0.0.1:8779/</a>.';
    }
  }

  function positionTooltip(event) {
    const panel = document.querySelector(".map-panel");
    const rect = panel.getBoundingClientRect();
    tooltip.style.left = event.clientX - rect.left + 12 + "px";
    tooltip.style.top = event.clientY - rect.top + 12 + "px";
  }

  // Drop overseas polygons (Canaries, Azores, etc.) so fitExtent stays on mainland Europe
  const VIEW_LON = [-20, 42];
  const VIEW_LAT = [35, 72];
  const RUS_MAX_LON = 60;

  function polygonCentroid(coords) {
    return d3.geoCentroid({ type: "Feature", geometry: { type: "Polygon", coordinates: coords } });
  }

  function keepPolygon(coords, iso) {
    const [lon, lat] = polygonCentroid(coords);
    if (iso === "RUS" && lon > RUS_MAX_LON) return false;
    return lon >= VIEW_LON[0] && lon <= VIEW_LON[1] && lat >= VIEW_LAT[0] && lat <= VIEW_LAT[1];
  }

  function mainlandFeature(feature) {
    const iso = NUM_TO_ISO3[+feature.id];
    const g = feature.geometry;
    if (!g) return null;

    let polys = [];
    if (g.type === "Polygon") polys = [g.coordinates];
    else if (g.type === "MultiPolygon") polys = g.coordinates;
    else return feature;

    polys = polys.filter((poly) => keepPolygon(poly, iso));
    if (!polys.length) return null;

    const geometry =
      polys.length === 1
        ? { type: "Polygon", coordinates: polys[0] }
        : { type: "MultiPolygon", coordinates: polys };

    return { ...feature, geometry };
  }

  d3.json("https://cdn.jsdelivr.net/npm/world-atlas@2/countries-50m.json").then((world) => {
    const features = topojson.feature(world, world.objects.countries).features;
    const europe = features
      .filter((f) => {
        const iso = NUM_TO_ISO3[+f.id];
        return iso && (byId.has(iso) || ISO3_TO_NAME[iso]);
      })
      .map(mainlandFeature)
      .filter(Boolean);

    const projection = d3.geoNaturalEarth1().fitExtent(
      [[24, 16], [936, 700]],
      { type: "FeatureCollection", features: europe }
    );
    projection.scale(projection.scale() * 1.1);

    const path = d3.geoPath(projection);

    function polygonFeatures(feature) {
      const g = feature.geometry;
      if (!g) return [];
      if (g.type === "Polygon") {
        return [{ type: "Feature", geometry: g, id: feature.id }];
      }
      if (g.type === "MultiPolygon") {
        return g.coordinates.map((coords) => ({
          type: "Feature",
          geometry: { type: "Polygon", coordinates: coords },
          id: feature.id,
        }));
      }
      return [feature];
    }

    function geoContainsFeature(feature, lon, lat) {
      return d3.geoContains(feature, [lon, lat]);
    }

    /** Anchor on largest land polygon (geo space), then project to screen. */
    function anchorPoint(feature) {
      const parts = polygonFeatures(feature);
      let bestGeo = null;
      let bestArea = -1;

      for (const part of parts) {
        const area = d3.geoArea(part);
        if (area <= 0) continue;
        const geo = d3.geoCentroid(part);
        if (geoContainsFeature(part, geo[0], geo[1]) && area > bestArea) {
          bestArea = area;
          bestGeo = geo;
        }
      }

      if (!bestGeo) {
        bestGeo = d3.geoCentroid(feature);
      }

      const pt = projection(bestGeo);
      return pt && Number.isFinite(pt[0]) && Number.isFinite(pt[1]) ? pt : path.centroid(feature);
    }

    function facePositions(feature, n) {
      const anchor = anchorPoint(feature);
      if (!anchor || !Number.isFinite(anchor[0])) return [];

      const [cx, cy] = anchor;
      const [[x0, y0], [x1, y1]] = path.bounds(feature);
      const spread = Math.max(10, Math.min(x1 - x0, y1 - y0) * 0.18);
      const positions = [[cx, cy]];

      for (let i = 1; i < n; i++) {
        const angle = ((i - 1) / Math.max(n - 1, 1)) * Math.PI * 2 - Math.PI / 2;
        const px = cx + spread * Math.cos(angle);
        const py = cy + spread * Math.sin(angle);
        const ll = projection.invert([px, py]);
        if (ll && geoContainsFeature(feature, ll[0], ll[1])) {
          positions.push([px, py]);
        } else {
          positions.push([cx + i * 4 - (n * 2), cy - 4]);
        }
      }

      return positions.slice(0, n);
    }

    const zoom = d3
      .zoom()
      .scaleExtent([1, 8])
      .on("zoom", (event) => g.attr("transform", event.transform));

    svg.call(zoom);

    function renderFaces() {
      const withData = europe.filter((f) => {
        const iso = NUM_TO_ISO3[+f.id];
        return iso && byId.has(iso);
      });
      const faceData = [];
      withData.forEach((feature) => {
        const iso = NUM_TO_ISO3[+feature.id];
        const row = byId.get(iso);
        const n = faceCount(row.immigrants2025);
        const pts = facePositions(feature, n);
        pts.forEach(([cx, cy], i) => {
          if (Number.isFinite(cx) && Number.isFinite(cy)) {
            faceData.push({ iso, cx, cy, i });
          }
        });
      });

      const faces = gFaces.selectAll("g.map-face").data(faceData, (d) => d.iso + "-" + d.i);
      faces.exit().remove();

      const merged = faces
        .enter()
        .append("g")
        .attr("class", "map-face")
        .attr("pointer-events", "none")
        .merge(faces);

      merged.attr("transform", (d) => `translate(${d.cx},${d.cy})`);

      const bob = merged
        .selectAll("g.face-bob")
        .data((d) => [d])
        .join("g")
        .attr("class", "face-bob");

      bob
        .selectAll("animateTransform")
        .data((d) => [d])
        .join("animateTransform")
        .attr("attributeName", "transform")
        .attr("type", "translate")
        .attr("dur", "2.4s")
        .attr("repeatCount", "indefinite")
        .attr("calcMode", "spline")
        .attr("keySplines", "0.45 0 0.55 1; 0.45 0 0.55 1")
        .attr("keyTimes", "0; 0.5; 1")
        .attr("values", "0,0; 0,-6; 0,0")
        .attr("begin", (d) => `${d.i * 0.35}s`);

      bob
        .selectAll("image")
        .data((d) => [d])
        .join("image")
        .attr("href", "media/meme-face.png")
        .attr("xlink:href", "media/meme-face.png")
        .attr("width", FACE_SIZE)
        .attr("height", FACE_SIZE)
        .attr("x", -FACE_SIZE / 2)
        .attr("y", -FACE_SIZE / 2);

      gFaces.raise();
    }

    gCountries
      .selectAll("path")
      .data(europe)
      .join("path")
      .attr("class", (d) => {
        const iso = NUM_TO_ISO3[+d.id];
        return "country" + (byId.has(iso) ? " has-data" : " no-data");
      })
      .attr("data-iso", (d) => NUM_TO_ISO3[+d.id] || "")
      .attr("d", path)
      .attr("fill", (d) => {
        const iso = NUM_TO_ISO3[+d.id];
        const row = byId.get(iso);
        return row ? colorScale(row.murdersPerMillion) : "#1a1418";
      })
      .on("click", function (event, d) {
        event.stopPropagation();
        const iso = NUM_TO_ISO3[+d.id];
        if (byId.has(iso)) showCountry(iso);
      })
      .on("mousemove", function (event, d) {
        const iso = NUM_TO_ISO3[+d.id];
        const row = byId.get(iso);
        const name = ISO3_TO_NAME[iso] || "Unknown";
        if (!row) {
          tooltip.hidden = true;
          return;
        }
        tooltip.hidden = false;
        tooltip.innerHTML =
          "<strong>" +
          name +
          "</strong><br/>Asylum 2025: " +
          fmt.format(row.immigrants2025) +
          "<br/>Est. immigrant-linked homicides: " +
          fmt.format(row.immigrantMurders) +
          "<br/>Icons: " +
          faceCount(row.immigrants2025) +
          " / 5";
        positionTooltip(event);
      })
      .on("mouseleave", () => {
        tooltip.hidden = true;
      });

    renderFaces();

    const defaultRow = byId.get("DEU") || countries[0];
    if (defaultRow) showCountry(defaultRow.id);
  });
})();
