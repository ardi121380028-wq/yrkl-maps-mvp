<script setup>
import { onBeforeUnmount, onMounted, ref, shallowRef } from 'vue'
import mapboxgl from 'mapbox-gl'
import { branding } from './config/branding'
import markerIcon from './assets/Icon.svg'

// Data sementara sebelum sumber data terhubung ke Firebase.
const corals = [
  {
    id: 1,
    name: 'Gosong Bekri',
    mapTitle: 'Peta Sebaran Terumbu Karang Lokasi Gosong Bekri Pahawang, Lampung',
    lifeCoral: '20.819 m²',
    totalArea: '192.507 m²',
    status: 'healthy',
    lng: 105.2058,
    lat: -5.6591,
    image: '/Gosong Bekri.jpg.jpeg'
  },
  {
    id: 2,
    name: 'Gosong Kalangan',
    mapTitle: 'Peta Sebaran Terumbu Karang Lokasi Gosong Kalangan Pahawang, Lampung',
    lifeCoral: '77.261 m²',
    totalArea: '231.977 m²',
    status: 'healthy',
    lng: 105.1853,
    lat: -5.6665,
    image: '/Gosong Kalangan 2.jpg.jpeg'
  },
  {
    id: 3,
    name: 'Pahawang Tea',
    mapTitle: 'Peta Sebaran Terumbu Karang Lokasi Pahawang Tea Pahawang, Lampung',
    lifeCoral: null,
    totalArea: '198.728 m²',
    status: 'healthy',
    lng: 105.2254,
    lat: -5.6482,
    image: '/Pahawang Tea 4.jpg.jpeg'
  }
]

// 2. STATE UNTUK SIDEBAR & UI
const selectedCoral = ref(null)
const activeMenu = ref('overview')
const isSidebarOpen = ref(false)
const isMonochromeEnabled = ref(false)
const isTopDown = ref(false)
const mapInstance = shallowRef(null)
const mapZoomLevel = ref(13)
let zoomFrame = 0
const mapViewerCoral = ref(null)
const mapZoom = ref(1)
const mapOffset = ref({ x: 0, y: 0 })
const isMapDragging = ref(false)
const dragOrigin = ref({ x: 0, y: 0 })

// State untuk Token Mapbox & Fallback Satelit/Medan 3D
const mapboxToken = ref(
  localStorage.getItem('yrkl_custom_mapbox_token') ||
  import.meta.env.VITE_MAPBOX_TOKEN ||
  'pk.eyJ1IjoiaXJzZ2VlZGVzaWduIiwiYSI6ImNtd2M0aDg0ajAwY2YyeXBxcmtnMXA2MHcifQ.5Dh6Ynsv_M4bDWLU8pLjvQ'
)
const isFallbackActive = ref(false)
const showTokenBanner = ref(false)
const showTokenModal = ref(false)
const inputToken = ref('')
const fallbackReason = ref('')

// 3. FUNGSI BUKA DETAIL: Dipanggil saat tombol di popup diklik
const openDetails = (coralId) => {
  selectedCoral.value = corals.find(c => c.id === coralId)
  activeMenu.value = 'nurseries'
  isSidebarOpen.value = true
}

const openMap = (coralId) => {
  const coral = corals.find(c => c.id === coralId)
  selectedCoral.value = coral
  activeMenu.value = 'nurseries'
  isSidebarOpen.value = true
  mapViewerCoral.value = coral
  mapZoom.value = 1
  mapOffset.value = { x: 0, y: 0 }
}

const closeMap = () => {
  mapViewerCoral.value = null
  mapZoom.value = 1
  mapOffset.value = { x: 0, y: 0 }
}

const zoomMap = (amount) => {
  mapZoom.value = Math.min(10, Math.max(1, Number((mapZoom.value + amount).toFixed(1))))
}

const handleMapWheel = (event) => {
  zoomMap(event.deltaY < 0 ? 0.1 : -0.1)
}

