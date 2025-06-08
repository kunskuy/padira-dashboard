<template>
  <main class="container" role="main" aria-label="Manajemen Harga Gabah dan Beras">
    <div class="headline-card">
      <h1 class="headline">Manajemen Harga Gabah dan Beras</h1>
    </div>

    <!-- Form Input/Edit -->
    <section class="form-section" aria-labelledby="form-title">
      <h2 id="form-title" class="section-title">{{ isEditing ? 'Edit Harga' : 'Tambah Harga Baru' }}</h2>
      <form @submit.prevent="submitPrice" class="form" novalidate>
        <div class="form-group">
          <label for="productType" class="label">Jenis Harga</label>
          <select id="productType" v-model="productType" required class="select" aria-required="true"
            aria-describedby="productTypeHelp">
            <option value="" disabled>Pilih opsi</option>
            <option value="Gabah">Harga Gabah</option>
            <option value="Beras">Harga Beras</option>
          </select>
          <small id="productTypeHelp" class="form-help">Pilih jenis harga yang sesuai.</small>
        </div>

        <div class="form-group">
          <label for="name" class="label">Nama Gabah/Beras</label>
          <input id="name" v-model="name" type="text" placeholder="Masukkan nama gabah atau beras" required
            class="input" aria-required="true" autocomplete="off" />
        </div>

        <div class="form-group">
          <label for="type" class="label">Jenis/Kualitas</label>
          <input id="type" v-model="type" type="text" placeholder="Masukkan jenis atau kualitas" required class="input"
            aria-required="true" autocomplete="off" />
        </div>

        <div class="form-group">
          <label for="price" class="label">Harga per Kg (Rp)</label>
          <input id="price" v-model.number="price" type="number" min="0" placeholder="Masukkan harga per kilogram"
            required class="input" aria-required="true" autocomplete="off" step="1000" />
        </div>

        <div class="button-group">
          <button type="submit" class="submit-button" :disabled="loading" :aria-busy="loading.toString()"
            aria-live="polite">
            {{ loading ? (isEditing ? 'Menyimpan...' : 'Menambahkan...') : (isEditing ? 'Update' : 'Tambah') }}
          </button>
          <button v-if="isEditing" type="button" @click="cancelEdit" class="cancel-button" :disabled="loading">
            Batal
          </button>
        </div>
      </form>

      <div v-if="message" :class="messageClass" role="alert" aria-live="assertive" tabindex="0">
        {{ message }}
      </div>
    </section>

    <!-- Data Table -->
    <section class="table-section" aria-labelledby="table-title">
      <header class="table-header">
        <h2 id="table-title" class="section-title">Daftar Harga</h2>
        <button @click="loadPrices" class="refresh-button" :disabled="loadingData" aria-live="polite"
          aria-disabled="loadingData.toString()">
          {{ loadingData ? 'Memuat...' : 'Refresh' }}
        </button>
      </header>

      <!-- Search Filter -->
      <div class="search-section">
        <input v-model="searchTerm" type="search" placeholder="Cari berdasarkan nama atau jenis..." class="search-input"
          aria-label="Cari harga berdasarkan nama atau jenis" />
        <select v-model="filterType" class="filter-select" aria-label="Filter berdasarkan jenis produk">
          <option value="">Semua Jenis</option>
          <option value="Gabah">Gabah</option>
          <option value="Beras">Beras</option>
        </select>
      </div>

      <div v-if="loadingData" class="loading" aria-live="polite" aria-busy="true">
        Memuat data...
      </div>

      <div v-else-if="filteredPrices.length === 0" class="no-data" role="status">
        Tidak ada data harga yang tersedia
      </div>

      <div v-else class="table-container">
        <table class="price-table" role="table" aria-describedby="table-title">
          <thead>
            <tr>
              <th scope="col">Jenis</th>
              <th scope="col">Nama</th>
              <th scope="col">Kualitas</th>
              <th scope="col">Harga (Rp/Kg)</th>
              <th scope="col">Tanggal Input</th>
              <th scope="col">Aksi</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in filteredPrices" :key="item.key">
              <td>{{ item.productType }}</td>
              <td>{{ item.name }}</td>
              <td>{{ item.type }}</td>
              <td class="price-cell">{{ formatPrice(item.price) }}</td>
              <td>{{ formatDate(item.inputTime) }}</td>
              <td class="action-cell">
                <button @click="editPrice(item.key, item)" class="edit-button"
                  :aria-label="`Edit harga untuk ${item.name}`">
                  Edit
                </button>
                <button @click="deletePrice(item.key)" class="delete-button"
                  :aria-label="`Hapus harga untuk ${item.name}`">
                  Hapus
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- Delete Confirmation Modal -->
    <div v-if="showDeleteModal" class="modal-overlay" @click="cancelDelete" role="dialog" aria-modal="true"
      aria-labelledby="modal-title" aria-describedby="modal-text" tabindex="-1">
      <div class="modal" @click.stop>
        <h3 id="modal-title" class="modal-title">Konfirmasi Hapus</h3>
        <p id="modal-text" class="modal-text">Apakah Anda yakin ingin menghapus harga ini?</p>
        <div class="modal-buttons">
          <button @click="confirmDelete" class="confirm-button" :disabled="loading" :aria-busy="loading.toString()">
            {{ loading ? 'Menghapus...' : 'Ya, Hapus' }}
          </button>
          <button @click="cancelDelete" class="cancel-modal-button" :disabled="loading">
            Batal
          </button>
        </div>
      </div>
    </div>
  </main>
