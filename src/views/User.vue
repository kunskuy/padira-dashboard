<template>
  <main class="container" role="main" aria-label="User Management">
    <!-- Header Section -->
    <div class="headline-card">
      <div class="section-header">
        <h1 class="headline">Manajemen Pengguna</h1>
      </div>
    </div>

    <!-- Stats Cards Section -->
    <section class="cards-section" aria-labelledby="stats-title">
      <div class="card-grid">
        <div class="dashboard-card">
          <div class="card-header">
            <h3 class="card-title">Total Petani</h3>
          </div>
          <div class="card-content">
            <div class="card-value">{{ stats.petani }}</div>
            <div class="card-desc">Petani terdaftar</div>
          </div>
        </div>

        <div class="dashboard-card">
          <div class="card-header">
            <h3 class="card-title">Total Penggilingan</h3>
          </div>
          <div class="card-content">
            <div class="card-value">{{ stats.penggilingan }}</div>
            <div class="card-desc">Penggilingan terdaftar</div>
          </div>
        </div>

        <div class="dashboard-card">
          <div class="card-header">
            <h3 class="card-title">Total Distributor</h3>
          </div>
          <div class="card-content">
            <div class="card-value">{{ stats.distributor }}</div>
            <div class="card-desc">Distributor terdaftar</div>
          </div>
        </div>

        <div class="dashboard-card">
          <div class="card-header">
            <h3 class="card-title">Total Pengguna</h3>
          </div>
          <div class="card-content">
            <div class="card-value">{{ stats.total }}</div>
            <div class="card-desc">Total pengguna</div>
          </div>
        </div>
      </div>
    </section>

    <!-- Users Table Section -->
    <section class="table-section" aria-labelledby="users-table-title">
      <header class="table-header">
        <h2 id="users-table-title" class="section-title">Daftar Pengguna</h2>
        <div class="header-actions">
          <div class="search-box">
            <input 
              type="text" 
              v-model="searchQuery" 
              placeholder="Cari pengguna..."
              class="search-input"
            />
          </div>
          <div class="filter-box">
            <select v-model="selectedRole" class="filter-select">
              <option value="">Semua Role</option>
              <option value="Petani">Petani</option>
              <option value="Penggilingan">Penggilingan</option>
              <option value="Distributor">Distributor</option>
            </select>
          </div>
        </div>
      </header>
      
      <div v-if="loading" class="loading-state">
        <div class="loading-spinner"></div>
        <p>Memuat data pengguna...</p>
      </div>

      <div v-else-if="error" class="error-state">
        <p>{{ error }}</p>
        <button @click="fetchUsers" class="retry-button">Coba Lagi</button>
      </div>

      <div v-else-if="filteredUsers.length === 0" class="no-data">
        {{ searchQuery || selectedRole ? 'Tidak ada pengguna yang sesuai dengan filter' : 'Belum ada pengguna terdaftar' }}
      </div>

      <div v-else class="table-container">
        <table class="price-table" role="table" aria-describedby="users-table-title">
          <thead>
            <tr>
              <th scope="col">Nama</th>
              <th scope="col">Email</th>
              <th scope="col">Role</th>
              <th scope="col">Tanggal Bergabung</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="user in filteredUsers" :key="user.id">
              <td>
                <div class="user-info">
                  <div class="user-avatar">
                    {{ user.name.charAt(0).toUpperCase() }}
                  </div>
                  <div class="user-details">
                    <span class="user-name">{{ user.name }}</span>
                  </div>
                </div>
              </td>
              <td>
                <span class="user-email">{{ user.email }}</span>
              </td>
              <td>
                <span class="role-badge" :class="getRoleClass(user.role)">
                  {{ user.role }}
                </span>
              </td>
              <td>
                <span class="join-date">{{ formatDate(user.createdAt) }}</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
  </main>
</template>

<script>
import { ref, onValue } from 'firebase/database'
import { database } from '../firebase.js'

