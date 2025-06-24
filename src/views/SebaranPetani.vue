<template>
    <main class="container" role="main" aria-label="Detail Lokasi Cilamaya Wetan">
        <div class="headline-card">
            <h1 class="headline">Sebaran Jumlah Petani</h1>
            <p class="subtitle">Kecamatan Cilamaya Wetan, Kabupaten Karawang, Jawa Barat</p>
        </div>

        <!-- Map Section -->
        <section class="map-section" aria-labelledby="map-title">
            <header class="map-header">
                <h2 id="map-title" class="section-title">Peta Sebaran Petani</h2>
                <div class="button-group">
                    <button @click="getCurrentLocation" class="location-button" :disabled="loadingLocation">
                        <component :is="loadingLocation ? 'LoaderIcon' : 'MapPinIcon'" class="icon" />
                        {{ loadingLocation ? 'Memuat...' : 'Lokasi Saya' }}
                    </button>
                    <button @click="fitToGeojsonBounds" class="refresh-button" :disabled="loadingData">
                        <component :is="loadingData ? 'LoaderIcon' : 'FocusIcon'" class="icon" />
                        {{ loadingData ? 'Memuat...' : 'Fokus Area' }}
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
                    <button @click="fitToGeojsonBounds" class="control-btn">
                        <FocusIcon class="icon" />
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
                <div class="info-item" v-if="geojsonData">
                    <BuildingIcon class="info-icon" />
                    <div>
                        <p class="info-label">Kecamatan</p>
                        <p class="info-value">Cilamaya Wetan</p>
                    </div>
                </div>
                <div class="info-item" v-if="geojsonData">
                    <MapIcon class="info-icon" />
                    <div>
                        <p class="info-label">Total Desa</p>
                        <p class="info-value">{{ totalDesa }} desa</p>
                    </div>
                </div>
                <div class="info-item" v-if="totalPoktan > 0">
                    <UsersIcon class="info-icon" />
                    <div>
                        <p class="info-label">Total Poktan</p>
                        <p class="info-value">{{ totalPoktan }} poktan</p>
                    </div>
                </div>
                <div class="info-item" v-if="totalLuasLahan > 0">
                    <SquareIcon class="info-icon" />
                    <div>
                        <p class="info-label">Total Luas Lahan</p>
                        <p class="info-value">{{ formatNumber(totalLuasLahan) }} ha</p>
                    </div>
                </div>
                <div class="info-item" v-if="totalPetani > 0">
                    <UserIcon class="info-icon" />
                    <div>
                        <p class="info-label">Total Petani</p>
                        <p class="info-value">{{ formatNumber(totalPetani) }} orang</p>
                    </div>
                </div>
                <div class="info-item" v-if="petaniDataError">
                    <AlertCircleIcon class="info-icon text-red-500" />
                    <div>
                        <p class="info-label">Status Data</p>
                        <p class="info-value text-red-500">{{ petaniDataError }}</p>
                    </div>
                </div>
            </div>
        </section>

        <!-- Village Info Modal -->
        <div v-if="selectedVillage" class="village-modal-overlay" @click="closeVillageModal">
            <div class="village-modal" @click.stop>
                <div class="village-modal-header">
                    <h4>Detail Desa</h4>
                    <button @click="closeVillageModal" class="close-btn">
                        <XIcon class="icon" />
                    </button>
                </div>
                <div class="village-modal-content">
                    <div class="village-info-row">
                        <BuildingIcon class="village-icon" />
                        <div>
                            <p class="village-label">Nama Desa</p>
                            <p class="village-value">{{ selectedVillage.NAMOBJ || 'Tidak tersedia' }}</p>
                        </div>
                    </div>
                    <div class="village-info-row">
                        <MapIcon class="village-icon" />
                        <div>
                            <p class="village-label">Kecamatan</p>
                            <p class="village-value">{{ selectedVillage.WADMKC || 'Cilamaya Wetan' }}</p>
                        </div>
                    </div>
                    <div class="village-info-row">
                        <MapPinIcon class="village-icon" />
                        <div>
                            <p class="village-label">Kabupaten</p>
                            <p class="village-value">{{ selectedVillage.WADMKK || 'Karawang' }}</p>
                        </div>
                    </div>
                    <div class="village-info-row">
                        <GlobeIcon class="village-icon" />
                        <div>
                            <p class="village-label">Provinsi</p>
                            <p class="village-value">{{ selectedVillage.WADMPR || 'Jawa Barat' }}</p>
                        </div>
                    </div>
                    <div class="village-info-row" v-if="selectedVillage.LUASWH">
                        <RulerIcon class="village-icon" />
                        <div>
                            <p class="village-label">Luas Wilayah</p>
                            <p class="village-value">{{ formatArea(selectedVillage.LUASWH) }} m²</p>
                        </div>
                    </div>
                    <div class="village-info-row" v-if="selectedVillage.KDCPUM">
                        <HashIcon class="village-icon" />
                        <div>
                            <p class="village-label">Kode Desa</p>
                            <p class="village-value">{{ selectedVillage.KDCPUM }}</p>
                        </div>
                    </div>

                    <div class="village-info-row" v-if="getVillageData(selectedVillage.NAMOBJ)">
                        <UsersIcon class="village-icon" />
                        <div>
                            <p class="village-label">Jumlah Poktan</p>
                            <p class="village-value">{{ getVillageData(selectedVillage.NAMOBJ).jumlah_poktan || 0 }}
                                poktan</p>
                        </div>
                    </div>

                    <div class="village-info-row" v-if="getVillageData(selectedVillage.NAMOBJ)">
                        <SquareIcon class="village-icon" />
                        <div>
                            <p class="village-label">Luas Lahan Pertanian</p>
                            <p class="village-value">{{ formatNumber(getVillageData(selectedVillage.NAMOBJ).luas_ha) ||
                                0 }} ha</p>
                        </div>
                    </div>

                    <div class="village-info-row" v-if="getVillageData(selectedVillage.NAMOBJ)">
                        <UserIcon class="village-icon" />
                        <div>
                            <p class="village-label">Jumlah Petani</p>
                            <p class="village-value">{{
                                formatNumber(getVillageData(selectedVillage.NAMOBJ).jumlah_petani) || 0 }} orang</p>
                        </div>
                    </div>

                    <!-- Jika data tidak ditemukan -->
                    <div v-if="!getVillageData(selectedVillage.NAMOBJ) && !petaniDataError" class="village-info-row">
                        <AlertCircleIcon class="village-icon text-yellow-500" />
                        <div>
                            <p class="village-label">Data Pertanian</p>
                            <p class="village-value text-gray-500">Data tidak tersedia</p>
                        </div>
                    </div>

                    <!-- Jika ada error loading data -->
                    <div v-if="petaniDataError" class="village-info-row">
                        <AlertCircleIcon class="village-icon text-red-500" />
                        <div>
                            <p class="village-label">Status Data</p>
                            <p class="village-value text-red-500">{{ petaniDataError }}</p>
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
    LayersIcon,
    LoaderIcon,
    FocusIcon,
    MapIcon,
    BuildingIcon,
    GlobeIcon,
    RulerIcon,
    HashIcon,
    XIcon,
    UsersIcon,
    SquareIcon,
    UserIcon,
    AlertCircleIcon
} from 'lucide-vue-next'

