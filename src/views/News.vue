<template>
  <main class="container" role="main" aria-label="Manajemen Berita">
    <div class="headline-card">
      <h1 class="headline">Manajemen Berita</h1>
    </div>

    <!-- Form Input/Edit -->
    <section class="form-section" aria-labelledby="form-title">
      <h2 id="form-title" class="section-title">{{ editingId ? 'Edit Berita' : 'Tambah Berita Baru' }}</h2>
      <form @submit.prevent="submitNews" class="form" novalidate>
        <div class="form-group">
          <label for="title" class="label">Judul Berita</label>
          <input
            id="title"
            v-model="title"
            type="text"
            placeholder="Masukkan judul berita"
            required
            class="input"
            aria-required="true"
            autocomplete="off"
          />
        </div>

        <div class="form-group">
          <label for="content" class="label">Isi Lengkap Berita</label>
          <textarea
            id="content"
            v-model="content"
            placeholder="Masukkan isi berita"
            required
            class="textarea"
            aria-required="true"
          ></textarea>
        </div>

        <div class="form-group">
          <label for="theme" class="label">Tema Berita</label>
          <select id="theme" v-model="theme" required class="select" aria-required="true">
            <option value="" disabled>Pilih tema</option>
            <option value="Pertanian">Pertanian</option>
            <option value="Ekonomi">Ekonomi</option>
          </select>
        </div>

        <!-- Image Upload Section -->
        <div class="form-group">
          <label for="image" class="label">Gambar Berita (Opsional)</label>
          <div class="image-upload-container">
            <input
              id="image"
              ref="imageInput"
              type="file"
              accept="image/*"
              @change="handleImageUpload"
              class="image-input"
            />
            <label for="image" class="image-upload-label">
              <svg class="upload-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6"></path>
              </svg>
              Pilih Gambar
            </label>
            <p class="image-hint">Maksimal 500KB - Format: JPG, PNG, GIF</p>
          </div>
          
          <!-- Image Preview -->
          <div v-if="imagePreview" class="image-preview-container">
            <img :src="imagePreview" alt="Preview gambar" class="image-preview" />
            <button type="button" @click="removeImage" class="remove-image-btn">
              <svg class="remove-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
              </svg>
            </button>
          </div>

          <!-- Image Upload Status -->
          <div v-if="imageUploadStatus" class="image-status" :class="imageUploadStatus.type">
            {{ imageUploadStatus.message }}
          </div>
        </div>

        <div class="button-group">
          <button type="submit" class="submit-button" :disabled="loading" :aria-busy="loading.toString()" aria-live="polite">
            {{ loading ? (editingId ? 'Mengupdate...' : 'Menyimpan...') : (editingId ? 'Update' : 'Tambah') }}
          </button>
          <button 
            v-if="editingId" 
            type="button" 
            @click="cancelEdit" 
            class="cancel-button"
            :disabled="loading"
          >
            Batal
          </button>
        </div>
      </form>

      <div v-if="message.text" :class="messageClass" role="alert" aria-live="assertive" tabindex="0">
        {{ message.text }}
      </div>
    </section>

    <!-- Data Table -->
    <section class="table-section" aria-labelledby="table-title">
      <header class="table-header">
        <h2 id="table-title" class="section-title">Daftar Berita</h2>
        <button @click="loadNews" class="refresh-button" :disabled="loadingNews" aria-live="polite" :aria-disabled="loadingNews.toString()">
          {{ loadingNews ? 'Memuat...' : 'Refresh' }}
        </button>
      </header>

      <!-- Search Filter -->
      <div class="search-section">
        <input v-model="searchTerm" type="search" placeholder="Cari berdasarkan judul atau isi berita..." class="search-input" aria-label="Cari berita berdasarkan judul atau isi" />
        <select v-model="filterTheme" class="filter-select" aria-label="Filter berdasarkan tema">
          <option value="">Semua Tema</option>
          <option value="Pertanian">Pertanian</option>
          <option value="Ekonomi">Ekonomi</option>
        </select>
      </div>

      <div v-if="loadingNews" class="loading" aria-live="polite" aria-busy="true">
        Memuat data...
      </div>

      <div v-else-if="filteredNews.length === 0" class="no-data" role="status">
        Tidak ada data berita yang tersedia
      </div>

      <div v-else class="table-container">
        <table class="price-table" role="table" aria-describedby="table-title">
          <thead>
            <tr>
              <th scope="col">Gambar</th>
              <th scope="col">Judul</th>
              <th scope="col">Tema</th>
              <th scope="col">Isi Berita</th>
              <th scope="col">Tanggal Input</th>
              <th scope="col">Aksi</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="news in filteredNews" :key="news.id">
              <td class="image-cell">
                <img 
                  v-if="news.image" 
                  :src="news.image" 
                  :alt="`Gambar ${news.title}`" 
                  class="table-image"
                  @click="viewImage(news.image, news.title)"
                />
                <span v-else class="no-image">Tidak ada gambar</span>
              </td>
              <td class="title-cell">{{ news.title }}</td>
              <td>
                <span class="theme-badge">{{ news.theme }}</span>
              </td>
              <td class="content-cell">{{ truncateContent(news.content) }}</td>
              <td>{{ formatDate(news.inputTime) }}</td>
              <td class="action-cell">
                <button @click="editNews(news)" class="edit-button" :aria-label="`Edit berita ${news.title}`">
                  Edit
                </button>
                <button @click="deleteNews(news.id, news.title)" class="delete-button" :aria-label="`Hapus berita ${news.title}`">
                  Hapus
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>

    <!-- Delete Confirmation Modal -->
    <div v-if="showDeleteModal" class="modal-overlay" @click="cancelDelete" role="dialog" aria-modal="true" aria-labelledby="modal-title" aria-describedby="modal-text" tabindex="-1">
      <div class="modal" @click.stop>
        <h3 id="modal-title" class="modal-title">Konfirmasi Hapus</h3>
        <p id="modal-text" class="modal-text">Apakah Anda yakin ingin menghapus berita "{{ deleteTitle }}"?</p>
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

    <!-- Image View Modal -->
    <div v-if="showImageModal" class="modal-overlay" @click="closeImageModal" role="dialog" aria-modal="true" aria-labelledby="image-modal-title">
      <div class="image-modal" @click.stop>
        <div class="image-modal-header">
          <h3 id="image-modal-title" class="modal-title">{{ imageModalTitle }}</h3>
          <button @click="closeImageModal" class="close-button">
            <svg class="close-icon" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"></path>
            </svg>
          </button>
        </div>
        <div class="image-modal-content">
          <img :src="imageModalSrc" :alt="imageModalTitle" class="modal-image" />
        </div>
      </div>
    </div>
  </main>