</template>

<script>
import { ref, set, remove, onValue, off } from 'firebase/database'
import { database } from '../firebase.js' // Adjust path if needed

export default {
  name: 'PriceCRUD',
  data() {
    return {
      // Form data
      productType: '',
      name: '',
      type: '',
      price: null,
      loading: false,
      loadingData: false,
      message: '',
      messageType: 'success', // 'success' or 'error'

      // CRUD state
      isEditing: false,
      editingKey: null,

      // Data management
      prices: {},
      searchTerm: '',
      filterType: '',

      // Delete modal
      showDeleteModal: false,
      deleteKey: null,
    }
  },
  computed: {
    messageClass() {
      return this.messageType === 'success' ? 'success-message' : 'error-message'
    },
    filteredPrices() {
      let filtered = Object.entries(this.prices).map(([key, value]) => ({
        key,
        ...value,
      }))

      // Filter by search term
      if (this.searchTerm) {
        const term = this.searchTerm.toLowerCase()
        filtered = filtered.filter(item =>
          item.name.toLowerCase().includes(term) ||
          item.type.toLowerCase().includes(term)
        )
      }

      // Filter by product type
      if (this.filterType) {
        filtered = filtered.filter(item => item.productType === this.filterType)
      }

      // Sort by input time (newest first)
      filtered.sort((a, b) => b.inputTime - a.inputTime)

      return filtered
    },
  },
  mounted() {
    this.loadPrices()
  },
  beforeUnmount() {
    // Clean up Firebase listener
    const pricesRef = ref(database, 'prices')
    off(pricesRef)
  },
  methods: {
    async loadPrices() {
      this.loadingData = true
      try {
        const pricesRef = ref(database, 'prices')
        onValue(
          pricesRef,
          snapshot => {
            const data = snapshot.val()
            this.prices = data || {}
            this.loadingData = false
          },
          error => {
            console.error('Error loading prices:', error)
            this.showMessage('Gagal memuat data harga', 'error')
            this.loadingData = false
          }
        )
      } catch (error) {
        console.error('Error setting up listener:', error)
        this.showMessage('Gagal memuat data harga', 'error')
        this.loadingData = false
      }
    },

    async submitPrice() {
      if (this.loading) return
      this.loading = true
      this.message = ''

      try {
        const priceData = {
          productType: this.productType,
          name: this.name.trim(),
          type: this.type.trim(),
          price: this.price,
          inputTime: this.isEditing
            ? this.prices[this.editingKey]?.inputTime || Date.now()
            : Date.now(),
          lastUpdated: Date.now(),
        }

        const priceKey = this.isEditing ? this.editingKey : Date.now().toString()
        const priceRef = ref(database, `prices/${priceKey}`)
        await set(priceRef, priceData)

        this.showMessage(
          this.isEditing ? 'Harga berhasil diperbarui!' : 'Harga berhasil ditambahkan!',
          'success'
        )
        this.resetForm()
      } catch (error) {
        console.error('Error submitting price:', error)
        this.showMessage('Gagal menyimpan harga. Silakan coba lagi.', 'error')
      } finally {
        this.loading = false
      }
    },

    editPrice(key, priceItem) {
      this.isEditing = true
      this.editingKey = key
      this.productType = priceItem.productType
      this.name = priceItem.name
      this.type = priceItem.type
      this.price = priceItem.price
      this.message = ''

      // Scroll to form
      const formSection = this.$el.querySelector('.form-section')
      if (formSection) {
        formSection.scrollIntoView({ behavior: 'smooth' })
      }
    },

    cancelEdit() {
      this.resetForm()
    },

    deletePrice(key) {
      this.deleteKey = key
      this.showDeleteModal = true
    },

    async confirmDelete() {
      if (!this.deleteKey || this.loading) return
      this.loading = true

      try {
        const priceRef = ref(database, `prices/${this.deleteKey}`)
        await remove(priceRef)
        this.showMessage('Harga berhasil dihapus!', 'success')
        this.cancelDelete()
      } catch (error) {
        console.error('Error deleting price:', error)
        this.showMessage('Gagal menghapus harga. Silakan coba lagi.', 'error')
      } finally {
        this.loading = false
      }
    },

    cancelDelete() {
      this.showDeleteModal = false
      this.deleteKey = null
    },

    resetForm() {
      this.productType = ''
      this.name = ''
      this.type = ''
      this.price = null
      this.isEditing = false
      this.editingKey = null
    },

    showMessage(text, type) {
      this.message = text
      this.messageType = type
      setTimeout(() => {
        this.message = ''
      }, 5000)
    },

    formatPrice(price) {
      return new Intl.NumberFormat('id-ID').format(price)
    },

    formatDate(timestamp) {
      if (!timestamp) return '-'
      return new Date(timestamp).toLocaleDateString('id-ID', {
        year: 'numeric',
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit',
      })
    },
  },
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
  /* Ubah dari 2rem menjadi 0 */
  color: #111827;
  text-align: center;
  padding: 0;
  /* Ubah dari 0 2rem menjadi 0 */
}

