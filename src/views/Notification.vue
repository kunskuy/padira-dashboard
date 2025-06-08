<template>
  <main class="container" role="main" aria-label="Manajemen Notifikasi">
    <div class="headline-card">
      <h1 class="headline">Manajemen Notifikasi</h1>
    </div>

    <!-- Form Input/Edit -->
    <section class="form-section" aria-labelledby="form-title">
      <h2 id="form-title" class="section-title">{{ isEditing ? 'Edit Notifikasi' : 'Tambah Notifikasi Baru' }}</h2>
      <form @submit.prevent="submitNotification" class="form" novalidate>
        <div class="form-group">
          <label for="notificationType" class="label">Jenis Notifikasi</label>
          <select id="notificationType" v-model="notificationType" required class="select" aria-required="true"
            aria-describedby="notificationTypeHelp">
            <option value="" disabled>Pilih opsi</option>
            <option value="Price">Notifikasi Harga</option>
            <option value="News">Notifikasi Berita</option>
          </select>
          <small id="notificationTypeHelp" class="form-help">Pilih jenis notifikasi yang sesuai.</small>
        </div>

        <div class="form-group">
          <label for="title" class="label">Judul Notifikasi</label>
          <input id="title" v-model="title" type="text" placeholder="Masukkan judul notifikasi" required
            class="input" aria-required="true" autocomplete="off" />
        </div>

        <div class="form-group">
          <label for="description" class="label">Deskripsi Notifikasi</label>
          <textarea id="description" v-model="description" placeholder="Masukkan deskripsi notifikasi" required
            class="textarea" aria-required="true" rows="4"></textarea>
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
        <h2 id="table-title" class="section-title">Daftar Notifikasi</h2>
        <button @click="loadNotifications" class="refresh-button" :disabled="loadingData" aria-live="polite"
          aria-disabled="loadingData.toString()">
          {{ loadingData ? 'Memuat...' : 'Refresh' }}
        </button>
      </header>

      <!-- Search Filter -->
      <div class="search-section">
        <input v-model="searchTerm" type="search" placeholder="Cari berdasarkan judul atau deskripsi..." class="search-input"
          aria-label="Cari notifikasi berdasarkan judul atau deskripsi" />
        <select v-model="filterType" class="filter-select" aria-label="Filter berdasarkan jenis notifikasi">
          <option value="">Semua Jenis</option>
          <option value="Price">Notifikasi Harga</option>
          <option value="News">Notifikasi Berita</option>
        </select>
      </div>

      <div v-if="loadingData" class="loading" aria-live="polite" aria-busy="true">
        Memuat data...
      </div>

      <div v-else-if="filteredNotifications.length === 0" class="no-data" role="status">
        Tidak ada data notifikasi yang tersedia
      </div>

      <div v-else class="table-container">
        <table class="notification-table" role="table" aria-describedby="table-title">
          <thead>
            <tr>
              <th scope="col">Jenis</th>
              <th scope="col">Judul</th>
              <th scope="col">Deskripsi</th>
              <th scope="col">Waktu Input</th>
              <th scope="col">Aksi</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in filteredNotifications" :key="item.key">
              <td>{{ getTypeLabel(item.notificationType) }}</td>
              <td>{{ item.title }}</td>
              <td class="description-cell">{{ truncateText(item.description, 50) }}</td>
              <td>{{ formatDate(item.inputTime) }}</td>
              <td class="action-cell">
                <button @click="editNotification(item.key, item)" class="edit-button"
                  :aria-label="`Edit notifikasi ${item.title}`">
                  Edit
                </button>
                <button @click="deleteNotification(item.key)" class="delete-button"
                  :aria-label="`Hapus notifikasi ${item.title}`">
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
        <p id="modal-text" class="modal-text">Apakah Anda yakin ingin menghapus notifikasi ini?</p>
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
import { database } from '../firebase.js'

