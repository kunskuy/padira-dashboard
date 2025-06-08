<template>
  <div class="header">
    <div class="profile">
      <div class="profile-info">
        <h3>{{ userData.name || 'Admin' }}</h3>
        <p>{{ userData.email || user?.email || 'admin@example.com' }}</p>
      </div>
      <div class="profile-avatar" @click="toggleDropdown" ref="profileAvatar">
        {{ getInitials(userData.name || user?.email) }}
      </div>
      
      <!-- Dropdown Menu -->
      <div v-if="showDropdown" class="profile-dropdown" ref="dropdown">
        <div class="dropdown-header">
          <div class="dropdown-avatar">
            {{ getInitials(userData.name || user?.email) }}
          </div>
          <div class="dropdown-info">
            <h4>{{ userData.name || 'Admin' }}</h4>
            <p>{{ userData.email || user?.email }}</p>
            <span class="role-badge">{{ userData.role || 'Admin' }}</span>
          </div>
        </div>
        <hr class="dropdown-divider">
        <button class="logout-btn" @click="showLogoutModal">
          <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"></path>
            <polyline points="16,17 21,12 16,7"></polyline>
            <line x1="21" y1="12" x2="9" y2="12"></line>
          </svg>
          Logout
        </button>
      </div>
    </div>

    <!-- Logout Confirmation Modal -->
    <div v-if="showLogoutConfirm" class="modal-overlay" @click="hideLogoutModal">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <div class="modal-icon">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"></path>
              <polyline points="16,17 21,12 16,7"></polyline>
              <line x1="21" y1="12" x2="9" y2="12"></line>
            </svg>
          </div>
          <h3>Konfirmasi Logout</h3>
        </div>
        <div class="modal-body">
          <p>Apakah Anda yakin ingin keluar dari sistem?</p>
        </div>
        <div class="modal-actions">
          <button class="btn-cancel" @click="hideLogoutModal">
            Batal
          </button>
          <button class="btn-confirm" @click="confirmLogout">
            Ya, Logout
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, get } from 'firebase/database'
import { database } from '../firebase.js'

export default {
  name: 'HeaderComponent',
  props: {
    user: {
      type: Object,
      default: null
    }
  },
  data() {
    return {
      showDropdown: false,
      showLogoutConfirm: false,
      userData: {}
    }
  },
  async mounted() {
    // Load user data from Firebase when component mounts
    if (this.user?.uid) {
      await this.loadUserData()
    }
    
    // Close dropdown when clicking outside
    document.addEventListener('click', this.handleClickOutside)
    // Close modal with Escape key
    document.addEventListener('keydown', this.handleKeydown)
  },
  beforeUnmount() {
    document.removeEventListener('click', this.handleClickOutside)
    document.removeEventListener('keydown', this.handleKeydown)
  },
  watch: {
    // Watch for user prop changes
    user: {
      handler(newUser) {
        if (newUser?.uid) {
          this.loadUserData()
        }
      },
      immediate: true
    }
  },
  methods: {
    async loadUserData() {
      try {
        const userRef = ref(database, `users/${this.user.uid}`)
        const snapshot = await get(userRef)
        const data = snapshot.val()
        
        if (data) {
          this.userData = {
            name: data.name,
            email: data.email,
            role: data.role
          }
        }
      } catch (error) {
        console.error('Error loading user data:', error)
        // Fallback to user object data
        this.userData = {
          name: this.user?.displayName || '',
          email: this.user?.email || '',
          role: 'Admin'
        }
      }
    },
    
    getInitials(name) {
      if (!name) return 'A'
      
      // If it's an email, get initials from the part before @
      if (name.includes('@')) {
        name = name.split('@')[0]
      }
      
      const words = name.split(' ')
      if (words.length >= 2) {
        return (words[0][0] + words[1][0]).toUpperCase()
      }
      return name.substring(0, 2).toUpperCase()
    },
    
    toggleDropdown() {
      this.showDropdown = !this.showDropdown
    },
    
    handleClickOutside(event) {
      if (this.$refs.profileAvatar && this.$refs.dropdown) {
        if (!this.$refs.profileAvatar.contains(event.target) && 
            !this.$refs.dropdown.contains(event.target)) {
          this.showDropdown = false
        }
      }
    },

    handleKeydown(event) {
      if (event.key === 'Escape' && this.showLogoutConfirm) {
        this.hideLogoutModal()
      }
    },
    
    showLogoutModal() {
      this.showDropdown = false
      this.showLogoutConfirm = true
    },

    hideLogoutModal() {
      this.showLogoutConfirm = false
    },

    confirmLogout() {
      this.showLogoutConfirm = false
      this.$emit('logout')
    }
  }
}
</script>

<style scoped>
/* Reset and base styles matching Dashboard.vue */
*,
*::before,
*::after {
  box-sizing: border-box;
}

.header {
  width: 100%;
  background: #ffffff;
  margin: 0;
  padding: 1rem;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  justify-content: flex-end;
  align-items: center;
  position: relative;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  color: #374151;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.profile {
  display: flex;
  align-items: center;
  position: relative;
}

.profile-info {
  margin-right: 1rem;
  text-align: right;
}