</template>

<script>
import { ref, set, remove, onValue, off } from 'firebase/database'
import { database } from '../firebase.js' // Adjust path if needed

export default {
  name: 'NewsCRUD',
  data() {
    return {
      title: '',
      content: '',
      theme: '',
      imageBase64: '',
      imagePreview: '',
      imageUploadStatus: null,
      loading: false,
      loadingNews: false,
      editingId: null,
      newsList: [],
      searchTerm: '',
      filterTheme: '',
      message: {
        text: '',
        type: '' // 'success' or 'error'
      },
      
      // Delete modal
      showDeleteModal: false,
      deleteId: null,
      deleteTitle: '',

      // Image view modal
      showImageModal: false,
      imageModalSrc: '',
      imageModalTitle: ''
    }
  },
  computed: {
    messageClass() {
      return this.message.type === 'success' ? 'success-message' : 'error-message'
    },
    filteredNews() {
      let filtered = [...this.newsList]

      // Filter by search term
      if (this.searchTerm) {
        const term = this.searchTerm.toLowerCase()
        filtered = filtered.filter(news =>
          news.title.toLowerCase().includes(term) ||
          news.content.toLowerCase().includes(term)
        )
      }

      // Filter by theme
      if (this.filterTheme) {
        filtered = filtered.filter(news => news.theme === this.filterTheme)
      }

      // Sort by input time (newest first)
      filtered.sort((a, b) => b.inputTime - a.inputTime)

      return filtered
    }
  },
  mounted() {
    this.loadNews()
  },
  beforeUnmount() {
    // Clean up listener
    const newsRef = ref(database, 'news')
    off(newsRef)
  },
  methods: {
    // Image handling methods
    handleImageUpload(event) {
      const file = event.target.files[0]
      if (!file) return

      // Validate file size (500KB = 500 * 1024 bytes)
      const maxSize = 500 * 1024
      if (file.size > maxSize) {
        this.imageUploadStatus = {
          type: 'error',
          message: 'Ukuran gambar terlalu besar. Maksimal 500KB.'
        }
        this.clearImageUploadStatus()
        return
      }

      // Validate file type
      if (!file.type.startsWith('image/')) {
        this.imageUploadStatus = {
          type: 'error',
          message: 'File harus berupa gambar (JPG, PNG, GIF).'
        }
        this.clearImageUploadStatus()
        return
      }

      // Convert to base64
      const reader = new FileReader()
      reader.onload = (e) => {
        this.imageBase64 = e.target.result
        this.imagePreview = e.target.result
        this.imageUploadStatus = {
          type: 'success',
          message: 'Gambar berhasil diunggah.'
        }
        this.clearImageUploadStatus()
      }
      reader.onerror = () => {
        this.imageUploadStatus = {
          type: 'error',
          message: 'Gagal membaca file gambar.'
        }
        this.clearImageUploadStatus()
      }
      reader.readAsDataURL(file)
    },

    removeImage() {
      this.imageBase64 = ''
      this.imagePreview = ''
      this.imageUploadStatus = null
      if (this.$refs.imageInput) {
        this.$refs.imageInput.value = ''
      }
    },

    clearImageUploadStatus() {
      setTimeout(() => {
        this.imageUploadStatus = null
      }, 3000)
    },

    viewImage(imageSrc, title) {
      this.imageModalSrc = imageSrc
      this.imageModalTitle = title
      this.showImageModal = true
    },

    closeImageModal() {
      this.showImageModal = false
      this.imageModalSrc = ''
      this.imageModalTitle = ''
    },

    loadNews() {
      this.loadingNews = true
      try {
        const newsRef = ref(database, 'news')
        
        onValue(
          newsRef, 
          (snapshot) => {
            const data = snapshot.val()
            if (data) {
              this.newsList = Object.keys(data).map(key => ({
                id: key,
                ...data[key]
              }))
            } else {
              this.newsList = []
            }
            this.loadingNews = false
          }, 
          (error) => {
            console.error('Error loading news:', error)
            this.showMessage('Gagal memuat berita', 'error')
            this.loadingNews = false
          }
        )
      } catch (error) {
        console.error('Error setting up listener:', error)
        this.showMessage('Gagal memuat berita', 'error')
        this.loadingNews = false
      }
    },

    async submitNews() {
      if (this.loading) return
      this.loading = true
      this.clearMessage()

      try {
        const newsData = {
          title: this.title.trim(),
          content: this.content.trim(),
          theme: this.theme,
          inputTime: this.editingId ? 
            this.newsList.find(n => n.id === this.editingId)?.inputTime || Date.now() : 
            Date.now(),
          lastUpdated: Date.now()
        }

        // Add image if provided
        if (this.imageBase64) {
          newsData.image = this.imageBase64
        }

        const newsId = this.editingId || Date.now().toString()
        const newsRef = ref(database, `news/${newsId}`)
        await set(newsRef, newsData)

        this.showMessage(
          this.editingId ? 'Berita berhasil diperbarui!' : 'Berita berhasil ditambahkan!', 
          'success'
        )

        this.resetForm()
      } catch (error) {
        console.error('Error submitting news:', error)
        this.showMessage('Gagal menyimpan berita. Silakan coba lagi.', 'error')
      } finally {
        this.loading = false
      }
    },

    editNews(news) {
      this.editingId = news.id
      this.title = news.title
      this.content = news.content
      this.theme = news.theme
      
      // Load image if exists
      if (news.image) {
        this.imageBase64 = news.image
        this.imagePreview = news.image
      }
      
      this.clearMessage()
      
      // Scroll to form
      const formSection = this.$el.querySelector('.form-section')
      if (formSection) {
        formSection.scrollIntoView({ behavior: 'smooth' })
      }
    },

    deleteNews(newsId, newsTitle) {
      this.deleteId = newsId
      this.deleteTitle = newsTitle
      this.showDeleteModal = true
    },

    async confirmDelete() {
      if (!this.deleteId || this.loading) return
      this.loading = true

      try {
        const newsRef = ref(database, `news/${this.deleteId}`)
        await remove(newsRef)
        
        this.showMessage('Berita berhasil dihapus!', 'success')
        
        // If we're editing the deleted news, cancel edit
        if (this.editingId === this.deleteId) {
          this.cancelEdit()
        }
        
        this.cancelDelete()
      } catch (error) {
        console.error('Error deleting news:', error)
        this.showMessage('Gagal menghapus berita. Silakan coba lagi.', 'error')
      } finally {
        this.loading = false
      }
    },

    cancelDelete() {
      this.showDeleteModal = false
      this.deleteId = null
      this.deleteTitle = ''
    },

    cancelEdit() {
      this.resetForm()
    },

    resetForm() {
      this.title = ''
      this.content = ''
      this.theme = ''
      this.editingId = null
      this.removeImage()
    },

    showMessage(text, type) {
      this.message = { text, type }
      setTimeout(() => {
        this.clearMessage()
      }, 5000)
    },

    clearMessage() {
      this.message = { text: '', type: '' }
    },

    truncateContent(content, maxLength = 100) {
      if (content.length <= maxLength) return content
      return content.substring(0, maxLength) + '...'
    },

    formatDate(timestamp) {
      if (!timestamp) return '-'
      return new Date(timestamp).toLocaleDateString('id-ID', {
        year: 'numeric',
        month: 'short',
        day: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      })
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
}

.textarea {
  min-height: 120px;
  resize: vertical;
}

.input:focus,
.select:focus,
.textarea:focus {
  outline: none;
  border-color: #6b7280;
  box-shadow: 0 0 0 1px #6b7280;
}

/* Image Upload Styles */
.image-upload-container {
  position: relative;
  margin-bottom: 1rem;
}

.image-input {
  display: none;
}

.image-upload-label {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
  padding: 1rem;
  border: 2px dashed #d1d5db;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: all 0.15s ease;
  background-color: #f9fafb;
  color: #6b7280;
  font-size: 0.875rem;
  font-weight: 500;
}

.image-upload-label:hover {
  border-color: #9ca3af;
  background-color: #f3f4f6;
}

.upload-icon {
  width: 1.25rem;
  height: 1.25rem;
}

.image-hint {
  margin-top: 0.5rem;
  font-size: 0.75rem;
  color: #6b7280;
  text-align: center;
}

.image-preview-container {
  position: relative;
  display: inline-block;
  margin-top: 1rem;
}

.image-preview {
  max-width: 200px;
  max-height: 200px;
  border-radius: 0.375rem;
  border: 1px solid #e5e7eb;
  object-fit: cover;
}

.remove-image-btn {
  position: absolute;
  top: -0.5rem;
  right: -0.5rem;
  background-color: #dc2626;
  color: white;
  border: none;
  border-radius: 50%;
  width: 1.5rem;
  height: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.remove-image-btn:hover {
  background-color: #b91c1c;
}

.remove-icon {
  width: 0.75rem;
  height: 0.75rem;
}

.image-status {
  margin-top: 0.5rem;
  padding: 0.5rem;
  border-radius: 0.25rem;
  font-size: 0.75rem;
  font-weight: 500;
}

.image-status.success {
  background-color: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
}

.image-status.error {
  background-color: #fee2e2;
  color: #991b1b;
  border: 1px solid #fca5a5;
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

.image-cell {
  width: 80px;
  text-align: center;
}

.table-image {
  width: 50px;
  height: 50px;
  object-fit: cover;
  border-radius: 0.25rem;
  cursor: pointer;
  transition: transform 0.15s ease;
}

.table-image:hover {
  transform: scale(1.1);
}

.no-image {
  font-size: 0.75rem;
  color: #9ca3af;
  font-style: italic;
}

.title-cell {
  font-weight: 600;
  color: #111827;
  max-width: 200px;
}

.content-cell {
  max-width: 250px;
  color: #6b7280;
}

.theme-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  font-size: 0.75rem;
  font-weight: 500;
  border-radius: 1rem;
  background-color: #111827;
  color: #ffffff;
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