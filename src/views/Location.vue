<template>
  <main class="container" role="main" aria-label="Lokasi Petani & Penggilingan">
    <div class="headline-card">
      <h1 class="headline">Lokasi Petani & Penggilingan</h1>
    </div>

    <!-- Map Section -->
    <section class="map-section" aria-labelledby="map-title">
      <header class="map-header">
        <h2 id="map-title" class="section-title">Peta Lokasi</h2>
        <div class="button-group">
          <button @click="getCurrentLocation" class="location-button" :disabled="loadingLocation">
            <component :is="loadingLocation ? 'LoaderIcon' : 'MapPinIcon'" class="icon" />
            {{ loadingLocation ? 'Memuat...' : 'Lokasi Saya' }}
          </button>
          <button @click="refreshData" class="refresh-button" :disabled="loadingData">
            <component :is="loadingData ? 'LoaderIcon' : 'RefreshCwIcon'" class="icon" />
            {{ loadingData ? 'Memuat...' : 'Refresh Data' }}
          </button>
        </div>
      </header>

      <div class="map-wrapper">
        <div id="map" class="map-container"></div>

        <!-- Floating controls -->
        <div class="floating-controls">
          <button @click="zoomIn" class="control-btn">
            <PlusIcon class="icon" />
          </button>
          <button @click="zoomOut" class="control-btn">
            <MinusIcon class="icon" />
          </button>
          <button @click="getCurrentLocation" class="control-btn" :disabled="loadingLocation">
            <component :is="loadingLocation ? 'LoaderIcon' : 'NavigationIcon'" class="icon" />
          </button>
          <button @click="toggleMapType" class="control-btn">
            <LayersIcon class="icon" />
          </button>
        </div>
      </div>

      <!-- Location info panel -->
      <div class="info-panel">
        <div class="info-item" v-if="currentLocation">
          <MapPinIcon class="info-icon" />
          <div>
            <p class="info-label">Koordinat Anda</p>
            <p class="info-value">{{ currentLocation.lat }}, {{ currentLocation.lng }}</p>
          </div>
        </div>
        <div class="info-item" v-if="products.length > 0">
          <PackageIcon class="info-icon" />
          <div>
            <p class="info-label">Total Produk</p>
            <p class="info-value">{{ products.length }} lokasi</p>
          </div>
        </div>
      </div>
    </section>

    <!-- Product Info Modal -->
    <div v-if="selectedProduct" class="product-modal-overlay" @click="closeProductModal">
      <div class="product-modal" @click.stop>
        <div class="product-modal-header">
          <h4>Detail Produk</h4>
          <button @click="closeProductModal" class="close-btn">
            <XIcon class="icon" />
          </button>
        </div>
        <div class="product-modal-content">
          <div class="product-info-row">
            <UserIcon class="product-icon" />
            <div>
              <p class="product-label">Email</p>
              <p class="product-value">{{ selectedProduct.userEmail }}</p>
            </div>
          </div>
          <div class="product-info-row">
            <PackageIcon class="product-icon" />
            <div>
              <p class="product-label">Jenis Produk</p>
              <p class="product-value">{{ selectedProduct.productType }}</p>
            </div>
          </div>
          <div class="product-info-row">
            <ScaleIcon class="product-icon" />
            <div>
              <p class="product-label">Berat</p>
              <p class="product-value">{{ selectedProduct.weight }} kg</p>
            </div>
          </div>
          <div class="product-info-row">
            <CoinsIcon class="product-icon" />
            <div>
              <p class="product-label">Harga</p>
              <p class="product-value">Rp {{ formatPrice(selectedProduct.price) }}</p>
            </div>
          </div>
          <div class="product-info-row">
            <MapPinIcon class="product-icon" />
            <div>
              <p class="product-label">Alamat</p>
              <p class="product-value">{{ selectedProduct.locationAddress }}</p>
            </div>
          </div>
          <div class="product-info-row">
            <ClockIcon class="product-icon" />
            <div>
              <p class="product-label">Waktu</p>
              <p class="product-value">{{ formatDate(selectedProduct.timestamp) }}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<script>
import {
  MapPinIcon,
  PlusIcon,
  MinusIcon,
  NavigationIcon,
  RotateCcwIcon,
  LayersIcon,
  LoaderIcon,
  RefreshCwIcon,
  PackageIcon,
  UserIcon,
  ScaleIcon,
  CoinsIcon,
  ClockIcon,
  XIcon
} from 'lucide-vue-next'

// Import Firebase (pastikan sudah dikonfigurasi di project Anda)
import { getDatabase, ref, onValue, off } from 'firebase/database'
import { getAuth } from 'firebase/auth'