.profile-info h3 {
  font-size: 1rem;
  font-weight: 600;
  margin: 0;
  color: #111827;
}

.profile-info p {
  font-size: 0.875rem;
  color: #6b7280;
  margin: 0;
}

.profile-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #e5e7eb;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.15s ease;
  color: #374151;
  border: 1px solid #d1d5db;
}

.profile-avatar:hover {
  background-color: #d1d5db;
  box-shadow: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
}

/* Dropdown Styles */
.profile-dropdown {
  position: absolute;
  top: 100%;
  right: 0;
  margin-top: 0.5rem;
  background: #ffffff;
  border-radius: 0.5rem;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
  min-width: 280px;
  z-index: 1000;
  border: 1px solid #e5e7eb;
}

.dropdown-header {
  padding: 1.5rem;
  display: flex;
  align-items: center;
}

.dropdown-avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background-color: #e5e7eb;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  margin-right: 0.75rem;
  font-size: 1.1rem;
  color: #374151;
  border: 1px solid #d1d5db;
}

.dropdown-info h4 {
  font-size: 1rem;
  font-weight: 600;
  margin: 0 0 0.25rem 0;
  color: #111827;
}

.dropdown-info p {
  font-size: 0.875rem;
  color: #6b7280;
  margin: 0 0 0.5rem 0;
}

.role-badge {
  display: inline-block;
  background-color: #f3f4f6;
  color: #374151;
  padding: 0.125rem 0.5rem;
  border-radius: 0.375rem;
  font-size: 0.75rem;
  font-weight: 500;
  border: 1px solid #d1d5db;
}

.dropdown-divider {
  margin: 0;
  border: none;
  border-top: 1px solid #e5e7eb;
}

.logout-btn {
  width: 100%;
  padding: 0.75rem 1.5rem;
  border: none;
  background: none;
  color: #374151;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  display: flex;
  align-items: center;
  transition: background-color 0.15s ease;
  border-radius: 0 0 0.5rem 0.5rem;
}

.logout-btn:hover {
  background-color: #f9fafb;
  color: #111827;
}

.logout-btn svg {
  margin-right: 0.5rem;
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  backdrop-filter: blur(2px);
}

.modal-content {
  background: #ffffff;
  border-radius: 0.75rem;
  box-shadow: 0 10px 25px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
  max-width: 400px;
  width: 90%;
  max-height: 90vh;
  overflow: hidden;
  border: 1px solid #e5e7eb;
  animation: modalFadeIn 0.15s ease-out;
}

@keyframes modalFadeIn {
  from {
    opacity: 0;
    transform: scale(0.95) translateY(-10px);
  }
  to {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}

.modal-header {
  padding: 1.5rem 1.5rem 1rem 1.5rem;
  text-align: center;
  border-bottom: 1px solid #f3f4f6;
}

.modal-icon {
  width: 48px;
  height: 48px;
  margin: 0 auto 1rem auto;
  background-color: #f3f4f6;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #374151;
  border: 1px solid #e5e7eb;
}

.modal-header h3 {
  font-size: 1.125rem;
  font-weight: 600;
  margin: 0;
  color: #111827;
}

.modal-body {
  padding: 1rem 1.5rem;
  text-align: center;
}

.modal-body p {
  font-size: 0.875rem;
  color: #6b7280;
  margin: 0;
  line-height: 1.5;
}

.modal-actions {
  padding: 1rem 1.5rem 1.5rem 1.5rem;
  display: flex;
  gap: 0.75rem;
  justify-content: center;
}

.btn-cancel,
.btn-confirm {
  padding: 0.625rem 1.25rem;
  border-radius: 0.5rem;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s ease;
  border: 1px solid;
  min-width: 100px;
}

.btn-cancel {
  background-color: #ffffff;
  color: #374151;
  border-color: #d1d5db;
}

.btn-cancel:hover {
  background-color: #f9fafb;
  border-color: #9ca3af;
  color: #111827;
}

.btn-confirm {
  background-color: #374151;
  color: #ffffff;
  border-color: #374151;
}

.btn-confirm:hover {
  background-color: #111827;
  border-color: #111827;
}

/* Mobile responsive styles */
@media (max-width: 768px) {
  .header {
    padding: 1rem;
    margin-bottom: 10px;
  }
  
  .profile-info {
    display: none;
  }
  
  .profile-dropdown {
    right: -1rem;
    min-width: 260px;
  }

  .modal-content {
    max-width: 350px;
    margin: 1rem;
  }

  .modal-actions {
    flex-direction: column;
  }

  .btn-cancel,
  .btn-confirm {
    width: 100%;
    min-width: auto;
  }
}

@media (max-width: 480px) {
  .header {
    padding: 0.75rem;
  }
  
  .profile-dropdown {
    right: -0.5rem;
    min-width: 240px;
  }

  .modal-content {
    max-width: 320px;
  }

  .modal-header {
    padding: 1.25rem 1.25rem 0.75rem 1.25rem;
  }

  .modal-body {
    padding: 0.75rem 1.25rem;
  }

  .modal-actions {
    padding: 0.75rem 1.25rem 1.25rem 1.25rem;
  }
}
</style>