<template>
  <SplashScreen v-if="showSplash" />
  <LoginPage v-else-if="!isAuthenticated" @login-success="handleLoginSuccess" />
  <div class="dashboard" v-else :style="{ opacity: showSplash ? 0 : 1 }">
    <Sidebar :activeSection="activeSection" @change-section="changeSection" />
    <div class="main-content">
      <Header 
        :title="activeSection" 
        :user="currentUser" 
        @logout="handleLogout" 
      />
      <component :is="currentView"></component>
    </div>
  </div>
</template>

<script>
import { onAuthStateChanged, signOut } from 'firebase/auth'
import { ref, get } from 'firebase/database'
import { auth, database } from './firebase.js'
import SplashScreen from './components/SplashScreen.vue'
import LoginPage from './views/LoginPage.vue'
import Sidebar from './components/Sidebar.vue'
import Header from './components/Header.vue'
import Dashboard from './views/Dashboard.vue'
import DataPertanian from './views/DataPertanian.vue'
import DataPenggilingan from './views/DataPenggilingan.vue'
import Price from './views/Price.vue'
import News from './views/News.vue'
import Notification from './views/Notification.vue'
import Location from './views/Location.vue'
import User from './views/User.vue'

export default {
  name: 'App',
  components: {
    SplashScreen,
    LoginPage,
    Sidebar,
    Header,
    Dashboard,
    DataPertanian,
    DataPenggilingan,
    Price,
    News,
    Notification,
    Location,
    User
  },
  data() {
    return {
      showSplash: true,
      activeSection: 'Dashboard',
      isAuthenticated: false,
      currentUser: null
    }
  },
  computed: {
    currentView() {
      const viewMap = {
        'Dashboard': Dashboard,
        'Data Pertanian': DataPertanian,
        'Data Penggilingan': DataPenggilingan,
        'Input Harga': Price,
        'Input Berita': News,
        'Input Notifikasi': Notification,
        'Lokasi': Location,
        'Pengguna': User
      }
      return viewMap[this.activeSection]
    }
  },
  mounted() {
    // Check authentication state
    onAuthStateChanged(auth, async (user) => {
      if (user) {
        // User is signed in, check if admin
        try {
          const userRef = ref(database, `users/${user.uid}`)
          const snapshot = await get(userRef)
          const userData = snapshot.val()
          
          if (userData && userData.role === 'admin') {
            this.currentUser = {
              uid: user.uid,
              email: user.email,
              displayName: user.displayName,
              photoURL: user.photoURL,
              ...userData // Include database data (name, role, etc.)
            }
            this.isAuthenticated = true
          } else {
            // Not admin, sign out
            await signOut(auth)
            this.isAuthenticated = false
            alert('Akses ditolak. Hanya admin yang dapat mengakses aplikasi ini.')
          }
        } catch (error) {
          console.error('Error checking user role:', error)
          this.isAuthenticated = false
        }
      } else {
        // User is signed out
        this.isAuthenticated = false
        this.currentUser = null
      }
      
      // Hide splash screen after auth check
      setTimeout(() => {
        this.showSplash = false
      }, 1000)
    })
  },
  methods: {
    changeSection(section) {
      this.activeSection = section
    },
    handleLoginSuccess() {
      // Auth state change will handle the rest
    },
    async handleLogout() {
      try {
        await signOut(auth)
        this.isAuthenticated = false
        this.currentUser = null
        this.activeSection = 'Dashboard' // Reset to default section
        
      } catch (error) {
        console.error('Error signing out:', error)
        alert('Terjadi kesalahan saat logout. Silakan coba lagi.')
      }
    }
  }
}
</script>

<style>
.dashboard {
  display: flex;
  transition: opacity 0.3s ease;
}
.main-content {
  flex: 1;
  padding: 1.5rem;
  overflow-y: auto;
  max-height: 100vh;
}
.section-content {
  display: flex;
  flex-direction: column;
}
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
  margin-bottom: 1.5rem;
}
.card {
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  padding: 1rem;
}
.card-header {
  margin-bottom: 1rem;
}
.chart-wrapper {
  position: relative;
  height: 300px;
}
</style>