export default {
  name: 'UserManagement',
  data() {
    return {
      users: [],
      loading: true,
      error: null,
      searchQuery: '',
      selectedRole: ''
    }
  },
  computed: {
    filteredUsers() {
      let filtered = this.users

      // Filter by search query
      if (this.searchQuery) {
        const query = this.searchQuery.toLowerCase()
        filtered = filtered.filter(user => 
          user.name.toLowerCase().includes(query) ||
          user.email.toLowerCase().includes(query)
        )
      }

      // Filter by role
      if (this.selectedRole) {
        filtered = filtered.filter(user => user.role === this.selectedRole)
      }

      return filtered.sort((a, b) => b.createdAt - a.createdAt)
    },
    stats() {
      const stats = {
        petani: 0,
        penggilingan: 0,
        distributor: 0,
        total: 0
      }

      this.users.forEach(user => {
        stats.total++
        switch (user.role) {
          case 'Petani':
            stats.petani++
            break
          case 'Penggilingan':
            stats.penggilingan++
            break
          case 'Distributor':
            stats.distributor++
            break
        }
      })

      return stats
    }
  },
  mounted() {
    this.fetchUsers()
  },
  methods: {
    async fetchUsers() {
      this.loading = true
      this.error = null

      try {
        const usersRef = ref(database, 'users')
        
        // Set up real-time listener
        onValue(usersRef, (snapshot) => {
          const data = snapshot.val()
          if (data) {
            this.users = Object.keys(data).map(key => ({
              id: key,
              ...data[key]
            })).filter(user => user.role !== 'admin') // Exclude admin users
          } else {
            this.users = []
          }
          this.loading = false
        }, (error) => {
          console.error('Error fetching users:', error)
          this.error = 'Gagal memuat data pengguna. Silakan coba lagi.'
          this.loading = false
        })
      } catch (error) {
        console.error('Error setting up users listener:', error)
        this.error = 'Terjadi kesalahan saat menghubungkan ke database.'
        this.loading = false
      }
    },
    formatDate(timestamp) {
      if (!timestamp) return '-'
      const date = new Date(timestamp)
      return date.toLocaleDateString('id-ID', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    },
    getRoleClass(role) {
      return {
        'petani': role === 'Petani',
        'penggilingan': role === 'Penggilingan',
        'distributor': role === 'Distributor'
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
  background-color: #f8f9fa;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.headline-card {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);
}

.section-header {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
}

.headline {
  font-size: 1.75rem;
  font-weight: 600;
  margin-bottom: 0;
  color: #111827;
  padding: 0;
  text-align: center;
}

.header-actions {
  display: flex;
  gap: 1rem;
  align-items: center;
}

.search-input, .filter-select {
  padding: 0.5rem 1rem;
  border: 1px solid #d1d5db;
  border-radius: 0.375rem;
  font-size: 0.875rem;
  outline: none;
  transition: all 0.2s;
  background: #ffffff;
  color: #374151;
}

.search-input {
  width: 250px;
}

.search-input:focus, .filter-select:focus {
  border-color: #6b7280;
  box-shadow: 0 0 0 3px rgba(107, 114, 128, 0.1);
}

/* Cards Section */
.cards-section {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);
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

/* Table Section */
.table-section {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.1);
}

.table-section:last-child {
  margin-bottom: 0;
  border-bottom: none;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: #111827;
}

.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.loading-state, .error-state {
  text-align: center;
  padding: 2rem;
  color: #6b7280;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 3px solid #f3f4f6;
  border-top: 3px solid #374151;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin: 0 auto 1rem;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.no-data {
  text-align: center;
  padding: 2rem;
  color: #6b7280;
  font-style: italic;
  font-size: 0.875rem;
}

.retry-button {
  margin-top: 1rem;
  padding: 0.5rem 1rem;
  background: #374151;
  color: white;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  font-size: 0.875rem;
  transition: background-color 0.2s;
}

.retry-button:hover {
  background: #111827;
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

.user-info {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.user-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: linear-gradient(135deg, #111827, #374151);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 0.875rem;
}

.user-name {
  font-weight: 500;
  color: #111827;
}

.user-email {
  color: #6b7280;
  font-size: 0.875rem;
}

.role-badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 500;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.role-badge.petani {
  background: #f3f4f6;
  color: #111827;
  border: 1px solid #d1d5db;
}

.role-badge.penggilingan {
  background: #e5e7eb;
  color: #374151;
  border: 1px solid #9ca3af;
}

.role-badge.distributor {
  background: #f9fafb;
  color: #1f2937;
  border: 1px solid #d1d5db;
}

.join-date {
  color: #6b7280;
  font-size: 0.875rem;
}

/* Responsive Design */
@media (max-width: 768px) {
  .container {
    padding: 1rem 0;
  }

  .headline-card,
  .cards-section,
  .table-section {
    padding: 1.5rem 1rem;
  }

  .headline {
    font-size: 1.5rem;
    padding: 0;
  }

  .section-header {
    flex-direction: column;
    align-items: stretch;
  }

  .header-actions {
    flex-direction: column;
  }

  .search-input {
    width: 100%;
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

  .table-container {
    overflow-x: scroll;
  }

  .user-info {
    flex-direction: column;
    align-items: flex-start;
    gap: 0.5rem;
  }

  .table-header {
    flex-direction: column;
    align-items: stretch;
  }

  .header-actions {
    justify-content: center;
  }
}
</style>