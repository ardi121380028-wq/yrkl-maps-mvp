# YRKL Tracker - Rumah Konservasi Laut Tracking

Aplikasi monitoring spasial dan kesehatan terumbu karang berbasis integrasi fotogrametri 3D di Kepulauan Pahawang, Lampung, Indonesia.

## Fitur Utama
- **3D Coral Mapping & Globe**: Peta interaktif dengan elevasi terrain 3D, proyeksi globe, dan citra satelit resolusi tinggi.
- **Auto-Failover Satellite & Terrain**: Sistem cadangan cerdas untuk memastikan visualisasi 3D tetap aktif secara optimal.
- **Titik Pantau Konservasi (Coral Nurseries)**:
  - Gosong Bekri
  - Gosong Kalangan
  - Pahawang Tea
- **High-Resolution Coral Health Map Viewer**: Penampil peta ortofoto drone detail dengan interaksi zoom dan pan.
- **Altitude HUD & Perspective Control**: Kontrol sudut pandang 2D (top-down) dan 3D perspective 45°.

## Teknologi
- [Vue 3](https://vuejs.org/) (Composition API, `<script setup>`)
- [Vite](https://vitejs.dev/)
- [Mapbox GL JS](https://docs.mapbox.com/mapbox-gl-js/)

## Menjalankan Proyek
```bash
# Instal dependensi
npm install

# Jalankan server pengembangan
npm run dev

# Bangun untuk produksi
npm run build
```