const startMapDrag = (event) => {
  if (event.button !== 0) return
  event.preventDefault()
  isMapDragging.value = true
  dragOrigin.value = {
    x: event.clientX - mapOffset.value.x,
    y: event.clientY - mapOffset.value.y
  }
  event.currentTarget.setPointerCapture(event.pointerId)
}

const moveMap = (event) => {
  if (!isMapDragging.value) return
  event.preventDefault()
  mapOffset.value = {
    x: event.clientX - dragOrigin.value.x,
    y: event.clientY - dragOrigin.value.y
  }
}

const stopMapDrag = (event) => {
  if (!isMapDragging.value) return
  isMapDragging.value = false
  if (event.currentTarget && event.currentTarget.hasPointerCapture(event.pointerId)) {
    event.currentTarget.releasePointerCapture(event.pointerId)
  }
}

const getZoomProgress = () => `${Math.round(Math.max(0, Math.min(1, (mapZoomLevel.value - 1) / 11)) * 100)}%`

const selectMenu = (menu) => {
  activeMenu.value = menu
  isSidebarOpen.value = true
  if (menu === 'overview') {
    selectedCoral.value = null
  }
}

const toggleSidebar = () => {
  isSidebarOpen.value = !isSidebarOpen.value
}

const setMapAppearance = () => {
  const map = mapInstance.value
  if (!map || !map.isStyleLoaded()) return

  for (const layer of Object.values(map.getStyle().layers || [])) {
    if (layer.type === 'raster') {
      try {
        map.setPaintProperty(layer.id, 'raster-saturation', isMonochromeEnabled.value ? -1 : 0)
        map.setPaintProperty(layer.id, 'raster-brightness-min', isMonochromeEnabled.value ? 0.12 : 0)
        map.setPaintProperty(layer.id, 'raster-brightness-max', isMonochromeEnabled.value ? 0.68 : 1)
      } catch {
        // Abaikan layer raster yang tidak mendukung brightness-min/max
      }
    }
  }
}

const toggleMonochrome = () => {
  isMonochromeEnabled.value = !isMonochromeEnabled.value
  setMapAppearance()
}

const toggleTopDown = () => {
  const map = mapInstance.value
  if (!map) return

  isTopDown.value = !isTopDown.value
  map.easeTo({ pitch: isTopDown.value ? 0 : 45, bearing: 0, duration: 700 })
}

const handleOpenDetail = (event) => {
  openDetails(event.detail)
}

const handleOpenMap = (event) => {
  openMap(event.detail)
}

// FUNGSI AKTIVASI FALLBACK SATELIT & MEDAN 3D SAAT TOKEN MAPBOX KENA RATE LIMIT / 429
const activateFallback = (reason = 'Batas Kuota Mapbox Tercapai (HTTP 429)') => {
  const map = mapInstance.value
  if (!map) return

  fallbackReason.value = reason
  showTokenBanner.value = true

  if (isFallbackActive.value) return
  isFallbackActive.value = true

  try {
    // 1. Fallback Citra Satelit Resolusi Tinggi (Esri World Imagery)
    if (!map.getSource('esri-fallback-sat')) {
      map.addSource('esri-fallback-sat', {
        type: 'raster',
        tiles: ['https://services.arcgisonline.com/arcgis/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}'],
        tileSize: 256,
        maxzoom: 19
      })
    }

    if (!map.getLayer('esri-fallback-layer')) {
      const layers = map.getStyle()?.layers || []
      const firstSymbol = layers.find(l => l.type === 'symbol')
      map.addLayer({
        id: 'esri-fallback-layer',
        type: 'raster',
        source: 'esri-fallback-sat'
      }, firstSymbol ? firstSymbol.id : undefined)
    }

    // 2. Fallback Medan 3D DEM (AWS Terrarium DEM)
    if (!map.getSource('aws-terrain-dem')) {
      map.addSource('aws-terrain-dem', {
        type: 'raster-dem',
        tiles: ['https://s3.amazonaws.com/elevation-tiles-prod/terrarium/{z}/{x}/{y}.png'],
        encoding: 'terrarium',
        tileSize: 256,
        maxzoom: 15
      })
    }
    map.setTerrain({ source: 'aws-terrain-dem', exaggeration: 1.6 })

    setMapAppearance()
  } catch (err) {
    console.warn('Fallback activation notice:', err)
  }
}

