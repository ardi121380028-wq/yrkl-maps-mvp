<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'
import mapboxgl from 'mapbox-gl'
import { branding } from './config/branding'

// 1. DUMMY DATA: Nanti ini bakal ditarik dari Firebase, sekarang kita hardcode dulu
const corals = ref([
  { 
    id: 1, 
    name: 'Pahawang Nursery 1', 
    date: 'July 2024', 
    count: 2930, 
    status: 'healthy', 
    lng: 105.211099, 
    lat: -5.654185 
  },
  { 
    id: 2, 
    name: 'Legundi Rescue', 
    date: 'August 2024', 
    count: 850, 
    status: 'damaged', 
    lng: 105.150000, 
    lat: -5.750000 
  }
])

// 2. STATE UNTUK SIDEBAR: Menyimpan data koral mana yang lagi diklik
const selectedCoral = ref(null)
const activeTab = ref('healthy')
const activeMenu = ref('overview')
const isSidebarOpen = ref(false)
const isMonochromeEnabled = ref(false)
const isTopDown = ref(false)
const mapInstance = ref(null)

// 3. FUNGSI BUKA DETAIL: Dipanggil saat tombol di popup diklik
const openDetails = (coralId) => {
  selectedCoral.value = corals.value.find(c => c.id === coralId)
  activeMenu.value = 'nurseries'
  isSidebarOpen.value = true
}

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
      map.setPaintProperty(layer.id, 'raster-saturation', isMonochromeEnabled.value ? -1 : 0)
      map.setPaintProperty(layer.id, 'raster-brightness-min', isMonochromeEnabled.value ? 0.12 : 0)
      map.setPaintProperty(layer.id, 'raster-brightness-max', isMonochromeEnabled.value ? 0.68 : 1)
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
  activeTab.value = 'healthy'
}

onMounted(() => {
  // Wajib masukin API Key lu
  mapboxgl.accessToken = 'pk.eyJ1IjoieXJrbCIsImEiOiJjbXVpNno2ZmgwNWMxMndwdXlqeXpkbWxiIn0.gdse692iUvEiuPYsEMSowA'

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
  map.addControl(new mapboxgl.NavigationControl({ showCompass: false }), 'bottom-right')
  map.addControl(new mapboxgl.FullscreenControl(), 'top-right')
  map.addControl(new mapboxgl.ScaleControl({ maxWidth: 120, unit: 'metric' }), 'bottom-left')

  map.on('style.load', () => {
    map.setFog({ color: 'rgb(186, 210, 235)', 'space-color': 'rgb(11, 11, 25)' })
    
    // LAYER TERRAIN & 3D BANGUNAN (Sesuai kodingan kita sebelumnya)
    map.addSource('mapbox-dem', { 'type': 'raster-dem', 'url': 'mapbox://mapbox.mapbox-terrain-dem-v1', 'tileSize': 512, 'maxzoom': 14 })
    map.setTerrain({ 'source': 'mapbox-dem', 'exaggeration': 3.0 }) 
    setMapAppearance()
  })

  // 4. THE MAGIC: Looping data buat bikin Marker & Popup
  corals.value.forEach((coral) => {
    // Desain HTML Popup (Mirip referensi gambar lu)
    const popupHTML = `
      <div class="custom-popup">
        <div class="popup-heading">
          <div class="popup-icon">✣</div>
          <div>
            <span class="popup-label">CORAL NURSERY</span>
            <h4>${coral.name}</h4>
          </div>
        </div>
        <div class="popup-divider"></div>
        <div class="popup-row"><span>Creation Date</span><b>${coral.date}</b></div>
        <div class="popup-row"><span>Corals Growing</span><b>${coral.count.toLocaleString()}</b></div>
        <button onclick="window.dispatchEvent(new CustomEvent('open-detail', {detail: ${coral.id}}))" class="btn-detail">
          Open Details <span aria-hidden="true">→</span>
        </button>
      </div>
    `;

    const popup = new mapboxgl.Popup({ offset: 25 }).setHTML(popupHTML)

    const markerElement = document.createElement('button')
    markerElement.className = 'coral-marker'
    markerElement.type = 'button'
    markerElement.setAttribute('aria-label', `Open ${coral.name}`)
    markerElement.innerHTML = '<span class="coral-branch coral-branch-a"></span><span class="coral-branch coral-branch-b"></span><span class="coral-branch coral-branch-c"></span>'
    markerElement.addEventListener('click', () => openDetails(coral.id))

    new mapboxgl.Marker({ element: markerElement, anchor: 'center' })
      .setLngLat([coral.lng, coral.lat])
      .setPopup(popup)
      .addTo(map)
  })

  // 5. LISTENER: Nangkep klik dari tombol di dalem popup Mapbox
  window.addEventListener('open-detail', handleOpenDetail)
})

