<template>
  <main class="container" role="main" aria-label="Dashboard">
    <div class="headline-card">
      <h1 class="headline">Dashboard</h1>
    </div>

    <!-- Cards Section -->
    <section class="cards-section" aria-labelledby="cards-title">
      <div class="card-grid">
        <div v-for="(card, index) in dashboardCards" :key="index" class="dashboard-card">
          <div class="card-header">
            <h3 class="card-title">{{ card.title }}</h3>
          </div>
          <div class="card-content">
            <div class="card-value">{{ card.value }}</div>
            <div class="card-desc">{{ card.desc }}</div>
          </div>
        </div>
      </div>
    </section>

    <!-- Charts Section -->
    <section class="charts-section" aria-labelledby="produksi-chart-title">
      <div class="chart-card">
        <div class="chart-header">
          <h3 id="produksi-chart-title" class="chart-title">Produksi Bulanan (Gabah vs Beras)</h3>
        </div>
        <div class="chart-wrapper">
          <canvas ref="produksiChart" id="produksiChart"></canvas>
        </div>
      </div>
    </section>

    <section class="charts-section" aria-labelledby="jenis-chart-title">
      <div class="chart-card">
        <div class="chart-header">
          <h3 id="jenis-chart-title" class="chart-title">Produksi Berdasarkan Jenis</h3>
        </div>
        <div class="chart-wrapper">
          <canvas ref="jenisChart" id="jenisChart"></canvas>
        </div>
      </div>
    </section>

    <!-- Table Section - Data Pertanian Terakhir -->
    <section class="table-section" aria-labelledby="pertanian-table-title">
      <header class="table-header">
        <h2 id="pertanian-table-title" class="section-title">Data Pertanian Terakhir</h2>
      </header>

      <div v-if="recentPertanianData.length === 0" class="no-data" role="status">
        Tidak ada data pertanian yang tersedia
      </div>

      <div v-else class="table-container">
        <table class="price-table" role="table" aria-describedby="pertanian-table-title">
          <thead>
            <tr>
              <th v-for="header in pertanianTableHeaders" :key="header" scope="col">{{ header }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, rowIndex) in recentPertanianData" :key="rowIndex">
              <td>{{ row.id }}</td>
              <td>{{ row.tanggal }}</td>
              <td>{{ row.jenisGabah }}</td>
              <td>{{ row.hasil }}</td>
              <td>{{ row.deskripsi }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- Table Section - Data Penggilingan Terakhir -->
    <section class="table-section" aria-labelledby="table-title">
      <header class="table-header">
        <h2 id="table-title" class="section-title">Data Penggilingan Terakhir</h2>
      </header>

      <div v-if="recentPenggilinganData.length === 0" class="no-data" role="status">
        Tidak ada data penggilingan yang tersedia
      </div>

      <div v-else class="table-container">
        <table class="price-table" role="table" aria-describedby="table-title">
          <thead>
            <tr>
              <th v-for="header in tableHeaders" :key="header" scope="col">{{ header }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, rowIndex) in recentPenggilinganData" :key="rowIndex">
              <td>{{ row.id }}</td>
              <td>{{ row.tanggal }}</td>
              <td>{{ row.jenisBeras }}</td>
              <td>{{ row.hasil }}</td>
              <td>{{ row.deskripsi }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
  </main>
</template>

<script>
import Chart from 'chart.js/auto'
import { ref, onValue, off } from 'firebase/database'
import { database } from '../firebase.js'

export default {
  name: 'DashboardView',
  data() {
    return {
      pertanianData: [],
      penggilinganData: [],
      usersData: [],
      dashboardCards: [
        { title: 'Total Produksi Gabah', value: '0', desc: 'Ton gabah' },
        { title: 'Total Produksi Beras', value: '0', desc: 'Ton beras' },
        { title: 'Jumlah Petani Terdaftar', value: '0', desc: 'Petani terdaftar' },
        { title: 'Jumlah Penggilingan Terdaftar', value: '0', desc: 'Penggilingan terdaftar' },
        { title: 'Jumlah Distributor Terdaftar', value: '0', desc: 'Distributor terdaftar' }
      ],
      recentPertanianData: [],
      recentPenggilinganData: [],
      pertanianTableHeaders: ['ID', 'Tanggal', 'Jenis Gabah', 'Hasil (Ton)', 'Deskripsi'],
      tableHeaders: ['ID', 'Tanggal', 'Jenis Beras', 'Hasil (Ton)', 'Deskripsi'],
      produksiChart: null,
      jenisChart: null,
      notesListener: null,
      usersListener: null,
      isComponentActive: false,
      dataLoaded: false,
      chartInitialized: false,
      retryCount: 0,
      maxRetries: 3
    }
  },
  async mounted() {
    console.log('Dashboard mounted')
    this.isComponentActive = true

    // Wait for DOM to be fully rendered
    await this.$nextTick()

    // Add small delay to ensure everything is ready
    setTimeout(() => {
      if (this.isComponentActive) {
        this.loadData()
        this.loadUsersData()
      }
    }, 100)
  },
  activated() {
    // This will be called when component becomes active (if using keep-alive)
    console.log('Dashboard activated')
    this.isComponentActive = true
    this.ensureChartsInitialized()
  },
  deactivated() {
    // This will be called when component becomes inactive (if using keep-alive)
    console.log('Dashboard deactivated')
    this.isComponentActive = false
  },
  beforeUnmount() {
    console.log('Dashboard beforeUnmount')
    this.isComponentActive = false

    // Destroy charts before component unmount
    this.destroyCharts()

    // Remove Firebase listeners
    if (this.notesListener) {
      off(ref(database, 'notes'), 'value', this.notesListener)
    }
    if (this.usersListener) {
      off(ref(database, 'users'), 'value', this.usersListener)
    }
  },
  methods: {
    loadData() {
      const notesRef = ref(database, 'notes')

      // Remove previous listener if exists
      if (this.notesListener) {
        off(notesRef, 'value', this.notesListener)
      }

      this.notesListener = (snapshot) => {
        if (!this.isComponentActive) return

        try {
          const data = snapshot.val() || {}
          this.pertanianData = []
          this.penggilinganData = []

          for (let id in data) {
            const item = data[id]
            const volume = parseFloat(item.weight) / 1000 || 0
            const date = item.dateString || ''
            const month = this.getMonthFromDate(date)

            if (item.userRole === 'Petani') {
              this.pertanianData.push({
                id, tanggal: date, jenisGabah: item.productType || '',
                deskripsi: item.description || '', hasilGabah: volume, month
              })
            } else if (item.userRole === 'Penggilingan') {
              this.penggilinganData.push({
                id, tanggal: date, jenisBeras: item.productType || '',
                deskripsi: item.description || '', hasilBeras: volume, month
              })
            }
          }

          this.updateDashboardCards()
          this.updateRecentPertanianData()
          this.updateRecentPenggilinganData()
          this.dataLoaded = true

          // Initialize charts with proper timing
          this.scheduleChartInitialization()
        } catch (error) {
          console.error('Error processing data:', error)
        }
      }

      onValue(notesRef, this.notesListener)
    },

    loadUsersData() {
      const usersRef = ref(database, 'users')

      // Remove previous listener if exists
      if (this.usersListener) {
        off(usersRef, 'value', this.usersListener)
      }

      this.usersListener = (snapshot) => {
        if (!this.isComponentActive) return

        try {
          const data = snapshot.val() || {}
          this.usersData = []

          for (let uid in data) {
            const user = data[uid]
            this.usersData.push({
              uid,
              role: user.role || '',
              email: user.email || '',
              name: user.name || ''
            })
          }

          this.updateUserCounts()
        } catch (error) {
          console.error('Error loading users data:', error)
        }
      }

      onValue(usersRef, this.usersListener)
    },

    scheduleChartInitialization() {
      // Clear any existing timeout
      if (this.chartInitTimeout) {
        clearTimeout(this.chartInitTimeout)
      }

      // Use multiple nextTick calls to ensure DOM is fully ready
      this.$nextTick(() => {
        this.$nextTick(() => {
          // Add a small delay to ensure canvas elements are fully rendered
          this.chartInitTimeout = setTimeout(() => {
            if (this.isComponentActive && this.dataLoaded) {
              this.initChartsWithRetry()
            }
          }, 150)
        })
      })
    },

    initChartsWithRetry() {
      if (!this.isComponentActive) return

      const success = this.initCharts()

      if (!success && this.retryCount < this.maxRetries) {
        this.retryCount++
        console.log(`Chart initialization failed, retrying... (${this.retryCount}/${this.maxRetries})`)

        setTimeout(() => {
          if (this.isComponentActive) {
            this.initChartsWithRetry()
          }
        }, 200 * this.retryCount) // Exponential backoff
      } else if (success) {
        this.chartInitialized = true
        this.retryCount = 0
        console.log('Charts initialized successfully')
      }
    },

    ensureChartsInitialized() {
      // Called when component becomes active again
      if (!this.chartInitialized && this.dataLoaded && this.isComponentActive) {
        this.scheduleChartInitialization()
      }
    },

    getMonthFromDate(dateString) {
      if (!dateString) return 0;

      // Handle format DD/MM/YYYY (seperti "06/06/2025")
      if (dateString.includes('/')) {
        const parts = dateString.split('/');
        if (parts.length === 3) {
          const month = parseInt(parts[1]) - 1; // Bulan dimulai dari 0 (Januari = 0)
          return month >= 0 && month <= 11 ? month : 0;
        }
      }

      // Handle format dengan nama bulan (seperti "6 Jun 2025")
      const monthNames = {
        'Jan': 0, 'Feb': 1, 'Mar': 2, 'Apr': 3, 'Mei': 4, 'Jun': 5,
        'Jul': 6, 'Agu': 7, 'Sep': 8, 'Okt': 9, 'Nov': 10, 'Des': 11
      };

      const parts = dateString.split(' ');
      if (parts.length >= 2) {
        const monthStr = parts[1];
        return monthNames[monthStr] !== undefined ? monthNames[monthStr] : 0;
      }

      // Fallback: coba parse sebagai Date object
      try {
        const date = new Date(dateString);
        if (!isNaN(date.getTime())) {
          return date.getMonth();
        }
      } catch (error) {
        console.warn('Failed to parse date:', dateString);
      }

      return 0; // Default ke bulan Januari jika parsing gagal
    },

    updateUserCounts() {
      const petaniCount = this.usersData.filter(user => user.role === 'Petani').length
      const penggilinganCount = this.usersData.filter(user => user.role === 'Penggilingan').length
      const distributorCount = this.usersData.filter(user => user.role === 'Distributor').length

      this.dashboardCards[2].value = petaniCount.toString()
      this.dashboardCards[3].value = penggilinganCount.toString()
      this.dashboardCards[4].value = distributorCount.toString()
    },

    updateDashboardCards() {
      const totalGabah = this.pertanianData.reduce((sum, item) => sum + item.hasilGabah, 0)
      const totalBeras = this.penggilinganData.reduce((sum, item) => sum + item.hasilBeras, 0)

      this.dashboardCards[0].value = totalGabah.toFixed(1)
      this.dashboardCards[1].value = totalBeras.toFixed(1)
    },

    updateRecentPertanianData() {
      this.recentPertanianData = this.pertanianData
        .sort((a, b) => new Date(b.tanggal) - new Date(a.tanggal))
        .slice(0, 3) // Ubah dari 5 menjadi 3
        .map(item => ({
          id: item.id,
          tanggal: item.tanggal,
          jenisGabah: item.jenisGabah,
          hasil: item.hasilGabah.toFixed(2),
          deskripsi: item.deskripsi
        }))
    },

    updateRecentPenggilinganData() {
      this.recentPenggilinganData = this.penggilinganData
        .sort((a, b) => new Date(b.tanggal) - new Date(a.tanggal))
        .slice(0, 3) // Ubah dari 5 menjadi 3
        .map(item => ({
          id: item.id,
          tanggal: item.tanggal,
          jenisBeras: item.jenisBeras,
          hasil: item.hasilBeras.toFixed(2),
          deskripsi: item.deskripsi
        }))
    },

    getMonthlyData() {
      const monthlyGabah = Array(12).fill(0)
      const monthlyBeras = Array(12).fill(0)

      this.pertanianData.forEach(item => {
        if (item.month >= 0 && item.month < 12) {
          monthlyGabah[item.month] += item.hasilGabah
        }
      })

      this.penggilinganData.forEach(item => {
        if (item.month >= 0 && item.month < 12) {
          monthlyBeras[item.month] += item.hasilBeras
        }
      })

      return { monthlyGabah, monthlyBeras }
    },

    getProductTypeData() {
      const gabahTypes = {}
      const berasTypes = {}

      this.pertanianData.forEach(item => {
        if (item.jenisGabah) {
          gabahTypes[item.jenisGabah] = (gabahTypes[item.jenisGabah] || 0) + item.hasilGabah
        }
      })

      this.penggilinganData.forEach(item => {
        if (item.jenisBeras) {
          berasTypes[item.jenisBeras] = (berasTypes[item.jenisBeras] || 0) + item.hasilBeras
        }
      })

      return { gabahTypes, berasTypes }
    },

    destroyCharts() {
      if (this.produksiChart) {
        try {
          this.produksiChart.destroy()
        } catch (error) {
          console.warn('Error destroying produksiChart:', error)
        }
        this.produksiChart = null
      }
      if (this.jenisChart) {
        try {
          this.jenisChart.destroy()
        } catch (error) {
          console.warn('Error destroying jenisChart:', error)
        }
        this.jenisChart = null
      }
      this.chartInitialized = false
    },

    initCharts() {
      if (!this.isComponentActive) {
        return false
      }

      try {
        // Destroy existing charts first
        this.destroyCharts()

        // Get canvas elements using refs
        const produksiCanvas = this.$refs.produksiChart
        const jenisCanvas = this.$refs.jenisChart

        if (!produksiCanvas || !jenisCanvas) {
          console.warn('Canvas elements not found')
          return false
        }

        // Check if canvas elements are actually in the DOM and visible
        if (!produksiCanvas.offsetParent || !jenisCanvas.offsetParent) {
          console.warn('Canvas elements not visible in DOM')
          return false
        }

        const produksiCtx = produksiCanvas.getContext('2d')
        const jenisCtx = jenisCanvas.getContext('2d')

        if (!produksiCtx || !jenisCtx) {
          console.warn('Canvas context not available')
          return false
        }

        const { monthlyGabah, monthlyBeras } = this.getMonthlyData()
        const { gabahTypes, berasTypes } = this.getProductTypeData()

        this.produksiChart = new Chart(produksiCtx, {
          type: 'line',
          data: {
            labels: ['Jan', 'Feb', 'Mar', 'Apr', 'Mei', 'Jun', 'Jul', 'Agu', 'Sep', 'Okt', 'Nov', 'Des'],
            datasets: [
              {
                label: 'Produksi Gabah (Ton)',
                data: monthlyGabah,
                borderColor: '#1f2937',
                backgroundColor: 'rgba(31, 41, 55, 0.1)',
                pointBackgroundColor: '#1f2937',
                pointBorderColor: '#ffffff',
                pointBorderWidth: 2,
                pointRadius: 5,
                pointHoverRadius: 7,
                tension: 0.4,
                fill: false,
                borderWidth: 3
              },
              {
                label: 'Produksi Beras (Ton)',
                data: monthlyBeras,
                borderColor: '#6b7280',
                backgroundColor: 'rgba(107, 114, 128, 0.1)',
                pointBackgroundColor: '#6b7280',
                pointBorderColor: '#ffffff',
                pointBorderWidth: 2,
                pointRadius: 5,
                pointHoverRadius: 7,
                tension: 0.4,
                fill: false,
                borderWidth: 3
              }
            ]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            animation: {
              duration: 0
            },
            plugins: {
              legend: {
                labels: {
                  color: '#374151',
                  font: {
                    weight: 'bold'
                  }
                }
              }
            },
            scales: {
              x: {
                ticks: {
                  color: '#6b7280'
                },
                grid: {
                  color: '#e5e7eb'
                }
              },
              y: {
                beginAtZero: true,
                ticks: {
                  color: '#6b7280'
                },
                grid: {
                  color: '#e5e7eb'
                }
              }
            }
          }
        })

        // Perbaikan untuk mendapatkan allTypes
        const allTypes = Array.from(new Set([
          ...Object.keys(gabahTypes),
          ...Object.keys(berasTypes)
        ])).slice(0, 8)

        this.jenisChart = new Chart(jenisCtx, {
          type: 'bar',
          data: {
            labels: allTypes,
            datasets: [
              {
                label: 'Gabah (Ton)',
                data: allTypes.map(type => gabahTypes[type] || 0),
                backgroundColor: '#374151',
                borderColor: '#1f2937',
                borderWidth: 1,
                hoverBackgroundColor: '#4b5563',
                hoverBorderColor: '#111827'
              },
              {
                label: 'Beras (Ton)',
                data: allTypes.map(type => berasTypes[type] || 0),
                backgroundColor: '#9ca3af',
                borderColor: '#6b7280',
                borderWidth: 1,
                hoverBackgroundColor: '#d1d5db',
                hoverBorderColor: '#374151'
              }
            ]
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            animation: {
              duration: 0
            },
            plugins: {
              legend: {
                labels: {
                  color: '#374151',
                  font: {
                    weight: 'bold'
                  }
                }
              }
            },
            scales: {
              x: {
                ticks: {
                  color: '#6b7280'
                },
                grid: {
                  color: '#e5e7eb'
                }
              },
              y: {
                beginAtZero: true,
                ticks: {
                  color: '#6b7280'
                },
                grid: {
                  color: '#e5e7eb'
                }
              }
            }
          }
        })

        return true
      } catch (error) {
        console.error('Error initializing charts:', error)
        return false
      }
    }
  }
}
</script>

<style scoped>
/* Reset and base */
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

/* Cards Section */
.cards-section {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.5rem;
}

.dashboard-card {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 0.5rem;
  padding: 1.5rem;
  transition: all 0.15s ease;
  position: relative;
  overflow: hidden;
}

.dashboard-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 3px;
  background: linear-gradient(90deg, #111827, #374151, #6b7280);
}

.dashboard-card:hover {
  box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.1), 0 8px 10px -6px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.card-header {
  margin-bottom: 1rem;
}

.card-title {
  font-size: 0.875rem;
  font-weight: 600;
  color: #6b7280;
  margin: 0;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.card-content {
  text-align: left;
}

.card-value {
  font-size: 2rem;
  font-weight: 700;
  color: #111827;
  margin-bottom: 0.25rem;
}

.card-desc {
  font-size: 0.875rem;
  color: #6b7280;
  font-weight: 500;
}

/* Charts Section */
.charts-section {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.chart-card {
  margin-bottom: 0;
}

.chart-header {
  margin-bottom: 1.5rem;
}

.chart-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #111827;
  margin: 0;
}

.chart-wrapper {
  position: relative;
  height: 300px;
  width: 100%;
}

@media (max-width: 768px) {
  .charts-section {
    padding: 1.5rem 1rem;
  }

  .chart-wrapper {
    height: 250px;
  }
}

/* Table Section */
.table-section {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.table-section:last-child {
  margin-bottom: 0;
  border-bottom: none;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  margin-bottom: 1.5rem;
  color: #111827;
}

.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.no-data {
  text-align: center;
  padding: 2rem;
  color: #6b7280;
  font-style: italic;
  font-size: 0.875rem;
}

.table-container {
  overflow-x: auto;
  border-radius: 0.375rem;
  border: 1px solid #e5e7eb;
}

.price-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
  background: #ffffff;
}

.price-table th,
.price-table td {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #e5e7eb;
}

.price-table th {
  font-weight: 600;
  color: #111827;
  background-color: #f9fafb;
  user-select: none;
}

.price-table tbody tr:hover {
  background-color: #f9fafb;
}

@media (max-width: 768px) {
  .container {
    padding: 1rem 0;
  }

  .headline-card,
  .cards-section,
  .charts-section,
  .table-section {
    padding: 1.5rem 1rem;
  }

  .headline {
    font-size: 1.5rem;
    padding: 0;
  }

  .card-grid {
    grid-template-columns: 1fr;
    gap: 1rem;
  }

  .dashboard-card {
    padding: 1rem;
  }

  .card-value {
    font-size: 1.5rem;
  }

  .chart-wrapper {
    height: 250px;
  }

  .table-container {
    overflow-x: scroll;
  }
}
</style>