export default {
  name: 'LocationView',
  components: {
    MapPinIcon,
    PlusIcon,
    MinusIcon,
    NavigationIcon,
    RotateCcwIcon,
    LayersIcon,
    LoaderIcon,
    RefreshCwIcon,
    PackageIcon,
    UserIcon,
    ScaleIcon,
    CoinsIcon,
    ClockIcon,
    XIcon
  },
  data() {
    return {
      map: null,
      currentLocation: null,
      loadingLocation: false,
      loadingData: false,
      userMarker: null,
      productMarkers: [],
      products: [],
      selectedProduct: null,
      defaultCenter: [-7.7956, 110.3695], // Yogyakarta coordinates
      defaultZoom: 13,
      mapType: 'light', // 'light' or 'dark'
      L: null, // Store Leaflet reference
      database: null,
      auth: null,
      productsRef: null
    }
  },
  async mounted() {
    await this.initFirebase()
    await this.initMap()
    this.loadProductsFromFirebase()
  },
  beforeUnmount() {
    // Clean up global function
    if (window.openProductDetail) {
      delete window.openProductDetail
    }

    // Clean up Firebase listener
    if (this.productsRef && this.database) {
      off(this.productsRef)
    }

    // Clean up markers and map
    if (this.map) {
      try {
        // Tutup semua popup dan hapus event listeners
        this.map.closePopup()

        // Clean up semua layer dengan popup
        this.map.eachLayer((layer) => {
          if (layer.off) {
            layer.off()
          }
          if (layer.getPopup && layer.getPopup()) {
            layer.closePopup()
            layer.unbindPopup()
          }
        })

        // Clean up product markers
        this.productMarkers.forEach(marker => {
          try {
            marker.off()
            if (marker.getPopup && marker.getPopup()) {
              marker.closePopup()
              marker.unbindPopup()
            }
            if (this.map.hasLayer(marker)) {
              this.map.removeLayer(marker)
            }
          } catch (error) {
            console.warn('Error cleaning up product marker:', error)
          }
        })
        this.productMarkers = []

        // Clean up user marker
        if (this.userMarker) {
          try {
            this.userMarker.off()
            if (this.userMarker.getPopup && this.userMarker.getPopup()) {
              this.userMarker.closePopup()
              this.userMarker.unbindPopup()
            }
            if (this.map.hasLayer(this.userMarker)) {
              this.map.removeLayer(this.userMarker)
            }
            this.userMarker = null
          } catch (error) {
            console.warn('Error cleaning up user marker:', error)
          }
        }

        // Hapus semua event listeners dari map
        this.map.off()

        // Remove map
        this.map.remove()
        this.map = null
      } catch (error) {
        console.warn('Error cleaning up map:', error)
      }
    }
  },
  methods: {
    async initFirebase() {
      try {
        this.database = getDatabase()
        this.auth = getAuth()
        this.productsRef = ref(this.database, 'products')
      } catch (error) {
        console.error('Error initializing Firebase:', error)
      }
    },

    loadProductsFromFirebase() {
      if (!this.productsRef || !this.auth.currentUser) {
        console.error('Firebase not initialized or user not authenticated')
        return
      }

      this.loadingData = true

      onValue(this.productsRef, (snapshot) => {
        try {
          const data = snapshot.val()
          this.products = []

          if (data) {
            Object.keys(data).forEach(key => {
              const product = {
                id: key,
                ...data[key]
              }

              // Filter hanya produk Gabah dan Beras
              if (product.productType === 'Gabah' || product.productType === 'Beras') {
                // Pastikan produk memiliki koordinat yang valid
                if (product.locationCoordinates) {
                  const coords = product.locationCoordinates.split(',')
                  if (coords.length === 2) {
                    product.lat = parseFloat(coords[0].trim())
                    product.lng = parseFloat(coords[1].trim())

                    // Hanya tambahkan jika koordinat valid
                    if (!isNaN(product.lat) && !isNaN(product.lng)) {
                      this.products.push(product)
                    }
                  }
                }
              }
            })
          }

          this.updateProductMarkers()
          this.loadingData = false
        } catch (error) {
          console.error('Error processing products data:', error)
          this.loadingData = false
        }
      }, (error) => {
        console.error('Error loading products:', error)
        this.loadingData = false
      })
    },

    updateProductMarkers() {
      if (!this.map || !this.L) return

      // Tutup semua popup yang terbuka sebelum menghapus markers
      this.map.closePopup()

      // Hapus marker produk yang ada dengan aman
      this.productMarkers.forEach(marker => {
        // Hapus event listeners terlebih dahulu
        marker.off()
        // Tutup popup jika ada
        if (marker.getPopup()) {
          marker.closePopup()
          marker.unbindPopup()
        }
        // Hapus marker dari map
        this.map.removeLayer(marker)
      })
      this.productMarkers = []

      // Tambahkan marker untuk setiap produk
      this.products.forEach(product => {
        const markerColor = this.getMarkerColor(product.productType)
        const markerIcon = this.createCustomMarker(markerColor, product.productType)

        const marker = this.L.marker([product.lat, product.lng], {
          icon: markerIcon
        })
          .addTo(this.map)

        // Buat popup content
        const popupContent = this.createPopupContent(product)
        marker.bindPopup(popupContent)

        // Event listener untuk membuka modal detail
        marker.on('click', (e) => {
          // Tutup popup yang mungkin terbuka
          this.map.closePopup()
          // Buka modal
          this.selectedProduct = product
          // Prevent event bubbling
          e.originalEvent.stopPropagation()
        })

        this.productMarkers.push(marker)
      })
    },

    getMarkerColor(productType) {
      const colors = {
        'Gabah': '#000000',
        'Beras': '#000000',
        default: '#6b7280'
      }
      return colors[productType] || colors.default
    },

    createCustomMarker(color, productType) {
      const iconHtml = `
        <div class="custom-marker" style="background-color: ${color}">
          <div class="marker-inner">
            ${productType.charAt(0)}
          </div>
        </div>
      `

      return this.L.divIcon({
        className: 'custom-marker-wrapper',
        html: iconHtml,
        iconSize: [30, 30],
        iconAnchor: [15, 30],
        popupAnchor: [0, -30]
      })
    },

    createPopupContent(product) {
      return `
        <div class="popup-content">
          <h4>${product.productType}</h4>
          <p><strong>Berat:</strong> ${product.weight} kg</p>
          <p><strong>Harga:</strong> Rp ${this.formatPrice(product.price)}</p>
          <p><strong>Dari:</strong> ${product.userEmail}</p>
          <button class="popup-btn" onclick="window.openProductDetail('${product.id}')">
            Lihat Detail
          </button>
        </div>
      `
    },

    refreshData() {
      // Tutup modal yang terbuka
      this.selectedProduct = null

      // Tutup semua popup
      if (this.map) {
        this.map.closePopup()
      }

      // Reload data dari Firebase
      this.loadProductsFromFirebase()
    },

    closeProductModal() {
      this.selectedProduct = null
    },

    formatPrice(price) {
      return new Intl.NumberFormat('id-ID').format(price)
    },

    formatDate(timestamp) {
      return new Date(timestamp).toLocaleString('id-ID', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      })
    },

    async initMap() {
      try {
        // Load Leaflet CSS and JS
        await this.loadLeaflet()

        // Initialize map
        this.map = this.L.map('map', {
          center: this.defaultCenter,
          zoom: this.defaultZoom,
          zoomControl: false, // We'll use custom controls
          attributionControl: true
        })

        // Add tile layer with CartoDB light theme (black & white style like Android)
        this.addTileLayer()

        // Try to get user's current location
        this.getCurrentLocation()

        // Tidak perlu setup global function openProductDetail lagi
      } catch (error) {
        console.error('Error initializing map:', error)
      }
    },

    async loadLeaflet() {
      // Load Leaflet CSS
      if (!document.querySelector('link[href*="leaflet"]')) {
        const cssLink = document.createElement('link')
        cssLink.rel = 'stylesheet'
        cssLink.href = 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.css'
        document.head.appendChild(cssLink)
      }

      // Load Leaflet JS
      if (!window.L) {
        return new Promise((resolve, reject) => {
          const script = document.createElement('script')
          script.src = 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.js'
          script.onload = () => {
            this.L = window.L
            resolve()
          }
          script.onerror = reject
          document.head.appendChild(script)
        })
      } else {
        this.L = window.L
      }
    },

    addTileLayer() {
      if (!this.map || !this.L) return

      // Remove existing tile layer
      this.map.eachLayer((layer) => {
        if (layer instanceof this.L.TileLayer) {
          this.map.removeLayer(layer)
        }
      })

      // CartoDB light theme for black & white appearance
      const tileLayer = this.L.tileLayer(
        'https://cartodb-basemaps-{s}.global.ssl.fastly.net/light_all/{z}/{x}/{y}.png',
        {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
          subdomains: ['a', 'b', 'c', 'd'],
          maxZoom: 19
        }
      )

      tileLayer.addTo(this.map)
    },

    getCurrentLocation() {
      if (!navigator.geolocation) {
        alert('Geolocation tidak didukung oleh browser ini.')
        return
      }

      this.loadingLocation = true

      navigator.geolocation.getCurrentPosition(
        (position) => {
          const lat = position.coords.latitude
          const lng = position.coords.longitude

          this.currentLocation = {
            lat: parseFloat(lat.toFixed(6)),
            lng: parseFloat(lng.toFixed(6))
          }

          // Center map to current location
          this.map.setView([lat, lng], 15)

          // Add or update user marker
          if (this.userMarker) {
            try {
              // Clean up existing marker
              this.userMarker.off()
              if (this.userMarker.getPopup && this.userMarker.getPopup()) {
                this.userMarker.closePopup()
                this.userMarker.unbindPopup()
              }
              if (this.map.hasLayer(this.userMarker)) {
                this.map.removeLayer(this.userMarker)
              }
            } catch (error) {
              console.warn('Error removing existing user marker:', error)
            }
          }

          // Custom marker icon
          const userIcon = this.L.divIcon({
            className: 'user-location-marker',
            html: '<div class="user-marker-inner"></div>',
            iconSize: [20, 20],
            iconAnchor: [10, 10]
          })

          // GANTI dengan kode sederhana ini:
          try {
            this.userMarker = this.L.marker([lat, lng], { icon: userIcon })
              .addTo(this.map)
            // Tidak ada event listener, jadi marker user tidak akan memunculkan modal
          } catch (error) {
            console.warn('Error creating user marker:', error)
          }

          this.loadingLocation = false
        },
        (error) => {
          console.error('Error getting location:', error)
          alert('Tidak dapat mengakses lokasi. Pastikan Anda mengizinkan akses lokasi.')
          this.loadingLocation = false
        },
        {
          enableHighAccuracy: true,
          timeout: 10000,
          maximumAge: 60000
        }
      )
    },

    zoomIn() {
      if (this.map) {
        this.map.zoomIn()
      }
    },

    zoomOut() {
      if (this.map) {
        this.map.zoomOut()
      }
    },

    resetView() {
      if (this.map) {
        this.map.setView(this.defaultCenter, this.defaultZoom)
      }
    },

    toggleMapType() {
      if (!this.map || !this.L) return

      this.mapType = this.mapType === 'light' ? 'dark' : 'light'

      if (this.mapType === 'dark') {
        // Dark theme
        this.map.eachLayer((layer) => {
          if (layer instanceof this.L.TileLayer) {
            this.map.removeLayer(layer)
          }
        })

        this.L.tileLayer(
          'https://cartodb-basemaps-{s}.global.ssl.fastly.net/dark_all/{z}/{x}/{y}.png',
          {
            attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
            subdomains: ['a', 'b', 'c', 'd'],
            maxZoom: 19
          }
        ).addTo(this.map)
      } else {
        this.addTileLayer()
      }
    }
  }
}
</script>