const saveCustomToken = () => {
  if (!inputToken.value.trim()) return
  localStorage.setItem('yrkl_custom_mapbox_token', inputToken.value.trim())
  window.location.reload()
}

const resetCustomToken = () => {
  localStorage.removeItem('yrkl_custom_mapbox_token')
  window.location.reload()
}

onMounted(() => {
  mapboxgl.accessToken = mapboxToken.value

  const map = new mapboxgl.Map({
    container: 'map',
    style: 'mapbox://styles/mapbox/satellite-streets-v12',
    center: [105.211099, -5.654185],
    zoom: 13,
    pitch: 45,
    projection: 'globe',
    attributionControl: false
  })
  mapInstance.value = map

  map.on('zoom', () => {
    if (zoomFrame) return
    zoomFrame = requestAnimationFrame(() => {
      mapZoomLevel.value = map.getZoom()
      zoomFrame = 0
    })
  })
  map.addControl(new mapboxgl.NavigationControl({ showCompass: false }), 'bottom-right')
  map.addControl(new mapboxgl.FullscreenControl(), 'top-right')

  map.on('style.load', () => {
    map.setFog({ color: 'rgb(186, 210, 235)', 'space-color': 'rgb(11, 11, 25)' })
    
    // LAYER TERRAIN MAPBOX DEM
    if (!map.getSource('mapbox-dem')) {
      map.addSource('mapbox-dem', {
        type: 'raster-dem',
        url: 'mapbox://mapbox.mapbox-terrain-dem-v1',
        tileSize: 512,
        maxzoom: 14
      })
    }
    map.setTerrain({ source: 'mapbox-dem', exaggeration: 1.8 })

    // LAYER 3D BANGUNAN (Fill Extrusion)
    if (!map.getLayer('3d-buildings') && map.getSource('composite')) {
      const layers = map.getStyle().layers || []
      const labelLayerId = layers.find(l => l.type === 'symbol' && l.layout?.['text-field'])?.id
      map.addLayer({
        id: '3d-buildings',
        source: 'composite',
        'source-layer': 'building',
        filter: ['==', 'extrude', 'true'],
        type: 'fill-extrusion',
        minzoom: 15,
        paint: {
          'fill-extrusion-color': '#aaa',
          'fill-extrusion-height': [
            'interpolate', ['linear'], ['zoom'],
            15, 0,
            15.05, ['get', 'height']
          ],
          'fill-extrusion-base': [
            'interpolate', ['linear'], ['zoom'],
            15, 0,
            15.05, ['get', 'min_height']
          ],
          'fill-extrusion-opacity': 0.6
        }
      }, labelLayerId)
    }

    setMapAppearance()
  })

  // Deteksi error 429 atau 401/403 dari Mapbox
  map.on('error', (e) => {
    const status = e?.error?.status
    if (status === 429 || status === 401 || status === 403 || e?.error?.message?.includes('429')) {
      activateFallback(status === 429 ? 'Batas Kuota Mapbox Tercapai (HTTP 429)' : 'Token Mapbox Ditolak (Error Autentikasi)')
    }
  })

  // Uji koneksi satelit Mapbox secara proaktif
  fetch(`https://api.mapbox.com/v4/mapbox.satellite/1/0/0.webp?access_token=${mapboxToken.value}`)
    .then(res => {
      if (res.status === 429 || res.status === 401 || res.status === 403) {
        activateFallback(`Batas Kuota Mapbox Tercapai (HTTP ${res.status})`)
      }
    })
    .catch(() => {})

  // 4. THE MAGIC: Looping data buat bikin Marker & Popup
  corals.forEach((coral) => {
    const popupHTML = `
      <div class="custom-popup">
        <div class="popup-heading">
          <div class="popup-icon"><img src="${markerIcon}" alt="" /></div>
          <div>
            <span class="popup-label">CORAL NURSERY</span>
            <h4>${coral.name}</h4>
          </div>
        </div>
        <div class="popup-divider"></div>
        <div class="popup-row"><span>Life Coral</span><b>${coral.lifeCoral || '—'}</b></div>
        <div class="popup-row"><span>Luasan Total</span><b>${coral.totalArea}</b></div>
        <div class="popup-actions">
          <button onclick="window.dispatchEvent(new CustomEvent('open-detail', {detail: ${coral.id}}))" class="btn-detail">Open Detail</button>
          <button onclick="window.dispatchEvent(new CustomEvent('open-map', {detail: ${coral.id}}))" class="btn-map">Open Map</button>
        </div>
      </div>
    `;

    const popup = new mapboxgl.Popup({ offset: 25 }).setHTML(popupHTML)

    const markerElement = document.createElement('button')
    markerElement.className = 'coral-marker'
    markerElement.type = 'button'
    markerElement.setAttribute('aria-label', `Show summary for ${coral.name}`)
    markerElement.innerHTML = `<span class="marker-orbit"></span><img src="${markerIcon}" alt="" />`

    new mapboxgl.Marker({ element: markerElement, anchor: 'center' })
      .setLngLat([coral.lng, coral.lat])
      .setPopup(popup)
      .addTo(map)
  })

  // 5. LISTENER: Nangkep klik dari tombol di dalem popup Mapbox
  window.addEventListener('open-detail', handleOpenDetail)
  window.addEventListener('open-map', handleOpenMap)
})