.form-section {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  margin-bottom: 1.5rem;
  color: #111827;
}

.form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-group {
  display: flex;
  flex-direction: column;
}

.label {
  font-weight: 500;
  color: #374151;
  margin-bottom: 0.5rem;
  font-size: 0.875rem;
}

.form-help {
  font-size: 0.75rem;
  color: #6b7280;
  margin-top: 0.25rem;
}

.input,
.select {
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  border-radius: 0.375rem;
  border: 1px solid #d1d5db;
  background-color: #ffffff;
  color: #111827;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.input:focus,
.select:focus {
  outline: none;
  border-color: #6b7280;
  box-shadow: 0 0 0 1px #6b7280;
}

.button-group {
  display: flex;
  gap: 0.75rem;
  justify-content: flex-start;
  margin-top: 1rem;
}

.submit-button,
.cancel-button {
  padding: 0.75rem 1.5rem;
  font-size: 0.875rem;
  font-weight: 500;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.submit-button {
  background-color: #111827;
  color: #ffffff;
  flex: 1;
}

.submit-button:hover:not(:disabled) {
  background-color: #374151;
}

.submit-button:disabled {
  background-color: #9ca3af;
  cursor: not-allowed;
}

.cancel-button {
  background-color: #6b7280;
  color: #ffffff;
  flex: none;
}

.cancel-button:hover:not(:disabled) {
  background-color: #4b5563;
}

.success-message,
.error-message {
  margin-top: 1.5rem;
  padding: 0.75rem 1rem;
  border-radius: 0.375rem;
  font-weight: 500;
  font-size: 0.875rem;
  text-align: center;
}

.success-message {
  background-color: #f3f4f6;
  color: #111827;
  border: 1px solid #d1d5db;
}

.error-message {
  background-color: #f9f9f9;
  color: #dc2626;
  border: 1px solid #d1d5db;
}

.table-section {
  background: #ffffff;
  margin: 0;
  padding: 2rem;
  border-top: 1px solid #e5e7eb;
}

.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

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
}

.refresh-button:hover:not(:disabled) {
  background-color: #374151;
}

.refresh-button:disabled {
  background-color: #9ca3af;
  cursor: not-allowed;
}

.search-section {
  display: flex;
  gap: 1rem;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.search-input {
  flex: 1;
  min-width: 180px;
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  border-radius: 0.375rem;
  border: 1px solid #d1d5db;
  background-color: #ffffff;
  color: #374151;
}

.filter-select {
  min-width: 150px;
  padding: 0.5rem 0.75rem;
  font-size: 0.875rem;
  border-radius: 0.375rem;
  border: 1px solid #d1d5db;
  background-color: #ffffff;
  color: #374151;
}

.loading,
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

.price-cell {
  font-weight: 600;
  color: #111827;
}

.action-cell {
  display: flex;
  gap: 0.5rem;
  justify-content: flex-start;
}

.edit-button,
.delete-button {
  padding: 0.25rem 0.75rem;
  font-size: 0.75rem;
  font-weight: 500;
  border: 1px solid;
  border-radius: 0.25rem;
  cursor: pointer;
  transition: all 0.15s ease;
}

.edit-button {
  background-color: #ffffff;
  color: #374151;
  border-color: #d1d5db;
}

.edit-button:hover {
  background-color: #f3f4f6;
  border-color: #9ca3af;
}

.delete-button {
  background-color: #ffffff;
  color: #dc2626;
  border-color: #d1d5db;
}

.delete-button:hover {
  background-color: #fef2f2;
  border-color: #fca5a5;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 50;
  padding: 1rem;
}

.modal {
  background: #ffffff;
  border-radius: 0.5rem;
  padding: 1.5rem;
  max-width: 400px;
  width: 100%;
  border: 1px solid #e5e7eb;
  outline: none;
}

.modal-title {
  font-size: 1.125rem;
  font-weight: 600;
  margin-bottom: 0.75rem;
  color: #111827;
}

.modal-text {
  margin-bottom: 1.5rem;
  color: #6b7280;
  font-size: 0.875rem;
}

.modal-buttons {
  display: flex;
  gap: 0.75rem;
}

.confirm-button,
.cancel-modal-button {
  padding: 0.5rem 1rem;
  font-size: 0.875rem;
  font-weight: 500;
  border: 1px solid;
  border-radius: 0.375rem;
  cursor: pointer;
  flex: 1;
  transition: all 0.15s ease;
}

.confirm-button {
  background-color: #dc2626;
  color: #ffffff;
  border-color: #dc2626;
}

.confirm-button:hover:not(:disabled) {
  background-color: #b91c1c;
  border-color: #b91c1c;
}

.confirm-button:disabled {
  background-color: #9ca3af;
  border-color: #9ca3af;
  cursor: not-allowed;
}

.cancel-modal-button {
  background-color: #ffffff;
  color: #374151;
  border-color: #d1d5db;
}

.cancel-modal-button:hover:not(:disabled) {
  background-color: #f3f4f6;
  border-color: #9ca3af;
}

.cancel-modal-button:disabled {
  background-color: #f9fafb;
  color: #9ca3af;
  cursor: not-allowed;
}

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
    /* Ubah dari 0 1rem menjadi 0 */
  }

  .form-section,
  .table-section {
    padding: 1.5rem 1rem;
  }

  .button-group {
    flex-direction: column;
    gap: 0.75rem;
  }

  .submit-button,
  .cancel-button {
    width: 100%;
  }

  .search-section {
    flex-direction: column;
  }

  .action-cell {
    flex-direction: column;
  }

  .table-header {
    flex-direction: column;
    gap: 1rem;
    align-items: stretch;
  }

  .table-container {
    overflow-x: scroll;
  }

  .modal {
    max-width: 100%;
    margin: 1rem;
  }
}
</style>