export default {
    name: 'SebaranPetani',
    components: {
        MapPinIcon,
        PlusIcon,
        MinusIcon,
        NavigationIcon,
        LayersIcon,
        LoaderIcon,
        FocusIcon,
        MapIcon,
        BuildingIcon,
        GlobeIcon,
        RulerIcon,
        HashIcon,
        XIcon,
        UsersIcon,
        SquareIcon,
        UserIcon,
        AlertCircleIcon
    },
    data() {
        return {
            map: null,
            currentLocation: null,
            loadingLocation: false,
            loadingData: false,
            userMarker: null,
            selectedVillage: null,
            geojsonData: null,
            geojsonLayer: null,
            totalDesa: 0,
            defaultCenter: [-6.235, 107.57], // Koordinat tengah Cilamaya Wetan
            defaultZoom: 12,
            mapType: 'light', // 'light' or 'dark'
            L: null, // Store Leaflet reference
            petaniData: [], // Store petani data
            petaniDataError: null // Store error message
        }
    },
    computed: {
        totalPoktan() {
            if (!this.petaniData || this.petaniData.length === 0) return 0
            return this.petaniData.reduce((total, data) => total + (data.jumlah_poktan || 0), 0)
        },
        totalLuasLahan() {
            if (!this.petaniData || this.petaniData.length === 0) return 0
            return this.petaniData.reduce((total, data) => total + (data.luas_ha || 0), 0)
        },
        totalPetani() {
            if (!this.petaniData || this.petaniData.length === 0) return 0
            return this.petaniData.reduce((total, data) => total + (data.jumlah_petani || 0), 0)
        }
    },
    async mounted() {
        await this.initMap()
        await this.loadGeojsonData()
        await this.loadPetaniData()
    },
    beforeUnmount() {
        // Clean up map
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

                // Clean up geojson layer
                if (this.geojsonLayer) {
                    this.geojsonLayer.eachLayer((layer) => {
                        if (layer.labelMarker) {
                            try {
                                if (this.map.hasLayer(layer.labelMarker)) {
                                    this.map.removeLayer(layer.labelMarker)
                                }
                                layer.labelMarker = null
                            } catch (error) {
                                console.warn('Error cleaning up label marker:', error)
                            }
                        }
                    })
                }

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
        async loadPetaniData() {
            try {
                console.log('Loading petani data...')
                const response = await fetch('https://raw.githubusercontent.com/kunskuy/padira-dashboard/main/public/data/DataPetani.json')

                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`)
                }

                const data = await response.json()
                this.petaniData = data
                console.log('Petani data loaded successfully:', data)
                this.petaniDataError = null

            } catch (error) {
                console.error('Error loading petani data:', error)
                this.petaniDataError = 'Data petani tidak dapat dimuat'
                this.petaniData = []
            }
        },


        getVillageData(villageName) {
            if (!villageName || !this.petaniData || this.petaniDataError) return null
            return this.petaniData.find(data =>
                data.desa.toLowerCase() === villageName.toLowerCase()
            )
        },

        formatArea(area) {
            return new Intl.NumberFormat('id-ID').format(Math.round(area))
        },

        formatNumber(number) {
            if (!number) return '0'
            return new Intl.NumberFormat('id-ID').format(number)
        },

        async loadGeojsonData() {
            this.loadingData = true

            try {
                console.log('Loading GeoJSON data...')
                const response = await fetch('https://raw.githubusercontent.com/kunskuy/padira-dashboard/main/public/maps/BatasDesaCilamayaWetan.geojson')

                if (!response.ok) {
                    throw new Error(`HTTP error! status: ${response.status}`)
                }

                const data = await response.json()
                this.geojsonData = data
                console.log('GeoJSON data loaded successfully')

                // Count total desa dengan validasi yang lebih ketat
                if (this.geojsonData && this.geojsonData.features) {
                    // Filter hanya features yang memiliki nama desa yang valid
                    const validVillages = this.geojsonData.features.filter(feature => {
                        return feature &&
                            feature.properties &&
                            feature.properties.NAMOBJ &&
                            feature.properties.NAMOBJ.trim() !== '' &&
                            feature.geometry &&
                            feature.geometry.coordinates
                    })

                    this.totalDesa = validVillages.length

                    // Debug: tampilkan nama-nama desa untuk verifikasi
                    console.log('Desa yang ditemukan:', validVillages.map(f => f.properties.NAMOBJ))
                    console.log('Total desa valid:', this.totalDesa)

                    // Update geojsonData dengan hanya features yang valid
                    this.geojsonData.features = validVillages
                }

                // Add to map
                this.addGeojsonToMap()

                // Fit map to geojson bounds
                this.fitToGeojsonBounds()

            } catch (error) {
                console.error('Error loading GeoJSON data:', error)
                alert('Gagal memuat data peta. Pastikan file tersedia di server.')
            } finally {
                this.loadingData = false
            }
        },

        addGeojsonToMap() {
            if (!this.map || !this.L || !this.geojsonData) return

            // Remove existing geojson layer
            if (this.geojsonLayer) {
                this.geojsonLayer.off()
                this.map.removeLayer(this.geojsonLayer)
            }

            const styleFunction = (feature) => {
                return {
                    fillColor: this.getVillageColor(feature.properties.NAMOBJ),
                    weight: 1, // Dikurangi dari 2 menjadi 1 (lebih tipis)
                    opacity: 1,
                    color: '#9CA3AF', // Diubah dari '#ffffff' menjadi abu-abu
                    dashArray: '',
                    fillOpacity: 0.7
                }
            }

            // Create geojson layer
            this.geojsonLayer = this.L.geoJSON(this.geojsonData, {
                style: styleFunction,
                onEachFeature: (feature, layer) => {
                    // Click event to show modal directly - NO POPUP
                    layer.on('click', (e) => {
                        this.selectedVillage = feature.properties
                        // Prevent event bubbling
                        e.originalEvent.stopPropagation()
                    })

                    // Hover effects
                    layer.on('mouseover', (e) => {
                        const layer = e.target
                        layer.setStyle({
                            weight: 2,
                            color: '#6B7280',
                            fillOpacity: 0.9
                        })
                    })

                    layer.on('mouseout', (e) => {
                        this.geojsonLayer.resetStyle(e.target)
                    })

                    // Tambahkan label nama desa
                    if (feature.properties.NAMOBJ) {
                        // Hitung center dari polygon untuk menempatkan label
                        const bounds = layer.getBounds()
                        const center = bounds.getCenter()

                        // Buat marker dengan divIcon untuk label
                        const labelIcon = this.L.divIcon({
                            className: 'village-label',
                            html: `<div class="village-label-text">${feature.properties.NAMOBJ}</div>`,
                            iconSize: [0, 0], // Tidak ada ukuran tetap
                            iconAnchor: [0, 0] // Tidak ada anchor point
                        })

                        // Tambahkan marker label ke peta
                        const labelMarker = this.L.marker(center, {
                            icon: labelIcon,
                            interactive: false // Label tidak bisa diklik
                        }).addTo(this.map)

                        // Simpan referensi label marker ke layer untuk cleanup nanti
                        layer.labelMarker = labelMarker
                    }
                }
            }).addTo(this.map)
        },

        getVillageColor(villageName) {
            // Generate different colors for different villages
            const colors = [
                '#FF6B6B', '#4ECDC4', '#45B7D1', '#96CEB4', '#FFEAA7',
                '#DDA0DD', '#98D8C8', '#F7DC6F', '#BB8FCE', '#85C1E9',
                '#F8C471', '#82E0AA', '#F1948A', '#85C1E9', '#D2B4DE'
            ]

            if (!villageName) return '#6b7280'

            // Simple hash function to get consistent color for same village
            let hash = 0
            for (let i = 0; i < villageName.length; i++) {
                hash = villageName.charCodeAt(i) + ((hash << 5) - hash)
            }
            return colors[Math.abs(hash) % colors.length]
        },

        fitToGeojsonBounds() {
            if (!this.map || !this.geojsonLayer) return

            try {
                const bounds = this.geojsonLayer.getBounds()
                if (bounds.isValid()) {
                    this.map.fitBounds(bounds, { padding: [20, 20] })
                }
            } catch (error) {
                console.warn('Error fitting to geojson bounds:', error)
            }
        },

        closeVillageModal() {
            this.selectedVillage = null
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

                // Add tile layer with CartoDB light theme
                this.addTileLayer()

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

                    try {
                        this.userMarker = this.L.marker([lat, lng], { icon: userIcon })
                            .addTo(this.map)
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
/* Reset and base styles matching Location.vue */
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
    margin-bottom: 0.5rem;
    color: #111827;
    text-align: center;
    padding: 0;
}

.subtitle {
    font-size: 1rem;
    color: #6b7280;
    text-align: center;
    margin: 0;
    font-weight: 400;
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
    flex-wrap: wrap;
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

/* Village Modal Styles */
.village-modal-overlay {
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

.village-modal {
    background: white;
    border-radius: 12px;
    max-width: 500px;
    width: 90%;
    max-height: 80vh;
    overflow-y: auto;
    box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.village-modal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1.5rem;
    border-bottom: 1px solid #e5e7eb;
}

.village-modal-header h4 {
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

.village-modal-content {
    padding: 1.5rem;
}

.village-info-row {
    display: flex;
    align-items: center;
    gap: 1rem;
    padding: 1rem 0;
    border-bottom: 1px solid #f3f4f6;
}

.village-info-row:last-child {
    border-bottom: none;
}

.village-icon {
    width: 20px;
    height: 20px;
    color: #6b7280;
    flex-shrink: 0;
}

.village-label {
    font-size: 0.75rem;
    color: #6b7280;
    margin: 0 0 0.25rem 0;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.05em;
}

:deep(.village-label) {
    background: transparent !important;
    border: none !important;
    box-shadow: none !important;
}

:deep(.village-label-text) {
    background: transparent;
    /* Hilangkan background putih */
    color: #111827;
    padding: 0;
    /* Hilangkan padding */
    border-radius: 0;
    /* Hilangkan border radius */
    font-size: 11px;
    font-weight: 700;
    /* Buat font lebih tebal agar lebih terlihat */
    text-align: center;
    white-space: nowrap;
    box-shadow: none;
    /* Hilangkan shadow */
    border: none;
    /* Hilangkan border */
    pointer-events: none;
    transform: translate(-50%, -50%);
    position: relative;
    left: 50%;
    top: 50%;
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
    /* Tambahkan text shadow putih untuk kontras */
}

.village-value {
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

    .village-modal {
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