onBeforeUnmount(() => {
  if (zoomFrame) cancelAnimationFrame(zoomFrame)
  window.removeEventListener('open-detail', handleOpenDetail)
  window.removeEventListener('open-map', handleOpenMap)
  mapInstance.value?.remove()
})
</script>

<template>
  <div id="app-container">
    <!-- BANNER PERINGATAN KUOTA TOKEN MAPBOX & STATUS FALLBACK -->
    <div v-if="showTokenBanner" class="token-warning-banner">
      <div class="token-banner-content">
        <span class="token-banner-icon">ℹ️</span>
        <div class="token-banner-text">
          <strong>{{ fallbackReason || 'Satelit Cadangan Aktif' }}</strong>
          <span>Menampilkan peta satelit (Esri) dan kontur medan 3D otomatis. Peta tetap dapat digunakan secara optimal.</span>
        </div>
      </div>
      <div class="token-banner-actions">
        <button type="button" class="btn-token-action" @click="showTokenModal = true">Ganti Token Mapbox</button>
        <button type="button" class="btn-token-close" aria-label="Tutup" @click="showTokenBanner = false">×</button>
      </div>
    </div>

    <!-- MODAL GANTI TOKEN MAPBOX -->
    <div v-if="showTokenModal" class="token-modal-backdrop" @click.self="showTokenModal = false">
      <div class="token-modal">
        <header class="token-modal-header">
          <h3>Konfigurasi Token Mapbox</h3>
          <button type="button" class="token-modal-close" @click="showTokenModal = false">×</button>
        </header>
        <div class="token-modal-body">
          <p>
            Akun Mapbox saat ini telah mencapai limit kuota (HTTP 429). Untuk menggunakan citra satelit bawaan Mapbox, Anda dapat membuat akun baru gratis di
            <a href="https://account.mapbox.com/" target="_blank" rel="noopener">mapbox.com</a> dan memasukkan token publik (<code>pk...</code>) di bawah ini:
          </p>
          <input
            v-model="inputToken"
            type="text"
            placeholder="pk.eyJ1Ijoi..."
            class="token-input"
          />
          <div class="token-modal-buttons">
            <button type="button" class="btn-modal-save" @click="saveCustomToken">Simpan &amp; Muat Ulang</button>
            <button type="button" class="btn-modal-reset" @click="resetCustomToken">Reset ke Awal</button>
          </div>
        </div>
      </div>
    </div>

    <nav class="navigation-rail" aria-label="Main navigation">
      <button class="rail-logo" aria-label="Rumah Konservasi Laut Tracking home" @click="selectMenu('overview')">
        <img :src="branding.logo" :alt="branding.appName" />
      </button>
      <div class="rail-menu">
        <button
          class="rail-item"
          :class="{ active: activeMenu === 'overview' }"
          aria-label="Overview"
          @click="selectMenu('overview')"
        >
          <span class="rail-icon">{{ branding.icons.overview }}</span><span class="rail-tooltip">Overview</span>
        </button>
        <button
          class="rail-item"
          :class="{ active: activeMenu === 'overlay' }"
          aria-label="Map overlays"
          @click="selectMenu('overlay')"
        >
          <span class="rail-icon">{{ branding.icons.overlay }}</span><span class="rail-tooltip">Overlay</span>
        </button>
        <button
          class="rail-item"
          :class="{ active: activeMenu === 'nurseries' }"
          aria-label="Coral nurseries"
          @click="selectMenu('nurseries')"
        >
          <span class="rail-icon">{{ branding.icons.nurseries }}</span><span class="rail-tooltip">Coral Nurseries</span>
        </button>
      </div>
      <button class="rail-item rail-bottom" aria-label="Help" @click="showTokenModal = true">
        <span class="rail-icon">⚙</span><span class="rail-tooltip">Pengaturan Token</span>
      </button>
    </nav>

    <!-- PANEL SIDEBAR -->
    <button
      class="sidebar-toggle"
      :class="{ open: isSidebarOpen }"
      type="button"
      :aria-label="isSidebarOpen ? 'Close sidebar' : 'Open sidebar'"
      :aria-expanded="isSidebarOpen"
      @click="toggleSidebar"
    >
      {{ isSidebarOpen ? '×' : '‹' }}
    </button>
    <aside id="sidebar" :class="{ 'is-open': isSidebarOpen }" aria-label="Map sidebar">
      <button v-if="isSidebarOpen" class="sidebar-close" type="button" aria-label="Close sidebar" @click="isSidebarOpen = false">×</button>
      <div v-if="activeMenu === 'overlay'" class="overlay-panel">
        <p class="eyebrow">DATA OVERLAYS</p>
        <h2>Map Layers</h2>
        <label class="layer-row">
          <span>Black &amp; White Map</span>
          <input :checked="isMonochromeEnabled" type="checkbox" role="switch" @change="toggleMonochrome" />
        </label>
      </div>
      <div v-else-if="activeMenu === 'overview' && !selectedCoral" class="menu-home">
        <div class="brand-mark">
          <img :src="branding.logo" :alt="branding.appName" />
        </div>
        <p class="eyebrow">RUMAH KONSERVASI LAUT</p>
        <h1>Rumah Konservasi Laut <span>Tracking</span></h1>
        <p class="empty-state">Reef health mapping berbasis integrasi 3D photogrammetry untuk monitoring spasial dalam mendukung pengelolaan dan konservasi terumbu karang berkelanjutan.</p>
      </div>

      <div v-else-if="activeMenu === 'nurseries' && selectedCoral" class="coral-details">
        <div class="details-header">
          <div>
            <p class="eyebrow">CORAL NURSERY</p>
            <h2>{{ selectedCoral.name }}</h2>
          </div>
        </div>
        <div class="detail-stats">
          <div><span>Life Coral</span><strong>{{ selectedCoral.lifeCoral || '—' }}</strong></div>
          <div><span>Luasan Total</span><strong>{{ selectedCoral.totalArea }}</strong></div>
        </div>
        <div class="sidebar-map-card">
          <div class="sidebar-map-heading">
            <span class="compass" aria-label="North compass">N</span>
            <div>
              <span class="map-label">PETA SEBARAN TERUMBU KARANG</span>
              <strong>{{ selectedCoral.name }}</strong>
            </div>
          </div>
          <div class="indonesia-map" role="img" :aria-label="`Indonesia overview map with ${selectedCoral.name} marker`">
            <img src="/Map Square.svg" alt="" />
            <span class="map-location-dot"></span>
          </div>
        </div>
        <div class="sidebar-description">
          <p class="eyebrow">KETERANGAN PETA</p>
          <h3>{{ selectedCoral.mapTitle }}</h3>
          <p>Peta sebaran menunjukkan area terumbu karang yang dipetakan di sekitar lokasi {{ selectedCoral.name }}, Pahawang, Lampung.</p>
        </div>
      </div>

    </aside>

    <button class="bearing-toggle" type="button" :class="{ active: isTopDown }" :aria-pressed="isTopDown" aria-label="Toggle top-down map view" @click="toggleTopDown">
      <span aria-hidden="true">⌖</span>
    </button>

    <!-- MAPBOX CONTAINER -->
    <div id="map"></div>
    <div v-if="mapZoomLevel < 6" class="zoomed-out-stars" aria-hidden="true">
      <span class="star-layer star-layer-back"></span>
      <span class="star-layer star-layer-mid"></span>
      <span class="star-layer star-layer-front"></span>
    </div>
    <div class="map-distance-hud" :style="{ '--zoom-progress': getZoomProgress() }" aria-label="Map scale indicator">
      <span>ALTITUDE</span>
      <div class="hud-track">
        <i></i>
        <div class="hud-ticks"><b v-for="tick in 10" :key="tick"></b></div>
      </div>
      <strong>{{ mapZoomLevel < 5 ? '12,619 KM' : mapZoomLevel < 9 ? '1,250 KM' : '5 KM' }}</strong>
    </div>

    <div v-if="mapViewerCoral" class="map-viewer-backdrop" :class="{ 'with-sidebar': isSidebarOpen }" @click.self="closeMap">
      <section class="map-viewer" aria-label="Large coral health map viewer">
        <header class="map-viewer-header">
          <div>
            <p class="eyebrow">CORAL HEALTH MAP</p>
            <h2>{{ mapViewerCoral.name }}</h2>
          </div>
          <div class="map-viewer-actions">
            <button type="button" @click="zoomMap(-0.5)" :disabled="mapZoom <= 1" aria-label="Zoom out">−</button>
            <span>{{ Math.round(mapZoom * 100) }}%</span>
            <button type="button" @click="zoomMap(0.5)" :disabled="mapZoom >= 10" aria-label="Zoom in">+</button>
            <button type="button" class="viewer-close" @click="closeMap" aria-label="Close map viewer">×</button>
          </div>
        </header>
        <div
          class="map-image-viewport"
          :class="{ dragging: isMapDragging }"
          @wheel.prevent="handleMapWheel"
          @pointerdown="startMapDrag"
          @pointermove="moveMap"
          @pointerup="stopMapDrag"
          @pointercancel="stopMapDrag"
          @lostpointercapture="stopMapDrag"
        >
          <img
            :src="mapViewerCoral.image"
            :alt="mapViewerCoral.mapTitle"
            :style="{ transform: `scale(${mapZoom}) translate(${mapOffset.x / mapZoom}px, ${mapOffset.y / mapZoom}px)` }"
            draggable="false"
          />
        </div>
      </section>
    </div>
  </div>
</template>