export default {
  name: 'NotificationCRUD',
  data() {
    return {
      // Form data
      notificationType: '',
      title: '',
      description: '',
      loading: false,
      loadingData: false,
      message: '',
      messageType: 'success', // 'success' or 'error'

      // CRUD state
      isEditing: false,
      editingKey: null,

      // Data management
      notifications: {},
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
    filteredNotifications() {
      let filtered = Object.entries(this.notifications).map(([key, value]) => ({
        key,
        ...value,
      }))

      // Filter by search term
      if (this.searchTerm) {
        const term = this.searchTerm.toLowerCase()
        filtered = filtered.filter(item =>
          item.title.toLowerCase().includes(term) ||
          item.description.toLowerCase().includes(term)
        )
      }

      // Filter by notification type
      if (this.filterType) {
        filtered = filtered.filter(item => item.notificationType === this.filterType)
      }

      // Sort by input time (newest first)
      filtered.sort((a, b) => b.inputTime - a.inputTime)

      return filtered
    },
  },
  mounted() {
    this.loadNotifications()
  },
  beforeUnmount() {
    // Clean up Firebase listener
    const notificationsRef = ref(database, 'notifications')
    off(notificationsRef)
  },
  methods: {
    async loadNotifications() {
      this.loadingData = true
      try {
        const notificationsRef = ref(database, 'notifications')
        onValue(
          notificationsRef,
          snapshot => {
            const data = snapshot.val()
            this.notifications = data || {}
            this.loadingData = false
          },
          error => {
            console.error('Error loading notifications:', error)
            this.showMessage('Gagal memuat data notifikasi', 'error')
            this.loadingData = false
          }
        )
      } catch (error) {
        console.error('Error setting up listener:', error)
        this.showMessage('Gagal memuat data notifikasi', 'error')
        this.loadingData = false
      }
    },

    async submitNotification() {
      if (this.loading) return
      this.loading = true
      this.message = ''

      try {
        const notificationData = {
          notificationType: this.notificationType,
          title: this.title.trim(),
          description: this.description.trim(),
          inputTime: this.isEditing
            ? this.notifications[this.editingKey]?.inputTime || Date.now()
            : Date.now(),
          lastUpdated: Date.now(),
        }

        const notificationKey = this.isEditing ? this.editingKey : Date.now().toString()
        const notificationRef = ref(database, `notifications/${notificationKey}`)
        await set(notificationRef, notificationData)

        this.showMessage(
          this.isEditing ? 'Notifikasi berhasil diperbarui!' : 'Notifikasi berhasil ditambahkan!',
          'success'
        )
        this.resetForm()
      } catch (error) {
        console.error('Error submitting notification:', error)
        this.showMessage('Gagal menyimpan notifikasi. Silakan coba lagi.', 'error')
      } finally {
        this.loading = false
      }
    },

    editNotification(key, notificationItem) {
      this.isEditing = true
      this.editingKey = key
      this.notificationType = notificationItem.notificationType
      this.title = notificationItem.title
      this.description = notificationItem.description
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

    deleteNotification(key) {
      this.deleteKey = key
      this.showDeleteModal = true
    },

    async confirmDelete() {
      if (!this.deleteKey || this.loading) return
      this.loading = true

      try {
        const notificationRef = ref(database, `notifications/${this.deleteKey}`)
        await remove(notificationRef)
        this.showMessage('Notifikasi berhasil dihapus!', 'success')
        this.cancelDelete()
      } catch (error) {
        console.error('Error deleting notification:', error)
        this.showMessage('Gagal menghapus notifikasi. Silakan coba lagi.', 'error')
      } finally {
        this.loading = false
      }
    },

    cancelDelete() {
      this.showDeleteModal = false
      this.deleteKey = null
    },

    resetForm() {
      this.notificationType = ''
      this.title = ''
      this.description = ''
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

    getTypeLabel(type) {
      return type === 'Price' ? 'Notifikasi Harga' : 'Notifikasi Berita'
    },

    truncateText(text, maxLength) {
      if (text.length <= maxLength) return text
      return text.substring(0, maxLength) + '...'
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
  color: #111827;
  text-align: center;
  padding: 0;
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
.select,
.textarea {
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  border-radius: 0.375rem;
  border: 1px solid #d1d5db;
  background-color: #ffffff;
  color: #111827;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  font-family: inherit;
  resize: vertical;
}

.input:focus,
.select:focus,
.textarea:focus {
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

.notification-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
  background: #ffffff;
}

.notification-table th,
.notification-table td {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #e5e7eb;
}

.notification-table th {
  font-weight: 600;
  color: #111827;
  background-color: #f9fafb;
  user-select: none;
}

.notification-table tbody tr:hover {
  background-color: #f9fafb;
}

.description-cell {
  max-width: 200px;
  word-wrap: break-word;
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