<script lang="ts">
  import { onMount } from 'svelte';
  import i18n from '$lib/i18n.json';
  import 'leaflet/dist/leaflet.css';

  type Point = { lat: number; lon: number };

  let mapEl: HTMLDivElement | null = null;
  let points: Point[] = [];
  let mid: Point | null = null;
  let latInput = '';
  let lonInput = '';
  let error = '';
  let lang: 'et' | 'en' = 'et';
  type Lang = keyof typeof i18n;
  const t = (key: keyof (typeof i18n)['et']) => i18n[lang as Lang][key];

  let L: typeof import('leaflet') | null = null;
  let map: import('leaflet').Map | null = null;
  let pointLayer: import('leaflet').LayerGroup | null = null;
  let midLayer: import('leaflet').CircleMarker | null = null;
  let lineLayer: import('leaflet').LayerGroup | null = null;
  let locating = false;
  let reverseAbort: AbortController | null = null;

  const parseCoord = (value: string) => {
    const n = Number(value);
    return Number.isFinite(n) ? n : null;
  };

  const isValidLat = (lat: number) => lat >= -90 && lat <= 90;
  const isValidLon = (lon: number) => lon >= -180 && lon <= 180;

  const escapeHtml = (value: string) =>
    value
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;')
      .replace(/'/g, '&#39;');

  const reverseGeocode = async () => {
    if (!mid || !midLayer) return;
    const { lat, lon } = mid;

    reverseAbort?.abort();
    const controller = new AbortController();
    reverseAbort = controller;

    if (!midLayer.getPopup()) midLayer.bindPopup('');
    midLayer.setPopupContent(`<div class="popup-body">${t('midpointLookupLoading')}</div>`);
    midLayer.openPopup();

    try {
      const url = new URL('https://nominatim.openstreetmap.org/reverse');
      url.searchParams.set('format', 'jsonv2');
      url.searchParams.set('lat', lat.toString());
      url.searchParams.set('lon', lon.toString());
      url.searchParams.set('zoom', '18');
      url.searchParams.set('addressdetails', '1');

      const res = await fetch(url.toString(), {
        signal: controller.signal,
        headers: { 'Accept-Language': lang === 'et' ? 'et' : 'en' }
      });
      if (!res.ok) throw new Error('reverse_geocode_failed');

      const data = (await res.json()) as { display_name?: string };
      const display = data?.display_name ? escapeHtml(data.display_name) : t('midpointLookupUnknown');
      const content = `
        <div class="popup">
          <div class="popup-title">${t('midpointLookupTitle')}</div>
          <div class="popup-body">${display}</div>
          <div class="popup-attrib">${t('midpointLookupAttribution')}</div>
        </div>`;
      midLayer.setPopupContent(content);
    } catch (err) {
      if (controller.signal.aborted) return;
      midLayer.setPopupContent(
        `<div class="popup-body">${t('midpointLookupError')}</div>`
      );
    }
  };

  const computeMidpoint = (list: Point[]) => {
    const count = list.length;
    if (!count) return null;
    const sum = list.reduce(
      (acc, p) => {
        acc.lat += p.lat;
        acc.lon += p.lon;
        return acc;
      },
      { lat: 0, lon: 0 }
    );
    return { lat: sum.lat / count, lon: sum.lon / count };
  };

  const refreshMarkers = () => {
    if (!map || !L) return;
    if (!pointLayer) pointLayer = L.layerGroup().addTo(map);
    if (!lineLayer) lineLayer = L.layerGroup().addTo(map);

    pointLayer.clearLayers();
    lineLayer.clearLayers();
    for (const p of points) {
      L.circleMarker([p.lat, p.lon], {
        radius: 6,
        weight: 2,
        color: '#1d4ed8',
        fillColor: '#60a5fa',
        fillOpacity: 0.9
      }).addTo(pointLayer);
    }

    mid = computeMidpoint(points);
    if (mid) {
      if (points.length > 1) {
        L.polyline(
          points.map((p) => [p.lat, p.lon]),
          { color: '#38bdf8', weight: 2, opacity: 0.7 }
        ).addTo(lineLayer);
      }
      for (const p of points) {
        L.polyline(
          [
            [mid.lat, mid.lon],
            [p.lat, p.lon]
          ],
          { color: '#f87171', weight: 1.5, opacity: 0.7, dashArray: '6 6' }
        ).addTo(lineLayer);
      }

      if (!midLayer) {
        midLayer = L.circleMarker([mid.lat, mid.lon], {
          radius: 8,
          weight: 2,
          color: '#b91c1c',
          fillColor: '#f87171',
          fillOpacity: 0.95
        }).addTo(map);
        midLayer.on('click', (event: import('leaflet').LeafletMouseEvent) => {
          L?.DomEvent.stopPropagation(event);
          reverseGeocode();
        });
      } else {
        midLayer.setLatLng([mid.lat, mid.lon]);
      }

    } else if (midLayer) {
      map.removeLayer(midLayer);
      midLayer = null;
      reverseAbort?.abort();
    }
  };

  const addPoint = (lat: number, lon: number) => {
    points = [...points, { lat, lon }];
    refreshMarkers();
  };

  const removePoint = (index: number) => {
    points = points.filter((_, i) => i !== index);
    refreshMarkers();
  };

  const addFromInput = () => {
    const lat = parseCoord(latInput.trim());
    const lon = parseCoord(lonInput.trim());

    if (lat === null || lon === null) {
      error = t('errorNumbers');
      return;
    }

    if (!isValidLat(lat) || !isValidLon(lon)) {
      error = t('errorRange');
      return;
    }

    error = '';
    latInput = '';
    lonInput = '';
    addPoint(lat, lon);
  };

  const clearPoints = () => {
    points = [];
    error = '';
    refreshMarkers();
  };

  const locateMe = () => {
    if (!map || !('geolocation' in navigator)) return;
    locating = true;
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        locating = false;
        if (!map) return;
        map.setView([pos.coords.latitude, pos.coords.longitude], 13);
      },
      () => {
        locating = false;
      },
      { enableHighAccuracy: true, timeout: 8000, maximumAge: 60000 }
    );
  };

  onMount(async () => {
    if (!mapEl) return;
    L = await import('leaflet');

    map = L.map(mapEl, {
      center: [58.7, 25.0],
      zoom: 9,
      zoomControl: false
    });

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      maxZoom: 19,
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);

    L.control.zoom({ position: 'bottomright' }).addTo(map);

    map.on('click', (event: import('leaflet').LeafletMouseEvent) => {
      addPoint(event.latlng.lat, event.latlng.lng);
    });

    refreshMarkers();

    locateMe();
  });