<style scoped>
/* Reset and base styles matching DataPertanian.vue */
*,
*::before,
*::after {
  box-sizing: border-box;
}

.container {
  width: 100%;
  margin: 0;
  padding: 2rem 0;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  color: #374151;
  background-color: #f5f5f5;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.headline-card {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.headline {
  font-size: 1.75rem;
  font-weight: 600;
  margin-bottom: 0;
  color: #111827;
  text-align: center;
  padding: 0;
}

.map-section {
  background: #ffffff;
  margin: 0;
  padding: 2rem;
  border-top: 1px solid #e5e7eb;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #111827;
}

.map-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.button-group {
  display: flex;
  gap: 0.5rem;
}

.location-button,
.refresh-button {
  background-color: #111827;
  color: #ffffff;
  padding: 0.5rem 1rem;
  font-size: 0.75rem;
  font-weight: 500;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
  white-space: nowrap;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.location-button:hover:not(:disabled),
.refresh-button:hover:not(:disabled) {
  background-color: #374151;
}

.location-button:disabled,
.refresh-button:disabled {
  background-color: #9ca3af;
  cursor: not-allowed;
}

.map-wrapper {
  position: relative;
  height: 650px;
  border-radius: 0.375rem;
  overflow: hidden;
  border: 1px solid #e5e7eb;
  margin-bottom: 1.5rem;
}

.map-container {
  width: 100%;
  height: 100%;
}

.floating-controls {
  position: absolute;
  top: 1rem;
  right: 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  z-index: 1000;
}

.control-btn {
  width: 44px;
  height: 44px;
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.control-btn:hover:not(:disabled) {
  background: #f3f4f6;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.15);
}

.control-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.icon {
  width: 18px;
  height: 18px;
  stroke-width: 2;
}

.info-panel {
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 0.375rem;
  background: #f9fafb;
  display: flex;
  gap: 2rem;
}

.info-item {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.info-icon {
  width: 20px;
  height: 20px;
  color: #6b7280;
  flex-shrink: 0;
}

.info-label {
  font-size: 0.75rem;
  color: #6b7280;
  margin: 0 0 0.25rem 0;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.info-value {
  font-size: 0.875rem;
  color: #111827;
  margin: 0;
  font-weight: 500;
  font-family: 'SF Mono', 'Monaco', 'Inconsolata', 'Roboto Mono', monospace;
}

/* Product Modal Styles */
.product-modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.product-modal {
  background: white;
  border-radius: 12px;
  max-width: 500px;
  width: 90%;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.product-modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.5rem;
  border-bottom: 1px solid #e5e7eb;
}

.product-modal-header h4 {
  font-size: 1.25rem;
  font-weight: 600;
  color: #111827;
  margin: 0;
}

.close-btn {
  background: none;
  border: none;
  padding: 0.5rem;
  cursor: pointer;
  border-radius: 6px;
  transition: background-color 0.2s;
}

.close-btn:hover {
  background: #f3f4f6;
}

.product-modal-content {
  padding: 1.5rem;
}

.product-info-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem 0;
  border-bottom: 1px solid #f3f4f6;
}

.product-info-row:last-child {
  border-bottom: none;
}

.product-icon {
  width: 20px;
  height: 20px;
  color: #6b7280;
  flex-shrink: 0;
}

.product-label {
  font-size: 0.75rem;
  color: #6b7280;
  margin: 0 0 0.25rem 0;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.product-value {
  font-size: 0.875rem;
  color: #111827;
  margin: 0;
  font-weight: 500;
}

/* Responsive design */
@media (max-width: 768px) {
  .container {
    padding: 1rem 0;
  }

  .headline-card {
    padding: 1.5rem 1rem;
  }

  .headline {
    font-size: 1.5rem;
    padding: 0;
  }

  .map-section {
    padding: 1.5rem 1rem;
  }

  .map-header {
    flex-direction: column;
    gap: 1rem;
    align-items: stretch;
  }

  .button-group {
    flex-direction: column;
    gap: 0.5rem;
  }

  .location-button,
  .refresh-button {
    width: 100%;
    justify-content: center;
  }

  .floating-controls {
    top: 0.5rem;
    right: 0.5rem;
  }

  .control-btn {
    width: 40px;
    height: 40px;
  }

  .info-panel {
    padding: 1rem;
    flex-direction: column;
    gap: 1rem;
  }

  .product-modal {
    width: 95%;
    margin: 1rem;
  }
}

@media (max-width: 480px) {
  .button-group {
    flex-direction: column;
  }
}
</style>

<style>
/* Global styles for Leaflet markers */
.user-location-marker {
  background: transparent;
  border: none;
}

.user-marker-inner {
  width: 20px;
  height: 20px;
  background: #000000;
  border: 3px solid white;
  border-radius: 50%;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(56, 58, 59, 0.7);
  }

  70% {
    box-shadow: 0 0 0 10px rgba(59, 130, 246, 0);
  }

  100% {
    box-shadow: 0 0 0 0 rgba(59, 130, 246, 0);
  }
}

/* Custom marker styles */
.custom-marker-wrapper {
  background: transparent;
  border: none;
}

.custom-marker {
  width: 30px;
  height: 30px;
  border-radius: 50% 50% 50% 0;
  border: 3px solid white;
  transform: rotate(-45deg);
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
  cursor: pointer;
  transition: transform 0.2s ease;
}

.custom-marker:hover {
  transform: rotate(-45deg) scale(1.1);
}

.marker-inner {
  font-size: 12px;
  font-weight: bold;
  color: white;
  transform: rotate(45deg);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

/* Popup styles */
.popup-content {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  min-width: 200px;
}

.popup-content h4 {
  margin: 0 0 0.5rem 0;
  font-size: 1rem;
  font-weight: 600;
  color: #111827;
}

.popup-content p {
  margin: 0.25rem 0;
  font-size: 0.875rem;
  color: #374151;
}

.popup-btn {
  background: #111827;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.2s;
  margin-top: 0.5rem;
  width: 100%;
}

.popup-btn:hover {
  background: #374151;
}

/* Customize Leaflet popup */
.leaflet-popup-content-wrapper {
  border-radius: 8px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.leaflet-popup-content {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-size: 0.875rem;
}
</style>