onBeforeUnmount(() => {
  window.removeEventListener('open-detail', handleOpenDetail)
  mapInstance.value?.remove()
})
</script>

<template>
  <div id="app-container">
    <nav class="navigation-rail" aria-label="Main navigation">
      <button class="rail-logo" aria-label="YRKL home" @click="selectMenu('overview')">
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
      <button class="rail-item rail-bottom" aria-label="Help">
        <span class="rail-icon">{{ branding.icons.help }}</span><span class="rail-tooltip">Help</span>
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
        <p class="eyebrow">MARINE CONSERVATION</p>
        <h1>YRKL <span>Tracker</span></h1>
        <p class="empty-state">Pilih salah satu titik nursery di peta untuk melihat detail.</p>
      </div>

      <div v-else-if="activeMenu === 'nurseries' && selectedCoral" class="coral-details">
        <div class="details-header">
          <div>
            <p class="eyebrow">CORAL NURSERY</p>
            <h2>{{ selectedCoral.name }}</h2>
          </div>
        </div>
        <div class="detail-stats">
          <div><span>Creation Date</span><strong>{{ selectedCoral.date }}</strong></div>
          <div><span>Corals Growing</span><strong>{{ selectedCoral.count.toLocaleString() }}</strong></div>
        </div>
        <div class="status-card" :class="selectedCoral.status">
          <span class="status-dot"></span>
          <div>
            <span class="status-caption">CURRENT STATUS</span>
            <h3 v-if="selectedCoral.status === 'healthy'">Healthy & Growing</h3>
            <h3 v-else>Bleaching / Damaged</h3>
          </div>
        </div>
        <div class="condition-tabs" role="tablist" aria-label="Coral condition">
          <button
            :class="{ active: activeTab === 'healthy' }"
            role="tab"
            :aria-selected="activeTab === 'healthy'"
            @click="activeTab = 'healthy'"
          >
            Healthy Coral
          </button>
          <button
            :class="{ active: activeTab === 'damaged' }"
            role="tab"
            :aria-selected="activeTab === 'damaged'"
            @click="activeTab = 'damaged'"
          >
            Damaged Coral
          </button>
        </div>
        <div class="coral-preview" :class="activeTab">
          <div class="preview-art" role="img" :aria-label="`${activeTab} coral image placeholder`">
            <span></span><span></span><span></span><span></span>
          </div>
          <div class="preview-copy">
            <span>{{ activeTab === 'healthy' ? 'HEALTHY CORAL' : 'DAMAGED CORAL' }}</span>
            <strong>{{ activeTab === 'healthy' ? 'Healthy growth' : 'Bleaching observed' }}</strong>
            <small>{{ activeTab === 'healthy' ? 'Strong color and active development' : 'Needs monitoring and restoration' }}</small>
          </div>
        </div>
      </div>

    </aside>

    <button class="bearing-toggle" type="button" :class="{ active: isTopDown }" :aria-pressed="isTopDown" aria-label="Toggle top-down map view" @click="toggleTopDown">
      <span aria-hidden="true">⌖</span>
    </button>

    <!-- MAPBOX CONTAINER -->
    <div id="map"></div>
  </div>
</template>