</script>

<svelte:head>
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
  <link
    href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600;700&display=swap"
    rel="stylesheet"
  />
</svelte:head>

<main>
  <div class="map" bind:this={mapEl}></div>

  <section class="panel">
    <div class="header">
      <div>
        <h1>{t('title')}</h1>
        <p class="hint">{t('hint')}</p>
      </div>
    </div>

    <div class="inputs">
      <label>
        <span>{t('latLabel')}</span>
        <input
          type="text"
          bind:value={latInput}
          placeholder={t('latPlaceholder')}
        />
      </label>
      <label>
        <span>{t('lonLabel')}</span>
        <input
          type="text"
          bind:value={lonInput}
          placeholder={t('lonPlaceholder')}
        />
      </label>
      <div class="actions">
        <button type="button" on:click={addFromInput}>
          {t('addPoint')}
        </button>
        <button type="button" class="secondary" on:click={clearPoints} disabled={!points.length}>
          {t('clearAll')}
        </button>
        <button type="button" class="ghost" on:click={locateMe} disabled={locating}>
          {locating ? t('locating') : t('locateMe')}
        </button>
      </div>
    </div>

    {#if error}
      <p class="error">{error}</p>
    {/if}

    <div class="stats">
      <div>
        <strong>{points.length}</strong>
        <span>{t('points')}</span>
      </div>
      <div>
        <strong>{mid ? `${mid.lat.toFixed(6)}, ${mid.lon.toFixed(6)}` : '—'}</strong>
        <span>{t('midpoint')}</span>
      </div>
    </div>

    {#if points.length}
      <ol class="list">
        {#each points as point, index}
          <li>
            <span class="point-label">#{index + 1}</span>
            <span class="point-coords">{point.lat.toFixed(6)}, {point.lon.toFixed(6)}</span>
            <button
              type="button"
              class="point-delete"
              on:click={() => removePoint(index)}
              aria-label="Delete point {index + 1}"
              title={t('deletePoint')}
            >
              ×
            </button>
          </li>
        {/each}
      </ol>
    {/if}
  </section>

  <div class="lang lang-floating">
    <button type="button" class:active={lang === 'et'} on:click={() => (lang = 'et')}>
      ET
    </button>
    <button type="button" class:active={lang === 'en'} on:click={() => (lang = 'en')}>
      EN
    </button>
  </div>
</main>

<style>
  :global(html, body) {
    margin: 0;
    padding: 0;
    height: 100%;
  }

  :global(body) {
    font-family: 'Space Grotesk', system-ui, sans-serif;
    background: #0b0f1c;
  }

  :global(#svelte) {
    height: 100%;
  }

  main {
    position: relative;
    height: 100vh;
    width: 100%;
    overflow: hidden;
  }

  .map {
    position: absolute;
    inset: 0;
  }

  .panel {
    position: absolute;
    top: 24px;
    right: 24px;
    width: min(360px, calc(100% - 48px));
    padding: 20px 20px 16px;
    background: rgba(12, 17, 32, 0.88);
    border: 1px solid rgba(148, 163, 184, 0.15);
    border-radius: 16px;
    color: #e2e8f0;
    box-shadow: 0 18px 50px rgba(15, 23, 42, 0.35);
    backdrop-filter: blur(12px);
    z-index: 9999;
  }

  h1 {
    margin: 0 0 4px;
    font-size: 1.4rem;
    letter-spacing: 0.02em;
  }

  .header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 16px;
  }

  .lang {
    display: inline-flex;
    gap: 6px;
    background: rgba(148, 163, 184, 0.12);
    border-radius: 999px;
    padding: 4px;
    border: 1px solid rgba(148, 163, 184, 0.2);
  }

  .lang-floating {
    position: absolute;
    left: 20px;
    bottom: 20px;
    z-index: 9999;
  }

  .lang button {
    border-radius: 999px;
    border: none;
    padding: 6px 10px;
    font-size: 0.7rem;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    background: transparent;
    color: #cbd5f5;
    cursor: pointer;
    transition: background 0.15s ease, color 0.15s ease;
  }

  .lang button.active {
    background: rgba(59, 130, 246, 0.85);
    color: #fff;
  }

  .lang button:hover {
    background: rgba(148, 163, 184, 0.2);
  }

  .hint {
    margin: 0 0 16px;
    font-size: 0.9rem;
    color: #94a3b8;
    white-space: pre-wrap;
  }

  :global(.leaflet-popup-content-wrapper) {
    background: rgba(12, 17, 32, 0.95);
    color: #e2e8f0;
    border-radius: 12px;
  }

  :global(.leaflet-popup-tip) {
    background: rgba(12, 17, 32, 0.95);
  }

  :global(.leaflet-popup-content) {
    margin: 10px 12px;
  }

  .popup-title {
    font-weight: 600;
    margin-bottom: 6px;
  }

  .popup-body {
    font-size: 0.85rem;
    line-height: 1.35;
  }

  .popup-attrib {
    margin-top: 6px;
    font-size: 0.7rem;
    color: #94a3b8;
  }

  .inputs {
    display: grid;
    gap: 12px;
  }

  label {
    display: grid;
    gap: 6px;
    font-size: 0.75rem;
    color: #cbd5f5;
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }

  input {
    border-radius: 10px;
    border: 1px solid rgba(148, 163, 184, 0.2);
    padding: 10px 12px;
    background: rgba(15, 23, 42, 0.7);
    color: #e2e8f0;
    font-size: 0.95rem;
  }

  input:focus {
    outline: none;
    border-color: rgba(59, 130, 246, 0.7);
    box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.25);
  }

  .actions {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
  }

  .actions button {
    flex: 1;
    border-radius: 10px;
    border: none;
    padding: 10px 12px;
    background: #2563eb;
    color: #fff;
    font-weight: 600;
    cursor: pointer;
    transition: transform 0.15s ease, box-shadow 0.15s ease;
  }

  .actions button:hover {
    transform: translateY(-1px);
    box-shadow: 0 8px 20px rgba(37, 99, 235, 0.3);
  }

  .actions button.secondary {
    background: rgba(148, 163, 184, 0.2);
    color: #e2e8f0;
  }

  .actions button.ghost {
    background: transparent;
    border: 1px solid rgba(148, 163, 184, 0.3);
    color: #cbd5f5;
  }

  .actions button:disabled {
    cursor: not-allowed;
    opacity: 0.6;
    box-shadow: none;
    transform: none;
  }

  .error {
    margin: 12px 0 0;
    color: #fca5a5;
    font-size: 0.85rem;
  }

  .stats {
    display: grid;
    grid-template-columns: 1fr;
    gap: 8px;
    margin-top: 16px;
    padding: 12px;
    background: rgba(15, 23, 42, 0.5);
    border-radius: 12px;
    font-size: 0.85rem;
  }

  .stats strong {
    display: block;
    font-size: 1rem;
    color: #f8fafc;
  }

  .list {
    margin: 16px 0 0;
    padding: 0;
    list-style: none;
    display: grid;
    gap: 8px;
    max-height: 200px;
    overflow: auto;
  }

  .list li {
    display: grid;
    grid-template-columns: auto 1fr auto;
    align-items: center;
    gap: 10px;
    font-size: 0.85rem;
    color: #e2e8f0;
    background: rgba(15, 23, 42, 0.4);
    border-radius: 10px;
    padding: 8px 10px;
  }

  .point-label {
    color: #93c5fd;
  }

  .point-coords {
    justify-self: start;
  }

  .point-delete {
    border: none;
    background: rgba(148, 163, 184, 0.15);
    color: #e2e8f0;
    width: 28px;
    height: 28px;
    border-radius: 999px;
    font-size: 1rem;
    line-height: 1;
    cursor: pointer;
  }

  .point-delete:hover {
    background: rgba(248, 113, 113, 0.25);
    color: #fecaca;
  }

  @media (max-width: 720px) {
    .panel {
      left: 16px;
      right: 16px;
      width: auto;
      top: auto;
      bottom: 16px;
    }
  }
</style>
